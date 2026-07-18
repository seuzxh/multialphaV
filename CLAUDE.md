# CLAUDE.md — multialphaV (RD-Agent 最新版二次开发)

> 本文件是 Claude Code（及兼容 Agent）在本项目中工作的**强制性**行为约束。
> 任何方案制定、代码修改、子任务派发前，必须先读完本文件并遵守全部条款。
> 与项目内 `.trae/rules/`、`AGENTS.md` 并存时，**本文件优先级最高**。

---

## 0. 项目身份与范围

| 项 | 值 |
|---|---|
| 项目代号 | **multialphaV** |
| 项目根目录 | `/home/zxh/projects/1.multialphaV/RD-Agent` |
| 上游官方仓库 | https://github.com/microsoft/RD-Agent |
| 开发仓库（fork） | https://github.com/seuzxh/RD-Agent |
| 官方文档 | https://rdagent.readthedocs.io/en/latest/index.html |
| 基础框架 | RD-Agent（最新 main 分支），非 0.8 旧版 |
| 业务场景 | **量化金融（fin_quant / Qlib）** —— 全栈扩展（数据层 / 评估回测 / 模型集成 / Alpha 因子挖掘） |
| 共存旧项目 | `/home/zxh/quant_projects/rdagent`（基于 rdagent 0.8），仅作数据共享与对照基准，不作为代码基线 |
| 数据策略 | 共享底层 Qlib 数据（`/home/zxh/qlib_data`），**因子层独立**（见 §7） |
| 环境策略 | 共享系统级资源（conda base / Qlib 数据 / GPU / Docker），**项目级独立 conda env `multialphav`**（见 §6） |

**核心边界**：本项目的所有二次开发必须**建立在最新 RD-Agent 框架的扩展能力之上**，禁止把 0.8 旧项目的实现模式当作默认范式。

---

## 1. 强制前置：知识获取（任何方案制定前必做）

> **第一性原理**：在不知道 rdagent"为什么这样设计"之前，不要改动"它怎么实现"。

### 1.1 必读资料（按顺序）

1. **官方 README**：https://github.com/microsoft/RD-Agent/blob/main/README.md
2. **官方文档站**（按章节定向阅读，不要泛读）：
   - Introduction：https://rdagent.readthedocs.io/en/latest/introduction.html
   - Qlib 量化场景：https://rdagent.readthedocs.io/en/latest/scens/quant_agent_fin.html
   - 框架架构：https://rdagent.readthedocs.io/en/latest/project_framework_introduction.html
   - 安装与配置：https://rdagent.readthedocs.io/en/latest/installation_and_configuration.html
3. **Qlib 本体资料**（rdagent 的 Qlib 场景建立在 Qlib 之上，理解 Qlib 是前置条件）：
   - 官方文档站：https://qlib.readthedocs.io/en/latest/（重点关注 Data Layer、Expression 引擎、Workflow/qrun、Model/Strategy 章节）
   - GitHub：https://github.com/microsoft/qlib（Release 节点、Breaking Change、Python 支持范围）
   - **本项目用 Qlib 0.9.7**（见 §6.4），官方 Python 支持范围 `>=3.8`（rdagent 侧限定 3.10，见 §6.2）
   - Qlib-Server（`qlib-server.readthedocs.io`）是服务端数据服务器，本项目离线因子挖掘场景**不用**
4. **本仓库源码**（修改前必读相邻模块）：
   - `rdagent/core/`：Loop / Agent / Scenario / Workspace / Developer / Experiment 等核心抽象
   - `rdagent/components/workflow/rd_loop.py`：RDLoop 主循环
   - `rdagent/scenarios/qlib/`：Qlib 场景的 Scenario / Developer / Experiment 实现
   - `rdagent/app/qlib_rd_loop/`：fin_quant CLI 入口与因子/模型 Loop 装配
   - `rdagent/components/coder/CoSTEER/`：CoSTEER 自演化子系统
   - `rdagent/components/agent/context7/`：**原生 context7 集成**（见 §3）
5. **历史变更**：相关 Issue / PR / Commit，特别是 fork 点之后上游的更新。

### 1.2 必须确认的上下文（每次任务前自问）

- 任务落在哪个场景？Quant / Data Science / LLM Fine-tune / Core
- 是否依赖外部系统：Qlib 数据 / Docker / Kaggle API / LLM API / GPU
- 是否有历史实验、因子库、Trace 需要复用
- 用户是否指定了实现路径或显式禁止某种做法

