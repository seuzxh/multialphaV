# Qlib 四场景验证日志

> 记录 factor / model / quant / factor_from_report 四个场景的真实执行验证过程。
> 每个场景用独立 git worktree 隔离，不污染主工作区。

---

## 前置条件（验证前已就绪）

| 项 | 状态 | 说明 |
|---|---|---|
| conda env multialphav | ✅ Python 3.10.20 + rdagent 0.8.1.dev29 | |
| LLM chat | ✅ glm-5.2 / 智谱 | 实测调用返回 "OK" |
| Embedding | ✅ litellm_proxy/doubao-embedding-vision / 火山方舟 | 修复 openai/ 前缀 bug 后正常（dim=2048）|
| Docker 镜像 | ✅ local_qlib:v2.1（qlib 0.9.7，含 MLFLOW_ALLOW_FILE_STORE） | |
| Docker GPU | ✅ 8 卡 H20 | |
| qlib 数据 | ✅ ~/.qlib/qlib_data/cn_data（真实目录，2020-2026） | |
| 回测日期分段 | ✅ train 2020-2022 / valid 2023-2024.6 / test 2024.7-2026.7 | .env 配了 18 个 QLIB_* 变量 |
| context7 MCP | ✅ 端口 8124 | Hoder-zyf 修改版，无 Redis 依赖 |

## 已发现并修复的问题

### 问题 1：generate.py debug 段时间切片（已修复 commit d027d9ca）

- **现象**：worktree 全新环境（无缓存 h5）跑 generate.py 时，第 17 行 `start_time="2018-01-01", end_time="2019-12-31"` 报 `KeyError: not in index`
- **根因**：数据只有 2020 起，2018-2019 段返回空 DataFrame，后续 `.loc[instrument_list]` 索引空 DataFrame 报错
- **修复**：第 17 行改为 `start_time="2024-01-01", end_time="2025-12-31"`（数据集最近的完整两年）
- **注意**：第 9 行 `loc["2008-12-29":]` 不需要改（pandas 对 DatetimeIndex 的 `.loc[str:]` 会自动从最近可用日期开始，不报错）
- **教训**：之前在主工作区测试时因复用了缓存的 daily_pv_all.h5 而没发现此问题，worktree 全新环境才暴露

### 问题 2：Embedding 路由错误（已修复 .env）

- **现象**：CoSTEER 知识库 embedding 调用报 `code: 1211, 模型不存在`，重试 10 次失败
- **根因**：`.env` 的 `EMBEDDING_MODEL=openai/doubao-embedding-vision`，`openai/` 前缀让 litellm 用 `OPENAI_API_BASE`（智谱 bigmodel.cn），但 embedding 模型在火山方舟，智谱没有此模型
- **修复**：改为 `litellm_proxy/doubao-embedding-vision` + 添加 `LITELLM_PROXY_API_KEY/BASE`（指向火山方舟）
- **上游代码无 bug**：上游假设 chat 和 embedding 同一服务商；我们多服务商配置（chat=智谱 / embedding=火山方舟）与上游假设不匹配
- **0.8 大写 RD-Agent 经验一致**：它也用 `litellm_proxy/` 前缀

### 问题 3：model_coder/conf.py extra_volumes 覆盖 bug（已修复 commit 889d26f1）

- **现象**：ModelRDLoop 实例化时报 `StopIteration`
- **根因**：`get_model_env()` 第 30 行 `env.conf.extra_volumes = extra_volumes.copy()` 无条件用空 `{}` 覆盖了 QlibDockerConf 默认挂载
- **修复**：改为 `if extra_volumes:` 才覆盖

---

## 场景验证记录

### 场景 1：factor（✅ 已完成）

| 项 | 结果 |
|---|---|
| worktree | rdagent-factor（已清理） |
| 命令 | `fin_factor --step-n 1 --no-checkout` |
| LLM 生成假设 | ✅ mom_10d / rev_5d / vol_20d |
| Embedding | ✅ 修复后正常 |
| CoSTEER 迭代 | ✅ 第一版 FAIL（raw close）→ 修正为 adj_close → SUCCESS |
| 产物 | 6 workspace + 3 factor.py + 2 conf_baseline.yaml |
| 执行步数 | 2/5（propose + coding，因 --step-n 1 停止） |
| 耗时 | ~4.5 分钟 |

### 场景 2：model（✅ 已完成）

| 项 | 结果 |
|---|---|
| worktree | rdagent-model |
| 命令 | `fin_model --step-n 1 --no-checkout` |
| LLM 生成模型 | ✅ GRUTimeSeriesBaseline（2 层 GRU + BatchNorm + Feedforward 64→32→1） |
| Embedding | ✅ litellm_proxy/doubao-embedding-vision 无报错 |
| 模型评审 | ✅ SUCCESS（执行成功，输出 shape (8,1) 正确） |
| 产物 | 3 workspace + model.py（GRU 模型代码） |
| 执行步数 | 2/5（propose + coding，因 --step-n 1 停止） |
| 耗时 | ~33 秒（比 factor 快，model 走 Docker GPU 直接验证） |

### 场景 3：quant（✅ 已完成）

| 项 | 结果 |
|---|---|
| worktree | rdagent-quant |
| 命令 | `fin_quant --step-n 1 --no-checkout` |
| LLM 生成因子 | ✅ 4 个：MOM_20（20日动量）/ REV_5（5日反转）/ VOL_STD_20（20日波动率）/ VOLUME_RATIO_5（量比） |
| Embedding | ✅ litellm_proxy/doubao-embedding-vision 全程无报错 |
| CoSTEER 评审 | ✅ 4 个因子全部 SUCCESS（第一次就通过，无需迭代修正） |
| 产物 | 4 workspace + 4 factor.py |
| 执行步数 | 2/5（propose + coding，因 --step-n 1 停止） |
| 耗时 | ~5 分 26 秒 |

### 场景 4：factor_from_report（✅ 已完成）

| 项 | 结果 |
|---|---|
| worktree | rdagent-frr |
| 命令 | `fin_factor_report --report-folder test_reports --no-checkout` |
| 第一次（147K PDF） | ⚠️ 提取 0 因子（688331 研报是公司分析报告，不含因子公式） |
| 第二次（1.6M PDF） | ✅ 从研报提取因子并完整执行 Loop |
| Embedding | ✅ 无报错 |
| LLM 因子提取 | ✅ 从 1.6M PDF 提取出因子 |
| 完整 Loop | ✅ **5/5 全步完成**（propose → coding → running → feedback → record） |
| 产物 | 9 workspace + 4 factor.py + 回测配置（conf_baseline / conf_combined） |
| 耗时 | 18 分 49 秒（含 PDF 解析 + 因子提取 + 回测 + LLM feedback） |
| 已知问题 | mlflow filesystem backend warning（不阻塞，只是 mlflow 记录降级） |
| 注意 | PDF 选择很重要——公司分析报告不含因子公式会提取 0 个；需用量化研报 |
