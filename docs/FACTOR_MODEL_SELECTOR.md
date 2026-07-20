# Factor 模型选择器（model_selector）技术方案

> 允许用户在 webUI 创建因子挖掘任务时选择验证模型（LightGBM/Linear/XGBoost/CatBoost），而非固定使用 LightGBM。
>
> 维护约定：模型清单、Jinja 模板、配置字段任一变更，必须同步本文档。

---

## 0. 一句话方案

webUI 的"验证模型"下拉选择 → `POST /upload` 的 `model_selector` 字段 → server 在 fork 子进程前设 `os.environ['QLIB_FACTOR_MODEL_SELECTOR']` → 子进程 `FactorBasePropSetting` 读到 → `factor_runner` 注入 qrun env → Jinja `{% if model_selector == ... %}` 渲染对应模型的 `task.model` YAML 段。

---

## 1. 背景

Factor / Factor-from-Report 场景原本固定用 LightGBM（`LGBModel`）评估因子质量。不同模型有不同特点：
- **LightGBM**：梯度提升，秒级，默认 SOTA
- **Linear**：闭式 OLS，毫秒级，最快基线验证
- **XGBoost / CatBoost**：梯度提升树，作为对照

需求：让用户在 webUI 按**任务**选择模型，快速对比不同模型下的因子表现。

---

## 2. 改动点（4 层）

### 2.1 配置层：`FactorBasePropSetting.model_selector`