---

## 2. Context7 作为上下文处理环境（强制）

> 本项目**指定 context7 作为官方上下文 / 文档检索环境**。

### 2.1 优先使用原生集成

最新版 rdagent 已**内置 context7 Agent**：[`rdagent/components/agent/context7/__init__.py`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/components/agent/context7/__init__.py)

- 类：`rdagent.components.agent.context7.Agent`（继承 `PAIAgent`）
- 后端：`MCPServerStreamableHTTP`（Streamable HTTP MCP）
- 配置：[`rdagent/components/agent/context7/conf.py`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/components/agent/context7/conf.py) 中的 `SETTINGS.url` / `SETTINGS.timeout` / `SETTINGS.enable_cache`
- 用途：将错误信息 / 代码上下文构造为增强查询，调用 context7 检索文档与示例。

**规则**：
- 任何涉及"框架 API 不确定"、"第三方库报错"、"需要查官方用法"的场景，**优先调用 context7 Agent**，而非凭记忆推断。
- 不要重新实现文档检索逻辑；扩展时复用 `PAIAgent` 基类与 `T(".prompts:...")` 模板机制。

### 2.2 MCP 工具直接访问

Claude Code 环境内已挂载 `mcp_context7` MCP 服务，提供：
- `resolve-library-id`：把库名解析为 context7 内部 ID
- `query-docs`：按库 ID 检索文档片段

调用前**必须**先用 LS / Read 读取 `~/.trae-cn/mcps/*/mcp_context7/tools/*.json` 确认 schema，参数放入 `args` 字段。

---

## 3. 架构分析框架（方案制定阶段必填）

> **目的**：确保方案是"rdagent 原生扩展"，而非"异构联想"。任何方案文档必须显式完成下表。

### 3.1 必须列出的核心组件清单

在方案中**逐项确认**将要使用 / 继承 / 扩展的 rdagent 组件：

| 层级 | rdagent 模块 | 关键基类 / 协议 | 本任务是否涉及 | 复用方式 |
|---|---|---|---|---|
| 核心 Loop | `rdagent/components/workflow/rd_loop.py` | `RDLoop` | ☐ | 继承 / 装配 / 不动 |
| Scenario | `rdagent/core/scenario.py` | `Scenario` | ☐ | 继承 `QlibFactorScenario` 等 |
| Developer | `rdagent/core/developer.py` | `Developer` / `EvolvingAgent` | ☐ | |
| Experiment | `rdagent/core/experiment.py` | `Experiment` / `ExpWorkspace` | ☐ | |
| Evaluator | `rdagent/core/evaluation.py` | `Evaluator` | ☐ | |
| Hypothesis / Proposal | `rdagent/core/proposal.py` | `Hypothesis` / `HypothesisGen` | ☐ | |
| Knowledge Base | `rdagent/core/knowledge_base.py` | `KnowledgeBase` | ☐ | |
| Coder (CoSTEER) | `rdagent/components/coder/CoSTEER/` | `CoSTEER` / `EvolvingStrategy` / `Evaluator` | ☐ | |
| Factor Coder | `rdagent/components/coder/factor_coder/` | `FactorCoder` | ☐ | |
| Model Coder | `rdagent/components/coder/model_coder/` | `ModelCoder` | ☐ | |
| Qlib Scenario | `rdagent/scenarios/qlib/experiment/` | `QlibFactorScenario` / `QlibFactorFromReportScenario` / `QlibModelScenario` / `QlibQuantScenario` | ☐ | 仅此四个属本项目范围 |
| App 入口 | `rdagent/app/qlib_rd_loop/` | `factor.py` / `factor_from_report.py` / `model.py` / `quant.py` | ☐ | |

### 3.2 必须遵守的扩展性 / 兼容性协议

1. **Loop 协议**：新 Loop 必须能被 `RDLoop.run()` 主流程调度；如改主循环，先用 CodeGraph `Show Callers` 评估影响面。
2. **Scenario 协议**：`Scenario` 子类必须实现框架要求的 `background` / `task` / `output_format` 等提示词接口，与 `Prompts` 系统协同。
3. **Pydantic 配置协议**：所有可配置项必须走 `rdagent/core/conf.py` + `rdagent/utils/factory.py` 的 settings 注入，**禁止硬编码**。
4. **Workspace 协议**：实验产物落地路径、Docker 挂载、缓存目录必须遵循 `ExpWorkspace` 约定。
5. **Logger 协议**：使用 `rdagent.log.rdagent_logger`，不要另起 logging 实例。
6. **Prompt 模板协议**：使用 `T(".prompts:xxx").r(...)` 加载 YAML 模板，不要把 prompt 写死在 .py 里。

