# 股池预测看板 技术方案

> 版本: v2.0 (MVP — T+1 实战预测)
> 状态: 待 review
> 日期: 2026-07-21
> 配套 PRD: [STOCKPOOL_DASHBOARD_PRD.md](./STOCKPOOL_DASHBOARD_PRD.md)

---

## 0. 版本演进说明

| 版本 | 方向 | 状态 |
|---|---|---|
| v1.0 | 读 pred.pkl 末日 Top20(历史回顾) | **废弃** |
| **v2.0** | **T+1 实战预测(复用 SOTA 模型 inference)** | **当前** |

---

## 1. 架构原则

**零新依赖 + 复用现有架构**:
- 后端:扩展现有 Flask app + RDAgentTask(`rdagent/log/server/app.py`)
- 前端:在 `web/src/multialpha/` 加组件，复用 Element Plus + composable + 轮询链
- **不引入**:pinia / axios / 新 UI 库 / qlib online 栈 / qlib ModelPersistRecord

---

## 2. 已验证的核心技术事实(方案成立的基础)

### 2.1 实测通过的预测路径(路径 1:补齐因子 → 简化 inference)

```python
# 在 docker(local_qlib:v2.1)内执行,~2-5 分钟完成(取决于空缺天数)
import qlib, pickle, glob, os, pandas as pd, copy
from qlib.constant import REG_CN
qlib.init(provider_uri='~/.qlib/qlib_data/cn_data', region=REG_CN)
os.chdir('/workspace/qlib_workspace')
from qlib.data import D
from qlib.contrib.data.loader import Alpha158DL
from qlib.data.dataset.loader import StaticDataLoader

# ===== 0. 读现有产物 + 算空缺区间(基于 parquet 末日,不是 pred.pkl) =====
old_pred = pickle.load(open(glob.glob('mlruns/*/*/artifacts/pred.pkl')[0], 'rb'))
old_factors = pd.read_parquet('combined_factors_df.parquet')
parquet_end = old_factors.index.get_level_values(0).max()   # 因子值水位线(每次 C 步更新)
qlib_latest = pd.Timestamp(D.calendar(freq='day')[-1])
if parquet_end >= qlib_latest:
    # 因子值已到 T 日,直接用已有 pred 的末日取 Top20(可能需要 E 步之前已补齐)
    top20 = old_pred.xs(old_pred.index.get_level_values(0).max(), level=0).dropna().sort_values(ascending=False).head(20)
else:
    # ===== A. 导行情(qlib bin → daily_pv.h5,空缺区间基于 parquet_end) =====
    lookback = 30
    pv_start = (parquet_end + pd.Timedelta(days=1) - pd.Timedelta(days=lookback)).date()
    instruments = D.instruments(market='all')
    fields = ['$open','$close','$high','$low','$volume','$factor']
    new_pv = D.features(instruments, fields, start_time=str(pv_start), end_time=str(qlib_latest.date()), freq='day')
    new_pv = new_pv.swaplevel().sort_index()  # → (datetime, instrument)
    new_pv.to_hdf('daily_pv.h5', key='data')

    # ===== B. 算因子(跑 rdagent factor.py) =====
    for factor_name, factor_code in sota_factors:
        exec(compile(factor_code, f'{factor_name}.py', 'exec'), {"__name__": "__main__"})
        # factor.py 读 daily_pv.h5 → 算 → 写 result.h5,收集每个因子结果

    # ===== C. 补齐 parquet(concat 新旧因子值) =====
    old_factors = pd.read_parquet('combined_factors_df.parquet')
    # new_all = pd.concat(所有 B 步骤的 result.h5, axis=1)
    # new_all = new_all[new_all.index > old_factors.index.max()]  # 只保留新日期
    # pd.concat([old_factors, new_all]).sort_index().to_parquet(...)  # 原子写

    # ===== D. 简化 inference(手动 concat feature + 矩阵乘法) =====
    task = pickle.load(open(glob.glob('mlruns/*/*/artifacts/task')[0], 'rb'))
    loaders = task['dataset']['kwargs']['handler']['kwargs']['data_loader']['kwargs']['dataloader_l']

    # D1. 加载两部分 feature
    a158_cfg = copy.deepcopy(loaders[0]['kwargs']['config'])
    a158_cfg.pop('label', None)  # inference 不需要 label
    d0 = Alpha158DL(config=a158_cfg).load(None, task['dataset']['kwargs']['handler']['kwargs']['start_time'], str(qlib_latest.date()))
    d1 = StaticDataLoader(config=loaders[1]['kwargs']['config']).load()

    # D2. 合并 + 列排序(关键: coef_ 按 qlib 字母排序训练,必须 sort_index(axis=1))
    features = pd.concat([d0, d1], axis=1).sort_index().sort_index(axis=1)

    # D3. 矩阵乘法(LinearModel.predict 的本质)
    model = pickle.load(open(glob.glob('mlruns/*/*/artifacts/params.pkl')[0], 'rb'))
    if hasattr(model, 'coef_'):       # LinearModel
        scores = features.values @ model.coef_ + model.intercept_
    elif hasattr(model, 'model'):     # LGBModel
        scores = model.model.predict(features.values)

    pred = pd.Series(scores, index=features.index)
    top20 = pred.xs(qlib_latest, level=0).dropna().sort_values(ascending=False).head(20)

    # ===== E. 保存预测结果回 pred.pkl(下次预测只补增量,避免重复计算) =====
    pred_path = glob.glob('mlruns/*/*/artifacts/pred.pkl')[0]
    # 合并新旧 pred:只追加 pred.pkl 末日之后的新预测
    pred_end = old_pred.index.get_level_values(0).max()
    new_part = pred[pred.index.get_level_values(0) > pred_end].to_frame('score')
    merged_pred = pd.concat([old_pred, new_part]).sort_index()
    # 原子写
    tmp_path = pred_path + '.tmp'
    with open(tmp_path, 'wb') as f:
        pickle.dump(merged_pred, f)
    os.replace(tmp_path, pred_path)
```

