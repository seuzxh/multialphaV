# 股池预测看板 PRD

> 版本: v2.0 (MVP — T+1 实战预测)
> 状态: 待 review
> 日期: 2026-07-21
> 范围: 仅 fin_factor 场景 · 复用 SOTA 模型 inference · 手动触发

---

## 0. 版本演进说明

| 版本 | 方向 | 状态 |
|---|---|---|
| v1.0 | 读 pred.pkl 末日 Top20(历史回顾) | **废弃**(无实战价值) |
| **v2.0** | **T+1 实战预测(复用 SOTA 模型 inference)** | **当前** |

---

## 1. 背景与目标

### 1.1 背景

rdagent fin_factor 场景产出 SOTA 实验，每个 SOTA 实验包含:
- 一组通过验证的因子(代码 + 表达式)
- 基于 LightGBM/LinearModel 训练的模型(qlib trainer 自动持久化为 `params.pkl`)
- test 段的回测指标(IC / 年化收益 / 最大回撤)

用户需要把这些 SOTA 因子**应用于实盘**:每日 T 日晚间，用 SOTA 模型对 T+1 日做股池预测，输出 Top20。

### 1.2 目标

建一个 Web 看板，让用户:
1. 选择一个**有 SOTA + 有 params.pkl** 的 fin_factor 实验
2. 点"预测 T+1"按钮，后端用该实验的已训练模型 + T 日最新行情做 inference
3. 查看 T+1 日 Top20 股池(代码 + 得分)
4. 历史预测记录可回看

### 1.3 非目标(本期不做)

- ❌ fin_model / fin_quant 场景
- ❌ 多模型 ensemble
- ❌ 模型每日重训(复用已训练模型 inference)
- ❌ 滚动重训 / walk-forward
- ❌ 定时自动触发(手动按钮)
- ❌ 实时行情 / K线 / 涨跌幅
- ❌ 历史预测的次日收益跟踪(留二期)
- ❌ 股票中文名称(留二期，通过 kline-fetcher 获取)

---

## 2. 用户故事

```
作为量化研究员，
T 日晚间(收盘后)打开看板，
选一个跑完的 SOTA 因子实验，
点"预测 T+1"按钮，
等待约 2-5 分钟后看到明日 Top20 股池名单，
据此决定次日交易。
```

---

## 3. 功能需求

### 3.1 功能 1:实验选择

**需求**:看板展示可选的 fin_factor 实验列表，用户选择一个作为预测的模型来源。

**可选实验判定**(三条件全满足):
1. **场景是 fin_factor**(过滤 fin_model / fin_quant / fin_factor_report)
2. **有 SOTA 产物**(`query_sota()` 返回含 `sota_factors`)
3. **模型可 load**(`experiment_workspace_path/mlruns/*/*/artifacts/params.pkl` 存在)

**交互**:
- 列表展示:实验名 / 创建时间 / SOTA 因子数 / 核心指标(IC / 年化收益)
- 三条件任一不满足的实验**不出现在列表**(避免误选)
- 选中后**自动锁定**该实验的 SOTA 模型作为预测源(本期"一实验一预测"，用户不换模型)

### 3.2 功能 2:T+1 预测(异步任务)

**需求**:选中实验后点"预测 T+1"按钮，触发后端异步预测任务。

#### 3.2.1 时间范围(核心业务逻辑)

预测涉及三个关键时间点:

| 时间点 | 含义 | 来源 |
|---|---|---|
| **parquet 末日** | 因子值最新日期(每次 C 步会更新) | `combined_factors_df.parquet` 的 datetime index 最大值 |
| **T 日** | qlib bin 行情数据最新日期 | `~/.qlib/qlib_data/cn_data` 的 calendar 末日 |
| **空缺区间** | `[parquet 末日 + 1, T 日]` | 因子值缺失的日期段(需 A-C 步补齐) |

**预测产出** = T 日 Top20(用户 T 日晚间据此做 T+1 日交易决策)。

> **为什么有空缺**:rdagent 跑 SOTA 实验时一次性算好因子值(`combined_factors_df.parquet`)。之后 qlib bin 行情每天更新,但因子值不会自动跟进——形成 `[parquet末日+1, T日]` 的空缺区间。
>
> **增量效率**:每次预测后 C 步会更新 parquet(因子值水位线),E 步会更新 pred.pkl(预测水位线)。下次预测的空缺区间从 `parquet 末日 + 1` 开始,**不会重复计算已补齐的日期**。

#### 3.2.2 执行流程(五步,后端全部在 docker 内执行)

