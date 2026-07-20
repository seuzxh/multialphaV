# multialpha webUI 性能审计报告

> 2026-07-21 审计，聚焦前端加载/渲染性能 + 多租户（10人+）场景。
> 审计后逐项优化，每项完成后标注 ✅ + commit hash。

---

## 问题清单（按严重程度）

### P0 — 严重（影响可用性 / 正确性 bug）

#### P0-1: `/trace` 的 `random.randint(1,10)` + pointer 用 user_ip（正确性 bug）
- **问题**：`app.py:392` `msg_num = random.randint(1,10)` 随机返回 1-10 条；`app.py:421-429` pointer 按 `request.remote_addr` 划分游标
- **影响**：
  - 高产出阶段 50+ 条消息需 5 次 poll（25s）才能拉完
  - **NAT 用户共享 IP → 共享 pointer → 后到的用户丢消息**（正确性 bug）
- **修复**：删 random.randint（全量返回增量）；pointer 改为前端传 cursor

#### P0-2: Flask dev server 不支持并发，10 用户可能卡死
- **问题**：`app.py:814` 用 `app.run()`，非生产级 WSGI
- **影响**：10 用户 × 5s/trace + 2s/stdout ≈ 4-6 req/s，dev server 延迟显著
- **建议**：生产部署改 gunicorn + gevent（文档建议，不改代码）

### P1 — 高（明显卡顿）

#### P1-3: `buildTraceView` 全量重算 + `loopMetrics` 嵌套循环
- **问题**：`trace-model.ts:6` `latest()` 每次 `[...messages].reverse().find()`；`trace-model.ts:35` `loopMetrics` 对每个 loop 全扫 messages
- **影响**：200 messages + 10 loops → 2000 次比较 + 10 次数组反转
- **修复**：latest() 改 Map 索引；loopMetrics 改单次遍历

#### P1-4: `messages.value = [...messages.value, ...updates]` 全数组重建
- **问题**：每次 poll 新建整个数组，触发所有 computed 失效
- **影响**：叠加 P1-3，每次 poll 全树 re-render
- **修复**：改 `push`（Vue ref 响应式支持）

#### P1-5: `AgentFlow.vue` 5 个 `latest(tag)` 重复扫描
- **问题**：`AgentFlow.vue:20` 在 computed 里 5 次 `[...messages].reverse()`
- **影响**：5 × 数组复制 + 反转
- **修复**：复用 view 已解析的字段

#### P1-6: iframe plotly chart 无 hash 约束，每次 poll 可能重建
- **问题**：`ResultWorkspace.vue:10` `:srcdoc="chartHtml"`，chartHtml 引用变即重建 iframe
- **影响**：每次 poll 50-150ms 重绘
- **修复**：加 chartVersion（内容 hash 变化才递增）

### P2 — 中（资源浪费）

#### P2-7: LogConsole + use-multialpha 双轮询无合并
- **问题**：2s stdout + 5s trace = 42 req/min
- **修复**：LogConsole 连续 N 次 416 退避到 8s

#### P2-8: 多 tab 打开同一 task = N 倍轮询
- **建议**：BroadcastChannel 选举主 tab（后续优化）

#### P2-9: LogConsole filtered 每次过滤 5000 行 + toLowerCase
- **建议**：预计算 lowercase 缓存（后续优化）

#### P2-12: LRU 缓存上限 2 太小
- **修复**：提到 5

### P3 — 低（首屏加载）

#### P3-15: katex 全量加载 + element-plus CSS 全量 + 无代码分割
- **建议**：katex 异步加载 + vite manualChunks（后续优化）

### P4 — 多租户资源

#### P4-17: Docker 并发无信号量 + auto_remove 关闭
- **建议**：加 task 并发限制 + cpu_count 显式设置

---

## 优化进度

| 编号 | 问题 | 状态 | commit |
|---|---|---|---|
| P0-1 | random.randint + pointer user_ip → cursor 模式 | ✅ 已修 | `bdd45209` |
| P1-3 | trace-model latest/loopMetrics 全量重算 | ✅ 已修 | `117081a1` |
| P1-4 | messages 全数组重建 → push | ✅ 已修 | `bdd45209` |
| P1-5 | AgentFlow 5× latest | ✅ 间接优化（latest 改线性扫描） | `117081a1` |
| P1-6 | iframe plotly 重建 → stableChartHtml | ✅ 已修 | `117081a1` |
| P2-7 | LogConsole 轮询退避 2s→8s | ✅ 已修 | `117081a1` |
| P2-12 | LRU 上限 2→5 | ✅ 已修 | `bdd45209` |
| P0-2 | Flask dev server → gunicorn | 📋 文档建议（部署时改） | — |
| P2-8 | 多 tab N 倍轮询 → BroadcastChannel | 📋 后续优化 | — |
| P3-15 | katex/element-plus 懒加载 | 📋 后续优化 | — |
| P4-17 | Docker 并发信号量 | 📋 后续优化 | — |