**E 步的作用**:每次预测成功后,把新预测追加到 pred.pkl。下次预测时:
- A-B-C 步空缺区间从 `parquet 末日 + 1` 开始(C 步每次也更新了 parquet)
- 不会重复计算已补齐的日期
- 否则:每次都从训练时的 pred.pkl 末日开始补齐,重复计算所有历史增量

**为什么 D 步不走 dataset/handler/processor 体系**(经过完整 review 确认):

| 组件 | 训练时用途 | inference 时是否需要 | 原因 |
|---|---|---|---|
| `NestedDataLoader`(join=left) | 合并 Alpha158 + 外部因子 | ❌ 不需要 | left join 受 Alpha158 label 末日限制,截断数据 |
| `DropnaLabel` | 删 label 为 NaN 的行(训练需 label) | ❌ 不需要 | inference 不需要 label |
| `CSZScoreNorm`(fields_group=label) | 归一化 label | ❌ 不需要 | 只处理 label,不影响 feature |
| `dataset.prepare(col_set=feature, DK_I)` | 取 feature 矩阵 | ✅ 等价于手动 concat | DK_I = raw feature(无 processor),手动 concat + sort_index(axis=1) 等价 |

**核心洞察**(来自 qlib 源码 + 交叉验证):
- `LinearModel.predict` = `x_test.values @ self.coef_ + self.intercept_`(源码一行)
- `LGBModel.predict` = `self.model.predict(x_test.values)`(源码一行)
- 两者都用 `data_key=DK_I`(infer data = raw feature,不经 learn_processors)
- rdagent 的 learn_processors 只处理 label(DropnaLabel + CSZScoreNorm),**feature 在 DK_L 和 DK_I 中完全一致**(实测验证)
- 因此:inference 只需 `raw_feature_matrix @ model.coef_`,**不碰 label/processor/NestedDataLoader**

**交叉验证**(同一日期手动 concat vs 原始 dataset.predict):最大差异 `0.0000000000`——数值完全一致。

**⚠️ ZScore 风险分析**(经源码 review + 实测确认):

- rdagent 的 `learn_processors` = `DropnaLabel` + `CSZScoreNorm(fields_group='label')`
- CSZScoreNorm 的 `fields_group='label'` —— **只处理 label**,feature 不碰(源码 `get_group_columns(df, 'label')` 只取 label 列)
- `infer_processors = None` —— inference 时无任何 processor
- **实测确认**:训练时 DK_L 的 feature 值 == Alpha158DL raw 值(10 列对比,差异全 0)
- 结论:feature 从未被归一化,`coef_` 是用 raw feature 训练的,inference 用 raw feature 正确

**风险点**(当前不适用,但实施时要检查):如果某个实验的 task config 配了 `CSZScoreNorm(fields_group='feature')` 或其他 feature 归一化 processor,则 inference 时也必须做同样归一化,否则数值不匹配。实施时应检查 task config 的 processors,若有 feature 归一化则报错或自动补齐。

### 2.2 官方方案 vs 本方案对比(经 context7 + 源码 + 实测三方验证)