遵循 qlib 源码确认的 predict 机制(find-docs 查 qlib github + 源码 review),只用 rdagent 产物(factor.py / pred.pkl / params.pkl / parquet),不走 rdagent 流程:

```
A. 导行情(补齐空缺区间的 qlib bin → daily_pv.h5)
   - 复用 generate.py 模式:D.features + swaplevel + sort_index
   - 导出 [parquet 末日 + 1 - lookback, T 日] 的行情
   - lookback = max(所有因子需要的回看天数) + buffer(如 MOM_20D 需前 20 天)

B. 算因子(跑 rdagent 的 factor.py 补齐因子值)
   - 从 SOTA pickle 取 factor.py 代码(sub_workspace_list[].file_dict["factor.py"])
   - 用步骤 A 的 daily_pv.h5 跑每个 factor.py → 每个因子的新值
   - 因子代码是 Python 函数(如 MOM_5D = pct_change(5)),独立可跑

C. 补齐 parquet(合并因子值)
   - 把步骤 B 的新因子值追加到 combined_factors_df.parquet
   - 复用 rdagent process_factor_data 的合并逻辑(utils.py:131)
   - 产出更新后的 parquet(因子值覆盖到 T 日)

D. 简化 inference(手动 concat feature + 矩阵乘法)
   - Alpha158DL(从 qlib bin 实时算 Alpha158 因子) + StaticDataLoader(从更新后 parquet 取 SOTA 因子)
   - 手动 pd.concat 合并 + sort_index(axis=1) 列排序
   - features @ model.coef_(LinearModel) 或 model.model.predict(LGBModel)
   - 不走 NestedDataLoader/handler/processor(见技术方案 §2.1 D 步说明)
   - 取 T 日 Top20

E. 保存预测结果回 pred.pkl(增量水位线,避免下次重复计算)
   - 把新预测追加到 pred.pkl(pred.pkl 末日之后的部分)
   - 下次预测时 A-C 步从 parquet 末日开始,不重复已算的日期
```

#### 3.2.3 耗时(已验证各环节可行)

| 环节 | 耗时 | 说明 |
|---|---|---|
| A. 导行情 | ~10-60 秒 | 仅导增量(空缺天数,首日最久) |
| B. 算因子 | ~10-30 秒 | 每个因子跑一次 factor.py |
| C. 补齐 parquet | ~5 秒 | pandas 合并 |
| D. 简化 inference | ~30 秒 | Alpha158DL 加载 + 矩阵乘法 |
| E. 保存 pred.pkl | ~2 秒 | pickle 追加 |
| **合计** | **首次 2-5 分钟 / 日常 ~1 分钟** | 有 E 步后日常只补 1 天增量 |

> 若 last_end == T 日(无空缺,当日已预测过),跳过 A-C,直接返回已有 Top20(去重)。

**交互**:
- 点按钮后展示**运行中状态**(loading + 进度提示,因耗时 2-5 分钟)
- 任务完成后自动展示 T 日 Top20
- 任务失败展示错误提示
- **同一实验 + 同一 T 日**不重复预测(去重:查历史记录 + pred.pkl 末日已是 T 日)

### 3.3 功能 3:Top20 展示

**展示字段**(MVP 三列):
| 字段 | 说明 | 来源 |
|---|---|---|
| 排名 | 1-20 | 计算得出 |
| 股票代码 | qlib 格式 `SH600601` | pred 的 instrument index |
| 得分 | 模型预测分(浮点) | model.predict 输出 |

> 名称列本期不做(见 §3.5)。预测日期(数据库最新日期，即 T 日)显示在表格标题。

### 3.4 功能 4:历史预测记录

**需求**:每次成功预测后存一条记录，看板可回看历史。

**记录内容**:
- 预测日期(实际跑预测时数据库的最新日期)
- 用的实验 trace_id
- Top20 列表
- 关键元信息(因子数 / 模型 IC)

**存储**:每日一个 JSON 文件，放 `git_ignore_folder/traces/Prediction History/<trace_id>-<YYYY-MM-DD>.json`(符合现有 trace_folder 约定)。

**展示**:看板提供"查看历史"入口，列表展示历史预测(按日期倒序)，点击查看当日 Top20。

### 3.5 股票名称(本期不做)

MVP 不展示名称。qlib `instruments/all.txt` 不含中文称谓。**后续通过 kline-fetcher 工具**获取代码→名称映射，加列。已与用户确认。

---

## 4. 关键技术事实(已通过实测验证)

