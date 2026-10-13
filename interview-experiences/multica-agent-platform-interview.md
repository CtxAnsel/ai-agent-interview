# Multica 类项目：AI Agent 二面项目深挖与系统设计

> **资料说明**：依据 [Multica 开源仓库](https://github.com/multica-ai/multica)、[AGENTS.md](https://github.com/multica-ai/multica/blob/main/AGENTS.md)、[CLI/Daemon 文档](https://github.com/multica-ai/multica/blob/main/CLI_AND_DAEMON.md)、[阿里云 AI Agent Handbook](https://github.com/aliyun/ai-agent-handbook) 和公开面经图片独立整理。题目是**模拟面试训练**，不代表本人真实工作经历。用户仓库 `CtxAnsel/multica` 是上游 `multica-ai/multica` 的 Fork，不能仅凭 Fork 推断个人贡献或线上指标。

## 一、项目速览（代码/文档可核实）

Multica 是开源 Managed Agents 协作平台：在看板分配 Issue 给 Coding Agent，通过 Runtime/daemon 执行，追踪进度与产物，沉淀 Skills。支持多个工作区及 Claude Code、Codex 等执行器。

技术栈：Next.js 16、Go（Chi、sqlc、WebSocket）、PostgreSQL/pgvector、pnpm + Turborepo。前端代码有 `apps/web`、`apps/desktop`、`packages/core`、`packages/ui`、`packages/views`。React Query 管服务端状态，Zustand 管 UI 本地状态，WebSocket 事件触发 Query invalidation。daemon 文档默认配置为轮询 3s、心跳 15s、Agent 超时 2h、最大并发 20；这些**不是实测吞吐或成功率**。

## 二、项目深挖题

### Q1：介绍你的 Agent 项目，解决什么业务问题？你负责什么？
**参考回答**：这是一个将编码 Agent 纳入团队工作流的平台。用户创建 Issue，平台为任务分配 Agent 与 Runtime，daemon 执行任务并回传状态，前端展示进度、结果和阻塞。与普通聊天机器人不同，它必须处理任务生命周期、执行环境、权限和结果验收。

**个人经历模板（按真实情况填写）**：我负责【模块】，解决【具体痛点】，主要改动【技术实现】，用【测试/监控】验证，结果【有来源的数据】。不要把开源仓库的所有能力说成自己的贡献。

### Q2：上线了吗？成功率、延迟、成本是多少？
**回答**：先明确是否真实上线；没有数据就直说，不编数字。建议建立指标：
- 任务验收成功率 = 验收通过数 / 已结束且纳入统计的任务数；
- 端到端 P95 = Issue 创建到验收完成的耗时；
- 排队 P95 = 入队到真正开始执行的耗时；
- 工具失败率 = 失败调用数 / 总调用数（按工具与错误码分层）；
- 单成功任务成本 = 模型与运行资源成本 / 成功任务数。

**追问**：成功率提升 10% 是百分点还是相对增幅？样本量和时间窗口是什么？

### Q3：RAG 召回不准，是 Chunk 还是 Embedding 的问题？
**回答**：用人工确认的黄金问答集，逐层排查：文档是否完整入库 → Chunk 是否包含答案 → 正确 Chunk 能否进入 Top-K（Recall@K）→ 排序/重排是否正确 → 模型是否依据证据作答。若答案被切断，先查分块；证据存在但召回不到，查 embedding、过滤条件和查询改写；召回到了但最终答错，查上下文构建和生成。每次只改变一个变量进行回归。

### Q4：工具调用失败率怎么监控？
**回答**：给任务和每次 Tool Span 绑定 trace_id、task_id、tool_name、耗时、错误类型、重试次数；对权限错误、超时、限流、依赖异常分别告警。HTTP 200 不等于任务成功，还需业务验收。副作用操作重试前要做幂等与状态确认。

### Q5：用户量增长 10 倍，哪里先扛不住？
**回答**：先用压测与 Trace 判断瓶颈在 API、WebSocket 扇出、数据库、队列、调度器、Runtime 并发还是模型限额。采取任务入队、租户限流、Runtime 容量调度、事件合并与背压、读写优化、横向扩容。daemon 默认并发 20 不能直接推导平台吞吐量。

## 三、多 Agent 系统设计

### Q6：多个 Agent 如何路由，谁决定调哪个？
**回答**：分成 Orchestrator 与 Scheduler。前者根据任务、能力、风险与依赖选择 Agent；后者根据 Runtime 在线状态、可用 CLI、并发余量、租户配额分配执行资源。队列用租约/幂等避免重复领取，最终以验收证据而不是模型自称“完成”来收尾。

**追问**：怎样避免 A、B 两个 Agent 互相委派？用任务 DAG、最大委派深度、截止时间、预算、去重和依赖环检测。

### Q7：并行还是串行？如何处理互相依赖？
**回答**：任务分解为 DAG，独立子任务并行，依赖任务等待上游产物验收；修改同一代码库时采用隔离工作区与合并冲突处理。并行是否划算要比较整体耗时、协调成本和冲突率。

### Q8：全局上下文与子任务上下文如何隔离？
**回答**：全局保留目标、约束、任务状态和经过验证的产物引用；子 Agent 只接收完成子任务所需的文档、工具、权限。大文件保存在 Workspace/Artifact，按需读取。租户隔离必须在服务端检索与资源授权层强制执行，不能只写 Prompt。

### Q9：一个 Agent 挂了，整个流程如何降级？
**回答**：任务状态机（queued/running/waiting/retrying/done/failed），心跳与租约识别失联；持久化 Checkpoint、外部操作结果和幂等键；安全重试或切换执行器，无法恢复则人工接管。已创建 PR 却丢回调时先远端对账，不能直接再创建。

### Q10：Token 成本与延迟怎么优化？
**回答**：模型按任务复杂度分级路由；按需检索和压缩上下文；缓存带权限与版本的确定性结果；并行独立任务但限制总预算；通过黄金集同时比较质量、P95 和单成功任务成本，不以 Token 下降替代质量指标。

## 四、前端背景专项

### Q11：WebSocket、React Query 和 Zustand 如何协作？
**回答**：仓库规范中 React Query 是服务端状态唯一来源，Zustand 管工作区选择、筛选、草稿等本地状态。WS 收到事件后按实体使对应 Query 失效并重取；断线重连拉快照，事件带序列号/版本防乱序，批量处理高频通知避免请求风暴。

### Q12：多端复用的包边界如何划分？
**回答**：`packages/core` 放 API、hooks、store；`packages/ui` 放纯 UI；`packages/views` 放跨端业务视图；`apps/web` 和 `apps/desktop` 处理平台差异。共享 Views 不应直接依赖 Next.js 路由，改由 NavigationAdapter 注入。

### Q13：如何做 Agent 实时任务面板？
**回答**：任务列表和详情订阅状态/日志事件，支持重连、去重、虚拟列表、日志分页和批量渲染。区分“模型说完成”与“业务验收完成”；通过 Web Vitals、事件到达延迟、重渲染次数、内存与消息积压评测前端性能。

## 五、业务场景题

### Q14：给一个企业场景设计 AI Agent？
**示例：研发 Issue → 修复 → 测试 → PR → 审批**。架构为看板/API → 编排与任务队列 → 调度器 → daemon/隔离工作区 → Coding Agent → 测试/PR → 人工审批与验收。关键措施：权限最小化、外部内容防提示注入、幂等写入、超时重试、任务可恢复、Trace 审计。指标：验收成功率、P95 完成时间、人工接管率、单成功任务成本。

**追问：Agent 答错怎么兜底？**：对可确定性验证的输出做测试和规则校验；低置信度或高风险操作转人工；失败样本进入 Badcase/黄金集；必要时回滚或补偿。

**追问：内容安全、合规怎么考虑？**：输入输出分类、敏感信息脱敏、租户隔离、权限逐次校验、数据保留和删除、审计与跨境数据要求；Prompt 不是安全边界。

## 六、算法手写（前端也会考）

### Q15：手写 LRU（TypeScript）
```ts
class LRU<K, V> {
  private data = new Map<K, V>();
  constructor(private readonly capacity: number) {
    if (!Number.isInteger(capacity) || capacity <= 0) throw Error("capacity");
  }
  get(k: K): V | undefined {
    if (!this.data.has(k)) return undefined;
    const v = this.data.get(k)!;
    this.data.delete(k);
    this.data.set(k, v);
    return v;
  }
  put(k: K, v: V) {
    this.data.delete(k);
    this.data.set(k, v);
    if (this.data.size > this.capacity) {
      this.data.delete(this.data.keys().next().value as K);
    }
  }
}
```
平均 O(1) get/put，空间 O(capacity)。追问：TTL、并发、容量 0 怎么处理？

### Q16：带超时的异步任务队列怎么设计？
**回答要点**：固定并发 worker 池、任务游标、结果按原始顺序保存；每个任务用 `Promise.race` 处理超时，配合 `AbortController` 通知取消；区分“调用方不再等待”和“底层操作真正停止”。必须限制并发、处理拒绝与资源清理。生产场景还需持久化队列、租约、幂等和重试退避。

## 七、二面回答模板：结果导向

每道项目题都按：**业务问题 → 我负责的边界 → 技术方案与权衡 → 指标口径 → 故障/降级 → 实际验证**。

面试前填写（不编造）：
- 真实职责与提交/PR：【待补】
- 是否生产上线：【待补】
- 任务规模与用户量：【待补】
- 任务成功率、P95 延迟、成本：【待补】
- 最难的一次故障与修复：【待补】

## 参考

- [Multica](https://github.com/multica-ai/multica)
- [阿里云 Handbook：任务编排](https://github.com/aliyun/ai-agent-handbook/tree/main/02-build)
- [阿里云 Handbook：运行与 Multi-Agent](https://github.com/aliyun/ai-agent-handbook/tree/main/03-run)
- [阿里云 Handbook：可观测与安全](https://github.com/aliyun/ai-agent-handbook/tree/main/04-governance)
- [阿里云 Handbook：黄金数据集与 Badcase](https://github.com/aliyun/ai-agent-handbook/tree/main/05-optimization)
