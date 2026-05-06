# 项目面试准备 — AgentBot & 教辅录题审核系统

---

## Part 1：AgentBot · 大模型智能体工具

### 项目概述

**定位**：基于 LangGraph 的 CLI Agent 框架，支持多轮工具调用、多模型提供商、长对话压缩、沙盒安全执行。  
**时间**：2025.10 — 至今  
**代码**：`/Users/qianlan/work/agent/agent-bot`

---

### 一、LangGraph 图结构

**核心实现**（`agentbot/core/agent.py`）：

```
StateGraph(AgentState)
  ├── node: agent      # 调用 LLM，决定是否使用工具
  ├── node: tools      # ToolNode 执行工具调用
  └── edge: tools_condition
        ├── → tools    # 有 tool_calls 时
        └── → END      # 无 tool_calls 时（直接回复）
```

**AgentState**（`agentbot/core/context.py`）：
- `messages`：使用 LangGraph `add_messages` reducer，自动合并消息列表
- `summary`：长对话压缩后的摘要文本

**面试问题**

**Q1：LangGraph 和直接用 LangChain 的 AgentExecutor 有什么区别？**

> AgentExecutor 是黑盒，内部循环不可见，难以在中间步骤插入自定义逻辑（如审计、条件分支）。LangGraph 把 Agent 建模为显式的有向图，每个节点和边都可以自定义，支持条件路由、并行节点、持久化 Checkpoint，更适合复杂 Agent 工作流。

**Q2：StateGraph 的 add_messages reducer 是什么？**

> LangGraph 的 State 字段可以指定 reducer 函数，决定如何合并新旧值。`add_messages` 是内置 reducer，把新消息追加到列表末尾，而不是覆盖。这样每次节点返回部分消息时，State 会自动累积，不需要手动管理消息列表。

**Q3：tools_condition 是怎么工作的？**

> `tools_condition` 是 LangGraph 内置的条件边函数，检查最后一条 AI 消息是否包含 `tool_calls` 字段。有则路由到 `tools` 节点执行工具，无则路由到 `END` 结束本轮。这实现了标准的 ReAct 循环。

---

### 二、长对话上下文压缩

**实现逻辑**（`agentbot/core/context.py`）：

1. 按 `HumanMessage` 边界把消息列表切分为"回合"
2. 总回合数超过 **40 轮**时触发压缩
3. 保留最近 **10 轮**完整消息
4. 丢弃的旧消息发给 LLM 生成摘要，写入 `AgentState.summary`
5. 下次对话时，`summary` 作为 System Message 前缀注入，保留历史语义

**效果**：长对话 Token 用量降低约 60%。

**面试问题**

**Q1：为什么不直接截断消息，而是生成摘要？**

> 直接截断会丢失早期对话的关键信息（如用户在第 1 轮说的需求）。摘要保留了语义，让 LLM 在后续对话中仍能参考早期上下文，同时大幅压缩 Token 数量。

**Q2：摘要生成用的是同一个 LLM 吗？会不会很慢？**

> 用的是同一个 LLM，但摘要生成是在触发压缩时同步执行的，用户会感知到一次稍长的等待。优化方向是用更小的模型（如 Haiku）做摘要，或者在后台异步预压缩。

**Q3：SQLite Checkpointer 是做什么的？**

> LangGraph 的 Checkpointer 在每次图执行后把完整的 AgentState 序列化存储。SQLite Checkpointer 把状态存到本地 SQLite 文件，支持跨会话恢复对话（按 `thread_id` 区分不同会话）。用户下次启动时可以继续上次的对话，而不是从头开始。

---

### 三、工具系统与安全沙盒

**12 个内置工具**：时间查询、计算器、任务调度、文件读写、Shell 执行、网络请求等，均通过 LangChain `@tool` 装饰器注册。

**安全机制**：
- **路径前缀校验**：文件和 Shell 操作限制在沙盒目录内，`os.path.abspath()` 后检查是否以沙盒路径为前缀，防止路径穿越（`../../etc/passwd`）
- **危险命令拦截**：正则匹配 `rm -rf`、`sudo`、`chmod 777` 等模式，直接拒绝执行
- **Shell 超时**：`subprocess.run(..., timeout=60)`，防止命令挂起

**面试问题**

