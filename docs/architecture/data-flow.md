# 数据流与执行架构

> 本文档梳理 multialphaV 平台（RD-Agent Qlib 场景）的完整数据流——从 qlib 原始数据到因子产出，覆盖 Docker 执行、generate.py 数据预处理、CoSTEER 因子迭代、qrun 回测等关键链路。

---

## 1. 整体架构图

```
用户执行 rdagent fin_factor/fin_model/fin_quant
    │
    ▼
RDLoop（主循环，5 步迭代）
    ├── 1. propose     LLM 生成因子/模型假设
    ├── 2. coding      CoSTEER 写代码 + 评审迭代
    │   ├── 读 daily_pv.h5（源数据）
    │   ├── 执行 factor.py/model.py（Docker 或 conda）
    │   └── 评审 → 修正 → 再执行（最多 10 轮演化）
    ├── 3. running     qrun 回测（Docker 内执行）
    ├── 4. feedback    LLM 总结回测结果
    └── 5. record      持久化 trace
```

## 2. 数据预处理：generate.py

### 触发时机
- Loop 初始化时（QlibFactorScenario.__init__）
- 仅当 daily_pv_all.h5 不存在时才执行（缓存复用）

### 执行环境
- **Docker 容器内**（QTDockerEnv），因为 multialphav env 不装 qlib
- 容器通过 extra_volumes 挂载 `~/.qlib → /root/.qlib/`（rw）

### 流程

```python
# generate.py 在 Docker 容器内执行
qlib.init(provider_uri="~/.qlib/qlib_data/cn_data")
instruments = D.instruments()           # csi300 股票池（681 只）
fields = ["$open","$close","$high","$low","$volume","$factor"]

# 全量数据（约 760 万行，213MB）
data = D.features(instruments, fields, freq="day")
       .swaplevel().sort_index()
       .loc["2008-12-29":]              # pandas 自动从最近可用日期开始
data.to_hdf("./daily_pv_all.h5", key="data")

# 调试样本（100 只股票 × 2 年，约 48 万行）
data = D.features(instruments, fields,
                  start_time="2024-01-01", end_time="2025-12-31")
       .loc[data.reset_index()["instrument"].unique()[:100]]
data.to_hdf("./daily_pv_debug.h5", key="data")
```

### 产出文件流向

```
generate.py 产出
├── daily_pv_all.h5   → git_ignore_folder/factor_implementation_source_data/daily_pv.h5
└── daily_pv_debug.h5 → git_ignore_folder/factor_implementation_source_data_debug/daily_pv.h5
                         ↓ FactorFBWorkspace.execute() 软链到因子工作区
                         <workspace>/daily_pv.h5
                         ↓ 因子代码读取
                         result.h5（因子值输出）
```

### 调试样本设计

| 维度 | 值 | 是否可配置 |
|---|---|---|
| 股票数 | 前 100 只（`unique()[:100]`） | ❌ 硬编码 |
| 时间段 | 2024-01-01 ~ 2025-12-31 | ❌ 硬编码（我们改自 2018-2019） |
| 股票池 | csi300（yaml 锚点） | ❌ 硬编码 |

调试样本用于 CoSTEER 快速迭代（48 万行 vs 全量 760 万行），验证因子代码正确性。

## 3. Docker 执行架构

### 镜像
- `local_qlib:v2.0`（10.7GB，qlib 0.9.7，8 卡 H20 GPU）
- Dockerfile 从 0.8 项目同步（阿里云镜像源 + requirements.lock + 本地 qlib-src.tar.gz）

### 数据挂载（官方默认）
```
宿主 ~/.qlib  →  容器 /root/.qlib/  (rw)
```
- `~/.qlib/qlib_data/cn_data` 是真实目录（从 /home/zxh/qlib_data 复制的三个核心子目录）
- 容器内 provider_uri `~/.qlib/qlib_data/cn_data` → 展开为 `/root/.qlib/qlib_data/cn_data` → 直接读挂载的真实数据

### 历史教训：三链死循环（已解决）

之前用软链时曾出现 Docker 内符号链接死循环：
```
链路1（宿主软链）: ~/.qlib/qlib_data/cn_data → /home/zxh/qlib_data
链路2（Dockerfile）: /home/zxh/qlib_data → /root/.qlib/qlib_data/cn_data
链路3（extra_volumes）: ~/.qlib → /root/.qlib/
→ 容器内解析形成环 → RuntimeError: Symlink loop
```

**解决方案**：数据直接复制到 `~/.qlib/qlib_data/cn_data`（真实目录非软链），回退所有 extra_volumes 修改到官方默认。详见 [VERIFICATION-log.md](../guide/VERIFICATION-log.md)。

## 4. LLM 调用架构

### 双服务商配置

| 功能 | 服务商 | 模型 | 环境变量前缀 |
|---|---|---|---|
| Chat（因子/模型假设生成） | 智谱 AI | `openai/glm-5.2` | `OPENAI_API_BASE`（bigmodel.cn） |
| Embedding（知识库向量检索） | 火山方舟 | `litellm_proxy/doubao-embedding-vision` | `LITELLM_PROXY_API_BASE`（volces.com） |

### litellm 前缀路由机制

litellm 根据模型名前缀决定用哪个 `api_key`/`api_base`：
- `openai/xxx` → `OPENAI_API_KEY` + `OPENAI_API_BASE`
- `litellm_proxy/xxx` → `LITELLM_PROXY_API_KEY` + `LITELLM_PROXY_API_BASE`

### 历史教训：Embedding 路由错误（已解决）

之前 `EMBEDDING_MODEL=openai/doubao-embedding-vision` 导致 litellm 用智谱 base 调 embedding（智谱没有 doubao → "模型不存在"）。修复为 `litellm_proxy/` 前缀。

## 5. CoSTEER 因子迭代机制

### 演化流程
```
LLM 生成因子假设（如 mom_10d = Close_t / Close_{t-10} - 1）
    ↓
CoSTEER 写 factor.py（初版代码）
    ↓
FactorFBWorkspace.execute()
    ├── 读 daily_pv.h5
    ├── 执行 factor.py
    └── 检查 result.h5 格式
    ↓
LLM 评审（代码 / 形状 / 值 / 执行反馈）
    ├── SUCCESS → 保留
    └── FAIL → 生成修正版 → 再执行（最多 10 轮）
```

### 真实示例（factor 场景）

第一版（FAIL）：
```python
df['mom_10d'] = df.groupby('instrument')['$close'].pct_change(periods=10)
# 评审：应使用复权价 adj_close = $close * $factor，而非 raw $close
```

第二版（SUCCESS）：
```python
df['adj_close'] = df['$close'] * df['$factor']
df['mom_10d'] = df.groupby('instrument')['adj_close'].pct_change(periods=10)
```
