# 因子挖掘场景 multialpha webUI 测试用例

> 测试场景：新建任务 → 填"挖掘基于成交量的动量因子" → 因子挖掘 → loop 3 → 启动
> 目标：覆盖因子挖掘全流程的界面展示、交互、接口验证
>
> 维护：测试执行后，每条标注 ✅通过 / ❌失败（附现象+根因）/ ⏭️跳过（附原因）

---

## 因子挖掘全流程 + 依赖接口

### 阶段 0：任务创建
- 前端动作：NewTaskDialog 填描述 + 选场景 + loop数 → POST /upload
- 依赖接口：`POST /upload`（multipart: scenario/loops/description/files）
- 预期：返回 `{id}`，task 启动，前端跳转到任务详情页

### 每个 Loop 的环节（以 loop N 为例）

| 环节 | 后端消息 tag | 前端展示组件 | 依赖接口 |
|---|---|---|---|
| **A. 假设生成** | `research.hypothesis` | TaskBrief（策略描述）+ AgentFlow 研究节点 + PipelineStages 研究 | POST /trace（轮询拿消息）|
| **B. 假设评审交互** | `user_interaction.request`（含 hypothesis 字段）| UserInteractionDialog 弹窗（hypothesis+reason 可编辑）| POST /user_interaction/submit（回传）|
| **C. 任务生成** | `research.tasks` | TaskBrief 初始因子徽章 + AgentFlow 设计节点 | POST /trace |
| **D. 代码实现** | `evolving.codes` | ResultWorkspace 代码 tab + AgentFlow 编码节点 | POST /trace |
| **E. 回测执行** | `feedback.metric` + `feedback.return_chart` | MetricsPanel 指标 + ResultWorkspace 曲线 tab + AgentFlow 回测节点 | POST /trace |
| **F. 反馈评审交互** | `user_interaction.request`（含 decision 字段）| UserInteractionDialog 弹窗（decision select + reason textarea）| POST /user_interaction/submit |
| **G. 反馈结论** | `feedback.hypothesis_feedback` | ResultWorkspace 结论 tab + AgentFlow 反馈节点 | POST /trace |
| **H. token 统计** | `token_cost` | TokenDashboard（累计 prompt/completion/calls）| POST /trace |

### 阶段终：任务完成
- 后端消息：`END`（end_code）
- 前端：DetailHeader 状态变 done、停止按钮消失、轮询停止
- 产物：ResultWorkspace 可下载 JSON（factors/codes/metrics/feedback）

### 跨环节
- **实时日志**：LogConsole（Range 轮询 GET /stdout）
- **loop 切换**：LoopSwitcher（切 selectedLoop，各面板按 loop 过滤）

---

## 测试用例清单

### TC-0 任务创建
- TC-0.1 POST /upload 带 description 返回 200 + id
- TC-0.2 前端跳转到任务详情页（不 404）
- TC-0.3 task 真正启动（stdout 开始增长）
- TC-0.4 description 生效（LLM 生成的 hypothesis 含"成交量/动量"关键词）

### TC-1 初始展示（loop 0 启动后）
- TC-1.1 feedback.config 消息 → TaskBrief 展示实验配置表（Dataset/Model/Factors/DataSplit）
- TC-1.2 DetailHeader 显示任务名 + 运行中状态 + 第0轮
- TC-1.3 PipelineStages 显示 4 阶段（研究 active，其余 idle）
- TC-1.4 LogConsole 展开 → 实时日志滚动（Range 206 响应）

### TC-2 假设生成（loop 0 环节 A）
- TC-2.1 research.hypothesis 消息出现
- TC-2.2 TaskBrief 展示 hypothesis 文本（strategy 字段）
- TC-2.3 AgentFlow 研究节点变 done（可点击）
- TC-2.4 点击研究节点 → 展开假设详情面板

### TC-3 假设评审交互（loop 0 环节 B）—— **核心验证点**
- TC-3.1 user_interaction.request 消息出现（含 hypothesis 字段）
- TC-3.2 UserInteractionDialog 弹窗显示（hypothesis + reason 两个 textarea）
- TC-3.3 编辑后点提交 → POST /user_interaction/submit 200
- TC-3.4 提交后进入 waiting 态（"R&D-Agent 正在生成假设" spinner）
- TC-3.5 task 继续运行（不再阻塞）

### TC-4 任务生成 + 代码实现（loop 0 环节 C/D）
- TC-4.1 research.tasks 消息出现
- TC-4.2 evolving.codes 消息出现
- TC-4.3 ResultWorkspace 代码 tab 显示因子代码（factor.py）
- TC-4.4 AgentFlow 设计+编码节点变 done

### TC-5 回测执行（loop 0 环节 E）
- TC-5.1 feedback.metric 消息出现（IC/Sharpe/年化/回撤）
- TC-5.2 MetricsPanel 展示指标（tone 着色：正数 up/负数 down）
- TC-5.3 feedback.return_chart 消息出现
- TC-5.4 ResultWorkspace 曲线 tab 显示 plotly 图（iframe srcdoc）
- TC-5.5 AgentFlow 回测节点变 done

### TC-6 反馈评审交互（loop 0 环节 F）—— **核心验证点**
- TC-6.1 user_interaction.request 消息出现（含 decision 字段）
- TC-6.2 UserInteractionDialog 弹窗（decision select + reason textarea）
- TC-6.3 提交后 task 继续

### TC-7 反馈结论 + token（loop 0 环节 G/H）
- TC-7.1 feedback.hypothesis_feedback 消息出现（decision true/false）
- TC-7.2 ResultWorkspace 结论 tab 展示 decision chip + 指标 + 反馈理由
- TC-7.3 AgentFlow 反馈节点变 done
- TC-7.4 token_cost 消息出现
- TC-7.5 TokenDashboard 展示 token 用量（无 NaN 错误）

### TC-8 loop 切换（loop 1/2 完成后）
- TC-8.1 LoopSwitcher 显示 3 个 loop 按钮 + IC 角标
- TC-8.2 点击 loop 0 → 各面板回看 loop 0 数据
- TC-8.3 点击 loop 2 → 各面板切换到 loop 2 数据

### TC-9 任务完成
- TC-9.1 END 消息出现
- TC-9.2 DetailHeader 状态变 done
- TC-9.3 轮询停止（POST /trace 不再发）
- TC-9.4 ResultWorkspace 下载 JSON 产物

### TC-10 接口验证（贯穿全流程）
- TC-10.1 GET /traces 列表更新（含新 task）
- TC-10.2 POST /trace 全量（all=true,reset=true）返回所有消息
- TC-10.3 POST /trace 增量（all=false）返回新消息
- TC-10.4 GET /stdout Range（206 + Content-Range）
- TC-10.5 POST /user_interaction/submit（200 + task 继续）
- TC-10.6 POST /control stop（如中途停止）

---

## 测试方法说明

由于 AI 无法操作浏览器，测试采用**后端 API 验证 + 日志分析**方式：
- 用 curl 模拟前端请求，验证每个接口的请求/响应格式
- 监控 task 的 stdout + /trace 消息流，确认每个环节的消息产生
- 对 user_interaction 环节，用 curl 模拟前端提交（POST /user_interaction/submit），验证 task 能继续
- 前端渲染（UI 视觉效果）无法验证，记录为"待浏览器确认"

---

**版本**：v1.0（2026-07-20）
**更新来源**：因子挖掘场景全面测试用例整理