---

## 4. 异构实现识别机制（核心质量门禁）

> **定义**：
> - **框架一致扩展**：直接继承 rdagent 基类、复用其 Loop / Scenario / CoSTEER 子系统、走原生配置与日志。
> - **异构实现**：绕过 rdagent 抽象，引入与框架设计哲学冲突的外部模式；或仅凭对 0.8 旧项目 / 其他框架的"联想"拼凑方案。

### 4.1 异构实现的红旗信号（出现即必须标记 ⚠️）

1. 在 rdagent 已有抽象（`Scenario` / `Developer` / `Loop`）之外，新建一套并行的调度体系。
2. 用通用 `requests` / 自研 HTTP 客户端调用 LLM，而非走 `rdagent.oai` 后端。
3. 自建 logging、配置加载、Docker 编排，不复用 `rdagent.log` / `rdagent.core.conf` / `rdagent.utils.docker`。
4. 把 0.8 旧项目里非 rdagent 的辅助脚本（如自研因子评估器）原样搬入，未与 `Evaluator` 协议对齐。
5. 在 Prompt 中硬编码中英文长文本，绕过 `.prompts:yaml` 模板。
6. 引入 langchain / llama-index 等编排框架承担 rdagent Loop 已承担的职责。

### 4.2 强制差异化说明模板

一旦方案中**任何一项**触发红旗，必须在该方案文档中单独列出：

```markdown
### ⚠️ 异构实现清单

| 异构点 | 涉及模块 | 偏离的 rdagent 规范 | 必要性论证 | 替代方案（更接近框架） | 决策 |
|---|---|---|---|---|---|
| 例：自研多因子组合器 | `multialpha/selector.py` | `rdagent.core.evaluation.Evaluator` 未提供组合维度 | 需跨因子协方差优化，Evaluator 是单因子粒度 | 子类化 `QlibFactorEvaluator` 并 override `evaluate()` | 采纳替代方案，撤销异构 |
```

**未经此清单登记的异构实现，Code Review 一律否决。**

### 4.3 优先级裁决

当出现"框架方式 vs 联想方式"两条路径时，按以下顺序裁决：
1. 框架原生扩展（继承 + override）
2. 框架扩展点不足时，**先在方案中论证缺口**（指出缺哪个 hook / 抽象方法），再决定是补框架还是局部异构
3. 仅当 1、2 都不可行且用户书面同意，才允许异构实现，并登记到 §4.2

---

## 5. CodeGraph 集成（代码检索 / 依赖分析 / 整理）

> 本项目**强制集成 CodeGraph** 作为代码理解的唯一权威索引工具，优先级高于 grep / ripgrep / 裸 Read。

### 5.1 索引约定

- 索引目录：`/home/zxh/projects/1.multialphaV/RD-Agent/.codegraph/`
- 索引数据库：`.codegraph/codegraph.db`（SQLite，**必须加入 `.gitignore`**）
- daemon 日志：`.codegraph/daemon.log`
- 索引范围：`rdagent/` 全部源码 + `multialpha/`（本项目扩展目录）
- 排除项：`traces/`、`git_ignore_folder/`、`*.pkl`、`__pycache__/`、`tests/`（conda env 在项目目录外，无需排除）

### 5.2 强制使用场景

| 场景 | 命令 | 必要性 |
|---|---|---|
| 修改 `rdagent/core/`、`rdagent/components/workflow/`、`rdagent/scenarios/qlib/` 前 | `CodeGraph: Show Callers` | **必做**，否则禁止改 |
| 理解陌生模块 | `CodeGraph: Show Callees` / `Search Symbol` | 优先于 `Read` 整文件 |
| 重命名 / 删除 / 改签名 | `CodeGraph: Show Impact` | **必做** |
| 查找抽象基类实现（如 `Scenario` 的所有子类） | `CodeGraph: Search Symbol` | 优先于 `Grep` |
| 跨项目对比（见 §7） | 两份索引分别查询后人工对齐 | 推荐 |

### 5.3 索引维护

- 首次接入：命令面板 → `CodeGraph: Re-index`，等待 `.codegraph/daemon.log` 出现 `index ready`。
- 依赖大变更（新增包 / 大规模重构）后必须重建索引。
- daemon 异常：先看 `.codegraph/daemon.log` 末尾 50 行，再考虑重启 Trae 窗口。

