# webUI（multialpha）API 迁移调研

> 适用：**multialpha webUI 子应用**适配新 RD-Agent 项目（`/home/zxh/projects/1.multialphaV/RD-Agent`）。
> 本文是**纯调研文档**，不含代码改动。所有结论附 file:/// 绝对路径 + 行号。
>
> 维护约定：webUI 前端或后端 server 的接口契约变更时同步本文档。本文档随讨论持续更新（见 `CLAUDE.md` §11「持续沉淀」）。

---

## 目录

- [0. 一句话结论](#0-一句话结论)
- [1. 背景与范围](#1-背景与范围)
- [2. multialpha 依赖的接口清单](#2-multialpha-依赖的接口清单)
- [3. 老 vs 新项目对比 + 格式校验](#3-老-vs-新项目对比--格式校验)
- [4. 待迁移项](#4-待迁移项)
  - [项 1：实时日志（/logs/sse）— 含性能分析](#项-1实时日志logssse含性能分析)
  - [项 2：token 用量面板（token_cost msg）](#项-2token-用量面板token_cost-msg)
- [5. 待办：老页面接口（已停用）](#5-待办老页面接口已停用)
- [6. 非迁移项（经核查无需迁移）](#6-非迁移项经核查无需迁移)
- [7. 证据链](#7-证据链)
- [8. 结论摘要](#8-结论摘要)

---

## 0. 一句话结论

multialpha 依赖 6 个后端接口，**5 个新项目已具备且格式一致**（直接可用）；**2 个功能缺口都有现成方案**——实时日志用 **Range 增量轮询 /stdout**（后端零改动，传输量 1×，性能优于 SSE）；token 面板断点在 `_obj_to_json` 一处分支（后端加 ~5 行即通）。msg 结构高度兼容，前端 `trace-model.ts` 可直接复用。

---

## 1. 背景与范围

### 1.1 webUI 架构
webUI 是 Vue 3.4 + Vite 8 的**多页应用（MPA）**，入口在 [`web/vite.config.ts`](file:///home/zxh/projects/1.multialphaV/RD-Agent/web/vite.config.ts)：
- `multialpha.html` → `web/src/multialpha/` 子应用（**唯一活动入口**）
- `index.html` → 老页面（合作者 commit `8331bcde` 已注释掉 main 入口，[`vite.config.ts#L20`](file:///home/zxh/projects/1.multialphaV/RD-Agent/web/vite.config.ts#L20) `// main: pathResolve('./index.html')`；老页面文件仍在但不再构建）

multialpha 子应用的 API 调用全部集中在 [`web/src/services/rdagent-api.ts`](file:///home/zxh/projects/1.multialphaV/RD-Agent/web/src/services/rdagent-api.ts)，[`web/src/multialpha/api.ts`](file:///home/zxh/projects/1.multialphaV/RD-Agent/web/src/multialpha/api.ts) 仅 re-export。

### 1.2 后端架构
- **新项目**：纯 Flask，[`rdagent/log/server/app.py`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/log/server/app.py)（12 路由）。无 patch 层、无 socketio、无 SSE/streaming。启动 `rdagent server_ui --port 19899`。
- **老项目**（`/home/zxh/quant_projects/rdagent`）：Flask + `app_patch.py` 补丁层（额外 `/logs/sse`、`/multialpha`、`/kanban` + 覆盖 `/upload`）。webUI 原为老项目前端所写。

### 1.3 本次范围
- **迁移范围**：multialpha 依赖的接口（§2）
- **待办记录**：老页面专用接口（§5，已停用）
- **不写代码**：本文只产出调研结论，是否实现迁移由后续决定

---

## 2. multialpha 依赖的接口清单

multialpha 子应用调用 **6 个后端接口**（全部走相对路径，dev 下由 [`vite.config.ts`](file:///home/zxh/projects/1.multialphaV/RD-Agent/web/vite.config.ts#L49) proxy，prod 下同源 Flask 托管）：

| # | HTTP | URL | 请求参数 | 响应格式 | 前端调用点 | 功能 |
|---|------|-----|----------|----------|------------|------|
| 1 | GET | `/traces` | 无 | `string[]`（如 `"Finance Data Building/abc-def"`） | [`rdagent-api.ts:27`](file:///home/zxh/projects/1.multialphaV/RD-Agent/web/src/services/rdagent-api.ts#L27) → `use-multialpha.ts:41` | trace 列表 |
| 2 | POST | `/trace` | JSON: `{id, all, reset}` | `TraceMessage[]`（`{tag, timestamp?, loop_id?, content}`） | [`rdagent-api.ts:28`](file:///home/zxh/projects/1.multialphaV/RD-Agent/web/src/services/rdagent-api.ts#L28) → `use-multialpha.ts:58,87` | trace 消息流（核心：所有业务数据走这里） |
| 3 | POST | `/upload` | multipart: `scenario`、`loops`、`description`、`files` | `{id?}` 或 `{error?}` | [`rdagent-api.ts:29`](file:///home/zxh/projects/1.multialphaV/RD-Agent/web/src/services/rdagent-api.ts#L29) → `use-multialpha.ts:129` | 上传启动任务 |
| 4 | POST | `/control` | JSON: `{id, action}`（实测仅 `"stop"`） | 任意 JSON | [`rdagent-api.ts:30`](file:///home/zxh/projects/1.multialphaV/RD-Agent/web/src/services/rdagent-api.ts#L30) → `use-multialpha.ts:137` | 控制进程 |
| 5 | GET | `/stdout?id=<traceId>` | query: `id` | `text/plain` 文件流（`as_attachment`） | [`rdagent-api.ts:31`](file:///home/zxh/projects/1.multialphaV/RD-Agent/web/src/services/rdagent-api.ts#L31) → `LogConsole.vue:12` | stdout 下载 |
| 6 | GET (SSE) | `/logs/sse?trace=<traceId>` | query: `trace` | `text/event-stream`，每行 `data: <logline>\n\n` | [`rdagent-api.ts:32`](file:///home/zxh/projects/1.multialphaV/RD-Agent/web/src/services/rdagent-api.ts#L32) → `LogConsole.vue:109` | 实时日志流 |

multialpha 依赖的 msg tag 集合（[`trace-model.ts`](file:///home/zxh/projects/1.multialphaV/RD-Agent/web/src/multialpha/trace-model.ts) 解析）：
`research.hypothesis` / `research.tasks` / `evolving.codes` / `evolving.feedbacks` / `feedback.config` / `feedback.metric` / `feedback.hypothesis_feedback` / `feedback.return_chart` / `token_cost` / `END`

---

## 3. 老 vs 新项目对比 + 格式校验

### 3.1 已具备且格式一致的接口（5 个）

| webUI 接口 | 新项目位置 | 格式校验结论 |
|---|---|---|
| `GET /traces` | [`app.py:367`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/log/server/app.py#L367) | ✅ 一致（`string[]`） |
| `POST /trace` | [`app.py:300`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/log/server/app.py#L300) | ✅ msg 结构一致（`tag`/`timestamp`/`content`）；`END` 消息兼容（新项目 content 多带 `error_msg`/`end_code`，前端只判 `tag==END`） |
| `POST /upload` | [`app.py:375`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/log/server/app.py#L375) | ✅ scenario 兼容——[`NewTaskDialog.vue:6`](file:///home/zxh/projects/1.multialphaV/RD-Agent/web/src/multialpha/components/NewTaskDialog.vue#L6) 用的 3 个值 `Finance Data Building` / `Finance Whole Pipeline` / `Finance Model Implementation` 都在新项目 4 个映射内（第 4 个 `Finance Data Building (Reports)` 未在 UI 暴露但 pdf 上传时触发） |
| `POST /control` | [`app.py:511`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/log/server/app.py#L511) | ✅ 一致——multialpha 只发 `stop`，新项目只支持 `stop`（[`app.py:521`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/log/server/app.py#L521) 硬编码 `if action != "stop": return 400`） |
| `GET /stdout` | [`app.py:349`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/log/server/app.py#L349) | ✅ 一致（`send_file` `as_attachment`，Flask 3.1.3 已默认支持 HTTP Range，见 §4 项 1） |

### 3.2 msg content 格式校验（trace-model.ts 依赖的 9 个 tag）

对比新项目 [`rdagent/log/ui/storage.py`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/log/ui/storage.py) 与老项目 `rdagent/log/ui/storage.py` 的 `_obj_to_json`：

| tag | 新项目 content 字段 | 老项目 content 字段 | 一致性 |
|---|---|---|---|
| `research.hypothesis` | hypothesis, reason, concise_reason, concise_justification, concise_observation, concise_knowledge | 同 | ✅ 结构一致 |
| `research.tasks`（Factor） | name, description, formulation, variables | 同 | ✅ |
| `research.tasks`（Model） | name, description, model_type, formulation, variables | 同 | ✅ |
| `evolving.codes` | evo_id, target_task_name, workspace | 同 | ✅ |
| `evolving.feedbacks` | evo_id, final_decision, execution, code, return_checking | 同 | ✅ |
| `feedback.config` | config | 同 | ✅ |
| `feedback.return_chart` | chart_html | 同 | ✅ |
| `feedback.metric` | result | 同 | ✅ |
| `feedback.hypothesis_feedback` | observations, hypothesis_evaluation, new_hypothesis, decision, reason, exception | 同 | ✅ |
| `END` | error_msg, end_code | 同 | ✅ |

**结论**：9 个 tag 的 content 结构全部一致，字段用英文 key（`trace-model.ts` 不依赖中文），前端解析逻辑可直接复用。

---

## 4. 待迁移项

### 项 1：实时日志（/logs/sse）— 含性能分析

#### 现状
[`LogConsole.vue:109`](file:///home/zxh/projects/1.multialphaV/RD-Agent/web/src/multialpha/components/LogConsole.vue#L109) 用 `new EventSource(logStreamUrl(traceId))` 连接 `GET /logs/sse?trace=<id>`。新项目后端：
- ❌ **无 `/logs/sse` 路由**（grep `sse`/`EventStream`/`text/event-stream`/`stream_with_context`/`Response(` 在 `rdagent/log/` 全 0 命中）
- ❌ 无 socketio（老 RD-Agent 的 `join`/`leave`/`new_msg` handler 未保留，新项目 import 只有 `Flask` + `flask_cors`）
- ❌ 无任何 streaming 机制（12 路由全是普通 JSON / `send_file`）

#### 新项目现成方案分析

| 方案 | 可行性 | 说明 |
|---|---|---|
| 移植老 SSE | ❌ 需后端 ~90 行 | 老项目 [`app_patch.py:169-256`](file:///home/zxh/quant_projects/rdagent/app_patch.py#L169) 实现，但新项目无 patch 层 |
| socketio | ❌ 无 | 框架未保留 |
| `/trace` 轮询复用 | ❌ 无日志 tag | `/trace` 的 msg 只有 10 个结构化业务 tag，无"日志行"类 tag，轮询拿不到日志 |
| **Range 增量轮询 /stdout** | ✅ **推荐** | Flask 3.1.3 `send_file(as_attachment=True)` 已默认支持 Range（`conditional=True`），**后端零改动** |

#### 性能分析（基于真实历史日志实测）

**日志大小量级**（样本：`/home/zxh/quant_projects/rdagent/traces/*.stdout`）：
- 单次 loop（loop_n=1）：0.5-0.7 MB，3000-6000 行，10-15 分钟
- 完整 loop（loop_n=10）：**4.7-7.0 MB**，3.6-5.7 万行，1.8-2.5 小时
- 最大样本：11.6 MB，7 万行

**日志增长特征**（实测 `nice-spear.stdout` 7MB/2.5h/56734行）：
- **强烈突发式**：97.8% 的墙上时间是 >1s 静默（等 LLM 响应），LLM 一响应就刷几千行
- 平均速率仅 0.75 KB/s；最大 gap 900s（15 分钟长 LLM 调用）
- 含义：大部分轮询落在静默期，文件未增长

**三方案传输量对比**（7MB 日志 / 2s 轮询 / 跑满 2.5h）：

| 方案 | 总传输量 | 放大倍数 | 后端改动 | 前端改动 | 其他代价 |
|---|---|---|---|---|---|
| 老 SSE（0.5s 增量推送） | ~7.4 MB | 1.06× | 移植 ~90 行 | EventSource 已有 | **每 viewer 占一个 worker 线程整个任务生命周期（2.5h）**，Flask dev 单线程会被一个 viewer 卡死 |
| 轮询全量下载 | **15.0 GB** | **2288×** ❌ | 0 | ~20 行 | 文件从 0 增长到 7MB 的积分效应；多 viewer 打满带宽；服务端重复读全文件 + 算 ETag |
| **Range 增量轮询** | **7.0 MB** | **1.0×** ✅ | **0** | **~20 行** | 无 |

**瓶颈严重性**：

| 文件大小 | 全量轮询（2s）总传输 | Range 增量总传输 |
|---|---|---|
| 0.5 MB（loop_n=1） | ~1.5 GB | 0.5 MB |
| 5 MB | 4.4 GB | 5 MB |
| **7 MB（实测 loop_n=10）** | **15 GB** | **7 MB** |
| 11.6 MB（实测最大） | 50 GB | 11.6 MB |
| 20 MB（假设） | 35 GB | 20 MB |

**结论**：全量下载方案不可取（2288× 放大）；SSE 传输虽低但占 worker 线程；**Range 增量轮询是唯一兼顾性能和改动量的方案**——传输量与文件同阶（1×），后端零改动。

#### 格式校验（Range 协议）
实测 Flask 3.1.3 的 `send_file(path, as_attachment=True)` 对 Range 请求的响应：

| 请求 | 状态码 | Content-Range | body |
|---|---|---|---|
| 普通 GET | 200 | — | 全文件 |
| `Range: bytes=0-99` | **206** | `bytes 0-99/{total}` | 100 字节 |
| `Range: bytes=500000-` | **206** | `bytes 500000-{end}/{total}` | 增量片段 |
| `Range: bytes={EOF}-`（超文件尾） | **416** | `bytes */{total}` | 错误体 |

#### 迁移建议
- **后端**：零改动（`send_file` 已支持 Range）
- **前端**：改 [`LogConsole.vue`](file:///home/zxh/projects/1.multialphaV/RD-Agent/web/src/multialpha/components/LogConsole.vue) 的 `connect()`（约 20 行）：
  - 首次：普通 `GET /stdout?id=`，从响应头 `Content-Range: bytes 0-N/{total}` 拿 total，记 `offset = N+1`
  - 后续：`fetch(stdoutUrl, { headers: { Range: 'bytes={offset}-' } })`，拿 206，解析 `Content-Range` 更新 offset
  - 边界：416 → offset 重置 0（文件被截断/重写）；200 → 全量回退
  - 轮询间隔 2s（静默期 Range 请求返回 0 字节，开销极小）
- **优先级**：高（multialpha 实时日志面板当前完全不可用）

---

### 项 2：token 用量面板（token_cost msg）

#### 现状
[`trace-model.ts:30`](file:///home/zxh/projects/1.multialphaV/RD-Agent/web/src/multialpha/trace-model.ts#L30) filter `tag==='token_cost'`，[`TokenDashboard.vue`](file:///home/zxh/projects/1.multialphaV/RD-Agent/web/src/multialpha/components/TokenDashboard.vue) 渲染 token 用量。前端字段映射有降级兜底：`Number(token.accumulated_prompt_tokens||token.prompt_tokens||0)`。

#### 关键发现：数据已存在，断点在序列化层

**数据流 6 步定位**：

| 步骤 | 状态 | 位置 |
|---|---|---|
| 1. 产生 | ✅ | [`litellm.py:214-223`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/oai/backend/litellm.py#L214) 每次 LLM 调用发 `{model, prompt_tokens, completion_tokens, cost, accumulated_cost}` |
| 2. 扇出 | ✅ | [`logger.py:132-136`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/log/logger.py#L132) 分发给 FileStorage + WebStorage |
| 3. 落盘 | ✅ | [`storage.py:38-66`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/log/storage.py#L38) FileStorage 无差别 pickle（**磁盘实证**：`log/.../token_cost/*.pkl`，内容 `{'model':'openai/glm-5.2','prompt_tokens':24,'completion_tokens':1,'cost':nan,'accumulated_cost':0.0}`） |
| 4. 实时推送 | ❌ **断** | [`ui/storage.py:66-279`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/log/ui/storage.py#L66) `_obj_to_json` 无 `token_cost` 分支 → 返回 `{}` → [`L41`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/log/ui/storage.py#L41) `if not data: return "skipped"` → 不 append msgs、不 POST `/receive` |
| 5. 历史回放 | ❌ **同样断** | [`app.py:240-256`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/log/server/app.py#L240) `read_trace` 复用同一个 `_obj_to_json`，token_cost 同样被过滤 |
| 6. 前端 | ❌ 空 | `trace-model.ts:30` filter 永远空数组 → token 面板全 0 |

**关键修正**：之前怀疑"老项目有 token_cost、新项目没有"。实测**老项目 `storage.py` 与新项目逐字相同**（diff 零差异，都只有 279 行，都无 token_cost 分支）。所以这不是"老有新无"，而是**两边都没接通，multialpha 是第一个想消费 token_cost 的前端**——它是一个"后端一直在发、中间层一直丢弃"的断链。

#### 格式校验
- litellm 发：`{prompt_tokens, completion_tokens, cost, accumulated_cost}`
- multialpha 要：`{accumulated_prompt_tokens, accumulated_completion_tokens, total_tokens, call_count}`
- 前端已有降级：单次 `prompt_tokens` 可显示（非累计）；`call_count` 用 `tokenMessages.length`（消息条数）兜底

#### 迁移建议
- **后端**：[`ui/storage.py`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/log/ui/storage.py) 的 `_obj_to_json` 加一个 `token_cost` 分支（约 5 行），直接透传 dict + 补 loop_id。**一处改动同时修实时推送（步骤4）和历史回放（步骤5）**，因为 `read_trace` 复用同一 `_obj_to_json`
- 用 litellm 原生字段（`prompt_tokens`/`completion_tokens`）+ 前端降级即可工作；若要显示累计值，在该分支内累加后塞 `accumulated_*` 字段
- **优先级**：中（token 面板不显示不影响核心功能，但浪费了已落盘的数据）

---

## 5. 待办：老页面接口（已停用）

合作者 commit [`8331bcde`](file:///home/zxh/projects/1.multialphaV/RD-Agent/web/vite.config.ts#L20) 已注释掉 vite main 入口，**老页面停用**（文件仍在但不再构建）。老页面专用接口，本次不分析、不迁移：

| 接口 | 调用点 | 状态 |
|---|---|---|
| `POST /user_interaction/submit` | [`utils/api.js:10`](file:///home/zxh/projects/1.multialphaV/RD-Agent/web/src/utils/api.js#L10)（仅老页面 Playground*.vue） | 新项目后端已有（[`app.py:488`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/log/server/app.py#L488)），格式一致。老页面停用后无消费者。**待评估**：若彻底删除老页面文件则此接口无消费者；若保留则接口已就绪 |
| vite dev proxy 补 `/user_interaction` 前缀 | [`vite.config.ts:49-56`](file:///home/zxh/projects/1.multialphaV/RD-Agent/web/vite.config.ts#L49) | 当前 proxy 未含 `/user_interaction`（含 `/traces`/`/trace`/`/upload`/`/control`/`/logs`/`/stdout` 6 前缀），老页面 dev 下会 404。随老页面停用不再紧迫 |

---

## 6. 非迁移项（经核查无需迁移）

### 6.1 中文翻译
- 新项目 `rdagent/` 全量 grep `translate`/`中文`/`glm-4-flash`/`i18n`：**0 命中**
- 老项目可见源码（`app_patch.py`/`webui_main.py`/`storage.py`）：**同样 0 命中**——之前怀疑的翻译机制在当前代码库不存在（可能是更早版本或归档代码）
- multialpha 字段映射用英文 key，不依赖翻译
- **结论**：无可迁之物，不作迁移。若需中文 UI 属新功能（建议前端 vue-i18n，不在后端引入翻译调用）

### 6.2 multialpha 专用后端接口
- 老项目 `/multialpha`、`/kanban`（[`app_patch.py:361-368`](file:///home/zxh/quant_projects/rdagent/app_patch.py#L361)）只是 `send_from_directory` serve 单文件 HTML
- 新架构由 vite 多入口（[`vite.config.ts`](file:///home/zxh/projects/1.multialphaV/RD-Agent/web/vite.config.ts) 的 `rollupOptions.input`）+ 前端路由接管，**无需后端路由**
- multialpha 前端复用通用 RD-Agent 接口，无 `/multialpha/*` 数据接口需求

---

## 7. 证据链

### multialpha 前端
- API 定义：[`web/src/services/rdagent-api.ts`](file:///home/zxh/projects/1.multialphaV/RD-Agent/web/src/services/rdagent-api.ts)（6 个接口，行 27-32）
- msg 解析：[`web/src/multialpha/trace-model.ts`](file:///home/zxh/projects/1.multialphaV/RD-Agent/web/src/multialpha/trace-model.ts)（9 tag + token_cost）
- SSE 消费：[`web/src/multialpha/components/LogConsole.vue:109`](file:///home/zxh/projects/1.multialphaV/RD-Agent/web/src/multialpha/components/LogConsole.vue#L109)
- scenario 值：[`web/src/multialpha/components/NewTaskDialog.vue:6`](file:///home/zxh/projects/1.multialphaV/RD-Agent/web/src/multialpha/components/NewTaskDialog.vue#L6)

### 新项目后端
- 主 server：[`rdagent/log/server/app.py`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/log/server/app.py)（12 路由：`/trace`:300, `/stdout`:349, `/traces`:367, `/upload`:375, `/receive`:468, `/user_interaction/submit`:488, `/control`:511, `/test`:551, `/traces/<path>/sota`:559）
- msg 序列化（token_cost 断点）：[`rdagent/log/ui/storage.py:66-279`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/log/ui/storage.py#L66)（`_obj_to_json`，L41 skip 逻辑）
- token 产生：[`rdagent/oai/backend/litellm.py:214-223`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/oai/backend/litellm.py#L214)
- stdout 重定向（loguru 实时写文件）：[`app.py:126-127`](file:///home/zxh/projects/1.multialphaV/RD-Agent/rdagent/log/server/app.py#L126)
- token_cost 落盘实证：`log/2026-07-19_02-27-16-967110/token_cost/1773521/2026-07-19_02-27-27-527562.pkl`

### 老项目
- SSE 实现：[`app_patch.py:169-256`](file:///home/zxh/quant_projects/rdagent/app_patch.py#L169)（`time.sleep(0.5)` 在 L250）
- `/multialpha`、`/kanban`：[`app_patch.py:361-368`](file:///home/zxh/quant_projects/rdagent/app_patch.py#L361)

### 性能实测样本
- loop_n=10 标杆：`/home/zxh/quant_projects/rdagent/traces/Finance Whole Pipeline/nice-spear.stdout`（7.0 MB / 56734 行 / 2.54h）
- 最大样本：`/home/zxh/quant_projects/rdagent/traces/Finance Data Building (Reports)/wise-rank.stdout`（11.6 MB / 70180 行）

---

## 8. 结论摘要

| 维度 | 结论 |
|---|---|
| multialpha 依赖接口 | 6 个，**5 个新项目已具备且格式一致**（直接可用） |
| 实时日志缺口 | 后端无 `/logs/sse`，但 **Range 增量轮询 `/stdout` 可替代**（后端零改动，传输量 1×，性能优于 SSE，无 worker 占用问题） |
| token 面板缺口 | 数据已落盘，断点在 `_obj_to_json` 一处分支（后端加 ~5 行即通，同时修实时推送 + 历史回放） |
| msg 结构 | 高度兼容，`trace-model.ts` 可直接复用（9 个 tag 字段全一致，英文 key） |
| 老页面 | 已停用（commit 8331bcde），专用接口记为待办 |
| 中文翻译 | 新老项目均无此机制，无可迁之物 |

**两个缺口的推荐方案都不需要移植老代码**——实时日志复用新项目已有的 `/stdout`（+ Range），token 面板补一个序列化分支。改动量小、性能优、向后兼容。

---

**版本**：v1.0（2026-07-20）
**适用目录**：`/home/zxh/projects/1.multialphaV`（根仓库）+ `/home/zxh/projects/1.multialphaV/RD-Agent`（代码仓库）
**配套文档**：[CLAUDE.md](file:///home/zxh/projects/1.multialphaV/CLAUDE.md)（开发行为约束）、[docs/QLIB_SCENARIOS.md](file:///home/zxh/projects/1.multialphaV/docs/QLIB_SCENARIOS.md)（Qlib 场景机制）、[docs/reference/API.md](file:///home/zxh/projects/1.multialphaV/docs/reference/API.md)（新项目接口参考）
**更新来源**：2026-07-20 调研沉淀：webUI multialpha 子应用适配新 RD-Agent 项目的接口对比 + 迁移方案（含 Range 增量轮询性能分析、token_cost 断链 6 步定位、老页面停用状态确认）
