# multialphaV 环境配置说明（.env Reference）

> 本文档说明 multialphaV 平台（基于 RD-Agent + Qlib 场景二次开发）**通过环境变量 / `.env` 文件配置的全部可调点**，回答"哪些行为是由哪条配置决定的"。
>
> - 配置载体：项目根目录 `.env`（`.gitignore` 已忽略；密钥只放本地）
> - 配置机制：Pydantic v2 `BaseSettings` + 自定义 `ExtendedBaseSettings`（`rdagent/core/conf.py`），支持父类 env 前缀继承
> - 加载入口：`rdagent/app/cli.py:13` → `load_dotenv(".env")`，在所有 Settings 单例构造前执行
> - 适用目录：`/home/zxh/projects/1.multialphaV/RD-Agent`
> - 维护约定：任何配置键 / 默认值 / 行为变更，必须同步本文档（CLAUDE.md §9 改后审查清单 B）

---

## 目录

- [0. 快速导航：我需要改 X，该动哪个配置？](#0-快速导航我需要改-x该动哪个配置)
- [1. 配置机制基础](#1-配置机制基础)
- [2. 当前 `.env` 实际配置逐条解读](#2-当前-env-实际配置逐条解读)
- [3. 全部配置键总览（按 Settings 类分组）](#3-全部配置键总览按-settings-类分组)
  - [3.1 全局运行配置（RDAgentSettings，无前缀）](#31-全局运行配置rdagentsettings无前缀)
  - [3.2 LLM 后端配置（LLMSettings，无前缀）](#32-llm-后端配置llmsettings无前缀)
  - [3.3 Qlib R&D 循环配置](#33-qlib-rd-循环配置)
  - [3.4 CoSTEER 子系统配置](#34-costeer-子系统配置)
  - [3.5 Docker / 执行环境配置](#35-docker--执行环境配置)
  - [3.6 日志配置（LOG_ 前缀）](#36-日志配置log_-前缀)
  - [3.7 Web UI 配置（UI_ 前缀）](#37-web-ui-配置ui_-前缀)
  - [3.8 context7 / RAG Agent 配置](#38-context7--rag-agent-配置)
  - [3.9 Benchmark 配置（BENCHMARK_ 前缀）](#39-benchmark-配置benchmark_-前缀)
- [4. 非 Settings 的环境变量（直接 os.environ 读取）](#4-非-settings-的环境变量直接-osenviron-读取)
- [5. 已知的"死配置"（在 .env 但代码不消费）](#5-已知的死配置在-env-但代码不消费)
- [6. 配置加载顺序与优先级](#6-配置加载顺序与优先级)
- [7. 推荐的最小 `.env` 模板](#7-推荐的最小-env-模板)

---

## 0. 快速导航：我需要改 X，该动哪个配置？

| 我要改的行为 | 对应配置 | 章节 |
|---|---|---|
| 换 LLM 模型（如 GLM→GPT-4o） | `CHAT_MODEL` | [3.2](#32-llm-后端配置llmsettings无前缀) |
| 换 LLM API 提供商 / Key | `OPENAI_API_KEY` / `OPENAI_API_BASE` 或 `CHAT_OPENAI_*` | [3.2](#32-llm-后端配置llmsettings无前缀) |
| 换 Embedding 模型 / Key | `EMBEDDING_MODEL` / `EMBEDDING_OPENAI_*` | [3.2](#32-llm-后端配置llmsettings无前缀) |
| 换 LLM 后端实现类 | `BACKEND` | [3.2](#32-llm-后端配置llmsettings无前缀) |
| 调整 LLM 温度 / 最大 token | `CHAT_TEMPERATURE` / `CHAT_MAX_TOKENS` | [3.2](#32-llm-后端配置llmsettings无前缀) |
| 切换 Qlib 训练/验证/测试日期区间 | `QLIB_FACTOR_TRAIN_START` 等（按场景前缀） | [3.3](#33-qlib-rd-循环配置) |
| 换因子/模型/全流水线的某个组件类 | `QLIB_FACTOR_CODER` 等 | [3.3](#33-qlib-rd-循环配置) |
| 调 CoSTEER 演化轮数 | `QLIB_FACTOR_EVOLVING_N`（循环级）或 `CoSTEER_MAX_LOOP`（子系统级） | [3.3](#33-qlib-rd-循环配置) / [3.4](#34-costeer-子系统配置) |
| Model 训练用 Docker 还是 Conda | `MODEL_CoSTEER_ENV_TYPE` | [3.4](#34-costeer-子系统配置) |
| 换 Qlib Docker 镜像 | `QLIB_DOCKER_IMAGE` | [3.5](#35-docker--执行环境配置) |
| 跳过 Docker 镜像构建（用现有镜像） | `QLIB_DOCKER_BUILD_FROM_DOCKERFILE=False` | [3.5](#35-docker--执行环境配置) |
| 改日志/trace 落盘路径 | `LOG_TRACE_PATH` | [3.6](#36-日志配置log_-前缀) |
| 改 UI trace 历史 / 静态资源目录 | `UI_TRACE_FOLDER` / `UI_STATIC_PATH` | [3.7](#37-web-ui-配置ui_-前缀) |
| 控制 context7 文档检索地址 | `CONTEXT7_URL` | [3.8](#38-context7--rag-agent-配置) |
| 改并行进程数 / 工作区路径 | `MULTI_PROC_N` / `WORKSPACE_PATH` | [3.1](#31-全局运行配置rdagentsettings无前缀) |
| 控制 GPU 设备 | `CUDA_VISIBLE_DEVICES` | [4](#4-非-settings-的环境变量直接-osenviron-读取) |

---

## 1. 配置机制基础

### 1.1 载体

- **`.env` 文件**（项目根目录）：人类编辑的主载体，键值对（`KEY=value`），支持引号字符串
- **同名 shell 环境变量**：优先级高于 `.env`（见 [§6](#6-配置加载顺序与优先级)）
- **密钥策略**：`.env` 在 `.gitignore` 中，**禁止提交**；真实 Key 只放本地

### 1.2 加载入口

```python
# rdagent/app/cli.py:11-13
from dotenv import load_dotenv
load_dotenv(".env")   # 必须在所有 BaseSettings 单例构造前执行
```

- 只有通过 `rdagent ...` CLI / `python -m rdagent.app.cli` 启动才会自动加载 `.env`
- 直接 `import rdagent.xxx` 不会加载 `.env`（除非显式 `load_dotenv()`），此时只读 shell 环境变量
- 即使不显式 `load_dotenv`，Pydantic `BaseSettings` 也会自动读取 CWD 下 `.env`，但显式调用保证加载时机

### 1.3 Settings 类约定

- 每个 Settings 类是一个 Pydantic 模型，字段名 → 环境变量名（加前缀）
- 每个 Settings 类有一个**模块级单例**（如 `RD_AGENT_SETTINGS`、`LLM_SETTINGS`），全局共享
- 自定义基类 `ExtendedBaseSettings`（`rdagent/core/conf.py`）改写 `settings_customise_sources`，使**父类的 env 前缀也被继承扫描**
- 前缀匹配默认**大小写不敏感**（如 `CoSTEER_` 和 `COSTEER_` 都生效）

---

## 2. 当前 `.env` 实际配置逐条解读

> 源文件：`/home/zxh/projects/1.multialphaV/RD-Agent/.env`（继承自 0.8 项目，按 CLAUDE.md §6.3 增量调整）

```dotenv
BACKEND=rdagent.oai.backend.LiteLLMAPIBackend
```
- **作用**：LLM 后端实现类的完整 import 路径。
- **归属**：`LLMSettings.backend`（[§3.2](#32-llm-后端配置llmsettings无前缀)）
- **可替换**：`rdagent.oai.backend.DeprecBackend`（旧版），或自定义后端

```dotenv
CHAT_OPENAI_BASE_URL=https://ark.cn-beijing.volces.com/api/coding/v3
OPENAI_API_BASE=https://ark.cn-beijing.volces.com/api/coding/v3
CHAT_MODEL=openai/glm-5.2
CHAT_TEMPERATURE=0.6
CHAT_MAX_TOKENS=16384
CHAT_OPENAI_API_KEY=<方舟 Coding Plan key>
OPENAI_API_KEY=<方舟 Coding Plan key>

# per-step 路由(官方 chat_model_map 机制,命中 logger._tag 子串 Loop_{li}.{step})
# temperature 依据厂商官方推荐(reasoning/agentic 模型高温训练评测):
#   minimax-m3=0.7(M3 thinking)/kimi-k2.7-code=1.0(官方禁低温)/deepseek-v4-flash=0.0(编码)/glm-5.2=0.6
CHAT_MODEL_MAP={"direct_exp_gen":{"model":"openai/minimax-m3","temperature":"0.7"},"coding":{"model":"openai/kimi-k2.7-code","temperature":"1.0"},"running":{"model":"openai/deepseek-v4-flash","temperature":"0.0"},"feedback":{"model":"openai/glm-5.2","temperature":"0.6"}}
```
- **作用**：把 chat LLM 切到**火山方舟 Coding Plan**（统一端点，多 model 聚合）。`openai/` 前缀是 LiteLLM 的 OpenAI 兼容协议标记。
- **归属**：均属 `LLMSettings`（[§3.2](#32-llm-后端配置llmsettings无前缀)）。
  - `CHAT_OPENAI_BASE_URL` / `CHAT_OPENAI_API_KEY`：**仅用于 chat** 的专用覆盖（优先级高于通用 `OPENAI_*`）
  - `OPENAI_API_BASE` / `OPENAI_API_KEY`：通用 OpenAI 兼容端点（chat 和 embedding 的回退）；本项目两者同指方舟
  - `CHAT_MODEL`：全局 chat 模型名（未命中 `CHAT_MODEL_MAP` 的调用走这里）
  - `CHAT_TEMPERATURE`：全局 fallback 采样温度（0.6，对齐 GLM-5.2 thinking 推荐）
  - `CHAT_MAX_TOKENS`：单次响应最大 token
  - `CHAT_MODEL_MAP`：per-step 模型路由 + 各 model 厂商推荐 temperature，详见 [§3.2](#32-llm-后端配置llmsettings无前缀)

```dotenv
EMBEDDING_MODEL=openai/doubao-embedding-vision
# 复用 OPENAI_API_KEY/BASE（与 chat 同指方舟，在 LLM 段已配）
```
- **作用**：embedding 走**火山方舟豆包**，与 chat 同 provider、同凭证。chat 切方舟后用 `openai/` 前缀即可直接复用 `OPENAI_API_KEY/BASE`，不再需要 `litellm_proxy/` 绕路。
- **归属**：`LLMSettings.embedding_*`（[§3.2](#32-llm-后端配置llmsettings无前缀)）。litellm 后端只认 model 前缀路由：`openai/` → `OPENAI_API_KEY/BASE`。
- **已删字段**：`EMBEDDING_OPENAI_API_KEY/BASE_URL`（litellm 后端不消费，仅废弃 `deprec.py` 用，[§3.2](#32-llm-后端配置llmsettings无前缀) 标注为死字段）；`LITELLM_PROXY_API_KEY/BASE`（`litellm_proxy/` 前缀专用，不再需要）。

```dotenv
# 注：market 值（csi300）写死在 scenarios/qlib/experiment/{factor,model}_template/conf_*.yaml 的 YAML 锚点，
# 不由环境变量控制；RDAGENT_MARKET 已删除（代码零引用，详见 ENV.md §5.1）。
```
- **原 `RDAGENT_MARKET=csi300` 已删除（2026-07-19）**：代码零引用，实际 market 写死在模板 YAML 锚点。详见 [§5.1](#51-rdagent_market完全死-已删除)。

```dotenv
# 注意：当前代码 get_factor_env() 实际改读 CONDA_DEFAULT_ENV，本字段未被消费（详见 ENV.md §5.2）；
#       保留此行作为意图标注，未来若修复 get_factor_env() 让 python_bin 生效即可直接生效。
FACTOR_CoSTEER_PYTHON_BIN=/home/zxh/miniconda3/envs/rdagent/bin/python
```
- **保留但加注释**：字段存在于 `FactorCoSTEERSettings.python_bin`，能被加载，但 `get_factor_env()` 读到后**不使用**——实际子进程解释器由 `CONDA_DEFAULT_ENV` 决定。CLAUDE.md §6.1/§6.2 记录的设计意图（指向 0.8 共享 env）需要保留，详见 [§5.2](#52-factor_costeer_python_bin加载但不使用-保留)。

```dotenv
MODEL_CoSTEER_ENV_TYPE=docker
```
- **作用**：Model CoSTEER 训练执行环境类型，`docker` 或 `conda`。
- **归属**：`ModelCoSTEERSettings.env_type`（[§3.4](#34-costeer-子系统配置)）
- **本项目选 docker**：因为主 env `multialphav` 不装 qlib/torch，训练在 `local_qlib:v2.0` 镜像里跑。

```dotenv
QLIB_DOCKER_IMAGE=local_qlib:v2.0
QLIB_DOCKER_BUILD_FROM_DOCKERFILE=False
```
- **作用**：Qlib Docker 容器配置。
  - `QLIB_DOCKER_IMAGE`：镜像名（系统级已构建，10.7GB，qlib 0.9.7）
  - `QLIB_DOCKER_BUILD_FROM_DOCKERFILE=False`：跳过从 Dockerfile 重建，直接用现有镜像
- **归属**：`QlibDockerConf`（[§3.5](#35-docker--执行环境配置)）

```dotenv
# CONTEXT7_URL=http://localhost:8124/mcp
# CONTEXT7_TIMEOUT=120
# CONTEXT7_ENABLE_CACHE=true
```
- **作用**：context7 文档检索 Agent 配置，全注释（用默认值）。
- **归属**：`rdagent/components/agent/context7/conf.py:Settings`（[§3.8](#38-context7--rag-agent-配置)）

---

## 3. 全部配置键总览（按 Settings 类分组）

### 3.1 全局运行配置（RDAgentSettings，无前缀）

单例 `RD_AGENT_SETTINGS`。文件：`rdagent/core/conf.py`。**变量名直接生效，无前缀**。

| 字段 | 类型 / 默认 | 决定什么 |
|---|---|---|
| `WORKSPACE_PATH` | `Path = ./git_ignore_folder/RD-Agent_workspace` | 实验工作区根目录（FBWorkspace 自动建在此） |
| `WORKSPACE_CKP_SIZE_LIMIT` | `int = 0` | 工作区检查点大小上限（0 = 不限） |
| `WORKSPACE_CKP_WHITE_LIST_NAMES` | `list[str]? = None` | 检查点保留白名单文件名 |
| `MULTI_PROC_N` | `int = 1` | 并行进程数（影响 `get_max_parallel()`） |
| `CACHE_WITH_PICKLE` | `bool = True` | pickle 缓存总开关（影响 Runner 重复实验跳过） |
| `PICKLE_CACHE_FOLDER_PATH_STR` | `str = ./pickle_cache/` | pickle 缓存目录 |
| `USE_FILE_LOCK` | `bool = True` | 文件锁（并发安全） |
| `STDOUT_CONTEXT_LEN` | `int = 400` | 子进程 stdout 上下文保留长度 |
| `STDOUT_LINE_LEN` | `int = 10000` | stdout 单行截断长度 |
| `ENABLE_MLFLOW` | `bool = False` | MLflow 实验跟踪 |
| `STEP_SEMAPHORE` | `int \| dict[str,int] = 1` | 步级并发信号量（限流） |
| `SUBPROC_STEP` | `bool = False` | 单步用子进程执行（`is_force_subproc()`） |
| `APP_TPL` | `str? = None` | app 模板根路径 |
| `MAX_INPUT_DUPLICATE_FACTOR_GROUP` | `int = 300` | 输入因子去重分组上限 |
| `MAX_OUTPUT_DUPLICATE_FACTOR_GROUP` | `int = 20` | 输出因子去重分组上限 |
| `MAX_KMEANS_GROUP_NUMBER` | `int = 40` | KMeans 聚类簇数上限 |
| `INITIAL_FATOR_LIBRARY_SIZE` | `int = 20` | 初始因子库大小（注：上游拼写为 `fator`） |
| `AZURE_DOCUMENT_INTELLIGENCE_KEY` | `str = ""` | Azure Document Intelligence key（PDF 解析用） |
| `AZURE_DOCUMENT_INTELLIGENCE_ENDPOINT` | `str = ""` | Azure DI endpoint |

### 3.2 LLM 后端配置（LLMSettings，无前缀）

单例 `LLM_SETTINGS`（`rdagent/oai/llm_conf.py`），子类 `LiteLLMSettings` 加 `LITELLM_` 前缀（单例 `LITELLM_SETTINGS`，即每个字段都可加 `LITELLM_` 前缀，如 `LITELLM_CHAT_MODEL`）。

#### 后端与模型选择

| 字段 | 默认 | 决定什么 |
|---|---|---|
| `BACKEND` | `rdagent.oai.backend.LiteLLMAPIBackend` | LLM 后端类路径（工厂 `get_api_backend()` 据此导入） |
| `CHAT_MODEL` | `gpt-4-turbo` | chat 模型名 |
| `EMBEDDING_MODEL` | `text-embedding-3-small` | embedding 模型名 |
| `REASONING_EFFORT` | `None` | 推理强度枚举 `low`/`medium`/`high` |
| `ENABLE_RESPONSE_SCHEMA` | `True` | 是否启用 JSON Schema 结构化响应 |
| `REASONING_THINK_RM` | `False` | 移除 reasoning think 段 |
| `LOG_LLM_CHAT_CONTENT` | `True` | 是否把 LLM 对话写入 trace |
| `CHAT_MODEL_MAP` | `{}` | per-step 模型路由 `{tag子串: {model, temperature?, max_tokens?, reasoning_effort?}}`；命中 `logger._tag`（格式 `Loop_{li}.{step}`）子串即覆盖 `CHAT_MODEL`。**限制**：①所有 model 必须同 provider（后端不传 per-call api_key/base）；②value 必须全字符串（类型 `dict[str, dict[str, str]]`，后端运行时 cast）；③JSON 单行；④未命中走 `CHAT_MODEL`。 |

#### 鉴权与端点（chat）

| 字段 | 默认 | 决定什么 |
|---|---|---|
| `OPENAI_API_KEY` | `""` | 通用 OpenAI key（chat+embedding 回退） |
| `OPENAI_API_BASE` | `""` | 通用 OpenAI base url |
| `CHAT_OPENAI_API_KEY` | `None` | chat 专用 key（**优先于** `OPENAI_API_KEY`） |
| `CHAT_OPENAI_BASE_URL` | `None` | chat 专用 base url（**优先于** `OPENAI_API_BASE`） |
| `CHAT_AZURE_API_BASE` | `""` | Azure OpenAI chat 端点 |
| `CHAT_AZURE_API_VERSION` | `""` | Azure API 版本 |
| `CHAT_USE_AZURE` | `False` | chat 走 Azure |
| `CHAT_USE_AZURE_TOKEN_PROVIDER` | `False` | chat 用 Azure 托管标识 |
| `MANAGED_IDENTITY_CLIENT_ID` | `None` | 托管标识 client id |
| `CHAT_USE_AZURE_DEEPSEEK` | `False` | chat 走 Azure DeepSeek |
| `CHAT_AZURE_DEEPSEEK_ENDPOINT` | `""` | Azure DeepSeek 端点 |
| `CHAT_AZURE_DEEPSEEK_KEY` | `""` | Azure DeepSeek key |
| `USE_AZURE` | `False` | **已弃用**（旧统一开关） |

#### 鉴权与端点（embedding）

| 字段 | 默认 | 决定什么 |
|---|---|---|
| `EMBEDDING_OPENAI_API_KEY` | `""` | embedding 专用 key（⚠️ **死字段**：`LiteLLMAPIBackend` 不消费，仅废弃 `deprec.py` 用；litellm 后端靠 model 前缀路由，`openai/` 读 `OPENAI_API_KEY`） |
| `EMBEDDING_OPENAI_BASE_URL` | `""` | embedding 专用 base url（⚠️ **死字段**，同上） |
| `EMBEDDING_AZURE_API_BASE` | `""` | Azure embedding 端点 |
| `EMBEDDING_AZURE_API_VERSION` | `""` | Azure embedding 版本 |
| `EMBEDDING_USE_AZURE` | `False` | embedding 走 Azure |
| `EMBEDDING_USE_AZURE_TOKEN_PROVIDER` | `False` | embedding 用托管标识 |
| `EMBEDDING_MAX_STR_NUM` | `50` | 单批 embedding 最大字符串数 |
| `EMBEDDING_MAX_LENGTH` | `8192` | embedding 单条最大长度 |

#### 调用参数

| 字段 | 默认 |
|---|---|
| `CHAT_MAX_TOKENS` | `None` |
| `CHAT_TEMPERATURE` | `0.5` |
| `CHAT_STREAM` | `True` |
| `CHAT_SEED` | `None` |
| `CHAT_FREQUENCY_PENALTY` | `0.0` |
| `CHAT_PRESENCE_PENALTY` | `0.0` |
| `CHAT_TOKEN_LIMIT` | `100000`（单次对话总 token 上限） |
| `MAX_PAST_MESSAGE_INCLUDE` | `10`（多轮历史携带条数） |
| `DEFAULT_SYSTEM_PROMPT` | `"You are an AI assistant who helps to answer user's questions."` |
| `SYSTEM_PROMPT_ROLE` | `"system"` |

#### 重试与缓存

| 字段 | 默认 | 决定什么 |
|---|---|---|
| `MAX_RETRY` | `10` | LLM 调用最大重试次数 |
| `RETRY_WAIT_SECONDS` | `1` | 重试间隔 |
| `TIMEOUT_FAIL_LIMIT` | `10` | 超时累计失败阈值 |
| `VIOLATION_FAIL_LIMIT` | `1` | 协议违规失败阈值 |
| `USE_CHAT_CACHE` | `False` | 读 chat 缓存 |
| `DUMP_CHAT_CACHE` | `False` | 写 chat 缓存 |
| `USE_EMBEDDING_CACHE` | `False` | 读 embedding 缓存 |
| `DUMP_EMBEDDING_CACHE` | `False` | 写 embedding 缓存 |
| `PROMPT_CACHE_PATH` | `./prompt_cache.db` | SQLite 缓存文件路径 |
| `USE_AUTO_CHAT_CACHE_SEED_GEN` | `False` | 自动生成缓存 seed |
| `INIT_CHAT_CACHE_SEED` | `42` | 初始缓存 seed |

#### 高级 / 旧版（一般不动）

- `USE_LLAMA2`、`LLAMA2_CKPT_DIR`、`LLAMA2_TOKENIZER_PATH`、`LLAMS2_MAX_BATCH_SIZE`（本地 llama2 推理）
- `USE_GCR_ENDPOINT`、`GCR_ENDPOINT_TYPE`（默认 `llama2_70b`）、`GCR_ENDPOINT_TEMPERATURE=0.7`、`GCR_ENDPOINT_TOP_P=0.9`、`GCR_ENDPOINT_DO_SAMPLE=False`、`GCR_ENDPOINT_MAX_TOKEN=100`
- GCR 模型端点系列：`LLAMA2_70B_ENDPOINT/KEY/DEPLOYMENT`、`LLAMA3_70B_*`、`PHI2_*`、`PHI3_4K_*`、`PHI3_128K_*`

> **LITELLM_ 前缀**：以上所有字段都可前缀化（如 `LITELLM_CHAT_MODEL`、`LITELLM_OPENAI_API_KEY`）。常用于在 LiteLLM 后端下用一套独立于裸 `LLM_SETTINGS` 的配置。

### 3.3 Qlib R&D 循环配置

文件：`rdagent/app/qlib_rd_loop/conf.py`。四个单例，分别带前缀：

| 单例 | 类 | 前缀 | 用途 |
|---|---|---|---|
| `FACTOR_PROP_SETTING` | `FactorBasePropSetting` | `QLIB_FACTOR_` | `fin_factor` 循环 |
| `FACTOR_FROM_REPORT_PROP_SETTING` | `FactorFromReportPropSetting` | `QLIB_FACTOR_`（继承） | `fin_factor_report` 循环 |
| `MODEL_PROP_SETTING` | `ModelBasePropSetting` | `QLIB_MODEL_` | `fin_model` 循环 |
| `QUANT_PROP_SETTING` | `QuantBasePropSetting` | `QLIB_QUANT_` | `fin_quant` 循环 |

#### 组件装配类路径（每个场景都有，按前缀替换）

下面用 `<X>` 代表 `QLIB_FACTOR_` / `QLIB_MODEL_` / `QLIB_QUANT_`：

| 字段 | 决定什么 | 示例（Factor） |
|---|---|---|
| `<X>SCEN` | Scenario 类（决定背景/输出格式/运行时环境） | `...QlibFactorScenario` |
| `<X>HYPOTHESIS_GEN` | 假设生成器类 | `...QlibFactorHypothesisGen` |
| `<X>HYPOTHESIS2EXPERIMENT` | 假设→实验转换器类 | `...QlibFactorHypothesis2Experiment` |
| `<X>CODER` | CoSTEER 编码器类 | `...QlibFactorCoSTEER` |
| `<X>RUNNER` | 实验 Runner 类 | `...QlibFactorRunner` |
| `<X>SUMMARIZER` | 反馈生成器类 | `...QlibFactorExperiment2Feedback` |
| `<X>EVOLVING_N` | CoSTEER 演化轮数（默认 `10`） | — |

> `BasePropSetting` 基类还有 `KNOWLEDGE_BASE` / `KNOWLEDGE_BASE_PATH` / `INTERACTOR` 字段（默认 None），用于注入知识库与交互器。

#### 训练/验证/测试日期区间（六字段，四个场景一致）

| 字段 | 默认 | 决定什么 |
|---|---|---|
| `<X>TRAIN_START` | `2008-01-01` | 训练集开始 |
| `<X>TRAIN_END` | `2014-12-31` | 训练集结束 |
| `<X>VALID_START` | `2015-01-01` | 验证集开始 |
| `<X>VALID_END` | `2016-12-31` | 验证集结束 |
| `<X>TEST_START` | `2017-01-01` | 测试集开始 |
| `<X>TEST_END` | `2020-08-01` | 测试集结束 |

例：`QLIB_FACTOR_TEST_END=2026-07-17`、`QLIB_QUANT_VALID_START=2020-01-01`。这些值最终注入 Qlib 模板的 `conf_*.yaml`，决定回测时间窗。

#### Quant 独有字段

| 字段 | 默认 | 决定什么 |
|---|---|---|
| `QLIB_QUANT_ACTION_SELECTION` | `bandit` | 联合循环的动作选择策略，枚举 `bandit` / `llm` / `random` |
| `QLIB_QUANT_QUANT_HYPOTHESIS_GEN` | `...FactorAndModelHypothesisGen` | 联合假设生成器 |
| `QLIB_QUANT_FACTOR_HYPOTHESIS2EXPERIMENT` | — | factor 子流水线 |
| `QLIB_QUANT_MODEL_HYPOTHESIS2EXPERIMENT` | — | model 子流水线 |
| `QLIB_QUANT_FACTOR_CODER` / `QLIB_QUANT_MODEL_CODER` | — | 子流水线编码器 |
| `QLIB_QUANT_FACTOR_RUNNER` / `QLIB_QUANT_MODEL_RUNNER` | — | 子流水线运行器 |
| `QLIB_QUANT_FACTOR_SUMMARIZER` / `QLIB_QUANT_MODEL_SUMMARIZER` | — | 子流水线反馈器 |

#### Factor-From-Report 独有字段（继承 `QLIB_FACTOR_` 前缀）

| 字段 | 默认 | 决定什么 |
|---|---|---|
| `QLIB_FACTOR_SCEN` | `...QlibFactorFromReportScenario` | 覆盖父类场景 |
| `QLIB_FACTOR_REPORT_RESULT_JSON_FILE_PATH` | `git_ignore_folder/report_list.json` | 研报清单 JSON |
| `QLIB_FACTOR_MAX_FACTORS_PER_EXP` | `6` | 单实验最多实现因子数 |
| `QLIB_FACTOR_REPORT_LIMIT` | `20` | 最多处理研报数 |

### 3.4 CoSTEER 子系统配置

#### CoSTEERSettings（前缀 `CoSTEER_`，单例 `CoSTEER_SETTINGS`）

文件：`rdagent/components/coder/CoSTEER/config.py`。大小写不敏感（`COSTEER_*` 也可）。

| 字段 | 默认 | 决定什么 |
|---|---|---|
| `CoSTEER_CODER_USE_CACHE` | `False` | CoSTEER 编码器缓存 |
| `CoSTEER_MAX_LOOP` | `10` | CoSTEER 单次演化最大循环 |
| `CoSTEER_FAIL_TASK_TRIAL_LIMIT` | `20` | 失败任务重试上限 |
| `CoSTEER_V1_QUERY_FORMER_TRACE_LIMIT` | `3` | v1 策略查历史 trace 条数 |
| `CoSTEER_V1_QUERY_SIMILAR_SUCCESS_LIMIT` | `3` | v1 查相似成功案例条数 |
| `CoSTEER_V2_QUERY_COMPONENT_LIMIT` | `1` | v2 策略查组件条数 |
| `CoSTEER_V2_QUERY_ERROR_LIMIT` | `1` | v2 查错误条数 |
| `CoSTEER_V2_QUERY_FORMER_TRACE_LIMIT` | `3` | v2 查历史 trace 条数 |
| `CoSTEER_V2_ADD_FAIL_ATTEMPT_TO_LATEST_SUCCESSFUL_EXECUTION` | `False` | v2 把失败尝试加入最近成功 |
| `CoSTEER_V2_ERROR_SUMMARY` | `False` | v2 错误摘要 |
| `CoSTEER_V2_KNOWLEDGE_SAMPLER` | `1.0` | v2 知识采样率 |
| `CoSTEER_KNOWLEDGE_BASE_PATH` | `None` | 已有知识库加载路径 |
| `CoSTEER_NEW_KNOWLEDGE_BASE_PATH` | `None` | 新知识库写入路径 |
| `CoSTEER_ENABLE_FILELOCK` | `False` | 知识库文件锁 |
| `CoSTEER_FILELOCK_PATH` | `None` | 文件锁路径 |
| `CoSTEER_MAX_SECONDS_MULTIPLIER` | `1000000` | 超时换算倍数 |

#### ModelCoSTEERSettings（前缀 `MODEL_CoSTEER_`，单例 `MODEL_COSTEER_SETTINGS`）

文件：`rdagent/components/coder/model_coder/conf.py`。继承 `CoSTEER_*` 全部字段。

| 字段 | 默认 | 决定什么 |
|---|---|---|
| `MODEL_CoSTEER_ENV_TYPE` | `conda` | Model 训练执行环境，`conda` 或 `docker`（本项目用 `docker`） |

> 切换为 `docker` 时，Model 训练在 `QlibDockerConf` 配置的镜像里跑；切换为 `conda` 时用 `QlibCondaEnv`。

#### FactorCoSTEERSettings（前缀 `FACTOR_CoSTEER_`，单例 `FACTOR_COSTEER_SETTINGS`）

文件：`rdagent/components/coder/factor_coder/config.py`。继承 `CoSTEER_*` 全部字段。

| 字段 | 默认 | 决定什么 |
|---|---|---|
| `FACTOR_CoSTEER_DATA_FOLDER` | `git_ignore_folder/factor_implementation_source_data` | 因子实现源数据目录 |
| `FACTOR_CoSTEER_DATA_FOLDER_DEBUG` | `..._debug` | 调试用源数据目录 |
| `FACTOR_CoSTEER_SIMPLE_BACKGROUND` | `False` | 简化背景 prompt |
| `FACTOR_CoSTEER_FILE_BASED_EXECUTION_TIMEOUT` | `3600` | 基于文件的执行超时（秒） |
| `FACTOR_CoSTEER_SELECT_METHOD` | `random` | 因子选择方法 |
| `FACTOR_CoSTEER_PYTHON_BIN` | `python` | ⚠️ 因子 CoSTEER 子进程解释器路径（**当前代码不使用，见 §5**） |

### 3.5 Docker / 执行环境配置

#### QlibDockerConf（前缀 `QLIB_DOCKER_`）

文件：`rdagent/utils/env.py`。Qlib 专用 Docker 容器配置，由 `QTDockerEnv` 使用。

| 字段 | 默认 | 决定什么 |
|---|---|---|
| `QLIB_DOCKER_BUILD_FROM_DOCKERFILE` | `True` | 是否从 Dockerfile 构建（本项目设 `False` 跳过） |
| `QLIB_DOCKER_IMAGE` | `local_qlib:latest` | 镜像名（本项目 `local_qlib:v2.0`） |
| `QLIB_DOCKER_MOUNT_PATH` | `/workspace/qlib_workspace/` | 容器内挂载点 |
| `QLIB_DOCKER_DEFAULT_ENTRY` | `qrun conf.yaml` | 容器入口命令 |
| `QLIB_DOCKER_SHM_SIZE` | `16g` | 共享内存大小 |
| `QLIB_DOCKER_ENABLE_CACHE` | `False` | Docker 运行缓存 |
| `QLIB_DOCKER_ENABLE_GPU` | `True` | GPU 设备映射 |
| `QLIB_DOCKER_MEM_LIMIT` | `48g` | 容器内存上限 |
| `QLIB_DOCKER_NETWORK` | `bridge` | 网络模式 |
| `QLIB_DOCKER_RUNNING_TIMEOUT_PERIOD` | `3600` | 单次运行超时（秒） |
| `QLIB_DOCKER_RETRY_COUNT` | `5` | 失败重试次数 |
| `QLIB_DOCKER_RETRY_WAIT_SECONDS` | `10` | 重试间隔 |
| `QLIB_DOCKER_SAVE_LOGS_TO_FILE` | `True` | 容器日志落盘 |

> Qlib 数据挂载是代码内置：宿主 `~/.qlib/` → 容器 `/root/.qlib/`（只读）。本项目通过软链 `~/.qlib/qlib_data/cn_data → /home/zxh/qlib_data` 让容器读到最新数据，零代码改动（CLAUDE.md §6.1）。

#### 通用 Env 系列（无前缀，一般不改）

`EnvConf` / `LocalConf` / `CondaConf` / `DockerConf` / `QlibCondaConf`（均无 env 前缀）。这些类的实例通常在代码里直接构造（如 `DockerConf(image=..., mount_path=..., default_entry=...)`），**不通过 .env 配置**。完整字段见 [API.md §3.6](file:///home/zxh/projects/1.multialphaV/API.md)。

唯一例外：`QlibCondaConf` 默认 `conda_env_name="rdagent4qlib"`，若要走 Conda 路径（`MODEL_CoSTEER_ENV_TYPE=conda`），需保证该 env 存在且装好 qlib。

### 3.6 日志配置（`LOG_` 前缀）

单例 `LOG_SETTINGS`（`rdagent/log/conf.py`）。

| 字段 | 默认 | 决定什么 |
|---|---|---|
| `LOG_TRACE_PATH` | `./log/<UTC timestamp>` | trace 落盘根目录（FileStorage 写入处） |
| `LOG_FORMAT_CONSOLE` | `None` | 自定义控制台格式（None 用默认） |
| `LOG_UI_SERVER_PORT` | `None` | 设端口则启用 `WebStorage`，把消息推送到该端口的 Flask UI |
| `LOG_STORAGES` | `{}` | 额外 storage 类路径→构造参数，如 `{"rdagent.log.ui.storage.WebStorage": [19899, "./log"]}` |

> `server_ui` 启动时会通过 `LOG_SETTINGS.set_ui_server_port(port)` 自动设此值，子进程经此把 trace 推回服务端。

### 3.7 Web UI 配置（`UI_` 前缀）

单例 `UI_SETTING`（`rdagent/log/ui/conf.py`）。

| 字段 | 默认 | 决定什么 |
|---|---|---|
| `UI_DEFAULT_LOG_FOLDERS` | `["./log"]` | UI 左侧默认日志目录列表 |
| `UI_BASELINE_RESULT_PATH` | `./baseline.csv` | 基准结果 CSV（UI 对比展示） |
| `UI_AIDE_PATH` | `./aide` | AIDE 数据路径 |
| `UI_AMLT_PATH` | `/data/share_folder_local/amlt` | AMLT 路径（内部 ML 平台） |
| `UI_STATIC_PATH` | `./git_ignore_folder/static` | 静态资源目录（HTML/JS/CSS） |
| `UI_TRACE_FOLDER` | `./git_ignore_folder/traces` | trace 根目录（HTTP 服务用此扫历史） |
| `UI_ENABLE_CACHE` | `True` | UI 缓存 |

> `UI_TRACE_FOLDER` 决定 HTTP `/traces` / `/upload` 的根目录，与 `LOG_TRACE_PATH` 是两个独立配置：前者是服务端聚合视图目录，后者是单个 rdagent 进程的写入目录。

### 3.8 context7 / RAG Agent 配置

#### context7（前缀 `CONTEXT7_`，单例 `SETTINGS`）

文件：`rdagent/components/agent/context7/conf.py`。**注意：这个类是裸 `BaseSettings`，不继承 `ExtendedBaseSettings`**。

| 字段 | 默认 | 决定什么 |
|---|---|---|
| `CONTEXT7_URL` | `http://localhost:8124/mcp` | context7 MCP 服务地址（Streamable HTTP） |
| `CONTEXT7_TIMEOUT` | `120` | MCP 请求超时（秒） |
| `CONTEXT7_ENABLE_CACHE` | `False` | 是否启用文档检索缓存 |

> 服务端需独立部署：`bun run dist/index.js --transport http --port 8124`（详见 conf 文件 docstring）。

#### RAG（前缀 `RAG_`，单例 `SETTINGS`）

文件：`rdagent/components/agent/rag/conf.py`。RAG Agent 与 context7 共享 MCP 后端。

| 字段 | 默认 | 决定什么 |
|---|---|---|
| `RAG_URL` | `http://localhost:8124/mcp` | RAG MCP 服务地址 |
| `RAG_TIMEOUT` | `120` | RAG 请求超时（秒） |

### 3.9 Benchmark 配置（`BENCHMARK_` 前缀）

单例 `BENCHMARK_SETTINGS`（`rdagent/components/benchmark/conf.py`）。Benchmark 测试用，量化 R&D 主流程不依赖。

| 字段 | 默认 | 决定什么 |
|---|---|---|
| `BENCHMARK_BENCH_DATA_PATH` | `./example.json` | benchmark 输入数据 |
| `BENCHMARK_BENCH_TEST_ROUND` | `10` | 测试轮数 |
| `BENCHMARK_BENCH_TEST_CASE_N` | `None` | 每轮用例数（None=全部） |
| `BENCHMARK_BENCH_METHOD_CLS` | `...factor_coder.FactorCoSTEER` | 被测方法类路径 |
| `BENCHMARK_BENCH_METHOD_EXTRA_KWARGS` | `{}` | 被测方法额外参数 |
| `BENCHMARK_BENCH_RESULT_PATH` | `./result` | 结果输出目录 |

---

## 4. 非 Settings 的环境变量（直接 `os.environ` 读取）

这些变量不经过 Pydantic Settings，被代码用 `os.environ.get(...)` / `os.getenv(...)` 直接读取。

| 变量 | 读取处 | 默认 | 决定什么 |
|---|---|---|---|
| `CUDA_VISIBLE_DEVICES` | `rdagent/utils/env.py:643, 982` | — | GPU 设备隔离。LocalEnv 自动透传给子进程；DockerEnv 据此选 `DeviceRequest(device_ids=...)`，未设则用全部 GPU |
| `CONDA_DEFAULT_ENV` | `rdagent/components/coder/factor_coder/config.py:40` | `None` | Factor CoSTEER 子进程的 conda env 名（**实际生效的解释器来源**，覆盖 `FACTOR_CoSTEER_PYTHON_BIN` 的本意） |
| `BACKEND` | `rdagent/app/utils/health_check.py:95` | — | health_check 检查该变量是否存在，缺失时告警提示去 `.env` 配置 |
| `DEEPSEEK_API_KEY` | `rdagent/app/utils/health_check.py:101` | — | 若存在，health_check 走 DeepSeek 配置分支（读 `CHAT_MODEL` / `EMBEDDING_MODEL` / `LITELLM_PROXY_API_KEY` / `LITELLM_PROXY_API_BASE` / `DEEPSEEK_API_BASE` / `OPENAI_API_BASE`） |
| `OPENAI_API_KEY` / `OPENAI_API_BASE` / `CHAT_MODEL` / `EMBEDDING_MODEL` | `rdagent/app/utils/health_check.py:113-117` | — | health_check 默认分支（OpenAI 风格）读取，用于测通 chat/embedding |
| `LITELLM_PROXY_API_KEY` / `LITELLM_PROXY_API_BASE` | `rdagent/app/utils/health_check.py:105-106` | — | health_check 的 embedding 测试端点（DeepSeek 分支） |
| `DEEPSEEK_API_BASE` | `rdagent/app/utils/health_check.py:107-108` | — | DeepSeek chat base url（health_check 用） |
| `OPENAI_API_KEY` | `rdagent/oai/backend/deprec.py:180, 183` | — | 旧后端的 key 优先级链回退（`chat_openai_api_key → openai_api_key → os.environ`） |
| `PYTHONHTTPSVERIFY` | `rdagent/oai/backend/deprec.py:156` | `""` | 旧后端 GCR 路径：未设时禁用 SSL 验证（**生产慎用**） |
| `{OPENAI/AZURE_AI/AZURE/LITELLM_PROXY}_API_KEY` / `_API_BASE` | `rdagent/oai/backend/pydantic_ai.py:50-51` | — | pydantic_ai Provider 构造时按 provider prefix 动态查 key/base |

---

## 5. 已知的"死配置"（在 .env 但代码不消费）

> 这些是当前 fork 点（`v0.8.0-30-g7cedd780`）下的真实状态，整理时已通过全仓 grep + 源码核查确认。
> 清理记录见各小节末"处置"。

### 5.1 `RDAGENT_MARKET`（完全死）✅ 已删除

- **原 `.env` 中**：`RDAGENT_MARKET=csi300`
- **代码消费**：**无任何引用**（Python / YAML / 模板均未读）
- **实际 market 来源**：硬编码在 `rdagent/scenarios/qlib/experiment/{factor,model}_template/conf_*.yaml` 的 YAML 锚点 `market: &market csi300`
- **处置（2026-07-19）**：已从 `.env` 删除。要换市场只能改模板 YAML。原位置保留注释说明去向。

### 5.2 `FACTOR_CoSTEER_PYTHON_BIN`（加载但不使用）⚠️ 保留

- **`.env` 中**：`FACTOR_CoSTEER_PYTHON_BIN=/home/zxh/miniconda3/envs/rdagent/bin/python`
- **代码消费**：`FactorCoSTEERSettings.python_bin` 字段存在，Pydantic 会加载该值；但 `rdagent/components/coder/factor_coder/config.py:39` 的 `get_factor_env()` 只用 `hasattr(conf, "python_bin")` 判存在，**然后忽略其值**，改用 `CondaConf(conda_env_name=os.environ.get("CONDA_DEFAULT_ENV"))`
- **处置（2026-07-19）**：**保留**。CLAUDE.md §6.1/§6.2 明确记录"有意指向 0.8 共享 env，避免重复安装 torch/qlib"为设计意图，删除会丢失该标注。已在 `.env` 加注释说明"当前代码未消费"，未来若修复 `get_factor_env()` 让 `python_bin` 生效即可直接起作用。

### 5.3 `.env.example` 中的过时项 ✅ 已删除

`.env.example` 模板里提到的 `FT_DOCKER_ENABLE_CACHE` / `DS_DOCKER_ENABLE_CACHE` 是 FineTune / DataScience 场景的配置，**本项目已裁剪这两个场景**（CLAUDE.md §0 / §10），故这两个变量在本仓库状态下无消费方。
**处置（2026-07-19）**：已从 `.env.example` 删除这两行。

---

## 6. 配置加载顺序与优先级

### 6.1 `ExtendedBaseSettings` 的源优先级

`rdagent/core/conf.py` 改写的 `settings_customise_sources` 决定优先级（高 → 低）：

```
1. 显式构造参数（__init__(field=value)）
2. 本类的 env settings（本类 env_prefix 下的变量）
3. 父类的 env settings（递归父类 env_prefix 下的变量）  ← ExtendedBaseSettings 新增
4. dotenv（.env 文件）
5. file secrets
```

**关键含义**：
- 父类字段也能被其前缀覆盖（如 `CoSTEER_MAX_LOOP` 能覆盖 `ModelCoSTEERSettings` 继承来的 `max_loop`）
- 显式 init 参数优先于 env，所以代码里 `DockerConf(image=...)` 会赢过 `QLIB_DOCKER_IMAGE`
- shell 环境变量 > `.env` 文件（同名字段）

### 6.2 加载时序

```
1. python 启动 → rdagent.app.cli 模块加载
2. load_dotenv(".env")  ← cli.py:13，把 .env 灌入 os.environ
3. 首次 import 各 Settings 模块 → 单例构造（LLM_SETTINGS / RD_AGENT_SETTINGS / ...）
4. 单例构造时按 6.1 优先级解析每个字段值
5. 此后全局共享这些单例
```

**坑点**：
- 如果你的代码在 `import rdagent.app.cli` 之前就 `import` 了某个 Settings 模块，`.env` 还没加载，单例会用 shell 环境变量 + Pydantic 自带的 `.env` 读取（仍能读到 CWD `.env`，但不会反映你后续的 `load_dotenv` 调用）
- 所以**始终通过 `rdagent ...` CLI 启动**是最安全的

### 6.3 前缀大小写

Pydantic 默认对 env 变量名**大小写不敏感**。所以 `CoSTEER_MAX_LOOP`、`COSTEER_MAX_LOOP`、`costeer_max_loop` 都能命中 `CoSTEERSettings.max_loop`。但 `QLIB_DOCKER_IMAGE` 这类全大写前缀建议保持一致写法以减少困惑。

---

## 7. 推荐的最小 `.env` 模板

> 本项目实际场景（Qlib 量化 + 火山方舟 Coding Plan 多 model + 豆包 embedding + Docker 训练）的最小必需集。其余配置使用代码默认值即可。

```dotenv
# ====== 必需：LLM 后端与模型（火山方舟 Coding Plan 统一端点） ======
BACKEND=rdagent.oai.backend.LiteLLMAPIBackend
CHAT_MODEL=openai/glm-5.2                          # 全局 fallback（未命中 CHAT_MODEL_MAP 的调用走这里）
CHAT_OPENAI_BASE_URL=https://ark.cn-beijing.volces.com/api/coding/v3
OPENAI_API_BASE=https://ark.cn-beijing.volces.com/api/coding/v3
CHAT_OPENAI_API_KEY=<方舟 Coding Plan key>
OPENAI_API_KEY=<方舟 Coding Plan key>
# per-step 路由（官方 chat_model_map 机制）+ temperature 用厂商官方推荐值
CHAT_MODEL_MAP={"direct_exp_gen":{"model":"openai/minimax-m3","temperature":"0.7"},"coding":{"model":"openai/kimi-k2.7-code","temperature":"1.0"},"running":{"model":"openai/deepseek-v4-flash","temperature":"0.0"},"feedback":{"model":"openai/glm-5.2","temperature":"0.6"}}

# ====== 必需：Embedding（方舟豆包，复用 chat 的方舟凭证） ======
EMBEDDING_MODEL=openai/doubao-embedding-vision
# 复用 OPENAI_API_KEY/BASE（与 chat 同指方舟，在 LLM 段已配，无需另设）

# ====== 必需：Model 训练执行环境 ======
MODEL_CoSTEER_ENV_TYPE=docker
QLIB_DOCKER_IMAGE=local_qlib:v2.0
QLIB_DOCKER_BUILD_FROM_DOCKERFILE=False

# ====== 可选：调用参数调优 ======
CHAT_TEMPERATURE=0.6
CHAT_MAX_TOKENS=16384

# ====== 可选：训练/验证/测试日期（默认 2008-01-01 ~ 2020-08-01） ======
# QLIB_FACTOR_TEST_END=2026-07-17
# QLIB_MODEL_TRAIN_START=2010-01-01
# QLIB_QUANT_VALID_END=2018-12-31

# ====== 可选：context7（用默认值可不写） ======
# CONTEXT7_URL=http://localhost:8124/mcp
# CONTEXT7_ENABLE_CACHE=true
```

**不必写进 .env 的项**（用代码默认值即可）：
- 所有 `WORKSPACE_*` / `MULTI_PROC_N` / `CACHE_*`（RDAgentSettings）—— 默认值适合单机开发
- 所有 `MAX_RETRY` / `RETRY_WAIT_SECONDS` —— 默认值合理
- 所有 `CoSTEER_*` 子系统调优项 —— 除非要做消融实验
- `RDAGENT_MARKET` —— 已删除的死配置（详见 [§5.1](#51-rdagent_market完全死-已删除)）
- `FACTOR_CoSTEER_PYTHON_BIN` —— 当前代码未消费的死配置（详见 [§5.2](#52-factor_costeer_python_bin加载但不使用-保留)），写也无害但不影响行为
- `LOG_*` / `UI_*` —— 默认值指向 `./log` / `./git_ignore_folder/`，符合本地开发习惯

---

## 附录：Settings 类索引（按 env 前缀）

| 前缀 | 类 | 单例 | 文件 |
|---|---|---|---|
| *(无)* | `RDAgentSettings` | `RD_AGENT_SETTINGS` | `rdagent/core/conf.py` |
| *(无)* | `LLMSettings` | `LLM_SETTINGS` | `rdagent/oai/llm_conf.py` |
| *(无)* | `EnvConf` / `LocalConf` / `CondaConf` / `DockerConf` / `QlibCondaConf` / `BasePropSetting` | — | `rdagent/utils/env.py` / `rdagent/components/workflow/conf.py` |
| `LITELLM_` | `LiteLLMSettings` | `LITELLM_SETTINGS` | `rdagent/oai/backend/litellm.py` |
| `QLIB_FACTOR_` | `FactorBasePropSetting` / `FactorFromReportPropSetting` | `FACTOR_PROP_SETTING` / `FACTOR_FROM_REPORT_PROP_SETTING` | `rdagent/app/qlib_rd_loop/conf.py` |
| `QLIB_MODEL_` | `ModelBasePropSetting` | `MODEL_PROP_SETTING` | `rdagent/app/qlib_rd_loop/conf.py` |
| `QLIB_QUANT_` | `QuantBasePropSetting` | `QUANT_PROP_SETTING` | `rdagent/app/qlib_rd_loop/conf.py` |
| `QLIB_DOCKER_` | `QlibDockerConf` | — | `rdagent/utils/env.py` |
| `CoSTEER_` | `CoSTEERSettings` | `CoSTEER_SETTINGS` | `rdagent/components/coder/CoSTEER/config.py` |
| `MODEL_CoSTEER_` | `ModelCoSTEERSettings` | `MODEL_COSTEER_SETTINGS` | `rdagent/components/coder/model_coder/conf.py` |
| `FACTOR_CoSTEER_` | `FactorCoSTEERSettings` | `FACTOR_COSTEER_SETTINGS` | `rdagent/components/coder/factor_coder/config.py` |
| `LOG_` | `LogSettings` | `LOG_SETTINGS` | `rdagent/log/conf.py` |
| `UI_` | `UIBasePropSetting` | `UI_SETTING` | `rdagent/log/ui/conf.py` |
| `CONTEXT7_` | `Settings`（裸 BaseSettings） | `SETTINGS` | `rdagent/components/agent/context7/conf.py` |
| `RAG_` | `Settings`（裸 BaseSettings） | `SETTINGS` | `rdagent/components/agent/rag/conf.py` |
| `BENCHMARK_` | `BenchmarkSettings` | `BENCHMARK_SETTINGS` | `rdagent/components/benchmark/conf.py` |

---

**版本**：v1.0（2026-07-19）
**适用目录**：`/home/zxh/projects/1.multialphaV/RD-Agent`
**源码依据**：本仓库 `rdagent/` 全量源码（fork 点 `v0.8.0-30-g7cedd780`）+ 实际 `.env` 文件
**配套文档**：[API.md](file:///home/zxh/projects/1.multialphaV/API.md)（接口总览）、[CLAUDE.md §6](file:///home/zxh/projects/1.multialphaV/CLAUDE.md)（环境策略与 .env 模板原始约定）