### 5.4 与原生 `SearchCodebase` 的分工

- CodeGraph：**精确的调用关系、符号依赖、变更影响**（语义索引）。
- `SearchCodebase`：**自然语言意图检索**（embedding），用于"哪里做了 X"类模糊查询。
- 两者互补，不可互相替代。

---

## 6. 环境配置（从 0.8 项目同步 + 增量）

### 6.1 系统级共享资源（只读引用，不复制）

| 资源 | 路径 / 值 | 说明 |
|---|---|---|
| Qlib 数据 | `/home/zxh/qlib_data` | 两项目共享，格式见 `qlib_data_format.md` |
| conda base | `/home/zxh/miniconda3` | 仅用于基础 Python 解释器 |
| Factor CoSTEER 解释器 | `/home/zxh/miniconda3/envs/rdagent/bin/python` | CoSTEER 子进程用，**保持指向 0.8 env** 以避免重复装包 |
| Docker Qlib 镜像 | `local_qlib:v2.0` | 已构建，`QLIB_DOCKER_BUILD_FROM_DOCKERFILE=False` 跳过重建 |
| GPU | 共享 | 通过 Docker device 映射使用 |

### 6.2 项目级独立 conda env（本项目专用）

```bash
# 创建独立的 conda 虚拟环境（Python 3.10，与官方 CI / 0.8 项目 / constraints 锁定文件一致）
cd /home/zxh/projects/1.multialphaV/RD-Agent
conda create -n multialphav python=3.10 -y
conda activate multialphav
pip install -e .  # 安装最新 rdagent 本体
pip install <multialphaV 扩展依赖>  # 按需增量
```