| 事实 | 验证方式 | 对方案影响 |
|---|---|---|
| `params.pkl` 能 load | docker 内 `pickle.load` 8 个 workspace 全部成功 | 模型可用 |
| 模型类型可能 LGBModel 或 LinearModel | 8 个 workspace 中 5 个 LGBModel / 3个 LinearModel | 后端要兼容多类型 |
| **LinearModel.predict = feature @ coef_ + intercept_** | qlib 源码确认(一行矩阵乘法) | inference 极简,不需 dataset/handler |
| **LGBModel.predict = model.predict(feature.values)** | qlib 源码确认 | 同上 |
| **feature 无需归一化** | 实测 DK_L feature == DK_I feature(learn_processors 只处理 label) | 手动 concat 即可,不需 processor |
| **手动 concat + @ coef_ 交叉验证通过** | 同一日期对比,最大差异 0.0000000000 | 方案数值正确 |
| NestedDataLoader join=left 截断末日 | 实测 handler._data 被 Alpha158 label 末日限制 | D 步不走 NestedDataLoader |
| 不能用反序列化的 dataset | pickle 丢 `_config`，`StaticDataLoader._data` 变 None | 不走 dataset artifact(见技术方案 §2.3) |
| 直接 predict 空缺区间返回空 | 实测 model.predict 对空缺区间返回 shape (0,) | **必须先补齐因子值**(A-C 步骤) |
| combined_factors_df.parquet 末日 < qlib bin | parquet 到训练末日,qlib bin 持续更新 | 因子值有空缺,需补齐 |
| 从 qlib bin 导出增量行情可行 | 实测 D.features + swaplevel + sort_index | A 步骤可行 |
| 跑 rdagent factor.py 算因子可行 | 实测 MOM_5D / VOL_20D 可算 | B 步骤可行 |
| 预测耗时 ~2-5 分钟 | 各环节实测累估(A 60s + B 30s + C 5s + D 30s) | 需异步任务 + 进度提示 |

### ⚠️ 实施时要注意的 quirks(验证中发现)

1. **D 步列顺序必须字母排序**:concat 后 `.sort_index(axis=1)`。qlib `dataset.prepare` 内部对列排序,`coef_` 按此顺序训练。不排序导致数值错误(实测差异 2-5 倍)。
2. **D 步不走 NestedDataLoader/handler/processor**:直接 `pd.concat([Alpha158DL, StaticDataLoader])` + 矩阵乘法。NestedDataLoader 的 left join 截断末日,DropnaLabel/CSZScoreNorm 引入 label 依赖(见技术方案 §2.1)。
3. **mlflow ExpManager 并发**:不经 R API,直接 `pickle.load` 读 artifacts。
4. **不同实验模型类型不同**:LGBModel 用 `model.model.predict(values)`,LinearModel 用 `values @ coef_ + intercept_`。
5. **StaticDataLoader 路径依赖 cwd**:docker 内 `os.chdir(workspace_path)` 后才能解析 parquet 相对路径。
6. **行情导出的 index 顺序**:`D.features` 返回 `(instrument, datetime)`,必须 `.swaplevel().sort_index()`。
7. **因子 lookback**:导行情时取 max(所有因子 lookback) + buffer(如 MOM_20D 需 20 天)。
8. **factor.py 执行**:设 `__name__='__main__'` 让入口函数被调用;每次执行后读取 result.h5 并清理。
9. **ZScore 检查**:当前实验 CSZScoreNorm 只处理 label(`fields_group='label'`),feature 不归一化,手动 concat 方案正确。但如果遇到配了 feature 归一化的实验,inference 需做同样归一化。实施时应检查 task config processors,发现 feature 归一化时报错提示。

---

## 5. 交互流程

```
T 日晚间，用户从侧栏进入看板
    ↓
[加载可选实验列表] GET /predict/experiments
   (后端聚合:遍历 fin_factor trace + query_sota + 检查 params.pkl)
    ↓
展示可选实验列表(名称/时间/因子数/IC/年化)
    ↓
用户选中一个实验
    ↓
用户点"预测 T+1"按钮
    ↓
[触发异步预测任务] POST /predict/run
    ↓
前端轮询任务状态(复用现有 trace 轮询链)
    ↓
后端:docker 内五步 pipeline(导行情 → 算因子 → 补齐 parquet → 简化 inference)
    ↓
任务完成(约 2-5 分钟)
    ↓
展示 T 日 Top20 表格 + 同步写入历史记录
```

**核心特性**:
- 全链路复用现有异步任务 + 轮询基础设施
- 单次预测 ~2-5 分钟(首次最久,后续若无空缺秒级返回)
- 预测产出是 T 日(qlib 最新日期)Top20,非训练末日

