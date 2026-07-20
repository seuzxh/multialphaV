# Qlib 四大场景：训练机制与模型来源

> 适用：**multialphaV** 项目内 RD-Agent 框架的 Qlib 量化场景理解。
> 与 `CLAUDE.md`（开发行为约束）配合：本文件管"**场景的客观机制**"，CLAUDE.md 管"**Agent 怎么改这些场景**"。
>
> 维护约定：场景定义、YAML 模板、CoSTEER 编码逻辑任一变更，必须同步本文档对应章节。本文档随团队讨论**持续更新**（见 `CLAUDE.md` §11「持续沉淀」条款）。

---

## 目录

- [0. 一句话结论](#0-一句话结论)
- [1. 四个场景全景](#1-四个场景全景)
- [2. 是否都涉及模型训练？](#2-是否都涉及模型训练)
- [3. 每个场景支持哪些模型？能选吗？](#3-每个场景支持哪些模型能选吗)
- [4. Model 场景：是全新设计，还是从清单挑选？](#4-model-场景是全新设计还是从清单挑选)
- [5. 证据链（代码引用）](#5-证据链代码引用)
- [6. 术语对照](#6-术语对照)

---

## 0. 一句话结论

四个 Qlib 场景**都要跑模型训练**（因为 Qlib workflow 的 `task.model` 是必填项，没有模型就出不了评估指标）。区别在于：

- **Factor / Factor-from-Report**：模型是**固定的基准评估器**（LightGBM），用户一般不去换它。
- **Model / Quant**：模型是 **LLM 从零设计并手写**的 PyTorch 网络，每轮迭代进化。**没有模型清单可选**。

---

## 1. 四个场景全景

四个场景都是 `rdagent.core.scenario.Scenario` 的子类，在
[`rdagent/app/qlib_rd_loop/conf.py`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/app/qlib_rd_loop/conf.py)
注册，各有独立的 RD-loop 入口脚本和 Qlib workflow YAML 模板。

| # | 场景类 | 定义文件 | conf.py 注册行 | RD-loop 入口 |
|---|---|---|---|---|
| ① | `QlibFactorScenario` | [`factor_experiment.py#L30`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/scenarios/qlib/experiment/factor_experiment.py#L30) | [`conf.py#L56`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/app/qlib_rd_loop/conf.py#L56) | `app/qlib_rd_loop/factor.py` |
| ② | `QlibModelScenario` | [`model_experiment.py#L26`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/scenarios/qlib/experiment/model_experiment.py#L26) | [`conf.py#L12`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/app/qlib_rd_loop/conf.py#L12) | `app/qlib_rd_loop/model.py` |
| ③ | `QlibFactorFromReportScenario` | [`factor_from_report_experiment.py#L7`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/scenarios/qlib/experiment/factor_from_report_experiment.py#L7) | [`conf.py#L98`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/app/qlib_rd_loop/conf.py#L98) | `app/qlib_rd_loop/factor_from_report.py` |
| ④ | `QlibQuantScenario` | [`quant_experiment.py#L41`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/scenarios/qlib/experiment/quant_experiment.py#L41) | [`conf.py#L116`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/app/qlib_rd_loop/conf.py#L116) | `app/qlib_rd_loop/quant.py` |

> ③ 继承自 ①（`QlibFactorFromReportScenario(QlibFactorScenario)`），差异在「因子从 PDF 报告提取」而非「因子从假设生成」。

配置实例化在 conf.py 文末（
[`conf.py#L173-L176`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/app/qlib_rd_loop/conf.py#L173-L176)
）：`FACTOR_PROP_SETTING` / `FACTOR_FROM_REPORT_PROP_SETTING` / `MODEL_PROP_SETTING` / `QUANT_PROP_SETTING`。

---

## 2. 是否都涉及模型训练？

**是——四个场景最终都要训练模型。**

每个场景跑的都是 Qlib 的完整 workflow（特征 → 训练模型预测收益 → 构建组合 → 回测评估 IC/Sharpe/回撤等）。`task.model` 是 workflow 必填项，没有模型就出不了评估指标，RD-loop 也就没有 feedback 可迭代。

但**"训练模型"是不是该场景的 R&D 焦点**，四个场景不一样：

| 场景 | 训练模型？ | 模型是 R&D 焦点？ |
|---|---|---|
| ① **Factor** | ✅ 用固定基准模型评估因子 | ❌ 焦点是**因子**，模型固定 |
| ② **Model** | ✅ 训练 agent 生成的模型 | ✅ 焦点就是**模型本身** |
| ③ **Factor-from-Report** | ✅ 同①，用基准模型评估 | ❌ 焦点是**从 PDF 提因子** |
| ④ **Quant** | ✅ 训练 | ✅ **因子 + 模型同时进化** |

---

## 3. 每个场景支持哪些模型？能选吗？

**没有一个静态的"模型下拉菜单"或枚举**，模型由各自的工作流 YAML 模板 + 场景定位决定。

### 3.1 Factor 系列（①③）—— 模型由 runner 动态选择

Factor 场景有**三个** YAML 模板，由
[`factor_runner.py`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/scenarios/qlib/developer/factor_runner.py)
根据实验状态动态选用（
[`factor_runner.py#L163-L193`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/scenarios/qlib/developer/factor_runner.py#L163-L193)
）：

| 模板 | model.class | 何时使用 |
|---|---|---|
| [`conf_baseline.yaml`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/scenarios/qlib/experiment/factor_template/conf_baseline.yaml#L64) | **`LGBModel`**（[`#L64`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/scenarios/qlib/experiment/factor_template/conf_baseline.yaml#L64)） | 无 base feature 时的默认基准 |
| [`conf_combined_factors.yaml`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/scenarios/qlib/experiment/factor_template/conf_combined_factors.yaml#L57) | **`LGBModel`**（[`#L57`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/scenarios/qlib/experiment/factor_template/conf_combined_factors.yaml#L57)） | 有组合因子但无历史 SOTA 模型 |
| [`conf_combined_factors_sota_model.yaml`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/scenarios/qlib/experiment/factor_template/conf_combined_factors_sota_model.yaml#L67) | **`GeneralPTNN`**（[`#L67`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/scenarios/qlib/experiment/factor_template/conf_combined_factors_sota_model.yaml#L67)） | 有组合因子 **且** trace 里存在历史 SOTA 模型实验（`QlibModelExperiment`） |

要点：
- 默认评估器是 **LightGBM**（`qlib.contrib.model.gbdt.LGBModel`），用户一般不去换。
- **多模型选择器**（`model_selector`，2026-07 新增）：`conf_baseline.yaml` 和 `conf_combined_factors.yaml` 的 `task.model` 段已用 Jinja `{% if model_selector == ... %}` 条件块参数化，支持 4 种 qlib 内置模型——通过环境变量 `QLIB_FACTOR_MODEL_SELECTOR` 切换（默认 `lgbm`）：

  | selector | class | 特点 |
  |---|---|---|
  | `lgbm`（默认） | `LGBModel` | LightGBM，与历史行为一致 |
  | `linear` | `LinearModel` | 闭式 OLS，毫秒级，最快基线 |
  | `xgboost` | `XGBModel` | XGBoost 梯度提升树 |
  | `catboost` | `CatBoostModel` | CatBoost（自动 GPU/CPU） |

  配置见 [`conf.py: FactorBasePropSetting.model_selector`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/app/qlib_rd_loop/conf.py#L94)；env 注入见 [`factor_runner.py: env_to_use`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/scenarios/qlib/developer/factor_runner.py#L100)。`FactorFromReportPropSetting` 自动继承；**副作用**：Quant 场景共用 `QlibFactorRunner` 也会受影响（默认 lgbm 不变行为）。pickle 缓存 key 已纳入 selector（[`_develop_cache_key`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/scenarios/qlib/developer/factor_runner.py#L64)），同因子不同 selector 互不干扰。
- **特例**：当历史 trace 里跑出过被采纳的 SOTA 模型实验（`exp.based_experiments` 含 `QlibModelExperiment`），Factor runner 会**复用那份模型代码**（`model.py`），切到 `GeneralPTNN` + `pt_model_uri: "model.model_cls"` 来跑组合因子评估——见
  [`factor_runner.py#L132-L166`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/scenarios/qlib/developer/factor_runner.py#L132-L166)。此分支不受 `model_selector` 影响。
- prompt 文档（[`prompts.yaml: qlib_factor_simulator`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/scenarios/qlib/experiment/prompts.yaml#L97)）给 LLM 的示例是：「train a model like **LightGBM, CatBoost, LSTM** or simple PyTorch model」——只是文字描述，不是代码层面的可选清单。

### 3.2 Model / Quant 场景（②④）—— 模型是 PyTorch，由 agent 生成

模型模板固定用 `GeneralPTNN` 包装器：

| 模板 | model.class | 关键机制 |
|---|---|---|
| [`conf_baseline_factors_model.yaml`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/scenarios/qlib/experiment/model_template/conf_baseline_factors_model.yaml#L62) | **`GeneralPTNN`**（[`#L62`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/scenarios/qlib/experiment/model_template/conf_baseline_factors_model.yaml#L62)） | `pt_model_uri: "model.model_cls"` |
| [`conf_sota_factors_model.yaml`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/scenarios/qlib/experiment/model_template/conf_sota_factors_model.yaml#L67) | **`GeneralPTNN`**（[`#L67`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/scenarios/qlib/experiment/model_template/conf_sota_factors_model.yaml#L67)） | 同上 |

`GeneralPTNN`（`qlib.contrib.model.pytorch_general_nn`）本身只是个**通用 PyTorch 包装器**，真正的网络结构是 `model.model_cls`——一个由 RD-Agent 在 loop 中**迭代生成/进化**的 `torch.nn.Module` 子类。具体机制见 §4。

Quant 场景同时装配了 `QlibFactorCoSTEER`（因子编码）和 `QlibModelCoSTEER`（模型编码），见
[`conf.py#L125-L145`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/app/qlib_rd_loop/conf.py#L125-L145)；
因子/模型分支由 `action_selection`（默认 `bandit`，可选 `llm`/`random`）决定，见
[`conf.py#L151`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/app/qlib_rd_loop/conf.py#L151)。

---

## 4. Model 场景：是全新设计，还是从清单挑选？

**全新设计（由 LLM 从零生成代码），不是从清单里挑。** 但有引导和约束，所以也不是"完全自由发挥"。

### 4.1 没有模型清单 / 注册表

[`model_coder/model.py`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/components/coder/model_coder/model.py)
下的 `ModelTask` 字段全是**描述性的自由文本**，由 LLM 填，没有任何枚举或 `choices=[...]`：

```python
# model_coder/model.py#L15-L37
class ModelTask(CoSTEERTask):
    def __init__(self, name, description, architecture, *,
                 hyperparameters, training_hyperparameters,
                 formulation=None, variables=None, model_type=None, ...):
```

`model_type` 只是个粗分类标签（`Tabular` / `TimeSeries` / `Graph` / `XGBoost`，见
[`model.py#L34-L36`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/components/coder/model_coder/model.py#L34-L36)
），不是模型清单。

### 4.2 模型代码是 LLM 现场写的

CoSTEER 的 evolving strategy 直接调用 LLM，**生成完整的 `model.py` 源码**（返回 JSON 里的 `code` 字段就是一段 Python）：

```python
# evolving_strategy.py#L72
code = json.loads(
    APIBackend(...).build_messages_and_create_chat_completion(
        user_prompt=user_prompt, system_prompt=system_prompt,
        json_mode=True, json_target_type=Dict[str, str],
    )
)["code"]
```

### 4.3 产出要求：一个 `torch.nn.Module` 子类

prompt 接口（[`prompts.yaml: qlib_model_interface`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/scenarios/qlib/experiment/prompts.yaml#L185)）要求 LLM：

> A class which is a sub-class of pytorch.nn.Module... Set a variable called "model_cls" to the class you defined.
> ```python
> class XXXModel(torch.nn.Module): ...
> model_cls = XXXModel
> ```

每轮 agent 要**从零写一个网络结构**（MLP / LSTM / GRU / Transformer… 由它自己决定），而不是去配置 qlib 自带的 `LGBModel`/`DNNModelPytorch` 之类的现成类。输入输出契约：
- Tabular 模型：输入 `(batch_size, num_features)` → 输出 `(batch_size, 1)`
- TimeSeries 模型：输入 `(batch_size, num_timesteps, num_features)` → 输出 `(batch_size, 1)`

### 4.4 有"引导"和"约束"，不是放飞

RAG 提示（[`model_proposal.py#L54`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/scenarios/qlib/proposal/model_proposal.py#L54)）给了方向性引导：

> "market data could be time-series, and **GRU model/LSTM model are suitable** for them. **Do not generate GNN model as for now.**"
>
> "If you believe that the previous model itself is good but the **training hyperparameters** or **model hyperparameters** are not optimal, you can return the **same model** and adjust these parameters instead."

含义：
- agent **倾向**于生成 GRU/LSTM 这类时序模型，被劝阻不要搞 GNN。
- 若只是调参不够好，允许"**复用上一个模型 + 改超参**"。

### 4.5 是"迭代进化"，不是一次性生成

```python
# model_proposal.py#L158
exp.based_experiments = [t[0] for t in trace.hist
                         if t[1] and isinstance(t[0], ModelExperiment)]
```
配合 `evolving_n: int = 10`（[`conf.py#L30`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/app/qlib_rd_loop/conf.py#L30)）——每一轮都把**历史实验 + 反馈**喂给 LLM，让它在上一轮代码基础上改进。

### 4.6 Model 场景总结

**"agent 当架构师，从零设计并手写 PyTorch 模型代码"**，不是"从 LightGBM/LSTM/Transformer 清单里勾选一个去训练"。它会参考历史反馈和 RAG 提示（偏好 GRU/LSTM、暂不碰 GNN、允许只调参不换结构），所以是**受引导的全新设计 + 迭代进化**。

---

## 5. 证据链（代码引用）

| 结论 | 代码位置 |
|---|---|
| 四场景在 conf.py 注册 | [`conf.py#L12,L56,L98,L116`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/app/qlib_rd_loop/conf.py#L12) |
| Factor 默认用 LGBM | [`factor_template/conf_baseline.yaml#L64`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/scenarios/qlib/experiment/factor_template/conf_baseline.yaml#L64) |
| Model/Quant 用 GeneralPTNN | [`model_template/conf_baseline_factors_model.yaml#L62`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/scenarios/qlib/experiment/model_template/conf_baseline_factors_model.yaml#L62) |
| Factor runner 动态选模板 | [`factor_runner.py#L163-L193`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/scenarios/qlib/developer/factor_runner.py#L163-L193) |
| Factor 复用 SOTA 模型 | [`factor_runner.py#L132-L166`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/scenarios/qlib/developer/factor_runner.py#L132-L166) |
| ModelTask 无枚举字段 | [`model_coder/model.py#L15-L37`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/components/coder/model_coder/model.py#L15-L37) |
| 模型代码由 LLM 生成 | [`model_coder/evolving_strategy.py#L72`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/components/coder/model_coder/evolving_strategy.py#L72) |
| 模型接口要求 torch.nn.Module | [`prompts.yaml#L185`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/scenarios/qlib/experiment/prompts.yaml#L185) |
| RAG 偏好 LSTM/GRU、禁 GNN | [`model_proposal.py#L54`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/scenarios/qlib/proposal/model_proposal.py#L54) |
| 迭代进化（based_experiments） | [`model_proposal.py#L158`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/scenarios/qlib/proposal/model_proposal.py#L158) |
| evolving_n=10 | [`conf.py#L30`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/app/qlib_rd_loop/conf.py#L30) |
| Quant 复用 QlibModelCoSTEER | [`conf.py#L125-L145`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/app/qlib_rd_loop/conf.py#L125-L145) |

---

## 6. 术语对照

| 术语 | 含义 |
|---|---|
| **Scenario** | RD-Agent 的场景抽象，定义一个 R&D loop 的背景/接口/输出格式 |
| **RD-loop** | Hypothesis → Experiment → Run → Feedback 的迭代循环 |
| **CoSTEER** | 子级自演化系统，负责把任务描述翻译成可运行代码（因子代码 / 模型代码） |
| **GeneralPTNN** | Qlib 的通用 PyTorch 模型包装器，网络结构通过 `pt_model_uri` 注入 |
| **model_cls** | `model.py` 里约定的变量名，指向 LLM 生成的 `torch.nn.Module` 子类 |
| **LGBModel** | Qlib 自带的 LightGBM 模型实现（`qlib.contrib.model.gbdt`），Factor 场景的默认评估器 |
| **LinearModel** | Qlib 自带的线性模型（`qlib.contrib.model.linear`），支持 ols/ridge/lasso/nnls 闭式解，Factor 场景可选的最快验证模型 |
| **XGBModel / CatBoostModel** | Qlib 自带的 XGBoost / CatBoost 梯度提升树实现，Factor 场景可选的树模型对照 |
| **model_selector** | Factor / Factor-from-Report 场景的模型选择器配置（`QLIB_FACTOR_MODEL_SELECTOR`，默认 `lgbm`），通过 Jinja 条件块切换 `task.model`。详见 §3.1 |
| **based_experiments** | 实验对象携带的"历史已采纳实验"列表，是迭代进化的上下文基础 |
| **SOTA 模型** | trace 中被 `decision=True` 采纳的最优模型实验，可被 Factor 场景复用 |

---

**版本**：v1.1（2026-07-20）
**适用目录**：`/home/zxh/projects/1.multialphaV`（根仓库，本文档所在地）+ `/home/zxh/projects/1.multialphaV/RD-Agent`（代码仓库）
**配套文档**：[CLAUDE.md](file:///home/zxh/projects/1.multialphaV/CLAUDE.md)（开发行为约束）、[docs/COLLABORATION.md](file:///home/zxh/projects/1.multialphaV/docs/COLLABORATION.md)（多人协作规范）、[docs/reference/ENV.md](file:///home/zxh/projects/1.multialphaV/docs/reference/ENV.md)（配置项参考）
**更新来源**：
- 2026-07-19 团队讨论「四 qlib 场景是否都训练模型 / Model 场景模型来源」结论沉淀
- 2026-07-20 实现沉淀：Factor 场景多模型选择器 `model_selector`（4 取值 lgbm/linear/xgboost/catboost，Jinja 参数化模板，cache key 纳入 selector；commit `d0348280`）
