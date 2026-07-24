# multialphaV 平台接口文档（API Reference）

> 本文档整理 multialphaV 平台（基于 RD-Agent 最新版 + Qlib 量化场景二次开发）对外提供的全部接口，分为四类：
> 1. **CLI 命令**（`rdagent ...`）—— 终端入口
> 2. **HTTP API**（Flask 服务，`rdagent server_ui`）—— Web/程序化访问
> 3. **Python 库 API**（核心抽象 + Qlib 场景 + LLM/Logger/Env）—— 二次开发用
> 4. **配置接口**（环境变量 / Pydantic Settings）—— 行为可调点
>
> 适用目录：`/home/zxh/projects/1.multialphaV/RD-Agent`
> 源码依据：rdagent 0.8.1.dev29（fork 点 `v0.8.0-30-g7cedd780`，裁剪非 Qlib 场景后）
> 维护约定：任何接口签名 / 参数 / 配置键变更，必须同步本文档（CLAUDE.md §9 改后审查清单 B）。

---

## 目录

- [1. CLI 命令接口](#1-cli-命令接口)
  - [1.1 全局入口](#11-全局入口)
  - [1.2 业务命令：Qlib 量化 R&D 循环](#12-业务命令qlib-量化-rd-循环)
  - [1.3 运维命令](#13-运维命令)
- [2. HTTP API（Flask 服务）](#2-http-apiflask-服务)
- [3. Python 库 API](#3-python-库-api)
  - [3.1 RDLoop 主循环](#31-rdloop-主循环)
  - [3.2 核心抽象（rdagent.core）](#32-核心抽象rdagentcore)
  - [3.3 Qlib 场景（rdagent.scenarios.qlib）](#33-qlib-场景rdagentscenariosqlib)
  - [3.4 LLM 后端（rdagent.oai）](#34-llm-后端rdagentoai)
  - [3.5 日志与 UI（rdagent.log）](#35-日志与-uirdagentlog)
  - [3.6 执行环境（rdagent.utils.env）](#36-执行环境rdagentutilsenv)
  - [3.7 context7 文档检索 Agent](#37-context7-文档检索-agent)
- [4. 配置接口](#4-配置接口)
  - [4.1 全局配置 RDAgentSettings](#41-全局配置-rdagentsettings)
  - [4.2 LLM 配置](#42-llm-配置)
  - [4.3 日志配置](#43-日志配置)
  - [4.4 UI 配置](#44-ui-配置)
  - [4.5 Qlib 循环配置](#45-qlib-循环配置)
  - [4.6 Docker 配置](#46-docker-配置)
  - [4.7 context7 配置](#47-context7-配置)

---

## 1. CLI 命令接口

### 1.1 全局入口

```bash
rdagent [COMMAND] [OPTIONS]
```

- 包入口：`pyproject.toml` 中 `[project.scripts] rdagent = "rdagent.app.cli:app"`
- 实现文件：`rdagent/app/cli.py`（基于 `typer`）
- 启动时自动加载当前目录 `.env`（`dotenv.load_dotenv(".env")`）
- 在 CLI 调用时**所有参数均可省略**，省略时使用默认值。

### 1.2 业务命令：Qlib 量化 R&D 循环

本平台保留四个 Qlib 场景入口（裁剪掉了 data_science / kaggle / finetune / general_model / rl 等非 Qlib 场景）。

#### `rdagent fin_factor` — Qlib 因子挖掘循环

```bash
rdagent fin_factor [--path PATH] [--step_n N] [--loop_n N]
                   [--all_duration DURATION] [--checkout/--no-checkout]
```

| 参数 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `--path` | `str?` | None | 已有 session 路径；提供则恢复运行，否则新建循环 |
| `--step_n` | `int?` | None | 恢复 session 时执行的步数 |
| `--loop_n` | `int?` | None | 本次运行的循环（loop）数量 |
| `--all_duration` | `str?` | None | 总运行时长限制（如 `"2h"`、`"30m"`） |
| `--checkout` / `--no-checkout` | `bool` | True | 恢复 session 时是否 git-checkout 工作区 |

实现：`rdagent/app/qlib_rd_loop/factor.py:main` → `FactorRDLoop`。

#### `rdagent fin_model` — Qlib 模型实现循环

```bash
rdagent fin_model [--path PATH] [--step_n N] [--loop_n N]
                  [--all_duration DURATION] [--checkout/--no-checkout]
```

参数同 `fin_factor`。实现：`rdagent/app/qlib_rd_loop/model.py:main` → `ModelRDLoop`。

#### `rdagent fin_quant` — Qlib 全流水线（因子+模型联合）循环

```bash
rdagent fin_quant [--path PATH] [--step_n N] [--loop_n N]
                  [--all_duration DURATION] [--checkout/--no-checkout]
```

参数同 `fin_factor`。该循环通过 Bandit/LLM/Random 控制器在"提出因子假设"与"提出模型假设"间自动选择（由 `QLIB_QUANT_ACTION_SELECTION` 配置）。实现：`rdagent/app/qlib_rd_loop/quant.py:main` → `QuantRDLoop`。

#### `rdagent fin_factor_report` — 从研报 PDF 提取因子并实现

```bash
rdagent fin_factor_report --report_folder FOLDER [--path PATH]
                          [--all_duration DURATION] [--checkout/--no-checkout]
```

| 参数 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `--report_folder` | `str?` | None | 研报 PDF 所在目录（递归扫描 `*.pdf`） |
| `--path` | `str?` | None | 恢复 session 路径 |
| `--all_duration` | `str?` | None | 总运行时长限制 |
| `--checkout` | `bool` | True | 同上 |

实现：`rdagent/app/qlib_rd_loop/factor_from_report.py:main` → `FactorReportLoop`。

### 1.3 运维命令

#### `rdagent ui` — 启动 Streamlit 日志 UI

```bash
rdagent ui [--port 19899] [--log_dir ""] [--debug]
```

| 参数 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `--port` | `int` | 19899 | 监听端口 |
| `--log_dir` | `str` | "" | trace 目录 |
| `--debug` | `flag` | False | 调试模式 |

实现：`rdagent/app/cli.py:ui` → `streamlit run rdagent/log/ui/app.py`。

#### `rdagent server_ui` — 启动 Flask 实时日志服务（含任务派发）

```bash
# ⚠️ 必须在 RD-Agent/ 目录下执行（load_dotenv 用相对路径找 .env）
cd ~/projects/multialphaV/RD-Agent
rdagent server_ui [--port 19899]
```

> **cwd 要求**：`cli.py` 的 `load_dotenv(".env")` 用相对路径，cwd 必须是 RD-Agent/ 仓库根目录。否则 `.env` 不会被加载，`CONDA_DEFAULT_ENV` / LLM Key 等配置缺失，task 子进程会因 `CondaConf` 校验失败而秒退。

实现：`rdagent/app/cli.py:server_ui` → `rdagent/log/server/app.py:main`。HTTP 路由见 [§2](#2-http-apiflask-服务)。

#### `rdagent health_check` — 环境自检

```bash
rdagent health_check [--check_env/--no-check-env]
                     [--check_docker/--no-check-docker]
                     [--check_ports/--no-check-ports]
```

| 参数 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `--check_env` | `bool` | True | 检查 LLM/Embedding API 可用性 |
| `--check_docker` | `bool` | True | 拉取并运行 `hello-world` 验证 Docker |
| `--check_ports` | `bool` | True | 检查 UI 端口（19899..19908）占用情况 |

实现：`rdagent/app/utils/health_check.py:health_check`。

#### `rdagent collect_info` — 收集系统信息

```bash
rdagent collect_info
```

依次输出 OS / Python / Docker 容器 / rdagent 及依赖版本信息。实现：`rdagent/app/utils/info.py:collect_info`。

#### `rdagent sota` — 查询 SOTA 实验产物

```bash
rdagent sota --log-path <log目录>
rdagent sota --trace-name <trace名>
rdagent sota --log-path <log目录> --output table
rdagent sota --log-path <log目录> --output code
```

| 参数 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `--log-path` | `str` | None | log 目录路径（含 `__session__/`），如 `log/2026-07-19_03-38-42` |
| `--trace-name` | `str` | None | trace 名称（扫描 `log/` 匹配） |
| `--output` | `str` | json | 输出格式：`json` / `table` / `code` |

两个路径参数二选一。`--log-path` 最可靠（直接指向 session）；`--trace-name` 会扫描 `log/` 匹配。

输出包含：SOTA loop ID、假设、反馈决策、回测指标（IC/年化/回撤）、因子代码（factor.py）、模型代码（model.py）、workspace 路径。

实现：`rdagent/app/cli.py:sota_cli` → `rdagent/log/sota_query.py:query_sota`。

---

## 2. HTTP API（Flask 服务）

启动：`rdagent server_ui --port 19899`（默认 `0.0.0.0:19899`，CORS 全开）。

服务端进程字典 `rdagent_processes: dict[str, RDAgentTask]` 以绝对 trace 路径为 key，记录每个 RD-Agent 子进程及其消息缓冲与 IPC 队列。Trace 根目录由 `UI_SETTING.trace_folder`（默认 `./git_ignore_folder/traces`）决定。

### 2.1 派发任务

`POST /upload`

**表单字段（multipart/form-data）**：

| 字段 | 必填 | 说明 |
|---|---|---|
| `scenario` | ✅ | 场景名，枚举值见下表 |
| `files` | ❌ | 上传的文件列表（多文件，pdf 模式传研报，optimize 模式传 .py 因子代码） |
| `competition` | ❌ | 比赛名（仅 Data Science 用） |
| `loops` | ❌ | 循环数（转 int） |
| `all_duration` | ❌ | 时长（小时数，自动补 `h` 后缀） |
| `description` | ❌ | 自然语言挖掘目标（fin_factor/fin_model/fin_quant 场景）。有值时 task 跳过交互式 init，直接用此描述驱动假设生成（写入 `plan["user_instruction"]`，由 `LLMHypothesisGen.gen` 渲染进 system_prompt）。fin_factor_report 不接收此字段 |

`scenario` 取值与后端 `target_name` 映射（**仅 Qlib 场景在本平台可用**）：

| scenario 字面值 | 后端 target | 说明 |
|---|---|---|
| `Finance Data Building` | `fin_factor` | 因子挖掘 |
| `Finance Model Implementation` | `fin_model` | 模型实现 |
| `Finance Whole Pipeline` | `fin_quant` | 因子+模型联合 |
| `Finance Data Building (Reports)` | `fin_factor_report` | 研报提因子 |

**响应 200**：`{"id": "<scenario>/<trace_name>"}`（`trace_name` 由 `randomname` 生成）。
**响应 400**：`{"error": "Unknown scenario"}` / `{"error": "Invalid file path"}`。

> 文件落盘：`<trace_root>/uploads/<scenario>/<trace_name>/<filename>`（`secure_filename` + 路径越界校验）。
> 任务在独立 multiprocessing 进程中执行，stdout 重定向至 `<trace_root>/<scenario>/<trace_name>.log`。

### 2.2 拉取 trace 消息

`POST /trace`

**JSON 请求体**：

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `id` | `str` | ✅ | trace id，形如 `"Finance Data Building/adoring-einstein"` |
| `all` | `bool` | ❌ | True 时返回全部累积消息 |
| `reset` | `bool` | ❌ | True 时将本 IP 的读取指针归零 |

**响应 200**：消息数组。每条消息结构 `{tag, timestamp, content}`。服务端按客户端 IP 维护读取指针，单次最多返回 1–10 条（随机批量），`all=true` 时一次性返回全部。
**响应 400**：`{"error": "Trace ID is required"}`。

子进程结束后会自动追加 `{tag: "END", content: {error_msg, end_code}}`。

### 2.3 列出历史 trace

`GET /traces`

无参数。响应：`["<scenario>/<trace_name>", ...]`。仅返回含 `*.pkl` 文件且不在 `uploads/` 下的 trace 目录。

### 2.4 下载子进程 stdout 日志

`GET /stdout?id=<trace_id>`

响应：`text/plain` 文件流（`as_attachment`）。404 if 文件不存在；400 if trace_id 非法或路径越界（须位于 `log_folder_path` 之下）。

**路径解析策略**（`_resolve_stdout_path`）：
1. **Running task**：从 `rdagent_processes` 内存查 task.stdout_path（精确路径）
2. **历史 task**（回退）：按 trace_id 拼路径 `<trace_root>/<scenario>/<name>.log`，文件存在则返回

> **HTTP Range 支持**：Flask 3.1.x 的 `send_file(as_attachment=True)` 已默认支持 Range 请求（`conditional=True`）。前端可用 `Range: bytes=<offset>-` 做增量拉取：
> - `206 Partial Content` + `Content-Range: bytes {start}-{end}/{total}` — 正常增量片段
> - `416 Range Not Satisfiable` + `Content-Range: bytes */{total}` — offset 超出文件尾（`total >= offset` 表示文件未增长；`total < offset` 表示文件被截断/重写，应 reset offset=0）
> - `200 OK`（无 Range 处理时）— 全文件回退
>
> multialpha webUI 的 `LogConsole.vue` 使用此机制做实时日志轮询（`fetchStdoutRange`，2s 间隔），传输量保持 1.0× 文件大小（vs 全量轮询的 2288× 放大）。历史 task 的 Range 下载同样支持（Strategy 2 回退）。

### 2.5 用户交互（双向 IPC）

`POST /user_interaction/submit` —— 前端将用户输入回传给正在运行的 rdagent 子进程。

**JSON 请求体**：

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `id` | `str` | ✅ | trace id |
| `payload` | `Any` | ✅ | 任意用户响应内容 |

服务端将 `payload` 放入对应任务的 `user_response_q`（非阻塞 put）。响应 200 `{"status":"success"}` / 400 / 500。

> 仅 `fin_factor` / `fin_model` / `fin_quant` 三个 target 支持用户交互；`fin_factor_report` 在 `_TARGETS_WITHOUT_USER_INTERACTION` 白名单中跳过 IPC。

### 2.6 控制运行中进程

`POST /control`

**JSON 请求体**：`{"id": "<trace_id>", "action": "stop"}`

当前仅支持 `action="stop"`：调用 `task.stop()` 终止子进程并追加 `END` 消息（`end_code=-1`）。响应 200 `{"status":"stopped"}` / 400（未知 action / 无运行进程）。

### 2.7 接收服务端转发消息（内部接口）

`POST /receive`

**JSON 请求体**：单条 `{id, msg}` 或数组 `[{id, msg}, ...]`。服务端把 `msg` 追加到对应 trace 的内存缓冲。响应 200 `{"status":"success"}`。

### 2.8 静态资源与调试

| 路由 | 方法 | 说明 |
|---|---|---|
| `/` | GET | 返回 `static_folder/index.html`（Web UI 入口） |
| `/favicon.ico` | GET | 站点图标 |
| `/<path:fn>` | GET | 静态文件（自动剥离 `static_path` 前缀） |
| `/test` | GET | 调试：返回所有任务的 `{msgs, pointers}` 摘要 |
| `/health` | GET | 轻量健康检查（LLM 配置 / Docker / Qlib 数据 / Conda env / MLflow） |
| `/traces/<trace_name>/sota` | GET | 查询某次实验的 SOTA 产物（假设/指标/因子代码/模型代码） |

#### `GET /health` — 轻量健康检查

不发起 LLM / Docker 实际调用，仅检查配置项和文件存在性。用于任务创建前快速验证环境。

**响应**（200）：

```json
{
  "overall": "pass" | "issues",
  "checks": [
    {"name": "LLM 配置", "icon": "🤖", "status": "pass"|"warn"|"fail",
     "detail": "chat=openai/glm-5.2, embedding=openai/doubao-embedding-vision, key=已设, base=..."},
    {"name": "Docker 环境", "icon": "🐳", "status": "pass"|"fail",
     "detail": "Docker 正常, 镜像: local_qlib:latest"},
    {"name": "Qlib 数据", "icon": "📊", "status": "pass"|"fail",
     "detail": "~/.qlib/qlib_data/cn_data, 6431 天数据 (2000-01-04 ~ 2026-07-20)"},
    {"name": "Conda 环境", "icon": "🐍", "status": "pass"|"warn",
     "detail": "CONDA_DEFAULT_ENV=rdagent4qlib"},
    {"name": "MLflow 配置", "icon": "📈", "status": "pass"|"warn",
     "detail": "MLFLOW_ALLOW_FILE_STORE=true"}
  ]
}
```

webUI TopBar 的"🩺 健康检查"按钮调用此接口，以弹窗展示每项 pass/warn/fail 状态。

#### `GET /traces/<trace_name>/sota` — SOTA 产物查询

```bash
# 通过 trace 名查询（webUI task）
curl http://localhost:19899/traces/Finance%20Data%20Building/plain-transformation/sota

# 通过 log_path 直接指定（CLI task，有 __session__/）
curl "http://localhost:19899/traces/any/sota?log_path=log/2026-07-19_03-38-42"
```

| 参数 | 位置 | 说明 |
|---|---|---|
| `trace_name` | URL path | trace 名（webUI 格式 `scenario/name`）或 log 目录名 |
| `log_path` | query | 直接指定 log 目录（跳过 trace_name 扫描） |

**查询策略**（按优先级）：
1. `log_path` query 参数 → 直接用 `query_sota` 加载 `__session__/`
2. trace_name 是有效 log 目录且含 `__session__/` → 同上
3. `find_session_by_trace_name` 扫描 `log/` → 同上
4. **回退：从消息流提取**（webUI task）—— `__session__/` 不存在时，从 `/trace` 消息流（`task.messages` 或 `read_trace` 从 FileStorage pickle 加载）提取 SOTA：找最后一条 `research.hypothesis` + `feedback.metric` + `evolving.codes` + `feedback.hypothesis_feedback`，组装成结构化结果（`source: "message_stream"`）

> **webUI task vs CLI task 的差异**：CLI task 的 session 存在 `log/<timestamp>/__session__/`（LoopBase.dump 格式），sota 查询直接加载；webUI task 的 FileStorage pickle 在 `<trace_root>/<scenario>/<name>/` 下，但 `__session__/` 不存在（session_folder 用的是默认 `LOG_SETTINGS.trace_path`）。回退策略 4 解决了这个差异。

**响应**（200）：SOTA 实验的结构化信息（loop ID、假设、反馈、指标、因子/模型代码、workspace 路径）。回退模式额外含 `"source": "message_stream"`。
**响应**（404）：session/消息流均不存在或无 SOTA 数据时返回 `{"error": "..."}`。

实现：`rdagent/log/server/app.py:get_sota` → `query_sota`（session 模式）/ `_sota_from_messages`（消息流回退模式）。

### 2.9 核心数据结构

```python
class RDAgentTask:
    # 进程控制
    process: multiprocessing.Process | None
    def start() -> None
    def is_alive() -> bool
    def get_end_code() -> int
    def stop() -> None
    # 消息缓冲（按客户端 IP 维护读取指针）
    messages: list[dict]
    pointers: defaultdict[str, int]
    # IPC 队列（与 rdagent 子进程双向通信）
    user_request_q: Queue   # 子进程 -> 服务端
    user_response_q: Queue  # 服务端 -> 子进程
```

消息标准形态：`{"tag": str, "timestamp": ISO-8601 str, "content": Any}`。

### 2.10 webUI 环节 × 接口 × 产物映射

> 以 **fin_factor（因子挖掘）** 为例，一个完整 loop 的数据流。每行对应一个环节：后端产生的消息 tag、前端展示它的组件、以及前端获取数据依赖的接口。
> 消息通过 `POST /trace` 轮询获取（5s 间隔，增量模式 `all:false`）；首次进入任务详情时全量拉取（`all:true, reset:true`）。

#### 全流程时序

```
任务创建（POST /upload）
  │
  ├─ feedback.config ──────────────────────→ 实验配置表
  │
  └─ Loop N（每轮重复）
       │
       ├─ research.hypothesis ─────────────→ 假设（策略方向）
       ├─ [user_interaction.request] ─────→ 假设评审弹窗（仅交互模式）
       ├─ research.tasks ─────────────────→ 因子定义（name/formula/variables）
       ├─ evolving.codes ─────────────────→ 因子代码（factor.py）
       ├─ evolving.feedbacks ─────────────→ 代码执行反馈
       ├─ feedback.metric ────────────────→ 回测指标（IC/Sharpe/年化/回撤）
       ├─ feedback.return_chart ──────────→ 回测曲线（plotly HTML）
       ├─ [user_interaction.request] ─────→ 反馈决策弹窗（仅交互模式）
       ├─ feedback.hypothesis_feedback ───→ 最终结论（decision/reason）
       └─ token_cost ─────────────────────→ token 用量统计
  │
  └─ END ─────────────────────────────────→ 任务完成
```

> `[user_interaction.request]` 标记的环节**仅在交互模式下产生**。当 `/upload` 带 `description` 时，task 全自动跑（跳过所有交互点），不产生这两条消息。

#### 环节 × tag × 前端组件 × 接口映射表

| 环节 | 消息 tag | content 关键字段 | 前端展示组件 | 依赖接口 |
|---|---|---|---|---|
| **实验配置** | `feedback.config` | `config`（markdown 表格：Dataset/Model/Factors/DataSplit）| TaskBrief（展开后 config chips）| POST /trace |
| **假设生成** | `research.hypothesis` | `hypothesis`, `reason`, `concise_reason`, `concise_justification`, `concise_observation`, `concise_knowledge` | TaskBrief（strategy 文本）+ AgentFlow 研究节点（点击展开假设详情）+ PipelineStages（研究阶段✓）| POST /trace |
| **假设评审交互** | `user_interaction.request` | `{hypothesis, reason, ...}`（Hypothesis 对象字段）| UserInteractionDialog（hypothesis+reason 可编辑 textarea）| POST /user_interaction/submit（回传编辑后的 payload）|
| **因子定义** | `research.tasks` | `[{name, description, formulation, variables}]`（数组）| TaskBrief（初始因子徽章，loop 0 的）+ AgentFlow 设计节点（stat=N 因子）| POST /trace |
| **因子代码** | `evolving.codes` | `[{evo_id, target_task_name, workspace: {filename: code}}]` | ResultWorkspace 代码 tab（文件选择器 + pre 显示 + 复制/下载）+ AgentFlow 编码节点（stat=N 文件）| POST /trace |
| **代码反馈** | `evolving.feedbacks` | `[{evo_id, final_decision, execution, code, return_checking}]` | AgentFlow 编码节点（内部 CoSTEER 演进数据，不直接展示）| POST /trace |
| **回测指标** | `feedback.metric` | `result`（JSON 字符串，含 IC/ICIR/annualized_return/max_drawdown/information_ratio）| MetricsPanel（指标 grid，tone 着色）+ ResultWorkspace 结论 tab（coreMetrics 前 4）+ AgentFlow 回测节点（stat=IC=X.XXX）+ LoopSwitcher 角标（IC 值）| POST /trace |
| **回测曲线** | `feedback.return_chart` | `chart_html`（plotly 内嵌 HTML）| ResultWorkspace 曲线 tab（iframe srcdoc 渲染）| POST /trace |
| **反馈决策交互** | `user_interaction.request` | `{decision, reason, ...}`（HypothesisFeedback 对象字段）| UserInteractionDialog（decision select[true/false] + reason textarea）| POST /user_interaction/submit |
| **最终结论** | `feedback.hypothesis_feedback` | `observations`, `hypothesis_evaluation`, `new_hypothesis`, `decision`, `reason`, `exception` | ResultWorkspace 结论 tab（decision chip + feedbackItems）+ AgentFlow 反馈节点（stat=已采纳/已拒绝）+ MetricsPanel（反馈摘要）| POST /trace |
| **token 用量** | `token_cost` | `model`, `prompt_tokens`, `completion_tokens`, `cost`, `accumulated_cost` | TokenDashboard（总/输入/输出/调用次数；`cost` 的 NaN 已 sanitize 为 0.0）| POST /trace |
| **任务完成** | `END` | `error_msg`, `end_code` | DetailHeader（状态→done）+ 前端停止轮询 | POST /trace（检测到 END 后不再请求）|

#### 贯穿全流程的接口

| 功能 | 接口 | 前端组件 | 说明 |
|---|---|---|---|
| 任务列表 | GET /traces | TaskSidebar + LandingTerminal（ticker） | 首页加载 + 刷新时调用 |
| 实时日志 | GET /stdout（Range）| LogConsole | 2s 轮询，`Range: bytes=<offset>-` 增量拉取（206/416 响应）|
| 任务控制 | POST /control | DetailHeader（停止按钮）| `action=stop` 终止 task |
| 任务创建 | POST /upload | NewTaskDialog | multipart 表单（scenario/loops/description/files）|

#### 交互模式 vs 自动模式（description 的影响）

| 模式 | 触发条件 | user_interaction.request | UserInteractionDialog |
|---|---|---|---|
| **自动模式** | `/upload` 带 `description` | ❌ 不产生（`_set_interactor` 未调用 → 所有交互点跳过）| 不触发 |
| **交互模式** | `/upload` 不带 `description`（或 CLI 带 `user_interaction_queues`）| ✅ 在假设评审 + 反馈决策环节产生 | 弹窗，用户编辑后 POST /user_interaction/submit |

---

## 3. Python 库 API

### 3.1 RDLoop 主循环

`rdagent/components/workflow/rd_loop.py`

```python
class RDLoop(LoopBase):  # metaclass=LoopMeta
    def __init__(self, PROP_SETTING: BasePropSetting) -> None
```

构造时根据 `PROP_SETTING` 通过 `import_class` 动态装配：`scen`、`hypothesis_gen`、`hypothesis2experiment`、`coder`、`runner`、`summarizer`，并初始化 `self.trace = Trace(scen=scen)`、`self.plan = {"features": ALPHA20, "feature_codes": {}}`。

**循环步骤方法**（每个返回 `dict[str, Any]`，作为下一步输入）：

| 方法 | 输入键 | 输出键 | 行为 |
|---|---|---|---|
| `async direct_exp_gen(prev_out)` | — | `propose`, `exp_gen` | 提出假设 + 转实验 + 注入 base_features |
| `coding(prev_out)` | `direct_exp_gen` | — | `coder.develop(exp)` |
| `running(prev_out)` | `coding` | — | `runner.develop(exp)` |
| `feedback(prev_out)` | `running` | — | 异常路径或 `summarizer.generate_feedback(...)` |
| `record(prev_out)` | `feedback` | — | `(exp, feedback)` 入 `trace.hist` |

**基类提供的能力**（继承自 `LoopBase`，`rdagent/utils/workflow/loop.py`）：
- `RDLoop.load(path, checkout=True)` / `.dump(path)` —— session 序列化（pickle）+ 可选 git checkout
- `async run(step_n=None, loop_n=None, all_duration=None)` —— 主调度
- `get_unfinished_loop_cnt() -> int`
- 类常量：`EXCEPTION_KEY = "_EXCEPTION"`、`LOOP_IDX_KEY = "_LOOP_IDX"`

**子类化协议**（本项目用到的四个）：

| 子类 | 文件 | `skip_loop_error` | 关键 override |
|---|---|---|---|
| `FactorRDLoop` | `app/qlib_rd_loop/factor.py` | `(FactorEmptyError, CoderError)` | `running` 校验非空 |
| `ModelRDLoop` | `app/qlib_rd_loop/model.py` | `(ModelEmptyError,)` | 无 |
| `QuantRDLoop` | `app/qlib_rd_loop/quant.py` | `(FactorEmptyError, ModelEmptyError)` | 按 `propose.action ∈ {"factor","model"}` 分支调度 factor/model 子流水线 |
| `FactorReportLoop` | `app/qlib_rd_loop/factor_from_report.py` | 继承 | override `direct_exp_gen`（从 PDF 提取）+ `coding` |

> `QuantRDLoop` 默认 plan：`{"features": ALPHA20, "feature_codes": {}}`（ALPHA20 来自 `rdagent.utils.qlib`）。

### 3.2 核心抽象（rdagent.core）

#### Scenario — `rdagent/core/scenario.py`

```python
class Scenario(ABC):
    @property
    def background(self) -> str: ...                          # abstract
    @property
    def rich_style_description(self) -> str: ...              # abstract
    def get_scenario_all_desc(self, task=None, filtered_tag=None,
                              simple_background=None) -> str: ...   # abstract
    def get_runtime_environment(self) -> str: ...             # abstract
    def get_source_data_desc(self, task=None) -> str: ...     # default ""
    @property
    def source_data(self) -> str: ...                          # = get_source_data_desc()
    @property
    def experiment_setting(self) -> str | None: ...           # default None
```

#### Task / Workspace / Experiment — `rdagent/core/experiment.py`

```python
class AbsTask(ABC):
    def __init__(self, name: str, version: int = 1) -> None
    @abstractmethod
    def get_task_information(self) -> str: ...

class Task(AbsTask):
    def __init__(self, name: str, version: int = 1,
                 description: str = "",
                 user_instructions: UserInstructions | None = None) -> None

class Workspace(ABC, Generic[ASpecificTask, ASpecificFeedback]):
    def __init__(self, target_task: ASpecificTask | None = None) -> None
    target_task / feedback / running_info  # 公开属性
    @abstractmethod
    def execute(self, *args, **kwargs) -> object | None: ...
    @abstractmethod
    def copy(self) -> Workspace: ...
    @property
    @abstractmethod
    def all_codes(self) -> str: ...
    @abstractmethod
    def create_ws_ckp(self) -> None: ...
    @abstractmethod
    def recover_ws_ckp(self) -> None: ...

class FBWorkspace(Workspace):
    DEL_KEY = "__DEL__"
    def __init__(self, *args, **kwargs) -> None
    def inject_files(self, **files: str) -> None         # value=="__DEL__" 删除
    def inject_from_workspace(self, workspace: FBWorkspace) -> None
    def execute(self, env: Env, entry: str) -> str       # 返回 stdout
    def run(self, env: Env, entry: str) -> EnvResult
    def create_ws_ckp(self) -> None                       # zip 进 self.ws_ckp
    def recover_ws_ckp(self) -> None

class Experiment(ABC, Generic[ASpecificTask, ASpecificWSForExperiment, ASpecificWSForSubTasks]):
    def __init__(self, sub_tasks: Sequence[ASpecificTask],
                 based_experiments: Sequence[...] = [],
                 hypothesis: Hypothesis | None = None) -> None
    # 公开属性：sub_tasks / sub_workspace_list / based_experiments /
    #          experiment_workspace / hypothesis / prop_dev_feedback /
    #          running_info / sub_results / local_selection / plan / user_instructions
    @property
    def result(self): ...
```

#### Developer — `rdagent/core/developer.py`

```python
class Developer(ABC, Generic[ASpecificExp]):
    def __init__(self, scen: Scenario) -> None
    self.scen: Scenario
    @abstractmethod
    def develop(self, exp: ASpecificExp) -> ASpecificExp: ...
```

#### Evaluator / Feedback — `rdagent/core/evaluation.py`

```python
class Feedback:
    def is_acceptable(self) -> bool: ...
    def finished(self) -> bool: ...
    def __bool__(self) -> bool: ...      # 默认 True

class Evaluator(ABC):
    @abstractmethod
    def evaluate(self, eo: EvaluableObj) -> Feedback: ...
```

#### Hypothesis / Trace / 提案器 — `rdagent/core/proposal.py`

```python
class Hypothesis:
    def __init__(self, hypothesis: str, reason: str,
                 concise_reason: str, concise_observation: str,
                 concise_justification: str, concise_knowledge: str) -> None

class ExperimentFeedback(Feedback):
    def __init__(self, reason: str, *, decision: bool,
                 code_change_summary: str | None = None,
                 eda_improvement: str | None = None,
                 exception: Exception | None = None) -> None
    @classmethod
    def from_exception(cls, e: Exception) -> ExperimentFeedback

class HypothesisFeedback(ExperimentFeedback):
    def __init__(self, reason: str, decision: bool,
                 code_change_summary: str = "",
                 *, observations: str | None = None,
                 hypothesis_evaluation: str | None = None,
                 new_hypothesis: str | None = None,
                 eda_improvement: str | None = None,
                 acceptable: bool | None = None,
                 exception: Exception | None = None) -> None

class Trace(Generic[ASpecificScen, ASpecificKB]):
    NodeType = tuple[Experiment, ExperimentFeedback]
    NEW_ROOT: tuple = ()
    SEL_LATEST_SOTA: tuple = (-1,)
    def __init__(self, scen: ASpecificScen,
                 knowledge_base: ASpecificKB | None = None) -> None
    # 属性：scen / hist / dag_parent / idx2loop_id /
    #       knowledge_base / current_selection
    def get_sota_hypothesis_and_experiment(self) -> tuple[Hypothesis|None, Experiment|None]
    def get_current_selection(self) -> tuple[int, ...]
    def set_current_selection(self, selection: tuple[int, ...]) -> None
    def get_parent_exps(self, selection=None) -> list[NodeType]
    def exp2idx(self, exp) -> int | list[int] | None
    def idx2exp(self, idx) -> Experiment | list[Experiment]
    def get_children(self, parent_idx=None) -> list[NodeType]
    def get_sota_experiment(self, node_id=None) -> Experiment | None

class HypothesisGen(ABC):
    def __init__(self, scen: Scenario) -> None
    @abstractmethod
    def gen(self, trace: Trace, plan: ExperimentPlan | None = None) -> Hypothesis: ...

class Hypothesis2Experiment(ABC, Generic[ASpecificExp]):
    @abstractmethod
    def convert(self, hypothesis: Hypothesis, trace: Trace) -> ASpecificExp: ...

class Experiment2Feedback(ABC):
    def __init__(self, scen: Scenario) -> None
    @abstractmethod
    def generate_feedback(self, exp: Experiment, trace: Trace,
                          exception: Exception | None = None) -> ExperimentFeedback: ...

class ExpGen(ABC):
    def __init__(self, scen: Scenario) -> None
    @abstractmethod
    def gen(self, trace: Trace) -> Experiment: ...
    async def async_gen(self, trace: Trace, loop: LoopBase) -> Experiment  # 默认实现，受 RD_AGENT_SETTINGS.get_max_parallel() 节流
    def reset(self) -> None  # 默认 no-op

class ExpPlanner(ABC, Generic[ASpecificPlan]):
    def __init__(self, scen: Scenario) -> None
    @abstractmethod
    def plan(self, trace: Trace) -> ASpecificPlan: ...
```

#### KnowledgeBase — `rdagent/core/knowledge_base.py`

```python
class KnowledgeBase:
    def __init__(self, path: str | Path | None = None) -> None
    self.path: Path | None
    def load(self) -> None       # 从 path 反序列化 __dict__
    def dump(self) -> None       # 把 __dict__ pickle 到 path
```

### 3.3 Qlib 场景（rdagent.scenarios.qlib）

#### 四个 Scenario 类（均**无构造参数**）

| 类 | 文件 | 父类 | 说明 |
|---|---|---|---|
| `QlibFactorScenario` | `scenarios/qlib/experiment/factor_experiment.py` | `Scenario` | 因子 R&D |
| `QlibModelScenario` | `scenarios/qlib/experiment/model_experiment.py` | `Scenario` | 模型 R&D（`source_data` 未实现） |
| `QlibQuantScenario` | `scenarios/qlib/experiment/quant_experiment.py` | `Scenario` | 联合，**所有描述方法带 `tag` 参数**（`None`/`"factor"`/`"model"`） |
| `QlibFactorFromReportScenario` | `scenarios/qlib/experiment/factor_from_report_experiment.py` | `QlibFactorScenario` | 仅 override `rich_style_description` |

公开属性 / 方法（前三个一致；QuantScenario 多 `tag` 形参）：

```python
scenario = QlibFactorScenario()
scenario.background               # -> str
scenario.source_data / get_source_data_desc(task=None)
scenario.output_format / interface / simulator
scenario.rich_style_description
scenario.experiment_setting
scenario.get_scenario_all_desc(task=None, filtered_tag=None, simple_background=None)
scenario.get_runtime_environment()

# QuantScenario 独有：
quant.background(tag=None)        # 还可作为方法调用
quant.get_scenario_all_desc(..., action=None)   # 多 action 参数
quant.get_runtime_environment(tag=None)
```

> Scenario 内部从 `FACTOR_PROP_SETTING` / `MODEL_PROP_SETTING` / `QUANT_PROP_SETTING` 读取训练/验证/测试日期区间；切换区间改环境变量即可。

#### Qlib Experiment 类

```python
class QlibFactorExperiment(FactorExperiment[FactorTask, QlibFBWorkspace, FactorFBWorkspace]):
    def __init__(self, *args, **kwargs) -> None
    # 构造时设置 experiment_workspace = QlibFBWorkspace(.../factor_template)
    self.base_features: dict[str, str] = {}        # Qlib operator 形式
    self.base_feature_codes: dict[str, str] = {}
    self.stdout: str = ""

class QlibModelExperiment(ModelExperiment[ModelTask, QlibFBWorkspace, ModelFBWorkspace]):
    def __init__(self, *args, **kwargs) -> None
    self.base_features: dict[str, str] = {}
    self.stdout: str = ""
```

#### QlibFBWorkspace

```python
class QlibFBWorkspace(FBWorkspace):
    def __init__(self, template_folder_path: Path, *args, **kwargs) -> None
    def execute(self, qlib_config_name: str = "conf.yaml",
                run_env: dict = {}, *args, **kwargs) -> tuple
    # 根据 MODEL_COSTEER_SETTINGS.env_type 选 QTDockerEnv 或 QlibCondaEnv
    # 返回 (result_series, stdout_log)
```

#### Developer 子类（CoSTEER / Runner / Feedback）

| 类 | 文件 | 关键方法 |
|---|---|---|
| `QlibFactorCoSTEER`（=`FactorCoSTEER`） | `developer/factor_coder.py` | `develop(exp) -> exp` |
| `QlibModelCoSTEER`（=`ModelCoSTEER`） | `developer/model_coder.py` | `develop(exp) -> exp` |
| `QlibFactorRunner(CachedRunner[QlibFactorExperiment])` | `developer/factor_runner.py` | `develop(exp)`（pickle 缓存）/ `deduplicate_new_factors(SOTA, new)` / `calculate_information_coefficient(...)` |
| `QlibModelRunner(CachedRunner[QlibModelExperiment])` | `developer/model_runner.py` | `develop(exp)`（pickle 缓存） |
| `QlibFactorExperiment2Feedback(Experiment2Feedback)` | `developer/feedback.py` | `generate_feedback(exp, trace) -> HypothesisFeedback` |
| `QlibModelExperiment2Feedback(Experiment2Feedback)` | `developer/feedback.py` | `generate_feedback(exp, trace) -> HypothesisFeedback` |

#### Proposal 子类

| 类 | 文件 |
|---|---|
| `QlibFactorHypothesisGen(FactorHypothesisGen)` / `QlibFactorHypothesis2Experiment(FactorHypothesis2Experiment)` | `proposal/factor_proposal.py` |
| `QlibModelHypothesisGen(ModelHypothesisGen)` / `QlibModelHypothesis2Experiment(ModelHypothesis2Experiment)` | `proposal/model_proposal.py` |
| `QlibQuantHypothesisGen(FactorAndModelHypothesisGen)` | `proposal/quant_proposal.py` — 按 `QUANT_PROP_SETTING.action_selection` (`"bandit"`/`"llm"`/`"random"`) 决定 action |
| `QlibQuantHypothesis(Hypothesis)` | 额外字段 `action ∈ {"factor","model"}` |
| `QuantTrace(Trace)` | 额外持 `EnvController()`（Bandit 控制器） |

#### Factor-From-Report Loader

```python
class FactorExperimentLoaderFromPDFfiles(FactorExperimentLoader):
    def load(self, file_or_folder_path: str) -> QlibFactorExperiment   # developer/factor_experiment_loader/pdf_loader.py
class FactorExperimentLoaderFromDict(FactorExperimentLoader):
    def load(self, factor_dict: dict) -> QlibFactorExperiment          # .../json_loader.py
class FactorExperimentLoaderFromJsonFile: ...
class FactorExperimentLoaderFromJsonString: ...
class FactorTestCaseLoaderFromJsonFile:
    def load(self, json_file_path) -> TestCases
```

### 3.4 LLM 后端（rdagent.oai）

#### 工厂入口 — `rdagent/oai/llm_utils.py`

```python
def get_api_backend(*args, **kwargs) -> BaseAPIBackend    # 按 LLM_SETTINGS.backend 动态导入实例化
APIBackend = get_api_backend                              # 别名，最常用调用方式
```

#### APIBackend 抽象基类 — `rdagent/oai/backend/base.py`

```python
class APIBackend(ABC):
    def __init__(self, use_chat_cache=None, dump_chat_cache=None,
                 use_embedding_cache=None, dump_embedding_cache=None) -> None
    # None 默认值回落到 LLM_SETTINGS 对应字段
    def build_chat_session(conversation_id=None, session_system_prompt=None) -> ChatSession
    def build_messages_and_create_chat_completion(
        user_prompt, system_prompt=None, former_messages=None,
        chat_cache_prefix="", shrink_multiple_break=False, *args, **kwargs) -> str
    def create_embedding(input_content: str | list[str], *args, **kwargs) -> list[float] | list[list[float]]
    def build_messages_and_calculate_token(user_prompt, system_prompt,
                                           former_messages=None, *, shrink_multiple_break=False) -> int
    @property
    def chat_token_limit(self) -> int
    # 抽象方法（子类实现）：
    #   supports_response_schema() -> bool
    #   _calculate_token_from_messages(messages) -> int
    #   _create_embedding_inner_function(input_content_list)
    #   _create_chat_completion_inner_function(messages, response_format=None, *args, **kwargs) -> (str, str|None)
```

默认实现：`LiteLLMAPIBackend`（`rdagent/oai/backend/litellm.py`），通过 `litellm` 转发；旧版 `DeprecBackend`（`deprec.py`）保留兼容。

#### ChatSession（多轮会话）

```python
class ChatSession:
    def __init__(self, api_backend, conversation_id=None, system_prompt=None) -> None
    def build_chat_completion_message(user_prompt) -> list[dict]
    def build_chat_completion(user_prompt, *args, **kwargs) -> str
    def get_conversation_id() -> str
```

#### 缓存层

```python
class SQliteLazyCache(SingletonBaseClass):
    def __init__(self, cache_location: str) -> None
    def chat_get(key) -> str | None / chat_set(key, value)
    def embedding_get(key) / embedding_set(content_to_embedding_dict)
    def message_get(conversation_id) -> list[dict] / message_set(conversation_id, message_value)

class SessionChatHistoryCache(SingletonBaseClass):
    def message_get(conversation_id) / message_set(conversation_id, message_value)
```

### 3.5 日志与 UI（rdagent.log）

#### Logger 单例 — `rdagent/log/__init__.py` + `logger.py`

```python
from rdagent.log import rdagent_logger     # 全局单例

class RDAgentLog(SingletonBaseClass):
    def __init__(self) -> None
    def refresh_storages_from_settings() -> None
    def rebind_console_to_current_streams() -> None
    def set_storages_path(path: str | Path) -> None
    def truncate_storages(time: datetime) -> None
    def log_object(obj: object, *, tag: str = "") -> None
    def info(msg: str, *, tag: str = "", raw: bool = False) -> None
    def warning(msg: str, *, tag: str = "", raw: bool = False) -> None
    def error(msg: str, *, tag: str = "", raw: bool = False) -> None
    @contextmanager
    def tag(self, tag: str) -> Generator[None, None, None]    # 空字符串会抛 ValueError
```

#### 存储抽象 — `rdagent/log/base.py`

```python
@dataclass
class Message:
    tag: str
    level: Literal["DEBUG","INFO","WARNING","ERROR","CRITICAL"]
    timestamp: datetime
    caller: str | None
    pid_trace: str | None
    content: object

class Storage(ABC):
    @abstractmethod
    def log(self, obj, tag="", timestamp=None) -> str | Path: ...
    @abstractmethod
    def iter_msg(self) -> Generator[Message, None, None]: ...
    @abstractmethod
    def truncate(self, time: datetime) -> None: ...

class View(ABC):
    @abstractmethod
    def display(self, s: Storage, watch: bool = False) -> None: ...
```

#### 文件 / Web 存储

```python
class FileStorage(Storage):                   # rdagent/log/storage.py
    def __init__(self, path: str | Path) -> None
    def log(self, obj, tag="", timestamp=None,
            save_type: Literal["json","text","pkl"] = "pkl", **kwargs) -> str | Path
    def iter_msg(self, tag=None, pattern=None) -> Generator[Message, None, None]

class WebStorage(Storage):                    # rdagent/log/ui/storage.py
    def __init__(self, port: int, path: str) -> None
    def log(self, obj, tag, timestamp=None, **kwargs) -> str | Path
```

### 3.6 执行环境（rdagent.utils.env）

```python
# 类型别名
CacheKeyFunc = Callable[[str | Path], list[list[str]]]

@dataclass
class EnvResult:
    def __init__(self, stdout: str, exit_code: int, running_time: float) -> None
    full_stdout / exit_code / running_time
    def update_stdout(stdout) -> None
    @property
    def stdout(self) -> str
    def hash_full_stdout(full_stdout) -> str

class EnvConf(ExtendedBaseSettings):
    default_entry: str                        # 必填
    env_dict: dict = {}
    extra_volumes: dict = {}
    running_timeout_period: int | None = 3600
    redirect_stdout_to_file: bool = False
    enable_cache: bool = True
    retry_count: int = 5
    retry_wait_seconds: int = 10
    exclude_chmod_paths: list[str] = []

class Env(Generic[ASpecificEnvConf]):
    def __init__(self, conf: ASpecificEnvConf) -> None
    def zip_a_folder_into_a_file(folder_path, zip_file_path) -> None
    def unzip_a_file_into_a_folder(zip_file_path, folder_path, files_to_extract=None) -> None
    def check_output(entry=None, local_path=".", env=None, ...) -> str
    def run(entry=None, local_path=".", env=None, ...) -> EnvResult
    def cached_run(...) -> EnvResult
    def dump_python_code_run_and_get_results(code, dump_file_names, local_path, env=None, ...) -> tuple[str, list]
    def refresh_env() -> None

class LocalConf(EnvConf):     bin_path: str = "" ; retry_count: int = 0 ; live_output: bool = True
class LocalEnv(Env[ASpecificLocalConf]): ...

class CondaConf(LocalConf):   conda_env_name: str（必填） ; default_entry: str = "python main.py"
class CondaEnv(LocalEnv): ...
class QlibCondaConf(CondaConf):
    conda_env_name = "rdagent4qlib"
    enable_cache = False
    default_entry = "qrun conf.yaml"
class QlibCondaEnv(LocalEnv[QlibCondaConf]):    # prepare() 自动建 env + 装 qlib/catboost/xgboost/torch

class DockerConf(EnvConf):
    build_from_dockerfile: bool = False
    dockerfile_folder_path: Path | None = None
    image: str                          # 必填
    mount_path: str                     # 必填
    default_entry: str                  # 必填
    extra_volumes: dict = {}
    extra_volume_mode: str = "ro"
    network: str | None = "bridge"
    shm_size: str | None = None
    enable_gpu: bool = True
    mem_limit: str | None = "48g"
    cpu_count: int | None = None
    save_logs_to_file: bool = True
    terminal_tail_lines: int = 20

class DockerEnv(Env[DockerConf]):
    def prepare(self, *args, **kwargs) -> None            # build 或 pull
    def _run(self, entry, local_path=".", env=None,
             running_extra_volume=MappingProxyType({}), **kwargs) -> tuple[str, int]

class QlibDockerConf(DockerConf):                          # env_prefix="QLIB_DOCKER_"
    build_from_dockerfile: bool = True
    image: str = "local_qlib:latest"
    mount_path: str = "/workspace/qlib_workspace/"
    default_entry: str = "qrun conf.yaml"
    # 自动挂载 ~/.qlib/ -> /root/.qlib/
    shm_size: str = "16g"
    enable_cache: bool = False

class QTDockerEnv(DockerEnv):
    def __init__(self, conf: DockerConf = QlibDockerConf()) -> None
    def prepare(self) -> None    # pull image + 缺失时下载 qlib CN 数据

# 模块级工具函数
def extract_dir_name_from_path_config(path_str: str) -> str
def cleanup_container(container, context: str = "") -> None
def normalize_volumes(vols, working_dir: str) -> dict
def pull_image_with_progress(image: str) -> None
```

### 3.7 context7 文档检索 Agent

`rdagent/components/agent/context7/__init__.py`

```python
from rdagent.components.agent.context7 import Agent

class Agent(PAIAgent):
    def __init__(self) -> None    # 自动连 SETTINGS.url（Streamable HTTP MCP）
    def query(self, query: str) -> str
    # 内部把 query 当作 error_message 走 _build_enhanced_query 构造增强检索 prompt
```

- 后端：`pydantic_ai.mcp.MCPServerStreamableHTTP`
- 用途：将"框架 API 不确定 / 第三方库报错"构造为增强查询，调 context7 检索官方文档片段
- 上游部署：`bun run dist/index.js --transport http --port 8124`（见 conf 模块 docstring）

---

## 4. 配置接口

所有配置基于 `ExtendedBaseSettings`（`rdagent/core/conf.py`），`settings_customise_sources` 已改写使父类 env 源可被子类继承。配置项通过 `.env` 或同名环境变量注入。

### 4.1 全局配置 RDAgentSettings

单例 `RD_AGENT_SETTINGS`，**无 env 前缀**（变量名直接生效）。文件：`rdagent/core/conf.py`。

| 字段 | 类型 / 默认 | 说明 |
|---|---|---|
| `workspace_path` | `Path = ./git_ignore_folder/RD-Agent_workspace` | 工作区根目录 |
| `workspace_ckp_size_limit` | `int = 0` | 工作区检查点大小上限 |
| `workspace_ckp_white_list_names` | `list[str]? = None` | 检查点白名单 |
| `multi_proc_n` | `int = 1` | 并行进程数 |
| `cache_with_pickle` | `bool = True` | pickle 缓存总开关 |
| `pickle_cache_folder_path_str` | `str = ./pickle_cache/` | pickle 缓存目录 |
| `use_file_lock` | `bool = True` | 文件锁 |
| `stdout_context_len` | `int = 400` | stdout 上下文长度 |
| `stdout_line_len` | `int = 10000` | stdout 单行长度上限 |
| `enable_mlflow` | `bool = False` | MLflow 跟踪 |
| `step_semaphore` | `int \| dict[str,int] = 1` | 步级并发信号量 |
| `subproc_step` | `bool = False` | 单步用子进程执行 |
| `app_tpl` | `str? = None` | app 模板根 |
| `max_input_duplicate_factor_group` | `int = 300` | 输入因子分组上限 |
| `max_output_duplicate_factor_group` | `int = 20` | 输出因子分组上限 |
| `max_kmeans_group_number` | `int = 40` | KMeans 簇数上限 |
| `initial_fator_library_size` | `int = 20` | 初始因子库大小（注意 typo `fator`） |
| `azure_document_intelligence_key` | `str = ""` | Azure DI key |
| `azure_document_intelligence_endpoint` | `str = ""` | Azure DI endpoint |

方法：`get_max_parallel() -> int`、`is_force_subproc() -> bool`。

### 4.2 LLM 配置

单例 `LLM_SETTINGS`（`LLMSettings`，`rdagent/oai/llm_conf.py`），**无 env 前缀**。子类 `LiteLLMSettings` 加 `LITELLM_` 前缀（`LITELLM_SETTINGS` 单例），即同一字段 `CHAT_MODEL` 也可写作 `LITELLM_CHAT_MODEL`。

#### 后端与模型

| 字段 | 默认 | 说明 |
|---|---|---|
| `BACKEND` | `rdagent.oai.backend.LiteLLMAPIBackend` | LLM 后端类路径 |
| `CHAT_MODEL` | `gpt-4-turbo` | chat 模型名 |
| `EMBEDDING_MODEL` | `text-embedding-3-small` | embedding 模型名 |
| `REASONING_EFFORT` | `None`（`"low"/"medium"/"high"`） | 推理强度 |
| `ENABLE_RESPONSE_SCHEMA` | `True` | 是否启用 JSON Schema 响应 |
| `REASONING_THINK_RM` | `False` | 移除 reasoning think 段 |
| `LOG_LLM_CHAT_CONTENT` | `True` | 记录 LLM 对话内容 |

#### 鉴权与端点

| 字段 | 默认 | 说明 |
|---|---|---|
| `OPENAI_API_KEY` | `""` | OpenAI key |
| `OPENAI_API_BASE` | `""` | OpenAI base url |
| `CHAT_OPENAI_API_KEY` | `None` | chat 专用 key（覆盖 OPENAI_API_KEY） |
| `CHAT_OPENAI_BASE_URL` | `None` | chat 专用 base url |
| `EMBEDDING_OPENAI_API_KEY` | `""` | embedding 专用 key |
| `EMBEDDING_OPENAI_BASE_URL` | `""` | embedding 专用 base url |
| `CHAT_USE_AZURE` / `EMBEDDING_USE_AZURE` | `False` | 启用 Azure OpenAI |
| `CHAT_AZURE_API_BASE` / `CHAT_AZURE_API_VERSION` | `""` | Azure chat |
| `EMBEDDING_AZURE_API_BASE` / `EMBEDDING_AZURE_API_VERSION` | `""` | Azure embedding |
| `CHAT_USE_AZURE_TOKEN_PROVIDER` / `EMBEDDING_USE_AZURE_TOKEN_PROVIDER` | `False` | Azure 托管标识 |
| `MANAGED_IDENTITY_CLIENT_ID` | `None` | 托管标识 client id |
| `CHAT_USE_AZURE_DEEPSEEK` | `False` | Azure DeepSeek |
| `CHAT_AZURE_DEEPSEEK_ENDPOINT` / `CHAT_AZURE_DEEPSEEK_KEY` | `""` | Azure DeepSeek 端点/key |

#### 调用参数

| 字段 | 默认 | 说明 |
|---|---|---|
| `CHAT_MAX_TOKENS` | `None` | 单次最大输出 token |
| `CHAT_TEMPERATURE` | `0.5` | 采样温度 |
| `CHAT_STREAM` | `True` | 流式输出 |
| `CHAT_SEED` | `None` | 随机种子 |
| `CHAT_FREQUENCY_PENALTY` / `CHAT_PRESENCE_PENALTY` | `0.0` | 频率/存在惩罚 |
| `CHAT_TOKEN_LIMIT` | `100000` | 单次对话 token 上限 |
| `DEFAULT_SYSTEM_PROMPT` | `"You are an AI assistant..."` | 默认 system prompt |
| `SYSTEM_PROMPT_ROLE` | `"system"` | system 消息 role 字段 |
| `MAX_PAST_MESSAGE_INCLUDE` | `10` | 多轮历史携带条数 |

#### 重试与缓存

| 字段 | 默认 | 说明 |
|---|---|---|
| `MAX_RETRY` | `10` | 最大重试次数 |
| `RETRY_WAIT_SECONDS` | `1` | 重试间隔 |
| `TIMEOUT_FAIL_LIMIT` | `10` | 超时失败阈值 |
| `VIOLATION_FAIL_LIMIT` | `1` | 协议违规失败阈值 |
| `USE_CHAT_CACHE` / `DUMP_CHAT_CACHE` | `False` | chat 缓存读写 |
| `USE_EMBEDDING_CACHE` / `DUMP_EMBEDDING_CACHE` | `False` | embedding 缓存读写 |
| `PROMPT_CACHE_PATH` | `./prompt_cache.db` | SQLite 缓存路径 |
| `USE_AUTO_CHAT_CACHE_SEED_GEN` | `False` | 自动生成缓存 seed |
| `INIT_CHAT_CACHE_SEED` | `42` | 初始 seed |

#### Embedding 长度

| 字段 | 默认 |
|---|---|
| `EMBEDDING_MAX_STR_NUM` | `50` |
| `EMBEDDING_MAX_LENGTH` | `8192` |

#### 高级 / 旧版

- `CHAT_MODEL_MAP: dict[str, dict[str,str]] = {}` —— 按 logger tag 切换模型
- `USE_LLAMA2`、`LLAMA2_CKPT_DIR`、`LLAMA2_TOKENIZER_PATH`、`LLAMS2_MAX_BATCH_SIZE`
- `USE_GCR_ENDPOINT`、`GCR_ENDPOINT_TYPE`（默认 `"llama2_70b"`）、`GCR_ENDPOINT_TEMPERATURE=0.7`、`GCR_ENDPOINT_TOP_P=0.9`、`GCR_ENDPOINT_DO_SAMPLE=False`、`GCR_ENDPOINT_MAX_TOKEN=100`，以及 `LLAMA2_70B_*` / `LLAMA3_70B_*` / `PHI2_*` / `PHI3_4K_*` / `PHI3_128K_*` 系列端点/key/deployment 字段
- `USE_AZURE`（已弃用）

### 4.3 日志配置

单例 `LOG_SETTINGS`（`LogSettings`，`rdagent/log/conf.py`），**env 前缀 `LOG_`**。

| 字段 | 默认 | 说明 |
|---|---|---|
| `LOG_TRACE_PATH` | `./log/<UTC timestamp>` | trace 落盘根目录 |
| `LOG_FORMAT_CONSOLE` | `None` | 自定义控制台格式 |
| `LOG_UI_SERVER_PORT` | `None` | 启用 WebStorage 推送到该端口 |
| `LOG_STORAGES` | `{}` | 额外 storage 类路径 -> 构造参数 |

方法：`set_ui_server_port(port: int | None) -> None`（开启/关闭 `WebStorage`）。

### 4.4 UI 配置

单例 `UI_SETTING`（`UIBasePropSetting`，`rdagent/log/ui/conf.py`），**env 前缀 `UI_`**。

| 字段 | 默认 | 说明 |
|---|---|---|
| `UI_DEFAULT_LOG_FOLDERS` | `["./log"]` | UI 默认日志目录列表 |
| `UI_BASELINE_RESULT_PATH` | `./baseline.csv` | 基准结果路径 |
| `UI_AIDE_PATH` | `./aide` | AIDE 路径 |
| `UI_AMLT_PATH` | `/data/share_folder_local/amlt` | AMLT 路径 |
| `UI_STATIC_PATH` | `./git_ignore_folder/static` | 静态资源目录 |
| `UI_TRACE_FOLDER` | `./git_ignore_folder/traces` | trace 根目录（HTTP 服务用） |
| `UI_ENABLE_CACHE` | `True` | UI 缓存 |

### 4.5 Qlib 循环配置

四个单例（`rdagent/app/qlib_rd_loop/conf.py`），每个通过对应 env 前缀覆盖：

| 单例 | 类 | env 前缀 |
|---|---|---|
| `FACTOR_PROP_SETTING` | `FactorBasePropSetting` | `QLIB_FACTOR_` |
| `FACTOR_FROM_REPORT_PROP_SETTING` | `FactorFromReportPropSetting` | `QLIB_FACTOR_` |
| `MODEL_PROP_SETTING` | `ModelBasePropSetting` | `QLIB_MODEL_` |
| `QUANT_PROP_SETTING` | `QuantBasePropSetting` | `QLIB_QUANT_` |

通用字段（前缀替换 `<X>` 为 `QLIB_FACTOR_` / `QLIB_MODEL_` / `QLIB_QUANT_`）：

| 字段 | 默认（factor/model） | 说明 |
|---|---|---|
| `<X>SCEN` | `...QlibFactorScenario` / `...QlibModelScenario` / `...QlibQuantScenario` | 场景类路径 |
| `<X>HYPOTHESIS_GEN` | `...QlibFactorHypothesisGen` 等 | 假设生成器类路径 |
| `<X>HYPOTHESIS2EXPERIMENT` | `...QlibFactorHypothesis2Experiment` 等 | 假设转实验类路径 |
| `<X>CODER` | `...QlibFactorCoSTEER` 等 | 编码器类路径 |
| `<X>RUNNER` | `...QlibFactorRunner` 等 | 运行器类路径 |
| `<X>SUMMARIZER` | `...QlibFactorExperiment2Feedback` 等 | 反馈生成器类路径 |
| `<X>EVOLVING_N` | `10` | CoSTEER 演化轮数 |
| `<X>TRAIN_START` / `<X>TRAIN_END` | `2008-01-01` / `2014-12-31` | 训练区间 |
| `<X>VALID_START` / `<X>VALID_END` | `2015-01-01` / `2016-12-31` | 验证区间 |
| `<X>TEST_START` / `<X>TEST_END` | `2017-01-01` / `2020-08-01` | 测试区间 |

`QuantBasePropSetting` 独有：

| 字段 | 默认 | 说明 |
|---|---|---|
| `QLIB_QUANT_ACTION_SELECTION` | `"bandit"` | 动作选择策略，枚举 `bandit` / `llm` / `random` |
| `QLIB_QUANT_QUANT_HYPOTHESIS_GEN` | `...FactorAndModelHypothesisGen` | 联合假设生成器 |
| `QLIB_QUANT_FACTOR_HYPOTHESIS2EXPERIMENT` / `QLIB_QUANT_MODEL_HYPOTHESIS2EXPERIMENT` | — | factor/model 各自 |
| `QLIB_QUANT_FACTOR_CODER` / `QLIB_QUANT_MODEL_CODER` | — | 同上 |
| `QLIB_QUANT_FACTOR_RUNNER` / `QLIB_QUANT_MODEL_RUNNER` | — | 同上 |
| `QLIB_QUANT_FACTOR_SUMMARIZER` / `QLIB_QUANT_MODEL_SUMMARIZER` | — | 同上 |

`FactorFromReportPropSetting` 独有（继承 `FactorBasePropSetting`）：

| 字段 | 默认 | 说明 |
|---|---|---|
| `QLIB_FACTOR_SCEN` | `...QlibFactorFromReportScenario` | 覆盖父类 |
| `QLIB_FACTOR_REPORT_RESULT_JSON_FILE_PATH` | `git_ignore_folder/report_list.json` | 研报清单 JSON |
| `QLIB_FACTOR_MAX_FACTORS_PER_EXP` | `6` | 单实验最多实现因子数 |
| `QLIB_FACTOR_REPORT_LIMIT` | `20` | 最多处理研报数 |

### 4.6 Docker 配置

`QlibDockerConf`（env 前缀 `QLIB_DOCKER_`，`rdagent/utils/env.py`）。

| 字段 | 默认 | 说明 |
|---|---|---|
| `QLIB_DOCKER_BUILD_FROM_DOCKERFILE` | `True` | 是否从 Dockerfile 构建 |
| `QLIB_DOCKER_IMAGE` | `local_qlib:latest` | 镜像名 |
| `QLIB_DOCKER_MOUNT_PATH` | `/workspace/qlib_workspace/` | 容器挂载点 |
| `QLIB_DOCKER_DEFAULT_ENTRY` | `qrun conf.yaml` | 容器入口命令 |
| `QLIB_DOCKER_SHM_SIZE` | `16g` | 共享内存 |
| `QLIB_DOCKER_ENABLE_CACHE` | `False` | Docker 运行缓存 |
| `QLIB_DOCKER_ENABLE_GPU` | `True` | GPU 映射 |
| `QLIB_DOCKER_MEM_LIMIT` | `48g` | 内存上限 |
| `QLIB_DOCKER_NETWORK` | `bridge` | 网络模式 |

> 运行时还会读环境变量 `CUDA_VISIBLE_DEVICES`（`DockerEnv._gpu_kwargs`）。

### 4.7 context7 配置

单例 `SETTINGS`（`Settings`，`rdagent/components/agent/context7/conf.py`），**env 前缀 `CONTEXT7_`**。

| 字段 | 默认 | 说明 |
|---|---|---|
| `CONTEXT7_URL` | `http://localhost:8124/mcp` | context7 MCP 服务地址 |
| `CONTEXT7_TIMEOUT` | `120` | 请求超时（秒） |
| `CONTEXT7_ENABLE_CACHE` | `False` | 是否启用缓存 |

---

## 附录 A：常用导入速查

```python
# CLI 入口
from rdagent.app.qlib_rd_loop.factor import main as fin_factor, FactorRDLoop
from rdagent.app.qlib_rd_loop.model import main as fin_model, ModelRDLoop
from rdagent.app.qlib_rd_loop.quant import main as fin_quant, QuantRDLoop
from rdagent.app.qlib_rd_loop.factor_from_report import main as fin_factor_report, FactorReportLoop

# 主循环 + 核心抽象
from rdagent.components.workflow.rd_loop import RDLoop
from rdagent.core.scenario import Scenario
from rdagent.core.experiment import Task, Workspace, FBWorkspace, Experiment, ExperimentPlan
from rdagent.core.developer import Developer
from rdagent.core.evaluation import Feedback, Evaluator
from rdagent.core.proposal import (Hypothesis, HypothesisFeedback, ExperimentFeedback,
                                   Trace, HypothesisGen, Hypothesis2Experiment,
                                   Experiment2Feedback, ExpGen, ExpPlanner)
from rdagent.core.knowledge_base import KnowledgeBase

# Qlib 场景
from rdagent.scenarios.qlib.experiment.factor_experiment import QlibFactorScenario, QlibFactorExperiment
from rdagent.scenarios.qlib.experiment.model_experiment import QlibModelScenario, QlibModelExperiment
from rdagent.scenarios.qlib.experiment.quant_experiment import QlibQuantScenario
from rdagent.scenarios.qlib.experiment.factor_from_report_experiment import QlibFactorFromReportScenario
from rdagent.scenarios.qlib.experiment.workspace import QlibFBWorkspace

# LLM / 日志 / 环境
from rdagent.oai.llm_utils import APIBackend, get_api_backend
from rdagent.oai.backend import LiteLLMAPIBackend
from rdagent.log import rdagent_logger
from rdagent.utils.env import (Env, EnvResult, LocalEnv, LocalConf,
                               CondaEnv, CondaConf, QlibCondaEnv, QlibCondaConf,
                               DockerEnv, DockerConf, QlibDockerConf, QTDockerEnv,
                               cleanup_container, normalize_volumes)

# context7
from rdagent.components.agent.context7 import Agent as Context7Agent

# 配置单例
from rdagent.core.conf import RD_AGENT_SETTINGS
from rdagent.oai.llm_conf import LLM_SETTINGS, LITELLM_SETTINGS
from rdagent.log.conf import LOG_SETTINGS
from rdagent.log.ui.conf import UI_SETTING
from rdagent.app.qlib_rd_loop.conf import (FACTOR_PROP_SETTING, MODEL_PROP_SETTING,
                                           QUANT_PROP_SETTING, FACTOR_FROM_REPORT_PROP_SETTING)
from rdagent.components.agent.context7.conf import SETTINGS as CONTEXT7_SETTINGS
```

## 附录 B：本平台相对上游的裁剪

依据 CLAUDE.md §0 / §10，本项目删除了以下非 Qlib 场景的接口（不再可用）：

- `rdagent fin_data_ds` / `fin_data_scheduling` / `ml` / `kaggle` / `finetune_model` / `general_model` / `evo_agent` / `insight` 等 CLI 命令
- `scenarios/{data_science,kaggle,finetune,general_model,rl}/` 全部场景类
- `components/coder/{data_science,finetune,rl}/` 全部编码器
- `log/ui/ds_*.py`、`log/mle_summary.py` 等 Data Science 专用 UI/汇总
- HTTP `/upload` 的 `scenario` 字段不再接受 `Data Science` / `Kaggle` / `Finetune` / `General Model` 等值

仅保留四个 Qlib 场景：`fin_factor` / `fin_model` / `fin_quant` / `fin_factor_report`。

---

**版本**：v1.0（2026-07-19）
**适用目录**：`/home/zxh/projects/1.multialphaV/RD-Agent`
**源码依据**：本仓库 `rdagent/` 全量源码（fork 点 `v0.8.0-30-g7cedd780`）
