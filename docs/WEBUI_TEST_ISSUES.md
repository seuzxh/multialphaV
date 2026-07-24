# multialpha webUI 测试问题清单

> 基于 2026-07-20 浏览器实测（后端 19899 + 前端 vite）+ 代码分析 + 老项目（官方 Vue PlaygroundPage）对照。
> 测试目标：保证因子挖掘场景的信息展示、交互、输出无问题。
>
> 维护约定：问题修复后在对应条目标注 ✅ 已修复（commit hash）+ 验证结论。按 `CLAUDE.md` §11 持续更新。

---

## 目录

- [0. 测试环境与已验证项](#0-测试环境与已验证项)
- [1. 已确认正常（核心链路）](#1-已确认正常核心链路)
- [2. 功能缺失（需补齐）](#2-功能缺失需补齐)
- [3. 待验证（被测试中断阻断）](#3-待验证被测试中断阻断)
- [4. 设计观察（非 bug）](#4-设计观察非-bug)
- [5. 测试方法局限](#5-测试方法局限)

---

## 0. 测试环境与已验证项

**环境**：
- 后端：`cd ~/projects/multialphaV/RD-Agent && rdagent server_ui --port 19899`（必须在 RD-Agent/ 目录下执行，确保 `.env` 被加载）。`.env` 里需有 `CONDA_DEFAULT_ENV=rdagent4qlib` + `MLFLOW_ALLOW_FILE_STORE=true`
- 前端：`npm run dev`（vite，端口浮动 8080/8081/8082），proxy 指向本地 19899
- 代码：main 分支 `d0b28772`（含 token_cost 修复 + Range 轮询 + description 修复）

**已验证通过的修复**（见 [WEBUI_API_MIGRATION.md](./WEBUI_API_MIGRATION.md)）：
- ✅ token_cost 消息序列化（`storage.py` 加分支，commit `95f673e2`）
- ✅ 实时日志 Range 增量轮询（`LogConsole.vue`，commit `295c4f54`，1.0× 传输）
- ✅ description → user_instruction 通道（commit `204427e4` + hotfix `d0b28772`）

---

## 1. 已确认正常（核心链路）

| # | 验证项 | 证据 |
|---|---|---|
| 1.1 | 后端 12 路由全部可达 | curl `/test` `/traces` `/trace` `/upload` `/control` `/stdout` 均返回预期状态码 |
| 2.1 | 前端 multialpha.html 加载 | HTTP 200，vite 编译无错，main.ts/rdagent-api.ts/LogConsole.vue 模块均 200 |
| 3.1 | vite proxy → 后端连通 | `/traces` `/trace` 经 proxy 返回 200 + 正确 body |
| 4.1 | **创建带 description 的任务跳过交互** | adaptive-map task：无 `user_interaction.request` 消息产生（description 修复生效）|
| 4.2 | **description 被用作挖掘目标** | adaptive-map stdout 显示 LLM 生成的 hypothesis 内容是"基于成交量的动量因子...放量突破信号"——正是用户填的 description |
| 4.3 | **task 真正启动运行** | adaptive-map stdout 从 2.4KB 增长到 70KB+，进入 hypothesis 生成（LLM 调用 minimax-m3）|
| 5.1 | Range 协议后端支持 | Python 端到端模拟：206/416/200 全部正确，1.0× 传输，offset 追踪准确 |
| 6.1 | token_cost 序列化单元测试 | `_obj_to_json` 对 token_cost tag 正确解析，loop_id/content 完整 |

---

## 2. 功能缺失（需补齐）

### 🔴 问题 2.1：user_interaction 交互闭环完全缺失（严重，但被 description 修复部分规避）

- **现象**：multialpha 前端**完全没有**处理 `user_interaction.request` 消息——`trace-model.ts` 不解析该 tag，没有弹窗/表单 UI，从不调用 `/user_interaction/submit`。
- **根因**：multialpha 设计时未实现此交互（老项目的官方 Vue `PlaygroundPage.vue` 有完整实现：三种 payload 形态 user_instruction/features/hypothesis+reason 各自的表单 + 回传逻辑）。
- **当前规避**：description 修复（commit `204427e4`）让有描述的任务跳过 `_interact_init_params`，避免 init 阶段卡死。**但运行中的反馈交互（`_interact_feedback`，rd_loop.py:154）仍会触发**——如果 task 配置了运行中假设反馈交互，会卡死。
- **影响**：
  - fin_factor/fin_model/fin_quant 用 description 启动时，**init 阶段不卡**（已修复）
  - 但 fin_factor 默认 `FactorBasePropSetting` 是否启用运行中反馈交互，需确认（`_interact_feedback` 的触发条件同 `_interact_init_params`：`hasattr(self, "user_request_q")`）
- **参考实现**：`/home/zxh/projects/1.multialphaV/RD-Agent/web/src/views/PlaygroundPage.vue` 的 `openUserInteraction`（L641）+ `submitUserInteractionForm`（L776）
- **建议**：如需完整交互，移植 PlaygroundPage 的 user_interaction 弹窗到 multialpha。当前用 description 可跑通主链路，暂不阻塞。
- **优先级**：中（主链路已通；仅在需要运行中反馈交互时才阻塞）

### 🟡 问题 2.2：pdf 上传路径（fin_factor_report）的 description 处理未确认

- **现象**：`NewTaskDialog.vue` 的 pdf 模式不收集 description（只上传 PDF + loops），后端 `fin_factor_report` 在 `_TARGETS_WITHOUT_USER_INTERACTION` 白名单里，不进交互。
- **潜在问题**：fin_factor_report 靠 PDF 内容驱动，不需要 description，理论上 OK。但未实测确认。
- **建议**：实测 pdf 上传创建任务，确认不卡、能产生消息。
- **优先级**：低

### 🟡 问题 2.3：TaskSidebar 状态过滤器不完整

- **现象**：TaskSidebar 的状态过滤 chip 只有「全部/完成/运行中」三项（components/TaskSidebar.vue），**无法过滤 idle/error**。
- **影响**：idle（待查看）和 error（异常）的任务只能从「全部」里看，无法快速筛选异常任务。
- **对比**：老项目官方 Vue 无此过滤器，算 multialpha 特有的半成品。
- **建议**：补充 idle/error 过滤 chip，或改为下拉多选。
- **优先级**：低（体验问题，不影响功能）

---

## 3. 待验证（被测试中断阻断）

测试期间 server 被停（task adaptive-map 中断在 hypothesis 生成阶段），以下项需重新测试验证：

| # | 待验证项 | 为什么没验证 | 重测方法 |
|---|---|---|---|
| 3.1 | hypothesis/tasks/codes/metric 消息能否正确渲染到各面板 | adaptive-map 只产生了 feedback.config（1条），hypothesis 还在 LLM 流式生成中就被中断 | 跑完一个完整 loop（loop_n=1，约 10-15 分钟），观察 TaskBrief/AgentFlow/ResultWorkspace/MetricsPanel |
| 3.2 | token_cost 消息是否在运行中进入 /trace 流（前端 TokenDashboard 能否显示） | task 未跑到产生 token_cost 的阶段（LLM 调用后才有） | 同上，跑完 loop 后检查消息流有无 token_cost tag |
| 3.3 | 实时日志（Range 轮询）在真实 running task 上的表现 | 之前用 Python 模拟验证了协议，但未在浏览器实测真实 task 的日志滚动 | 浏览器打开 multialpha.html，跑 task，点开 LogConsole 看日志是否实时滚动 |
| 3.4 | loop 切换（LoopSwitcher）各面板联动 | 需要多 loop 的 task（loop_n>1） | 跑 loop_n=3 的 task，完成后切 loop 看各面板是否按 loop 过滤 |
| 3.5 | 任务完成（END 消息）后看板状态 + 历史回看 | task 未跑完 | 跑完 loop_n=1，看 DetailHeader 状态变 done、停止按钮消失、能否回看 |
| 3.6 | 停止任务（/control stop）的 UI 反馈 | 未实测 | 跑一个 task，点 DetailHeader「停止」，看是否乐观置 done + ElMessage |

---

## 4. 设计观察（非 bug）

这些是 multialpha 的设计选择，不是问题：

| 观察 | 说明 |
|---|---|
| Landing 页隐藏右侧 results 面板 | `goHome()` 显式隐藏，设计如此 |
| 任务完成无浏览器通知（仅 ElMessage toast） | 老项目官方 Vue 用 ElNotification + localStorage；multialpha 简化为 toast。非 bug |
| selectedLoop 一旦用户手动切换后不再自动跟随新 loop | `use-multialpha.ts` poll 里只在 selectedLoop 为 null 时设 max loop，用户选择优先。合理设计 |
| `/trace` 增量返回条数随机 1-10 | 后端 `app.py:306` `random.randint(1,10)`，前端 5s 轮询固定。会导致单次拉取波动，不影响功能 |
| pdf/optimize 路径前端不显示 scenario 选择器 | use-multialpha 硬编码映射（pdf→Reports, optimize→fin_factor），用户无法覆盖。设计如此 |
| multialpha 不用 socketio（直接 polling） | 老项目有 socketio 代码但后端是裸 Flask 从不 emit，实际也是 polling fallback。multialpha 更诚实 |

---

## 5. 测试方法局限

本次测试的局限（影响结论的覆盖度）：

1. **AI 无法直接操作浏览器**：所有"前端渲染"验证基于代码分析 + API 层 curl 模拟，未在真实浏览器看 UI 渲染效果。视觉问题（布局错乱、样式异常、交互卡顿）无法发现。
2. **task 未跑完整 loop**：adaptive-map 在 hypothesis 生成阶段被 server 停止中断，未产生 research.hypothesis/evolving.codes/feedback.metric/token_cost 等业务消息，所以看板各面板的数据渲染**未实测**。
3. **Range 轮询用 Python 模拟**：协议层正确性已证（1.0× 传输、416 边界），但浏览器 EventSource→fetch 切换后的 UI 表现（日志滚动、错误态显示）未实测。

**建议补充测试**：由人在浏览器跑一个完整的 fin_factor task（loop_n=1，填描述），观察 §3 的 6 个待验证项。

---

**版本**：v1.0（2026-07-20）
**适用目录**：`/home/zxh/projects/1.multialphaV`（根仓库）+ `/home/zxh/projects/1.multialphaV/RD-Agent`（代码仓库）
**配套文档**：[WEBUI_API_MIGRATION.md](./WEBUI_API_MIGRATION.md)（接口迁移调研）、[docs/QLIB_SCENARIOS.md](./QLIB_SCENARIOS.md)（Qlib 场景机制）
**更新来源**：2026-07-20 测试沉淀：multialpha webUI 因子挖掘场景测试（核心链路已通：description 跳过交互✅；功能缺口：user_interaction 闭环缺失；6 项待验证因 task 中断未完成）
