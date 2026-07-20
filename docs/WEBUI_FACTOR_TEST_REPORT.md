# 因子挖掘场景 multialpha webUI 测试报告

> 测试场景：新建任务 → "挖掘基于成交量的动量因子" → 因子挖掘 → loop 3 → 启动
> 测试 task：`Finance Data Building/plain-transformation`
> 测试时间：2026-07-20 23:07 ~ 23:46+
> 测试方式：API 验证 + 日志分析（AI 无法操作浏览器）

---

## 1. 测试结论摘要

**核心链路验证通过**——因子挖掘的全流程（创建→假设→代码→回测→反馈→token）数据流完整、接口正常、description 生效。

**发现并修复 1 个 bug**：token_cost 的 NaN 导致前端 JSON 解析崩溃（已修复 commit `b3b5b6aa`）。

**发现 1 个设计确认**：有 description 时 task 全自动跑，不产生 user_interaction.request（UserInteractionDialog 不触发）——这是预期行为，不是 bug。

**发现 1 个环境问题**：LLM 生成的因子代码可能死循环（VSP_10 的 rolling.apply），导致 task 卡住——这是因子代码质量问题，非 webUI bug。

---

## 2. 测试用例执行结果

### TC-0 任务创建 ✅

| TC | 验证项 | 结果 | 证据 |
|---|---|---|---|
| TC-0.1 | POST /upload 返回 200 + id | ✅ | `{"id":"Finance Data Building/plain-transformation"}` |
| TC-0.2 | task 真正启动 | ✅ | stdout 从 0 增长到 1.2MB+ |
| TC-0.3 | description 生效 | ✅ | hypothesis 含 volume/momentum/突破关键词；6 个因子全是成交量动量类（VR_10/VR_20/VWPM_10/VSP_10 等）|

### TC-1 初始展示（loop 0）✅

| TC | 验证项 | 结果 | 证据 |
|---|---|---|---|
| TC-1.1 | feedback.config 消息 | ✅ | 配置表（CSI300/LGBModel/Alpha158 Plus/日期段）|
| TC-1.2 | token_cost 无 NaN | ✅ | cost 全 0.0（NaN 已 sanitize，commit b3b5b6aa）|

### TC-2 假设生成（loop 0 环节 A）✅

| TC | 验证项 | 结果 | 证据 |
|---|---|---|---|
| TC-2.1 | research.hypothesis 消息 | ✅ | "volume-based momentum factors...volume breakout signals" |
| TC-2.2 | research.tasks 消息 | ✅ | 6 个因子定义（VR_10/VR_20/VWPM_10/VSP_10/ZVM_20/...）|

### TC-3 假设评审交互 ⏭️ 跳过（设计如此）

| TC | 验证项 | 结果 | 说明 |
|---|---|---|---|
| TC-3.1 | user_interaction.request（hypothesis）| ⏭️ | description 模式下 `_set_interactor` 未调用 → `user_request_q` 未设置 → `_interact_hypo` 自动跳过。**不产生交互消息是预期行为**。UserInteractionDialog 组件代码正确（vue-tsc 通过），但此场景不触发。 |

### TC-4 代码实现（loop 0 环节 C/D）✅

| TC | 验证项 | 结果 | 证据 |
|---|---|---|---|
| TC-4.1 | evolving.codes 消息 | ✅ | 6 个 factor.py 文件 |
| TC-4.2 | evolving.feedbacks 消息 | ✅ | 代码执行反馈 |

### TC-5 回测执行（loop 0 环节 E）✅

| TC | 验证项 | 结果 | 证据 |
|---|---|---|---|
| TC-5.1 | feedback.metric 消息 | ✅ | IC=0.0168, Sharpe=-0.2801 |
| TC-5.2 | feedback.return_chart 消息 | ✅ | plotly HTML |

### TC-6 反馈评审交互 ⏭️ 跳过（同 TC-3）