**Q1：路径穿越攻击是什么？你怎么防的？**

> 路径穿越是通过 `../` 跳出预期目录访问任意文件。防御方式：先用 `os.path.abspath()` 解析出绝对路径（消除 `..`），再检查是否以沙盒目录为前缀。如果不是，拒绝操作。

**Q2：正则拦截危险命令可靠吗？**

> 不完全可靠，攻击者可以用变形命令绕过（如 `r\m -rf`、`$(rm -rf /)`）。更健壮的方案是用白名单而非黑名单，只允许特定命令集合。我的场景是 CLI 工具，用户是开发者，正则拦截主要防止 LLM 幻觉产生的误操作，不是对抗恶意用户。

---

### 四、多模型提供商适配

**Provider 工厂**（`agentbot/core/provider.py`）：统一接口适配 OpenAI / Anthropic / 阿里云（通义）/ 其他 OpenAI 兼容 API，运行时通过配置文件热切换，不需要重启。

**面试问题**

**Q：怎么实现运行时热切换模型？**

> LangGraph 图在 `compile()` 时绑定 LLM 对象。热切换的实现是：不重新编译图，而是在每次 `graph.invoke()` 时通过 `config` 参数传入当前 LLM 实例，节点内部从 `config` 读取而非使用编译时绑定的对象。这样切换模型只需更新配置，不需要重建图。

---

### 五、WikiAgent 子智能体

**架构**：自研轻量 ReAct Agent（非 LangGraph），`ThreadPoolExecutor` 并行执行多个工具调用。

**三类子智能体**：
- **Ingest Agent**：摄入 PDF/URL，提取实体和交叉引用，写入结构化 Markdown
- **Query Agent**：语义查询知识库，返回相关页面
- **Health Check Agent**：扫描死链、孤立页面、矛盾内容并自动修复

**面试问题**

**Q：为什么 WikiAgent 不用 LangGraph，而是自研？**

> WikiAgent 的工作流比较简单固定（摄入/查询/检查三条路径），不需要 LangGraph 的复杂状态管理和条件路由。自研的轻量 ReAct 循环代码量更少，依赖更少，更容易理解和维护。LangGraph 适合复杂的多分支、有状态的 Agent，简单场景用自研反而更清晰。

---

## Part 2：教辅录题审核一体化系统

### 项目概述

**定位**：打通教材上传 → OCR 识别 → 录题工作台 → 质检 → 入库全链路的一体化平台。  
**时间**：2025.06 — 至今  
**技术栈**：Vue 3 + TypeScript + Fastify + Prisma + PostgreSQL + Redis + BullMQ + MinerU + Prometheus + Grafana

---

### 一、自动切题模块

**流程**：
1. 教材 PDF 上传到 MinIO 对象存储
2. BullMQ 队列触发 MinerU 解析任务
3. MinerU 输出块级坐标（每个文本块的页码、位置、类型）+ 目录结构
4. 按"章节标题 + 题号"规则分段，识别题干、选项、答案、解析
5. 大模型（LLM）做结构化抽取，输出标准化 JSON
6. 结果回写工作台，支持人工校对

**题答配对**：基于"章节标题 + 题号"的双重 key 匹配，解决答案和题目在 PDF 中分离的问题（如题目在第 10 页，答案在第 80 页）。

**面试问题**

**Q1：MinerU 是什么？为什么用它而不是直接用 pdfplumber？**

> MinerU 是专门针对学术/教辅 PDF 的解析工具，能输出块级坐标（每个文本块的精确位置），并识别公式、表格、图片等复杂元素。pdfplumber 只能提取文本流，无法保留空间位置信息，对双栏排版和公式处理很差。录题场景需要精确的块坐标来支持工作台的"框选 OCR"功能。

**Q2：大模型抽取结构化内容，准确率怎么保证？**

> 三层保障：
> 1. Prompt 工程：给 LLM 提供严格的 JSON Schema 和少样本示例
> 2. 提交前校验：前端和后端都有字段完整性校验，缺失必填字段时阻止提交
> 3. 人工兜底：工作台支持人工修改 LLM 抽取结果，所有内容最终经人工确认才入库

**Q3：题答配对失败怎么处理？**

> 配对失败的题目标记为"待人工配对"状态，在工作台以高亮显示，由录题员手动关联答案。系统记录配对失败率，用于后续优化匹配算法。