---

## 6. 页面结构

**入口**:主站左侧栏加"股池预测"导航项(与现有"任务列表"并列)。

```
┌─────────────────────────────────────────────────┐
│  顶栏:"股池预测"标题 + 返回主站                  │
├──────────────┬──────────────────────────────────┤
│              │  当前实验信息:                   │
│  实验列表    │  - 实验名 / 创建时间              │
│  (左栏)      │  - 因子数 / IC / 年化             │
│              ├──────────────────────────────────┤
│  - 实验 A    │  [预测 T+1 按钮] [查看历史]       │
│  - 实验 B ✓  ├──────────────────────────────────┤
│  - ...       │  预测状态:运行中... / 完成 / 错误 │
│              ├──────────────────────────────────┤
│              │  T+1 Top20 表格(完成时显示):     │
│              │  ┌────┬──────┬──────┐            │
│              │  │排名│代码  │得分  │            │
│              │  ├────┼──────┼──────┤            │
│              │  │ 1  │SH60..│0.0888│            │
│              │  │...                                │
│              │  └────┴──────┴──────┘            │
└──────────────┴──────────────────────────────────┘
```

---

## 7. 边界与错误处理

| 场景 | 处理 |
|---|---|
| 无可选实验 | 列表空态:"暂无可用的 SOTA 因子实验" |
| 选中的实验 params.pkl 缺失 | API 1 已过滤(has_model 字段)，不会出现在列表 |
| qlib bin 行情不可用(数据未更新) | 任务报错,提示"qlib 行情数据加载失败,请先更新 qlib 数据" |
| 空缺区间为空(last_end == T 日) | 跳过 A-C 步骤,直接返回已有 Top20(当日已预测过) |
| factor.py 执行失败(因子代码报错) | 任务报错,提示"因子 {name} 计算失败",附错误详情 |
| parquet 合并失败(列名/index 不对齐) | 任务报错,提示"因子值合并失败" |
| 预测时模型 load 失败(pickle 损坏/版本不兼容) | 任务报错，提示"模型加载失败，请重跑 SOTA 实验" |
| ExpManager 并发错误 | 后端不经 R API,直接 pickle.load 读 artifacts(见 §4 quirks) |
| 同日重复预测 | 后端检测到当日已有该实验的预测记录，提示"今日已预测过，查看历史记录?" |
| 预测超时(>10 分钟) | 任务标记失败 |

---

## 8. 验收标准

1. **侧栏入口**:主站侧栏有"股池预测"导航项，点击进入看板。
2. **实验列表**:列表只显示 fin_factor + 有 SOTA + 有 params.pkl 的实验。
3. **触发预测**:选中实验 + 点"预测 T+1"按钮，触发异步任务，状态可见(运行中/完成/错误)。
4. **Top20 展示**:任务完成后 ≤ 5 分钟内展示 Top20 表格(排名/代码/得分)，按得分降序。
5. **预测日期正确**:Top20 的日期 = qlib bin 最新日期(T 日),不是训练时的历史末日。
6. **数据准确**:Top20 的代码和得分与手动跑五步 pipeline(导行情→算因子→补齐→predict)的结果一致。
7. **历史记录**:每次成功预测写一条 JSON 记录，"查看历史"能看到往日 Top20。
8. **错误处理**:行情缺失/因子失败/模型损坏/重复预测等场景有明确提示，不展示空白或堆栈。

---

## 9. 后续迭代方向(本期不做)

- **股票名称**:kline-fetcher 获取代码→名称映射，Top20 加名称列
- **历史预测次日收益跟踪**:T+2 回头看 T+1 预测的实际涨跌
- **多模型 ensemble**:多个 SOTA 实验综合预测
- **定时自动触发**:每日收盘后 cron 自动跑
- **fin_model / fin_quant 场景支持**
- **滚动重训**:用最新 N 个月数据重新训练，缓解数据漂移
- **集成 qlib OnlineManager**:若 rdagent 调整 dataset 反序列化机制使 `update_online_pred` 可用

---

## 10. 关联文档

- 技术方案:[STOCKPOOL_DASHBOARD_TECH.md](./STOCKPOOL_DASHBOARD_TECH.md)
- 后端 SOTA 查询:`rdagent/log/sota_query.py`
- 后端异步任务:`rdagent/log/server/app.py` RDAgentTask
- 前端架构:`RD-Agent/web/src/multialpha/`
- 方案验证记录:见本 PRD §4 + 技术方案 §2