文件：[`rdagent/app/qlib_rd_loop/conf.py`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/app/qlib_rd_loop/conf.py#L95)

```python
model_selector: str = "lgbm"
"""Model for factor validation.
- 'lgbm'    : LightGBM (default, current behavior)
- 'linear'  : LinearModel (closed-form OLS, fastest)
- 'xgboost' : XGBModel (gradient boosting tree)
- 'catboost': CatBoostModel (gradient boosting tree, auto GPU/CPU)
Set via env var QLIB_FACTOR_MODEL_SELECTOR.
"""
```

- Pydantic Settings 字段，env_prefix=`QLIB_FACTOR_`，通过 `QLIB_FACTOR_MODEL_SELECTOR` 环境变量覆盖
- `FactorFromReportPropSetting` 自动继承
- **副作用**：`QlibFactorRunner` 三场景共用（factor/from_report/quant），quant 也会受影响（默认 lgbm 不变）

### 2.2 模板层：Jinja `{% if %}` 条件块

文件：[`rdagent/scenarios/qlib/experiment/factor_template/conf_baseline.yaml`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/scenarios/qlib/experiment/factor_template/conf_baseline.yaml) + `conf_combined_factors.yaml`

```yaml
task:
    model:
{%- if model_selector == "linear" %}
        class: LinearModel
        module_path: qlib.contrib.model.linear
        kwargs:
            estimator: ols
{%- elif model_selector == "xgboost" %}
        class: XGBModel
        module_path: qlib.contrib.model.xgboost
        kwargs:
            objective: reg:squarederror
            eta: 0.1
            max_depth: 6
{%- elif model_selector == "catboost" %}
        class: CatBoostModel
        module_path: qlib.contrib.model.catboost_model
        kwargs:
            loss: RMSE
{%- else %}
        class: LGBModel
        module_path: qlib.contrib.model.gbdt
        kwargs:
            loss: mse
            colsample_bytree: 0.8879
            ...（完整 LGBM 超参）
{%- endif %}
```

- 用 `{%-` / `-%}` 紧凑语法（消除控制块空行）
- qrun 在容器内渲染（`Template(config)`，无 `trim_blocks`）
- 默认（`lgbm` 或空字符串）走 `else` 分支，**与原有行为完全一致**（向后兼容）

### 2.3 Runner 层：env 注入 + cache key

文件：[`rdagent/scenarios/qlib/developer/factor_runner.py`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/scenarios/qlib/developer/factor_runner.py)

```python
env_to_use = {
    ...,
    "model_selector": fbps.model_selector,  # ← 传给 qrun 渲染 YAML
}
```

cache key 纳入 selector（避免同因子不同模型命中同一缓存）：

```python
def _develop_cache_key(self, exp):
    base_key = CachedRunner.get_cache_key(self, exp)
    selector = FactorBasePropSetting().model_selector
    return md5_hash(f"{base_key}\nmodel_selector={selector}")
```

> **注意**：原计划的 `override get_cache_key` 行不通——`@cache_with_pickle` 装饰器在类定义时绑定基类函数引用，子类 override 不生效。改用 `_develop_cache_key` 方法 + 重新装饰 `develop`（在类体内裸名引用）。

### 2.4 Server 层：`/upload` 接收 model_selector

文件：[`rdagent/log/server/app.py`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/log/server/app.py)

```python
model_selector = request.form.get("model_selector")
if model_selector:
    os.environ["QLIB_FACTOR_MODEL_SELECTOR"] = model_selector
elif "QLIB_FACTOR_MODEL_SELECTOR" in os.environ:
    os.environ.pop("QLIB_FACTOR_MODEL_SELECTOR", None)  # reset to avoid leak

task.start()  # fork 的子进程继承 os.environ 快照
```

- 子进程是 multiprocessing.Process（fork 模式），在 `task.start()` 那一刻拷贝父进程的 `os.environ`
- 无值时 reset env（避免上一个任务的设置泄漏到下一个）

### 2.5 前端层：NewTaskDialog 下拉选择器

文件：[`web/src/multialpha/components/NewTaskDialog.vue`](file:///home/zxh/projects/1.multialphaV/RD-Agent/web/src/multialpha/components/NewTaskDialog.vue)

```html
<el-form-item v-if="form.method==='text'||form.method==='optimize'" label="验证模型">
  <el-select v-model="form.modelSelector" style="width:100%">
    <el-option label="LightGBM（默认）" value="lgbm"/>
    <el-option label="Linear（闭式 OLS，最快）" value="linear"/>
    <el-option label="XGBoost" value="xgboost"/>
    <el-option label="CatBoost" value="catboost"/>
  </el-select>
</el-form-item>
```

- 仅 text / optimize 模式显示（pdf 模式走 fin_factor_report，不进交互不选模型）
- `use-multialpha.ts` 的 createTask 仅在非 lgbm 时传 `model_selector`（lgbm 是后端默认，省传输）

---

## 3. 数据流（端到端）

```
webUI NewTaskDialog（下拉选 linear）
  │
  │  form.modelSelector = 'linear'
  ▼
use-multialpha createTask
  │
  │  FormData: model_selector=linear
  ▼
POST /upload
  │
  │  os.environ['QLIB_FACTOR_MODEL_SELECTOR'] = 'linear'
  │  task.start() → fork 子进程
  ▼
子进程 fin_factor main()
  │
  │  import FactorBasePropSetting → 读 os.environ → model_selector='linear'
  ▼
QlibFactorRunner.develop()
  │
  │  env_to_use['model_selector'] = 'linear'
  ▼
qrun（docker 容器内）
  │
  │  Jinja 渲染 {% if model_selector == "linear" %} → LinearModel
  ▼
qlib.contrib.model.linear.LinearModel 训练 + 回测
```

---

## 4. 支持的模型清单

| selector | class | module_path | 训练特点 | 适用场景 |
|---|---|---|---|---|
| `lgbm`（默认） | `LGBModel` | `qlib.contrib.model.gbdt` | 梯度提升，秒级 | 默认 SOTA |
| `linear` | `LinearModel` | `qlib.contrib.model.linear` | 闭式 OLS，毫秒~秒级，纯CPU | 最快验证 |
| `xgboost` | `XGBModel` | `qlib.contrib.model.xgboost` | 梯度提升树，秒级 | 树模型对照 |
| `catboost` | `CatBoostModel` | `qlib.contrib.model.catboost_model` | 梯度提升树，自动GPU/CPU | 树模型对照 |

> `identity`（无训练）和 `mlp`（DNNModelPytorch，需 input_dim 注入）未纳入第一版。

---

## 5. 兼容性

- **默认 `lgbm` 零变化**：Jinja else 分支渲染出与原模板完全相同的 LGBModel 段
- **CLI 直跑不受影响**：不传 `QLIB_FACTOR_MODEL_SELECTOR` 时走默认 lgbm
- **fin_factor_report 不受影响**：其 YAML 模板未参数化
- **SOTA 复用分支不受影响**：`conf_combined_factors_sota_model.yaml` 没有 `model_selector` 占位符

---

## 6. 并发安全说明

`os.environ` 是进程级全局。如果两个 `/upload` 请求同时到达，后设的 env 会覆盖前一个。但：
- Flask dev server 是**单线程顺序处理**（threaded=True 但 GIL 限制）
- `task.start()` 是同步调用，fork 在那一刻拷贝 env 快照
- 已 fork 的子进程不受后续 `os.environ` 修改影响

因此**典型使用场景下安全**。如果未来切换到多 worker 部署（gunicorn），需要改用 multiprocessing 的 `env=` 参数（而非 os.environ）或通过 kwargs 传递。

---

## 7. 验证记录

| 验证项 | 结果 | 证据 |
|---|---|---|
| Jinja 渲染（4 selector × 2 模板） | ✅ | Python 端到端测试全过 |
| qlib 模型实例化（4 selector） | ✅ | `init_instance_by_config` 全过 |
| cache key 随 selector 变化 | ✅ | 4 key 全不同 |
| lgbm 默认向后兼容 | ✅ | else 分支渲染 = 原模板 |
| lgbm 冒烟（model_selector:lgbm） | ✅ | stdout `model_selector:lgbm` |
| linear 冒烟（model_selector:linear） | ✅ | stdout `model_selector:linear` |
| vue-tsc 类型检查 | ✅ | 0 错误 |

---

## 8. 相关 commit

| commit | 内容 |
|---|---|
| `d0348280` | 核心功能（conf + YAML + factor_runner） |
| `25de8ffe` | mlflow docker 修复（冒烟时发现） |
| `0c37bebf` | webUI 支持（NewTaskDialog + /upload + use-multialpha） |

---

**版本**：v1.0（2026-07-21）
**适用目录**：`/home/zxh/projects/1.multialphaV/RD-Agent`
**配套文档**：[docs/QLIB_SCENARIOS.md](file:///home/zxh/projects/1.multialphaV/docs/QLIB_SCENARIOS.md)（§3.1 model_selector）、[docs/reference/API.md](file:///home/zxh/projects/1.multialphaV/docs/reference/API.md)（§2.1 description + model_selector 字段）、[docs/reference/ENV.md](file:///home/zxh/projects/1.multialphaV/docs/reference/ENV.md)（§3.3 QLIB_FACTOR_MODEL_SELECTOR）
