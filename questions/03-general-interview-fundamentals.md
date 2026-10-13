# AI Agent 二面：非项目类通用基础面试题（附答案）

> 来源：用户提供的“前端冲字节 AI Agent 二面复盘”两张截图。**本篇只整理非项目经历、非项目指标深挖的通用知识与能力题**。题目为截图转述，参考答案为独立编写，并非面试官官方答案。涉及具体企业场景的题目以通用设计模板回答，不假定任何项目经历。

## 目录

1. [多 Agent 如何路由和调度](#q1-多个-agent-怎么路由和调度谁决定调用哪个-agent)
2. [并行任务与相互依赖](#q2-agent-并行还是串行互相依赖怎么处理)
3. [全局与子任务上下文隔离](#q3-全局上下文和子任务上下文怎么隔离)
4. [Agent 故障与降级](#q4-一个-agent-挂了整个流程怎么降级兜底)
5. [Token 成本与延迟优化](#q5-token-成本怎么控延迟怎么优化)
6. [LRU 缓存手写](#q6-手写-lru-缓存)
7. [带超时的异步任务队列](#q7-实现一个带超时的异步任务队列)
8. [企业场景 Agent 设计](#q8-给某个企业业务场景设计一个-ai-agent-你的方案)
9. [Agent 答错如何兜底](#q9-agent-答错了用户体验怎么兜底)
10. [内容安全与合规](#q10-内容安全合规怎么考虑)

---

## 一、系统设计基础

### Q1. 多个 Agent 怎么路由和调度？谁决定调用哪个 Agent？

**30 秒回答：** 区分两层：编排器 Orchestrator 负责根据任务类型、能力、风险和依赖决定**由哪个角色处理**；调度器 Scheduler 根据在线状态、容量、并发配额和成本决定**在哪个 Runtime 执行**。不是让模型直接掌握底层资源权限。

**展开：**
1. 任务先分类、拆解成子任务，确定输入、输出、验收条件和权限。
2. 能力注册表记录 Agent 的专长、支持工具、模型与权限边界。
3. 任务路由优先考虑确定性规则；开放性任务可由模型辅助选择，但最终经服务端策略校验。
4. 调度器检查 Runtime 心跳、可用并发、租户限额与队列优先级，领取时使用租约与幂等。
5. 失败时重试或降级，最终用验收证据确认完成。

**常见追问：** 怎么防止 Agent 互相无限委派？限制最大深度、循环检测、任务预算、截止时间，并保证每个子任务有明确 owner。

**参考**：[Handbook 第 4 章](https://github.com/aliyun/ai-agent-handbook/tree/main/02-build)、[第 11 章](https://github.com/aliyun/ai-agent-handbook/tree/main/03-run)。

### Q2. Agent 并行还是串行？互相依赖怎么处理？

**30 秒回答：** 用 DAG 表示任务依赖。无依赖的节点可以并行；有依赖的节点在上游产物验收后再启动。并行受并发配额、共享资源冲突和成本限制，并不是越多越好。

**展开：**
- 任务拆解时明确依赖、输入、输出、超时和验收条件。
- 对共享代码/数据使用隔离工作区、版本控制、乐观锁或资源锁。
- 聚合结果时校验来源与版本；失败节点可局部重试，避免整图重跑。
- 指标：端到端耗时、冲突率、重试率、成本/成功任务。

**追问：** 为什么 Promise.all 不能直接解决？它只是单进程 Promise 协调，不提供跨进程持久化、分布式调度、故障恢复或资源隔离。

### Q3. 全局上下文和子任务上下文怎么隔离？

**30 秒回答：** 全局保留目标、约束、任务 DAG 和经过验证的产物引用；子 Agent 只获取执行该子任务必需的上下文、工具和权限。大结果存到 Workspace/Artifact，通过引用按需读取。

**展开：** 分清 Session、Task State、Workspace、Memory 和 Knowledge；上下文要有来源与信任等级，敏感数据必须在检索和工具服务端做授权过滤，不能仅依靠 Prompt 声明隔离。

**追问：** 子 Agent 需要上游信息怎么办？传结构化交接契约（目标、已验证事实、产物位置、未解决问题、允许操作），不要复制整个聊天记录。

**参考**：[Handbook 第 5 章](https://github.com/aliyun/ai-agent-handbook/tree/main/02-build)。

### Q4. 一个 Agent 挂了，整个流程怎么降级兜底？

**30 秒回答：** 用持久化任务状态机、心跳/租约、Checkpoint 和幂等机制恢复；能安全重试就重试，不能安全重试就对账或转人工；保证其他独立任务可继续。

**展开：**
- 任务状态：queued、running、waiting、retrying、completed、failed、cancelled。
- 识别失败类型：模型错误、工具超时、Runtime 失联、预算耗尽、外部写入状态未知。
- 重试前先确认外部副作用，避免重复创建订单、PR、退款等。
- 降级策略：备用模型/执行器、缩小任务范围、只读模式、人工审批或终止。
- 用户侧显示当前状态、失败原因、可恢复动作。

**追问：** 工具超时但实际成功怎么办？使用幂等键、远端状态查询、事务/补偿方案，不能直接盲重试。

### Q5. Token 成本怎么控？延迟怎么优化？

**30 秒回答：** 把质量作为约束，优化模型选择、上下文预算、工具调用、并行度和缓存；评估“任务成功率 + P95 延迟 + 单成功任务成本”，不能只看 Token 数。

**展开：**
- 模型分层路由：简单任务使用较低成本模型，复杂任务升级。
- Context Builder：只放当前必需信息，摘要/卸载长历史，RAG 按需召回。
- 工具缓存：对稳定、只读、权限可控的结果缓存并做版本失效。
- 并行独立工具调用，避免串行等待；限制最大步骤、重试和总预算。
- 监控 prompt/completion tokens、缓存命中率、模型耗时、工具 P95 和重试开销。

**追问：** 缓存为什么可能有风险？数据过期、跨租户泄漏、权限变更、错误答案复用。

**参考**：[Handbook 第 13 章](https://github.com/aliyun/ai-agent-handbook/tree/main/04-governance)、[第 18–23 章](https://github.com/aliyun/ai-agent-handbook/tree/main/05-optimization)。

---

## 二、前端算法手写

### Q6. 手写 LRU 缓存

**思路：** 哈希表 + 双向链表可实现平均 O(1) 读写；在 JavaScript 中利用 Map 的插入顺序也可实现常见面试版本。

```ts
class LRUCache<K, V> {
  private cache = new Map<K, V>();

  constructor(private readonly capacity: number) {
    if (!Number.isInteger(capacity) || capacity <= 0) {
      throw new Error("capacity must be a positive integer");
    }
  }

  get(key: K): V | undefined {
    if (!this.cache.has(key)) return undefined;
    const value = this.cache.get(key)!;
    this.cache.delete(key);
    this.cache.set(key, value);
    return value;
  }

  put(key: K, value: V): void {
    this.cache.delete(key);
    this.cache.set(key, value);
    if (this.cache.size > this.capacity) {
      const oldest = this.cache.keys().next().value as K;
      this.cache.delete(oldest);
    }
  }
}
```

**复杂度：** 平均 get/put O(1)，空间 O(capacity)。

**追问：** 为什么不用普通对象？对象没有同等直观的键类型与最近访问顺序维护机制。生产中还要考虑 TTL、容量为 0、并发与淘汰回调。

### Q7. 实现一个带超时的异步任务队列

**考点：** 并发上限、超时、错误隔离、结果顺序、取消语义。

```ts
type Task<T> = (signal: AbortSignal) => Promise<T>;

async function runWithLimit<T>(
  tasks: Task<T>[],
  concurrency: number,
  timeoutMs: number
): Promise<PromiseSettledResult<T>[]> {
  if (!Number.isInteger(concurrency) || concurrency < 1) throw new Error("invalid concurrency");
  if (!Number.isFinite(timeoutMs) || timeoutMs <= 0) throw new Error("invalid timeout");

  const results: PromiseSettledResult<T>[] = new Array(tasks.length);
  let cursor = 0;

  async function worker(): Promise<void> {
    while (cursor < tasks.length) {
      const index = cursor++;
      const controller = new AbortController();
      let timer: ReturnType<typeof setTimeout> | undefined;
      try {
        const timeout = new Promise<never>((_, reject) => {
          timer = setTimeout(() => {
            controller.abort();
            reject(new Error("task timeout"));
          }, timeoutMs);
        });
        const value = await Promise.race([tasks[index](controller.signal), timeout]);
        results[index] = { status: "fulfilled", value };
      } catch (reason) {
        results[index] = { status: "rejected", reason };
      } finally {
        if (timer !== undefined) clearTimeout(timer);
      }
    }
  }

  await Promise.all(
    Array.from({ length: Math.min(concurrency, tasks.length) }, () => worker())
  );
  return results;
}
```

**重要边界：** `Promise.race` 超时仅使等待方停止等待；`AbortController` 只是协作取消信号，底层任务如果忽略 signal，仍可能继续执行。因此此实现限制的是“被等待的任务槽位”，**不保证不可取消的底层工作实际并发数永远不超过上限**。生产级任务队列需由底层操作支持取消或在隔离执行器中强制终止，并处理持久化、租约、重试和幂等。

**追问：** 如何保持输出顺序？按任务原始 index 写入 results；为什么使用 allSettled 风格？避免一个失败中断其他任务。

---

## 三、企业场景与安全

### Q8. 给某个企业业务场景设计一个 AI Agent，你的方案？

**建议用通用场景：企业内部 IT 工单处理 Agent。**

**回答结构：**
1. **需求**：识别员工报障、检索知识、查询设备/账号状态、提出方案；高风险操作需审批。
2. **架构**：Web/IM 入口 → API/身份认证 → Orchestrator → RAG/Tools → 工单系统与知识库 → 验收与人工接管。
3. **任务流**：分类 → 检索 → 诊断 → 低风险自动操作或人工确认 → 验证结果 → 更新工单。
4. **权限与安全**：用户身份透传、租户/部门数据隔离、敏感字段脱敏、逐次工具鉴权、审计。
5. **可靠性**：任务状态机、幂等、超时重试、失败降级、人工兜底。
6. **评估**：一次解决率、自动化闭环率、P95 响应时间、错误操作率、人工接管率、单工单成本。

**追问：** 为什么不是全部自动化？涉及权限变更、数据删除、资金或不可逆操作时，应限制自主权限，设置 HITL 审批。

### Q9. Agent 答错了，用户体验怎么兜底？

**30 秒回答：** 分层兜底：回答前通过检索证据和规则约束降低错误；回答时明确不确定性、提供引用与人工转接；涉及外部操作时做权限校验和结果验证；回答后记录 Badcase 并回归测试。

**具体措施：**
- 知识问答：证据不足时不编造，显示来源与知识更新时间。
- 工具操作：区分“计划执行”“已执行”“已验收”，关键操作需确认。
- 体验层：给出失败原因、可重试入口、恢复进度和人工接管。
- 质量层：黄金集 + 人工标注 + 轨迹评估 + Badcase 归因，持续修复。

**追问：** 仅设置置信度阈值够吗？不够，模型自评置信度可能失准；需要基于证据、规则、环境事实与业务风险共同判断。

### Q10. 内容安全、合规怎么考虑？

**30 秒回答：** 将 Agent 同时视为可能被攻击的系统和可能越权行动的主体，采用输入、模型、工具、数据、执行环境的纵深防护，关键授权在服务端强制执行。

**展开：**
- 输入：Prompt Injection/恶意检索内容防护，不把外部文档指令当作系统指令。
- 身份权限：最小权限、按用户/租户/资源逐次校验，高风险工具需要二次授权。
- 数据：分类分级、脱敏、加密、访问审计、保留期限与删除流程；必要时评估数据出境。
- 执行：沙箱、网络出站控制、密钥隔离、危险操作审批与回滚。
- 输出：敏感信息泄漏检测、结构化结果校验和可追溯证据。
- 治理：日志脱敏、红队测试、安全事件处置、定期权限复核。

**追问：** 为什么“系统 Prompt 禁止泄漏数据”不够？因为 Prompt 不是访问控制机制，必须在工具和资源层执行权限。

**参考**：[Handbook 第 6 章](https://github.com/aliyun/ai-agent-handbook/tree/main/02-build)、[第 14 章](https://github.com/aliyun/ai-agent-handbook/tree/main/04-governance)。

---

## 面试复习建议

- **第一轮**：Q1–Q5 每题练习 30 秒定义 + 2 分钟展开，能画出 DAG/任务状态机。
- **第二轮**：Q6–Q7 现场手写并解释边界条件与复杂度。
- **第三轮**：Q8–Q10 按“业务目标 → 架构 → 风险 → 兜底 → 指标”完整口述。

**范围说明**：截图中的“项目上线了吗、效果数据、RAG 实际定位、工具调用实际失败率、用户量涨十倍时当前架构瓶颈”等属于项目深挖题，本篇不重复；RAG 通用排障方法可参阅 [项目型面经](../interview-experiences/multica-agent-platform-interview.md)。截图中的“反问”是候选人向面试官提问，不属于技术基础题。