---

### 二、BullMQ 异步任务队列

**使用场景**：PDF 拆页、MinerU 解析、自动切题、OCR 调用——这些任务耗时从几秒到几分钟不等，不能阻塞 HTTP 请求。

**架构**：
- Fastify 路由接收请求 → 立即返回 `jobId` → BullMQ 入队
- Worker 进程消费队列，执行实际任务
- 前端通过 SSE（Server-Sent Events）轮询任务状态

**队列监控**：
- 自定义 Prometheus 指标：队列积压数、任务等待时长 P95、任务执行时长 P95、Worker 心跳
- Grafana 面板可视化，配置 Worker 离线告警

**面试问题**

**Q1：BullMQ 和 Celery 有什么区别？**

> BullMQ 是 Node.js 生态的队列库，基于 Redis，API 设计更现代（Promise/async-await）。Celery 是 Python 生态，支持更多 Broker（RabbitMQ/Redis），功能更丰富（定时任务、任务链、Canvas）。选 BullMQ 是因为后端是 Node.js/TypeScript，生态一致，不需要跨语言。

**Q2：任务失败了怎么处理？**

> BullMQ 内置重试机制：配置 `attempts`（最大重试次数）和 `backoff`（退避策略，如指数退避）。超过重试次数的任务进入 `failed` 队列，触发告警通知，同时在工作台显示"处理失败"状态，支持手动重新触发。

**Q3：SSE 和 WebSocket 怎么选？**

> 任务状态推送是单向的（服务端 → 客户端），SSE 足够，实现比 WebSocket 简单（HTTP 长连接，无需握手协议）。WebSocket 适合双向实时通信（如聊天）。Fastify 原生支持 SSE，不需要额外库。

---

### 三、Prometheus + Grafana 监控

**自定义指标**（`ops-monitor` 服务）：

| 指标 | 类型 | 说明 |
|------|------|------|
| `http_request_duration_seconds` | Histogram | HTTP 请求耗时，按路由分组 |
| `bullmq_queue_depth` | Gauge | 队列积压任务数 |
| `bullmq_job_wait_duration_seconds` | Histogram | 任务等待时长 |
| `bullmq_job_active_duration_seconds` | Histogram | 任务执行时长 |
| `worker_heartbeat_timestamp` | Gauge | Worker 最后心跳时间 |

**告警规则**：
- API P95 延迟 > 2s 持续 5 分钟
- Worker 心跳超过 60s 未更新（Worker 离线）
- 机器 CPU > 85% 持续 10 分钟

**面试问题**

**Q1：Histogram 和 Gauge 有什么区别？**

> Gauge 是瞬时值，可以任意增减（如队列积压数、当前连接数）。Histogram 记录观测值的分布，自动统计 bucket 计数和总和，用于计算百分位数（P95/P99）。请求延迟用 Histogram，因为需要 P95 而不是平均值（平均值会被少量极慢请求拉高，掩盖真实情况）。

**Q2：为什么要单独开发 ops-monitor 服务？**

> Fastify 内置的 `prom-client` 只能采集 HTTP 层指标。BullMQ 队列积压、Worker 心跳这些指标需要主动查询 Redis 状态，不在 HTTP 请求路径上，所以需要一个独立的采集服务定期轮询并暴露给 Prometheus。

---

### 四、技术栈速查

| 技术 | 用途 | 关键点 |
|------|------|--------|
| Fastify v5 | HTTP 框架 | 插件化架构，SSE，高性能 |
| Prisma | ORM | 类型安全，迁移管理 |
| PostgreSQL | 主数据库 | 题目、用户、审核记录 |
| Redis | 缓存 + 队列 Broker | BullMQ 依赖，草稿缓存 |
| BullMQ | 异步任务队列 | 重试、优先级、并发控制 |
| MinerU | PDF 解析 | 块级坐标，公式识别 |
| MinIO | 对象存储 | PDF 文件存储 |
| Prometheus + Grafana | 监控告警 | 自定义指标，P95，Worker 心跳 |
| Vue 3 + TypeScript | 前端 | 录题工作台，PDF 双栏浏览 |
| Turborepo | Monorepo 构建 | pnpm workspace，增量构建 |
