# Multica 类项目：全栈开发 + Agent 平台架构二面深挖

> **适用方向：C 全栈开发 + D AI Agent 平台架构。** 根据用户选择调整备考重点。内容基于公开的 [Multica](https://github.com/multica-ai/multica) 项目结构与 [阿里云 AI Agent Handbook](https://github.com/aliyun/ai-agent-handbook) 的工程方法独立编写，**不表示用户实际实现了所有模块**；面试时必须用本人真实职责、PR 和数据替换占位内容。

## 1. 项目介绍：90 秒可复述版本

> 我参与的是一类 Managed Agents 协作平台，目标是把多个 Coding Agent 纳入统一的任务协作流程。它不仅提供聊天入口，更关注 Issue 分配、Agent/Runtime 注册、任务执行、实时进度、权限隔离与交付验收。架构上前端是 Next.js/React，后端使用 Go，数据库是 PostgreSQL；本地 daemon 负责调用 Agent CLI 并回传运行事件。我的实际职责是【填写本人负责的前后端与架构模块】，主要技术挑战是【真实挑战】，通过【实现方式】解决，并用【真实数据/测试证据】验证。对于没有亲自参与的模块，我可以解释架构原理，但不会把它当作个人成果。

**面试官会追问**：你的改动在哪些目录？谁负责调度？任务重试会不会重复写数据？如何证明结果有效？有什么量化指标？

## 2. 架构全景：从用户创建 Issue 到 Agent 交付

```text
Next.js Web / Electron Desktop
   | REST + WebSocket
Go API / Auth / Workspace
   |          \
   |           --> Event stream / Trace / UI status
Task / Issue state + Orchestrator
   |
Queue + Scheduler (capacity, quota, capability)
   |
Runtime registry + lease + heartbeat
   |
Local/Cloud Daemon -> isolated workspace -> Coding Agent CLI
   |                                    |
   +------------ Tool results / Artifact / PR
                  |
          Verify / HITL / Done
                  |
          PostgreSQL + logs + metrics
```

> 这是一种**推荐的系统设计表达**，不是对 Multica 每个内部实现细节的确认。实际讲项目要以代码为准。

## 3. 核心深挖题与回答

### Q1. Orchestrator、Scheduler、Runtime 三者怎么分工？

**标准答案**：Orchestrator 负责业务任务状态、依赖和委派决策；Scheduler 负责把可执行任务匹配到满足能力、配额和健康条件的 Runtime；Runtime/Daemon 负责隔离执行、汇报心跳、事件与结果。分层后模型决策不会直接决定资源权限。

**追问**：谁保证不会重复执行？  
**回答**：数据库事务/唯一约束、任务租约、幂等键与结果对账共同保证；仅靠前端禁用按钮不够。对于跨系统写操作通常不能声称“天然 exactly once”。

### Q2. Agent 执行 30 分钟后 daemon 宕机，如何恢复？

**标准答案**：保存 task_id、状态、阶段、Checkpoint、工具副作用记录和产物引用。心跳超时后标记 suspect，再经租约到期重新分配。恢复先查询外部状态（如 PR 是否已创建），避免重复提交；能安全重试则续跑，否则人工介入。前端要区分“失联”“恢复中”“失败”，而不是一直转圈。

**追问**：为什么单纯重放消息不够？  
**回答**：模型历史不等于外部世界状态，工具可能已经成功执行却丢了回执。

### Q3. 并发调度与公平性怎么做？

**标准答案**：按租户设置并发与 Token/费用预算，调度器检查 Runtime 能力、可用槽位、任务优先级和等待时长。可采用加权公平队列、优先级老化防饥饿；用租约处理 worker 离线。压测需覆盖热点租户和突发流量。

**追问**：用户量十倍增长先扩什么？  
**回答**：先测 API QPS、WS 连接数、任务积压、DB 锁等待、Runtime 槽位与模型限额，找到真正瓶颈再扩容。

### Q4. 如何设计任务状态机和数据库表？

**标准答案**：至少拆分 Issue（业务需求）、Task（执行意图）、Run（一次执行尝试）、Runtime（执行节点）、Event（有序状态事件）、Artifact（产物）、Approval（审批）。状态变更需服务端校验合法迁移，Run 保留失败历史；事件有 sequence 和关联 ID，支持重连后的补拉与审计。

**追问**：为什么不只在 Issue 上放一个 status？  
**回答**：一个 Issue 可以多次重试、委派、等待审批和产生多个产物，单字段无法表达尝试历史和并发子任务。

### Q5. WebSocket 断线、重复、乱序，前后端怎么处理？

**标准答案**：服务器发送带 task_id、event_id/seq、版本的事件；客户端先按序去重，发现缺口后拉取快照或增量补偿。React Query 是服务端数据的权威缓存，WS 触发精确 invalidate；Zustand 只保留筛选、草稿等本地状态。高频进度合并更新，日志虚拟化渲染。

**追问**：事件送达就等于状态已写入吗？  
**回答**：不等于；设计时明确持久化和发布顺序，必要时用事务 Outbox 保证可重放。

### Q6. 跨租户 Agent 权限隔离怎么做？

**标准答案**：每次 API、任务领取、工具执行、检索和 Artifact 读取都绑定 user/tenant/workspace/task 身份；数据库查询加作用域过滤，服务端验证资源归属。Runtime 使用最小权限凭证，Secret 不写入模型上下文或日志；高风险动作必须审批。Prompt 不承担最终授权。

**追问**：如果恶意仓库 README 指示 Agent 上传密钥怎么办？  
**回答**：把仓库内容当不可信输入，限制网络出站、工具授权与凭证作用域；对敏感动作做策略检查和人工确认。

### Q7. Tool Calling 失败如何分类与重试？

**标准答案**：分参数错误、鉴权失败、限流、依赖超时、业务拒绝与未知结果。参数/权限问题不盲重试；429/暂时性错误做带抖动的指数退避；有副作用操作先核对执行结果并使用幂等键。每次尝试形成 Tool Span，统计失败率与 P95。

### Q8. Agent 结果质量怎么量化？

**标准答案**：对 Coding Agent 用测试通过率、PR 可合并率、人工修改量、缺陷回归率、任务验收率；对系统用排队 P95、端到端 P95、失败/人工接管率和单成功任务成本。建立黄金任务集，区分任务完成与文本看起来合理，Badcase 修复后跑回归。

**追问**：如何证明改进有效？  
**回答**：统一任务集与评分口径，对比基线和候选，报告样本量、时间窗、相对变化/百分点及代价。

### Q9. 多 Agent 并行任务如何管理依赖和冲突？

**标准答案**：将子任务建模成 DAG，独立任务并行、依赖任务按验收条件解锁。各 Agent 用隔离工作区/分支；合并时执行测试和冲突检查；产物有版本、owner、审计记录。避免共享完整上下文，按需传递任务契约与产物引用。

### Q10. Agent 的 Token 成本与延迟怎么优化？

**标准答案**：分层模型路由、Context 预算与压缩、检索过滤、工具结果缓存（按权限和版本）、独立子任务并行、取消无效循环。评估必须同时看任务成功率、P95 延迟、成本/成功任务，不能只看 Token 总量。

## 4. 结合截图的字节式追问

### Q11. “只讲方案不讲数据”如何避免？

面试前准备每个项目的三类数字：**效果（验收成功率）/延迟（P95）/成本（每成功任务）**。如未上线，明确说明仅有压测、测试集或预估，并标注实验条件。**禁止捏造生产指标**。

### Q12. 给一个完整企业场景设计 Agent

场景：研发 Issue 自动修复。需求边界 → Issue 接入 → 任务分解 → Runtime 选择 → 沙箱执行 → 测试验证 → PR → 人工审批 → 交付。必须补上错误兜底、审计、隔离、监控、成本预算与回滚/补偿。

### Q13. RAG 召回不准如何排查？

黄金集固定问题与正确证据，逐层排查入库/切分 → Embedding/索引/过滤 → Top-K 召回 → Rerank → Context → 生成。Chunk 不含答案先改分块；证据存在但召回不到查检索；检索到了但答错查生成。

## 5. 算法与工程手写

- LRU：Map + 双向链表，或 JS Map 插入顺序；get/put 平均 O(1)。
- 并发任务队列：固定 worker 池、Promise.race 超时、AbortController 协作取消、错误隔离、重试与幂等。
- 加分题：实现 WebSocket 事件按序去重与断线补偿；实现带租约的任务领取（SQL 事务/唯一约束）。

详见 [Multica 项目型面经](multica-agent-platform-interview.md) 的算法部分。

## 6. 一页纸复盘表（面试前填写）

| 维度 | 你自己的证据 |
| --- | --- |
| 真实负责的前端模块 | 【目录/组件/PR】 |
| 真实负责的后端模块 | 【API/表结构/服务/PR】 |
| 真实参与的 Agent 架构决策 | 【背景/方案/取舍】 |
| 最难的一次故障 | 【现象/定位/修复/验证】 |
| 上线状态与规模 | 【上线环境、任务量、租户数】 |
| 效果、延迟、成本 | 【成功率、P95、单成功任务成本】 |
| 能拿出来讲的代码 | 【具体路径与测试】 |

## 延伸阅读

- [阿里云 Handbook 构建篇](https://github.com/aliyun/ai-agent-handbook/tree/main/02-build)：任务编排、Context、Action Plane。
- [阿里云 Handbook 运行篇](https://github.com/aliyun/ai-agent-handbook/tree/main/03-run)：Runtime、状态、异步与 Multi-Agent。
- [阿里云 Handbook 治理篇](https://github.com/aliyun/ai-agent-handbook/tree/main/04-governance)：可观测与安全。
- [阿里云 Handbook 调优篇](https://github.com/aliyun/ai-agent-handbook/tree/main/05-optimization)：黄金集、轨迹、Badcase。