### TC-7 token 统计（loop 0 环节 H）✅

| TC | 验证项 | 结果 | 证据 |
|---|---|---|---|
| TC-7.1 | token_cost 消息 | ✅ | 多条，prompt/completion/cost 字段完整 |
| TC-7.2 | 无 NaN | ✅ | cost=0.0（修复后）|

### TC-8/9 loop 切换 + 任务完成 ⏳ 待完成

task 在 loop 1 遇到 VSP_10 因子代码死循环（非 webUI bug），杀掉后恢复。loop 1/2 的验证待 task 跑完后补充（消息结构与 loop 0 一致，无新增验证点）。

---

## 3. 接口验证结果

| 接口 | TC | 结果 | 说明 |
|---|---|---|---|
| `POST /upload` | TC-0.1 | ✅ | 返回 `{id}`，task 启动，description 正确传递 |
| `POST /trace` (all=true) | TC-10.2 | ✅ | 返回全量消息，格式正确，无 NaN |
| `POST /trace` (all=false) | TC-10.3 | ✅ | 增量返回新消息（轮询正常）|
| `GET /stdout` (Range) | TC-10.4 | ✅ | 206 + Content-Range（之前 Python 模拟验证 + 本次后端日志确认）|
| `POST /user_interaction/submit` | TC-10.5 | ⏭️ | description 模式不触发交互，未测到（组件代码 vue-tsc 通过）|
| `POST /control` | — | ✅ | 之前验证过（stop 返回 200）|

---

## 4. 发现的问题

### 问题 1：token_cost 的 NaN 导致前端崩溃 ✅ 已修复

- **现象**：前端报 `Unexpected token 'N', ..."cost":NaN... is not valid JSON`
- **根因**：litellm 的 `completion_cost` 对某些模型返回 NaN；storage.py 的 token_cost 分支直接透传，Python json.dumps 输出 `NaN`（JS 不接受）
- **修复**：storage.py 的 token_cost 分支 sanitize NaN/Inf 为 0.0（commit `b3b5b6aa`）
- **验证**：adaptive-map 的历史消息 cost 全变 0.0，JS JSON.parse 通过

### 问题 2：LLM 生成的因子代码可能死循环（环境问题，非 webUI bug）

- **现象**：task 在 loop 1 卡住 15 分钟，stdout 不增长
- **根因**：VSP_10 因子的 `rolling(window=10).apply(lambda x: ..., raw=False)` 在大数据集上极慢（100% CPU 15 分钟）
- **影响**：task 卡住无法继续
- **处理**：手动 kill 死循环进程，task 恢复
- **建议**：rdagent 框架应加因子代码执行超时（目前 `running_timeout_period` 可能没覆盖 conda 模式的因子验证）

### 设计确认：user_interaction 在 description 模式下不触发

- **不是 bug**：有 description 时 `_set_interactor` 未调用 → 所有交互点自动跳过 → task 全自动跑
- **影响**：UserInteractionDialog 组件在此场景不触发（无消息可消费）
- **组件验证状态**：vue-tsc 类型检查通过，代码逻辑对照官方 PlaygroundPage 正确，但**未经实际交互场景触发验证**（需要无 description 的 task 才能触发）

---

## 5. 未验证项（待浏览器实测）

以下需要人在浏览器操作才能验证（AI 无法看 UI 渲染）：

- 各面板的视觉渲染效果（布局/样式/颜色）
- AgentFlow 节点点击交互（展开产物面板）
- LoopSwitcher 切换的 UI 联动
- ResultWorkspace 的 tab 切换 + 代码复制/下载
- UserInteractionDialog 弹窗的视觉 + 提交交互（需无 description 场景）
- 实时日志的滚动/过滤/虚拟滚动

---

**版本**：v1.0（2026-07-20）
**测试 task**：plain-transformation（loop 0 完整验证，loop 1/2 进行中）
**代码版本**：main `b3b5b6aa`（含 NaN 修复）