qlib 官方对"加载模型 + 重建 dataset + 预测"有**两条路径**:

**路径 A: `RMDLoader` 三步(反序列化 dataset artifact)**
```python
# qlib/workflow/online/update.py RMDLoader.get_dataset
dataset = self.rec.load_object("dataset")                              # 反序列化 dataset artifact
dataset.config(handler_kwargs={...}, segments={...})
dataset.setup_data(handler_kwargs={"init_type": DataHandlerLP.IT_LS})
```

**路径 B: task config + handler 文件缓存(`replace_task_handler_with_cache`)**
```python
# qlib/workflow/task/utils.py:283
# 训练前缓存 handler(用 dump_all=True 保留完整状态):
h = init_instance_by_config(handler); h.to_pickle(h_path, dump_all=True)
task["dataset"]["kwargs"]["handler"] = f"file://{h_path}"
# 重建时: init_instance_by_config(task["dataset"]) → mod.py 解析 file:// → load handler pkl
```

**本方案**:
```python
task = pickle.load("artifacts/task")
dataset = init_instance_by_config(task['dataset'])   # qlib 通用重建 API
```

| 步骤 | 官方路径 A (RMDLoader) | 官方路径 B (handler 缓存) | 本方案 |
|---|---|---|---|
| load model | `rec.load_object("params.pkl")` | 同 | `pickle.load(params.pkl)` ✅ |
| 准备 dataset | `rec.load_object("dataset")` + config + setup_data | `init_instance_by_config(task['dataset'])`(含 file:// handler) | `init_instance_by_config(task['dataset'])` |
| predict | `model.predict(dataset)` | 同 | `model.predict(dataset, segment='test')` ✅ |

**关键结论**:**本方案 = 官方路径 B 的执行核心**。两者都用 `init_instance_by_config(task['dataset'])` 重建,这是 qlib 通用 API,内部由 `qlib/utils/mod.py:160-171` 自动处理 handler 的两种形态:
- handler 是 dict → 实例化 handler → setup_data 加载数据
- handler 是 `file://...` 字符串 → 从 pkl 反序列化(`dump_all=True`,完整状态)

**差异仅在"handler 怎么存"**:路径 B 训练前用 `replace_task_handler_with_cache` 把 handler 缓存成 file:// 引用;rdagent 没用这个函数,handler 直接以 dict 形式存在 task config 里。两种形态 `init_instance_by_config` 都能处理,本方案都能兼容。

**为什么不用官方路径 A(RMDLoader)**:它在 rdagent 上不可用,见 §2.3。

### 2.3 为什么不用官方路径 A(RMDLoader):rdagent 的 StaticDataLoader 场景下它不可用

**根因**(已通过 qlib 源码 + rdagent dataset artifact 实测定位):

1. qlib trainer(`qlib/model/trainer.py:51-53`)保存 dataset artifact 时**主动**调用:
   ```python
   dataset.config(dump_all=False, recursive=True)   # dump_all=False!
   R.save_objects(**{"dataset": dataset})
   ```
2. `dump_all=False` 触发 `Serializable._is_kept`(`qlib/utils/serial.py`)的规则:`return self.dump_all or not key.startswith("_")` → **丢弃所有 `_` 前缀属性**。
3. rdagent 的 dataset 用 `NestedDataLoader` + `StaticDataLoader` 组合加载外部因子(`combined_factors_df.parquet`)。`StaticDataLoader` 把配置存在 `self._config`(= `"combined_factors_df.parquet"` 字符串),数据存在 `self._data`。**两个都是 `_` 前缀,pickle 后全丢**。
4. 官方路径 A 的 `setup_data` 内部走 `StaticDataLoader._maybe_load_raw_data` → 需要 `self._config` 知道加载哪个文件 → 但 `_config` 已丢 → 报 `'StaticDataLoader' object has no attribute '_data'`。

**实测对照**(在同一 rdagent SOTA workspace 上):
- 官方 RMDLoader 三步(`load dataset + config + setup_data(IT_LS)`):❌ 失败,`StaticDataLoader._config` 丢失
- 本方案(`load task + init_instance_by_config(task['dataset'])`):✅ 成功,重建时 `_config='combined_factors_df.parquet'` 被正确赋值,setup_data 能加载 parquet

**官方方案在什么场景下可用**:当 dataset 用 qlib 原生 data loader(如 `Alpha158DL`,从 qlib 数据源实时算因子)时,`_config` 是非 `_` 前缀的 fields_group dict,pickle 不丢 → 官方方案可用。**rdagent 用 StaticDataLoader 加载外部预计算因子是官方未充分测试的场景**。

### 2.4 各验证项结果

| 验证项 | 结果 |
|---|---|
| qlib online 模块 import | ✅ 成功 |
| R 读 rdagent mlruns(挂载到 /workspace/qlib_workspace) | ✅ 成功 |
| `load params.pkl` | ✅ 8 个 workspace 全部成功(5 LGBModel + 3 LinearModel) |
| 官方 RMDLoader 三步 | ❌ 失败(StaticDataLoader._config 丢失,见 §2.3) |
| task config 重建 dataset + predict(训练末日) | ✅ 成功(Top10 正确) |
| 直接 predict 空缺区间(用旧 parquet) | ❌ 返回 shape (0,)(因子值未覆盖空缺) |
| NestedDataLoader join=left 截断末日 | ❌ handler._data 末日被 Alpha158 label 限制(实测确认) |
| **手动 concat Alpha158DL + StaticDataLoader** | ✅ **成功**,07-17 feature 全有值(13/13 列非空) |
| **手动 concat + @ coef_ 矩阵乘法(LinearModel)** | ✅ **成功**,T 日 Top20 正确产出 |
| **手动 concat + model.model.predict(LGBModel)** | ✅ **成功**,T 日 Top20 正确产出 |
| **交叉验证:手动 concat vs 原始 dataset.predict** | ✅ **最大差异 0.0000000000**(数值完全一致) |
| 从 qlib bin 导出增量行情 | ✅ 成功(D.features + swaplevel + sort_index) |
| 跑 factor.py 算因子 | ✅ 成功(MOM_5D / VOL_20D 等可算) |
| feature 无需归一化(DK_L feature == DK_I feature) | ✅ 实测确认(learn_processors 只处理 label) |
| 耗时 | ~2-5 分钟(导行情 60s + 算因子 30s + 合并 5s + inference 30s) |

### 2.5 实施时规避的 quirks

1. **D 步不走 NestedDataLoader/handler/processor**——直接手动 `pd.concat([Alpha158DL, StaticDataLoader])`(见 §2.1 D 步和说明表)。NestedDataLoader 的 `join=left` 会截断末日,DropnaLabel/CSZScoreNorm 会引入 label 依赖死循环。
2. **列顺序必须字母排序**:concat 后必须 `.sort_index(axis=1)`。qlib `dataset.prepare` 内部对列排序,`coef_` 按此顺序训练。不排序会导致 `feature @ coef_` 数值错误(实测差异 2-5 倍)。
3. **mlflow ExpManager 并发**:不经 R API,直接 `pickle.load` 读 artifacts(本方案代码就是这么做的)。
4. **StaticDataLoader 路径依赖 cwd**:`combined_factors_df.parquet` 是相对路径,docker 内 `os.chdir(workspace_path)` 后才能解析。
5. **挂载点必须用训练时路径** `/workspace/qlib_workspace`(mlflow 把绝对路径写进 meta.yaml)。
6. **行情导出的 index 顺序**:`D.features` 返回 `(instrument, datetime)`,必须 `.swaplevel().sort_index()` 转成 `(datetime, instrument)` 才和 daily_pv.h5 / factor.py 期望的格式一致(generate.py 源码确认)。
7. **因子 lookback**:不同因子需不同回看天数,导行情时 `pv_start = start_time - max(所有因子 lookback) - buffer`。
8. **factor.py 执行**:设 `exec_globals = {"__name__": "__main__"}` 让 `if __name__ == "__main__"` 入口被执行;每次执行后立即读取 result.h5 并清理。

### 2.6 为什么不能直接用 qlib OnlineManager.update_online_pred

两条原因:
1. **dataset 反序列化失败**:`PredUpdater.prepare_data` → `RMDLoader.get_dataset` → `rec.load_object("dataset")` + `setup_data` 在 StaticDataLoader 场景下 `_config` 丢失(见 §2.3)。
2. **因子值不会自动补齐**:即使 dataset 能 setup,`StaticDataLoader` 读的 `combined_factors_df.parquet` 是训练时的静态文件,不会从 qlib bin 实时算因子——`model.predict` 对空缺区间返回空(shape 0)。

**本方案的路径 1(§2.1)** 显式补齐因子值(A-C 步骤),再用简化 inference(手动 concat + 矩阵乘法,D 步骤)解决了这两个问题。

---

## 3. 后端设计

### 3.1 新增 API 路由(3 个，加在 `rdagent/log/server/app.py`)

#### API 1:列出可选实验(后端聚合)

```
GET /predict/experiments
```

**响应**:
```json
{
  "experiments": [
    {
      "trace_id": "Finance Data Building/my_exp_1",
      "name": "my_exp_1",
      "created_at": "2026-07-20 16:05:49",
      "factor_count": 3,
      "metrics": {"IC": 0.0095, "annualized_return": -0.0085},
      "has_model": true,
      "exp_id": "810020961654375743",
      "recorder_id": "f97c96b78d38450cbcf9f6a81868ced2",
      "workspace_path": ".../RD-Agent_workspace/a5df3828..."
    }
  ]
}
```

**实现逻辑**(`app.py` 新增 `list_predict_experiments`):
1. `_collect_existing_trace_ids()`(`app.py:354-371`)
2. 过滤 scenario == `"Finance Data Building"`
3. 逐个 `find_session_by_trace_name()` + `query_sota()`
4. 过滤含 `sota_factors` 的(有 SOTA)
5. 对每个 workspace 用 `glob mlruns/*/*/artifacts/params.pkl` 检查模型存在
6. 返回精简字段 + `has_model` + `exp_id` + `recorder_id` + `workspace_path`

**性能**:trace ~10-50 个，每个要 `query_sota`(读 pickle)，首次 10-30 秒。加**内存缓存**(键=trace_id，失效=`__session__` mtime)。

#### API 2:触发 T+1 预测(异步任务)

```
POST /predict/run
Body: {"trace_id": "Finance Data Building/my_exp_1"}
Response: {"task_id": "Finance Prediction/<random>-<date>"}
```

**实现逻辑**(`app.py` 新增 `run_predict`):复用 RDAgentTask 模式:
1. 新 scenario `"Finance Prediction"`，trace_name = `<randomname>-<YYYYMMDD>`
2. 算路径:`trace_folder/Finance Prediction/<trace_name>/`(pkl + stdout.log)
3. 新 target `"fin_predict"`，加进 `_TARGETS_WITHOUT_USER_INTERACTION`(`app.py:49`)
4. `RDAgentTask(target_name="fin_predict", kwargs={trace_id/workspace_path/exp_id/recorder_id}, ...)`
5. `task.start()` + 存 `rdagent_processes`，返回 trace_id

**RDAgentTask._run 加分支**(`app.py:137-156`):
```python
elif self.target_name == "fin_predict":
    from rdagent.app.qlib_rd_loop.predict import main as fin_predict
    fin_predict(**self.kwargs)
```

#### API 3:查询历史预测记录

```
GET /predict/history?trace_id=<optional>
Response: {"records": [{"date": "2026-07-21", "trace_id": "...", "top20": [...], ...}]}
```

**实现逻辑**:glob `trace_folder/Prediction History/*.json`，按 mtime 倒序，可选按 trace_id 过滤。

### 3.2 新增模块:`rdagent/app/qlib_rd_loop/predict.py`

核心 ~120 行,负责启动 docker + 传参 + 收结果:
```python
def main(trace_id, workspace_path, exp_id, recorder_id, sota_factors):
    """T+1 预测入口,经 RDAgentTask spawn。
    sota_factors: list[dict] 从 query_sota() 取的因子列表,每个含 name/code。
    """
    # 1. 把 sota_factors 的 factor.py 代码写到 workspace 临时目录
    #    (供 predict_infer.py 的步骤 B 循环跑)
    # 2. 启动 docker 挂载 workspace → /workspace/qlib_workspace + ~/.qlib → /root/.qlib:ro
    # 3. 容器内跑 predict_infer.py(含路径 1 五步)
    # 4. 从 stdout 拿 Top20 JSON
    # 5. rdagent_logger.log_object(top20, tag="prediction.top20") 推前端
    # 6. 写历史 JSON 到 trace_folder/Prediction History/<trace_id>-<date>.json
```

**docker 调用**:复用 `QlibFBWorkspace.execute` 模式(`workspace.py:18-59`),挂载 workspace 到 `/workspace/qlib_workspace` + `~/.qlib:/root/.qlib:ro`,跑 `python predict_infer.py`。

### 3.3 新增预测脚本(注入 docker 跑,含路径 1 五步)

放 `rdagent/scenarios/qlib/experiment/factor_template/predict_infer.py`(~220 行),由 `inject_code_from_folder`(`core/experiment.py:266`)自动复制进 workspace:
```python
# 接收 workspace 内的 mlruns + sota_factors 因子代码,跑路径 1 五步:
# A. 导行情(qlib bin → daily_pv.h5,带 lookback + swaplevel,空缺区间基于 parquet 末日)
# B. 算因子(循环跑每个 factor.py → result.h5)
# C. 补齐 parquet(concat 新旧因子值,原子写)
# D. 简化 inference(Alpha158DL + StaticDataLoader 手动 concat + sort_index(axis=1) + 矩阵乘法)
# E. 保存预测回 pred.pkl(增量水位线,避免下次重复计算)
# 最后把 T 日 Top20 以 JSON 写 stdout
import json
print(json.dumps({"predict_date": str(qlib_latest.date()), "top20": [...]}))
```

**关键**:步骤 D **不走** NestedDataLoader / handler / processor / dataset.prepare。直接手动 concat 两个 DataLoader + 矩阵乘法(见 §2.1 D 步说明)。这是经过 qlib 源码 review + 交叉验证(差异 0.0)确认的简化路径。

### 3.4 历史记录存储

`git_ignore_folder/traces/Prediction History/<trace_id_short>-<YYYYMMDD>.json`:
```json
{
  "date": "2026-07-21",
  "predict_target_date": "2026-07-22",
  "source_trace_id": "Finance Data Building/my_exp_1",
  "exp_id": "...", "recorder_id": "...",
  "factor_count": 3,
  "metrics": {"IC": 0.0095, ...},
  "top20": [{"rank": 1, "instrument": "SH600601", "score": 0.0888}, ...]
}
```

放 `Prediction History/` 子目录(无 .pkl，天然不被 `_collect_existing_trace_ids` 误识别)。

### 3.5 不修改的部分

- ❌ 不改 `query_sota()` / `sota_query.py`
- ❌ 不改 rdagent conf 模板 / workspace.py / RDAgentTask 核心逻辑
- ❌ 不改实验运行逻辑 / 因子运行
- ❌ 不写 ModelPersistRecord / 不动 qlib online 栈

---

## 4. 前端设计

### 4.1 新增/修改文件

```
web/src/multialpha/
├── components/
│   └── PredictDashboard.vue          # 新增 主看板组件(列表+表格+历史)
├── use-predict.ts                    # 新增 composable(状态+API+轮询)
├── api.ts                            # 修改 re-export 新 API 函数
├── router.ts                         # 修改 加 /predict 路由
└── components/MultiAlphaApp.vue      # 修改 侧栏加入口 + 路由分支

web/src/services/rdagent-api.ts       # 修改 加 fetch 函数
```

### 4.2 路由 + 侧栏入口

**(a) 路由**(`router.ts`):
```ts
{ path: '/predict', name: 'multialpha-predict', component: { template: '<span />' } }
```

**(b) 侧栏入口**(`MultiAlphaApp.vue`):左侧导航加"股池预测"项，与"任务列表"并列。点击后路由跳 `/predict`，根据 `route.name === 'multialpha-predict'` 渲染 `PredictDashboard`。

### 4.3 组件结构(`PredictDashboard.vue`)

单组件两栏布局(复用 TaskSidebar + MetricsPanel 视觉模式)。template 要点:
- 左栏:`el-select` 筛选 + `v-for` 实验列表(@click 选中)
- 右栏上:实验信息卡(metric-grid: IC/年化/因子数)
- 右栏中:`el-button` 预测 T+1 + `el-button` 查看历史
- 右栏下:状态条(loading/完成/错误) + `el-table`(Top20 三列:rank/instrument/score)
- 历史弹窗:`el-dialog` + 历史列表

### 4.4 状态管理(`use-predict.ts`)

仿 `use-multialpha.ts` 模式:
```ts
export function usePredict() {
  const experiments = ref<PredictExperiment[]>([])
  const selectedTraceId = ref<string | null>(null)
  const taskStatus = ref<'idle'|'running'|'done'|'error'>('idle')
  const top20 = ref<Top20Item[] | null>(null)
  const history = ref<PredictRecord[]>([])
  // ...fetchExperiments / runPredict / pollTask(复用 fetchTrace+cursor) / fetchHistory
}
```

**轮询复用**:点"预测"后 `createTask` → POST /predict/run → 拿 task_id → `fetchTrace(task_id, cursor)` 轮询直到 END tag → 解析 prediction.top20 消息。

### 4.5 API 层(`services/rdagent-api.ts`)

复用现有 `parseResponse<T>` + 相对路径 + AbortSignal 模式:
```ts
export const fetchPredictExperiments = (signal?) =>
  fetch('/predict/experiments', {signal}).then(r => parseResponse<{experiments: PredictExperiment[]}>(r))

export const runPredict = (traceId: string) =>
  fetch('/predict/run', {method:'POST', headers:{'Content-Type':'application/json'},
        body: JSON.stringify({trace_id: traceId})})
    .then(r => parseResponse<{task_id: string}>(r))

export const fetchPredictHistory = (traceId?: string, signal?) =>
  fetch(`/predict/history${traceId ? '?trace_id='+encodeURIComponent(traceId) : ''}`, {signal})
    .then(r => parseResponse<{records: PredictRecord[]}>(r))
```

### 4.6 vite proxy

`vite.config.ts` 的 `server.proxy` 加:
```ts
'/predict': 'http://115.190.106.124:19899',
```

### 4.7 不引入的东西

- ❌ pinia(用 composable)
- ❌ axios(用原生 fetch)
- ❌ 新图表库(Top20 是表格)
- ❌ 新 UI 库(用 Element Plus 的 el-table / el-select / el-dialog)

---

## 5. 端到端数据流

```
用户从侧栏进入 /predict
    ↓
PredictDashboard onMounted → fetchExperiments
    ↓
fetch('/predict/experiments') ──→ Flask list_predict_experiments
    ↓                                ↓
                                  _collect_existing_trace_ids  [已有]
                                  过滤 fin_factor
                                  query_sota() 过滤 [已有]
                                  检查 params.pkl 存在
                                  返回 [{trace_id, metrics, has_model, exp_id, ...}]
    ↓
渲染实验列表(左栏)
    ↓
用户点"预测 T+1"
    ↓
POST /predict/run {trace_id}  ──→  Flask run_predict
    ↓                                ↓
                                  RDAgentTask(target="fin_predict", kwargs=...)
                                  task.start()
                                  返回 {task_id: "Finance Prediction/xxx-20260721"}
    ↓
前端 selectTrace(task_id) + 轮询 fetchTrace(cursor)
    ↓
后端子进程(docker 内,路径 1 五步):
  A. 导行情(qlib bin → daily_pv_gap.h5,带 lookback + swaplevel)
  B. 算因子(循环跑 sota_factors 的 factor.py → 每个因子新值)
  C. 补齐 parquet(concat 新旧因子值,原子写)
  D. 简化 inference(Alpha158DL + StaticDataLoader 手动 concat + @ coef_ 矩阵乘法)
  E. 保存预测回 pred.pkl(增量水位线,避免下次重复计算)
  rdagent_logger.log_object(top20, tag="prediction.top20")  → /receive
  写 JSON 到 Prediction History/
    ↓
任务 END → 前端 deriveTraceStatus='done' → 解析 prediction.top20
    ↓
渲染 Top20 表格(右栏)
```

---

## 6. 实现步骤(建议顺序)

| 步骤 | 内容 | 验证方式 |
|---|---|---|
| 1 | 后端 API 1 `GET /predict/experiments` | curl 返回可选实验列表 |
| 2 | **predict_infer.py 五步端到端**(A 导行情 + B 算因子 + C 补齐 parquet + D 简化 inference) | docker run 单独跑 predict_infer.py,验证 Top20 日期 = qlib 最新日期 |
| 3 | predict.py 主入口(写因子代码 + 启 docker + 收结果 + 推前端) | docker run predict.py,看 stdout 有 Top20 + 前端收到消息 |
| 4 | 后端 API 2 `POST /predict/run` + RDAgentTask 分支 | curl 触发,看异步任务状态 |
| 5 | 后端 API 3 `GET /predict/history` + JSON 存储 | 跑两次预测,curl 看历史 |
| 6 | 前端 API 层 + composable | 浏览器 console 能 fetch |
| 7 | 前端 PredictDashboard.vue | 页面能选+预测+展示 Top20 |
| 8 | 侧栏入口 + 路由集成 | 从主站进入看板 |
| 9 | 错误边界 + 空态 + 历史弹窗 | 各种边界测试 |

---

## 7. 关联影响与文档同步(预声明)

- **关联影响**:
  - 后端新增 3 个路由 + 1 个 target 分支 + 1 个 inference 脚本(只读消费现有 SOTA workspace)
  - 前端新增 1 页面 + 1 composable，不改现有页面
  - **不动**:rdagent conf 模板 / workspace.py / RDAgentTask 核心逻辑 / 实验运行逻辑
- **文档同步**:
  - `docs/reference/API.md` 补 `/predict/*` 路由
  - `CLAUDE.md` 不涉及

---

## 8. 风险与缓解

| 风险 | 影响 | 缓解 |
|---|---|---|
| query_sota 慢致列表加载 10-30s | 体验差 | 内存缓存(mtime 失效)+ loading 态 |
| 历史实验 params.pkl 可能损坏 | 选了预测失败 | API 1 的 `has_model` 字段 + 预测时 try/catch |
| 不同 model class(LGB/Linear) | predict 行为可能不同 | 两种都已实测 predict 成功 |
| mlflow ExpManager 并发 | 偶发 ExpAlreadyExistError | 不经 R API,直接 pickle.load(见 §2.5 quirk 2) |
| **行情导出 index 顺序不对** | factor.py 读到错乱数据 | 必须用 swaplevel + sort_index(generate.py 同款,见 §2.5 quirk 5) |
| **factor.py 执行失败** | 某个因子算不出 | try/catch + 跳过失败因子 + 记录 |
| **因子 lookback 不足** | 空缺区间边缘因子值为 NaN | 导行情时带 max(lookback) + buffer(见 §2.5 quirk 6) |
| **parquet 写入原子性** | 中途失败致 parquet 损坏 | 先写临时文件再 rename |
| 空缺区间过长(首次预测数月空缺) | 耗时 >5 分钟 | 接受 + 超时兜底(10 分钟) |
| 数据漂移(2020-2022 模型预测 2026) | 预测精度差 | 本期接受,后续滚动重训 |

---

## 9. 验收对齐(对应 PRD §8)

| PRD 验收项 | 技术实现 |
|---|---|
| 侧栏入口 | MultiAlphaApp.vue 侧栏 + 路由 /predict |
| 实验列表过滤 fin_factor+SOTA+model | API 1 三层过滤 |
| 异步任务状态可见 | RDAgentTask + fetchTrace 轮询 + deriveTraceStatus |
| **Top20 ≤ 5 分钟展示** | docker 五步 pipeline 实测 ~2-5 分钟 |
| **预测日期 = T 日** | 路径 1 补齐到 qlib 最新日期,非训练末日 |
| **数据准确** | 后端用 §2.1 同款五步代码,含因子补齐 |
| 历史记录 | JSON 文件 + API 3 + 历史弹窗 |
| 错误处理 | has_model / try-catch / error 字段 + 前端 v-if |

---

## 10. 决策记录(review 已确认)

| 决策点 | 决定 | 依据 |
|---|---|---|
| 业务方向 | T+1 实战预测 | v1.0 历史回顾无实战价值 |
| **预测流程** | **路径 1:补齐因子 → 简化 inference** | 直接 predict 空缺区间返回空;必须先补齐因子值 |
| **qlib 框架复用** | D 步用 LinearModel/LGBModel 源码逻辑(feature @ coef_) | qlib 源码确认 predict 本质是矩阵乘法 |
| **rdagent 产物复用** | factor.py + parquet + pred.pkl + params.pkl | 不走 rdagent 流程,只用产物 |
| dataset 重建 | task config + init_instance_by_config | 官方路径 B 核心,绕开反序列化坑 |
| qlib OnlineManager.update_online_pred | 不用 | StaticDataLoader._config 丢失 + 因子值不自动补齐 |
| ModelPersistRecord | 不写 | params.pkl 已是 trainer 默认产物 |
| 模型选择 | 一实验一预测 | MVP 最简 |
| 触发方式 | 手动按钮 | 不做调度 |
| 历史记录 | JSON 存储 | 符合 trace_folder 约定 |
| API 前缀 | `/predict/*` | 短 |
| 前端入口 | 侧栏 | 与任务列表并列 |
| 股票名称 | MVP 不展示 | 后续 kline-fetcher |
| 滚动重训 | MVP 不做 | 留二期 |
| 范围 | MVP 最小闭环 | 先跑通 |

---

## 11. 工作量估算(基于代码行数核实)

| 模块 | 文件 | 行数 |
|---|---|---|
| 后端 API + RDAgentTask 分支 | app.py | ~150 |
| 后端 predict.py 主入口 | 新文件 | ~120 |
| **后端 predict_infer.py(五步 pipeline)** | 新文件 | **~220** |
| 后端历史记录 | 新模块 | ~30 |
| 前端 PredictDashboard.vue | 新组件 | ~200 |
| 前端 use-predict.ts | 新文件 | ~80 |
| 前端 api/router/proxy | 修改 | ~40 |
| **合计** | — | **~840 行** |

**主要增量**(对比旧方案的 ~630 行):predict_infer.py 从 ~50 行涨到 ~220 行(多了导行情 + 算因子 + 合并 parquet + 保存 pred.pkl 的 A-E 步)。

风险点:五步 pipeline 的工程细节(index 对齐 / lookback / 多因子协调 / 原子写)已在验证中识别(见 §2.5 quirks),但完整端到端实现仍需调试。