> **Python 版本选择依据**：官方 CI 仅测 3.10/3.11（[`.github/workflows/ci.yml`](file:///RD-Agent/.github/workflows/ci.yml)），constraints 仅有 3.10/3.11 锁定文件，0.8 项目用 3.10.20。选 3.10 实现三方一致，零跨版本风险。
>
> **禁止**把本项目依赖装进 `/home/zxh/miniconda3/envs/rdagent`（那是 0.8 的领地）。
> Factor CoSTEER 子进程沿用 0.8 env 是有意为之（共享 torch/qlib），勿改。

### 6.3 `.env` 配置模板（本项目根目录）

> 复制自 0.8 项目并按本项目增量调整；**禁止把真实 Key 提交进 git**，仅放本地 `.env`（已在 `.gitignore`）。

```dotenv
# ==================== 后端 ====================
BACKEND=rdagent.oai.backend.LiteLLMAPIBackend

# ==================== LLM（智谱 AI / GLM） ====================
CHAT_OPENAI_BASE_URL=https://open.bigmodel.cn/api/coding/paas/v4
OPENAI_API_BASE=https://open.bigmodel.cn/api/coding/paas/v4
CHAT_MODEL=openai/glm-5.2
CHAT_TEMPERATURE=0.5
CHAT_MAX_TOKENS=16384
#CHAT_OPENAI_API_KEY=        # 从 0.8 项目 .env 继承，勿入库
#OPENAI_API_KEY=

# ==================== Embedding（火山方舟 / 豆包） ====================
EMBEDDING_MODEL=openai/doubao-embedding-vision
EMBEDDING_OPENAI_BASE_URL=https://ark.cn-beijing.volces.com/api/coding/v3
#EMBEDDING_OPENAI_API_KEY=   # 从 0.8 项目 .env 继承，勿入库

# ==================== Qlib / 市场 ====================
RDAGENT_MARKET=csi300
# provider_uri 在代码中显式传 /home/zxh/qlib_data（见 qlib_data_format.md §5）

# ==================== Factor CoSTEER 子进程 ====================
# 有意指向 0.8 共享 env，避免重复安装 torch/qlib
FACTOR_CoSTEER_PYTHON_BIN=/home/zxh/miniconda3/envs/rdagent/bin/python

# ==================== Model 训练执行环境 ====================
MODEL_CoSTEER_ENV_TYPE=docker
QLIB_DOCKER_IMAGE=local_qlib:v2.0
QLIB_DOCKER_BUILD_FROM_DOCKERFILE=False

# ==================== context7（原生 Agent） ====================
# 见 rdagent/components/agent/context7/conf.py，默认值通常足够
# CONTEXT7_URL=
# CONTEXT7_TIMEOUT=
# CONTEXT7_ENABLE_CACHE=true
```

### 6.4 关键版本基线（核对命令）

| 组件 | 版本 | 核对命令 |
|---|---|---|
| Python | 3.10（与官方 CI / 0.8 项目 / constraints 一致） | `python --version` |
| rdagent | fork 点 commit（当前 `v0.8.0-29-g8f60d6ea`） | `git describe --tags --always`（在 RD-Agent 仓库内） |
| rdagent 包版本（pip 元数据） | setuptools-scm 动态生成（当前 `0.8.1.dev29`） | `python -c "from importlib.metadata import version; print(version('rdagent'))"` |
| qlib | 0.9.7（在 Docker 镜像内，主 env 不装） | `docker run --rm local_qlib:v2.0 python -c "import qlib;print(qlib.__version__)"` |
| pydantic | 2.x（rdagent 依赖，未 pin） | `python -c "import pydantic; print(pydantic.VERSION)"` |

> **同步升级对比**：rdagent 用 `git describe` 的 commit 标识（含 tag + commit 数 + hash）跟踪 fork 点。对比上游时：`git fetch upstream && git log --oneline v0.8.0-29-g8f60d6ea..upstream/main` 可看出上游新增了多少 commit。
>
> **注意**：`rdagent.__version__` 属性不存在（setuptools-scm 动态生成不注入包级属性），不要用 `python -c "import rdagent; rdagent.__version__"`。
| pydantic | 与 rdagent `pyproject.toml` 锁定一致 | `pip show pydantic` |

---

## 7. 与 rdagent 0.8 项目的兼容性评估与差异化管理

> 0.8 项目位置：`/home/zxh/quant_projects/rdagent`（**只读参照**，不作为运行时依赖）。

### 7.1 三层资产关系

> **Qlib 数据层背景**：Qlib 的 `provider_uri` 根目录下三个子目录构成其 Data Layer —— `calendars/`（时间轴）、`instruments/`（股票池/成分股）、`features/`（按 `$field` 存储的 `.bin` 因子值，纯 float32）。这三者是 Qlib 表达式引擎（如 `Mean($close, 5)`）求值的底座；因子表达式本身是"计算逻辑"，`.bin` 文件是"原料"。本项目与 0.8 项目共享原料、独立产出计算逻辑。详见 [Qlib Data Layer 文档](https://qlib.readthedocs.io/en/latest/component/data.html)。

| 资产层 | 关系 | 处置策略 |
|---|---|---|
| **Qlib 底层数据**（`/home/zxh/qlib_data` 的 features/calendars/instruments） | **完全共享** | 双方只读；任一方修改必须双方验证（见 `qlib_data_format.md` §6 checklist） |
| **因子层**（各自产出的 Alpha 表达式 / .bin 因子文件 / 因子库） | **完全独立** | 跨项目复用必须走 §7.4 转换规则 |
| **模型 / 实验 Trace / 知识库** | **完全独立** | 不互通；如需对照，仅以最终指标（IC / IR / Sharpe）做基准对照 |

### 7.2 API 版本适配矩阵（方案制定阶段必填）

每次扩展涉及 rdagent API 时，必须在方案中登记：

| rdagent API（最新版） | 0.8 中的对应 | 签名差异 | 适配策略 |
|---|---|---|---|
| 例：`RDLoop(...)` | `RDLoop(...)`（0.8） | 0.8 无 `selector` 入参 | 本项目用最新签名；不向 0.8 回写 |
| ...（逐项填） | | | |

> 填写依据：本项目用 CodeGraph 查询；0.8 项目用 `Grep`/`Read`（其目录不在本项目 CodeGraph 索引内）。

### 7.3 功能模块映射（核心子系统对照）

| 子系统 | 最新版（本项目基线） | 0.8 项目 | 差异要点 |
|---|---|---|---|
| 主循环 | `rdagent/components/workflow/rd_loop.py` | 同名 | 主线已演进，以最新为准 |
| Qlib 因子 Loop | `rdagent/app/qlib_rd_loop/factor.py` | 同路径旧版 | 入参 / prompts 已变 |
| Qlib 模型 Loop | `rdagent/app/qlib_rd_loop/model.py` | 同路径旧版 | 同上 |
| CoSTEER | `rdagent/components/coder/CoSTEER/` | 旧版 | 策略与评估器接口已重构 |
| context7 | `rdagent/components/agent/context7/`（原生） | 0.8 无 | **本项目独有**，0.8 不兼容 |
| 知识管理 | `rdagent/components/knowledge_management/` | 旧版 | 向量库 schema 可能不同 |
| Web UI | `rdagent/log/ui/` | 旧版 | 端口、组件已变 |

### 7.4 数据 / 因子格式转换规则（跨项目搬运必走）

1. **因子表达式（字符串形式）**：两项目通用的 Qlib 表达式可直接复用（如 `Mean($close, 5)`）。
2. **因子 .bin 文件**：格式严格遵循 `qlib_data_format.md` §2（纯 float32，无 header，首 4 字节为 `start_index`）。跨项目拷贝前**必须**用 §6 checklist 验证 `start_index < len(calendar)`。
3. **Trace / 日志产物**：两项目 schema 不同，**禁止直接拷贝**；如需对照，仅提取最终指标 JSON。
4. **Hypothesis / Proposal 对象**：Pydantic schema 跨版本可能不兼容，禁止 pickle 直传；以 JSON 序列化为准并显式做字段映射。

### 7.5 禁止的反模式

- ❌ 直接 `cp -r /home/zxh/quant_projects/rdagent/rdagent/* ` 到本项目覆盖最新代码。
- ❌ 把 0.8 的 `.env` 整体复制过来（context7 / 新版 LLM 后端配置会缺失）。
- ❌ 把 0.8 的 traces / knowledge_base 直接软链到本项目（schema 不兼容）。
- ❌ 在方案文档里写"参考 0.8 实现"而不附 §7.2 适配矩阵。

---

## 8. 开发原则（六条强制条款）

### 原则 1：奥卡姆剃刀（开发） + 墨菲定律（验收）

- **开发阶段**：如无必要，勿增实体。每个新组件、新参数、新抽象必须有显式业务需求支撑。三行重复优于一次过早抽象。
- **验收阶段**：凡是可能出错的，一定会出错。必须设计：
  - 边界测试（空数据、单股、全 NaN、start_index 溢出）
  - 异常场景测试（LLM 超时、Docker 不可用、Qlib 数据缺失、GPU OOM）
  - 并发 / 重入测试（Loop 被外部中断后能否续跑）

### 原则 2：第一性原理决策

解决问题、修 Bug、设计架构时：
- 先回到 rdagent 的核心设计理念（Hypothesis → Experiment → Feedback → Knowledge 的 R&D 闭环）。
- 基于"框架为什么这样设计"做判断，**不基于**"其他框架（langchain / autogen / 0.8 项目）是怎么做的"。
- 经验主义方案必须先翻译为框架语言（落到 `Scenario` / `Loop` / `Evaluator` 哪一层），否则视为异构。

### 原则 3：多 Agent 对抗性审查（复杂任务自动触发）

**触发条件**（满足任一即触发）：
- 需要 > 3 个模块协作
- 涉及 `rdagent/core/` 或 `rdagent/components/workflow/` 的核心功能变更
- 涉及与 0.8 项目的兼容性 / 数据格式
- 触及 Docker 镜像、Qlib 数据写入、LLM 后端切换

**审查机制**：触发后**必须**派发 **≥ 3 个独立审查 Agent**，各覆盖一个维度：

| 审查 Agent | 维度 | 必查项 |
|---|---|---|
| Agent-A：框架一致性 | rdagent 原生扩展度 | 是否触发 §4 红旗？是否填了 §3 表？ |
| Agent-B：健壮性 / 可靠性 | 异常、并发、回滚 | 是否覆盖墨菲定律全部场景？ |
| Agent-C：跨项目兼容性 | 0.8 数据 / API | 是否填了 §7.2 矩阵？转换规则是否完备？ |

三份报告必须留存（`docs/review/<task>-<agent>.md`），**任一 Agent 否决即不得合入**，需回到方案阶段。

### 原则 4：严格限定任务范围（DO NOT send optional commentary）

- 每个响应**只**针对当前任务目标。
- 禁止附带"顺便建议"、"未来可考虑"、"额外优化"等无关信息。
- 发现的旁支问题最多一句话提示，不展开；是否处理由用户决定。
- **每行代码改动都必须能追溯到用户请求**（参考 `CLAUDE.md` §3 Surgical Changes）。

### 原则 5：精准提问（一次一问，达 95% 信心再动手）

在给出实质性方案前：
1. **一次只问一个问题**（多选项可批量，但每个问题独立）。
2. 根据回答**持续追问**，直到通过下面的"需求确认矩阵"达到 ≥ 95% 信心。

**需求确认矩阵**（每项 0–100% 自评，加权平均 ≥ 95% 才动手）：

| 维度 | 权重 | 必须明确到 |
|---|---|---|
| 功能边界 | 25% | 做什么、不做什么、与现有功能的交叠 |
| 数据契约 | 20% | 输入 / 输出 schema、Qlib 字段、因子格式 |
| 框架落点 | 20% | 落到 §3 表的哪一行、继承哪个基类 |
| 失败语义 | 15% | 异常如何上报、是否可重试、是否污染共享数据 |
| 验收标准 | 10% | 用什么指标 / 测试判定完成 |
| 范围禁区 | 10% | 用户明确禁止什么 |

若某维度无法通过提问提升到要求水位，**必须显式声明假设**并在方案中标注 `ASSUMPTION:`，由用户在方案评审时确认或推翻。

### 原则 6：科斯定理派发子 Agent

把任务交给子 Agent（Task 工具）前，必须评估：
- **交易成本**（协调开销）：子任务边界是否清晰？产出是否易验证？若需要反复对齐，交易成本 > 收益，应自己做。
- **外部性**：子 Agent 的改动是否波及他人模块？若有负外部性（如改公共基类），必须内化（在本 Agent 内完成或加锁）。
- **资源优化**：并行可独立的子任务才并行；存在共享状态的必须串行。

**判定流程**：
1. 子任务产出能否用 ≤ 3 句话验收？能 → 可派发；不能 → 自己做。
2. 子任务是否触及 `rdagent/core/`？是 → **禁止派发**（核心改动必须主 Agent 把控）。
3. 多个子任务是否有共享文件 / 共享状态？有 → 串行；无 → 并行。
4. 派发时**必须**给子 Agent 完整上下文（本文件要点 + 任务目标 + 验收标准），子 Agent 无对话历史。

### 原则 7：复杂问题用 sequential-thinking MCP 先拆解再执行

**触发条件**（满足任一即触发，先拆解后动手）：
- 问题涉及 ≥ 3 个相互依赖的子系统（如 Loop × CoSTEER × Qlib Scenario）
- 问题陈述存在歧义、目标状态不清晰、或"完整范围初期不可见"
- 修复方案可能有多个分支路径，需要在比较后抉择
- 改动可能产生连锁影响（核心模块、跨项目数据契约）
- 排查 Bug 时根因不明、需要"假设—验证—修正"多轮推演

**工具**：Claude Code 环境内已挂载 `mcp_Sequential_Thinking` MCP 服务，工具名 `sequentialthinking`。

**必填参数**（调用前必读 schema：`~/.trae-cn/mcps/*/mcp_Sequential_Thinking/tools/sequentialthinking.json`）：
- `thought`（str）：当前思考步骤的内容
- `thoughtNumber`（int）：当前步骤序号，从 1 起
- `totalThoughts`（int）：预估总步数（可随进展上调/下调）
- `nextThoughtNeeded`（bool）：是否还需下一步（仅当真正得出满意答案时设 false）

**可选参数**（用于修正与分支）：
- `isRevision=true` + `revisesThought=N`：修正第 N 步的推理
- `branchFromThought=N` + `branchId=...`：从第 N 步分叉探索新方向
- `needsMoreThoughts=true`：到达预估末步时发现还需更多思考

**使用纪律**：
1. **拆解顺序**：先列出子问题树 → 估计 `totalThoughts` → 逐 thought 推进，每步聚焦一个子问题。
2. **允许回溯**：发现先前判断错误立即用 `isRevision` 修正，不要硬走到底；这是该工具区别于线性思维的核心价值。
3. **明确产出**：拆解结束后**必须**输出可执行的子任务清单（落到 §3 表的哪一行 / 哪个模块），再进入 §9 流程的第 2 步以后。
4. **禁止替代人类决策**：sequential-thinking 只做问题拆解与方案推演，涉及用户意图的抉择仍走原则 5 的精准提问。
5. **不与原则 1 冲突**：奥卡姆剃刀约束的是"实体数量"，sequential-thinking 约束的是"思考步数"；前者防过度设计，后者防思考跳跃。
6. 简单任务（单模块、目标清晰、≤ 2 个子系统）**无需触发**本原则，避免工具滥用。

---

## 9. 通用开发流程（每次任务必走）

```
1. 知识获取（§1）→ 必读资料 + CodeGraph 查调用方
2. 复杂问题拆解（§8 原则 7）→ 若触发条件成立，用 sequential-thinking MCP 产出子任务清单；简单任务跳过
3. 需求确认（§8 原则 5）→ 一次一问直至 95% 信心
4. 架构落点（§3）→ 填核心组件表 + 扩展协议
5. 异构审查（§4）→ 红旗自检；有则登记
6. 兼容评估（§7）→ 与 0.8 的 API/数据矩阵
7. 实现 → 奥卡姆剃刀；每步可验证
8. 复杂任务触发多 Agent 审查（§8 原则 3）
9. 验收 → 墨菲定律全覆盖
10. 交付摘要 → 改了什么 / 验证证据 / 文件链接
```

---

## 10. 禁止事项（硬性红线）

- ❌ 未读官方文档与相关源码就制定方案。
- ❌ 跳过 §3 表与 §4 异构自检直接写代码。
- ❌ 把 0.8 项目代码当默认范式拷贝。
- ❌ 在代码 / 配置 / 日志中硬编码 API Key、Token、密码（一律走 `.env`）。
- ❌ 修改 `rdagent/core/`、`rdagent/components/workflow/` 前未用 CodeGraph `Show Callers`。
- ❌ 用 grep 替代 CodeGraph 查调用关系（CodeGraph 可用时）。
- ❌ 把 prompt 硬编码在 .py 中（必须走 `.prompts:yaml`）。
- ❌ 触发 §8 原则 3 的复杂任务却跳过多 Agent 审查。
- ❌ 未经用户确认就执行 `git commit` / `git push` / 破坏性 git 操作。
- ❌ 主动创建 README / docs，除非用户明确要求或属于必要交付物。
- ❌ 修改或引用非 Qlib 场景目录（`data_science` / `kaggle` / `finetune` / `general_model` / `rl`），违反 §0 场景禁区。
- ❌ 在响应中夹带可选评论 / 无关建议（违反 §8 原则 4）。

---

## 11. 交付与代码引用规范

- **执行摘要**：完成了什么、修改了哪些文件、验证证据（日志 / 指标 / 测试输出）。
- **代码引用**：所有文件与行范围使用绝对路径 markdown 链接：
  - 文件：`[文件名](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/core/scenario.py)`
  - 行范围：`[scenario.py#L10-L30](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/core/scenario.py#L10-L30)`
- **文档同步**：行为变更必须同步更新受影响的 `docs/` 章节与 `.trae/rules/`。
- **Git 操作**：不主动 commit / push；如需提交先确认无敏感信息泄露。

---

## 12. 常用自检命令

```bash
# 进入项目独立 conda env
conda activate multialphav
cd /home/zxh/projects/1.multialphaV/RD-Agent

# 版本核对（注意：rdagent.__version__ 不存在，用 git describe 或 importlib.metadata）
git describe --tags --always                    # rdagent fork 点 commit（如 v0.8.0-29-g8f60d6ea）
python --version                                # Python 3.10.x
python -c "import pydantic; print(pydantic.VERSION)"  # pydantic 2.x

# Qlib 数据可用性（qlib 在 Docker 内，主 env 不装；数据软链生效后用 0.8 env 核对）
/home/zxh/miniconda3/envs/rdagent/bin/python -c "import qlib; from qlib.constant import REG_CN; qlib.init(provider_uri='/home/zxh/qlib_data', region=REG_CN); from qlib.data import D; print(len(D.calendar()))"

# fin_quant 帮助（确认 CLI 装配正确）
rdagent fin_quant --help

# CodeGraph 索引状态
ls -la .codegraph/ && tail -n 50 .codegraph/daemon.log

# Docker 镜像可用性
docker images | grep local_qlib
```

---

**版本**：v1.0
**适用目录**：`/home/zxh/projects/1.multialphaV/RD-Agent`
**上游依据**：[microsoft/RD-Agent](https://github.com/microsoft/RD-Agent)、[官方文档](https://rdagent.readthedocs.io/en/latest/index.html)、本项目源码、`/home/zxh/quant_projects/rdagent/.trae/rules/qlib_data_format.md`

---

**版本**：v1.0
**适用目录**：`/home/zxh/projects/1.multialphaV/RD-Agent`
**上游依据**：[microsoft/RD-Agent](https://github.com/microsoft/RD-Agent)、[官方文档](https://rdagent.readthedocs.io/en/latest/index.html)、本项目源码、`/home/zxh/quant_projects/rdagent/.trae/rules/qlib_data_format.md`
