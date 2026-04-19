# 张哲

- 手机：13160006317
- 邮箱：3220857484@qq.com
- 学校：南京理工大学
- 学历：软件工程 · 硕士在读

**关键词**：Agent 框架, LangGraph, MCP, Tool-use, RAG, Memory, 智能运维, Django, Vue3

---

南京理工大学软件工程硕士在读，方向为 LLM Agent 开发与应用。熟悉 LangGraph 状态图编排、MCP 工具协议与 RAG 检索增强，有 Agent 框架开发、工具插件化、记忆管理与审计监控的实践经验。独立开发开源 Agent 框架 AgentBot，主导教辅录题审核系统开发；在 KDD Cup、天翼云息壤杯等竞赛中有模型微调与检索优化实战经验。技术栈覆盖 Python/Node.js 后端、Vue 3 前端及 Docker 容器化部署。

---

## 教育背景

- **南京理工大学** — 软件工程 硕士（2024.09 -- 至今）
- **南通大学** — 软件工程 本科（2019.09 -- 2023.06）

## 实习经历

### 中国电子科技集团公司第二十八研究所（智能模型资源管理平台研发实习）｜2025.09 -- 至今

- 参与航天领域 AI 模型全生命周期管理平台开发（Django + Vue 3），负责语义检索、推荐系统、缓存与监控模块。
- 针对模型资源增长后关键词检索召回率下降的问题，基于 Qwen3-Embedding-0.6B + FAISS 构建语义检索服务，支持自然语言任务描述检索模型库；通过 Django Signal 监听元数据变更触发增量索引重建，索引读写采用快照隔离实现无锁并发，持久化通过原子文件替换保证崩溃一致性。
- 实现三路混合推荐引擎（协同过滤 + 内容匹配 + 热度排序），引入 Redis 缓存推荐结果，按用户维度设置 TTL 并通过事件驱动主动失效；写入端采用 Pipeline 批量操作降低 RTT，缓存未命中时由 Celery 异步回填，配合分布式锁防止并发穿透。
- 基于 DBSCAN 聚类对系统 CPU、内存、磁盘、网络、请求延迟 5 维指标进行异常检测，识别偏离正常模式的异常时段；结合线性回归对资源用量做趋势外推，在容量触达阈值前生成预警工单，为运维预留扩容窗口。

## 项目经历

### AgentBot · LLM Agent 框架（开源个人项目）｜2026.2 -- 至今

- 基于 LangGraph StateGraph 搭建的 CLI Agent，核心是 agent ↔ tools 循环工作流，支持多轮工具调用与条件路由；通过统一的 Provider 工厂适配 OpenAI / Anthropic / 阿里云等 6 家模型 API，运行时可热切换。
- 记忆分两层：长期记忆用 Markdown 文件存用户偏好，LLM 通过 Tool Call 自己决定什么时候更新，跨会话保留；短期记忆基于 SQLite Checkpointer 存完整对话，按回合切分，超 40 轮后自动让 LLM 把旧对话压缩成摘要再裁掉原文，长对话 Token 用量降低约 60%。
- 内置 12 个工具（时间、计算器、任务调度、文件读写、Shell 执行等），工具调用走 LangChain @tool 标准接口；文件和 Shell 操作限制在沙盒目录内，通过路径前缀校验拦截穿越，正则匹配拦截 rm -rf 等危险命令，Shell 执行设 60s 超时。
- 技能（Skill）是可热插拔的外部扩展，放一个带 SKILL.md 的目录就能自动扫描注册为 LangChain Tool；执行前 LLM 先读 SKILL.md 了解用法和副作用再决定是否调用，自建 20 组评测集验证该机制将误操作率从 50% 降至 10%；同时支持通过 MCP 协议接入外部工具服务。
- 所有 LLM 输入、工具调用、工具返回、AI 回复、系统动作共 5 类事件写入 JSONL 审计日志（单例 + 内存队列 + 守护线程异步落盘），配套 Rich 终端可实时查看 Agent 每一步的决策过程。

### 教辅录题审核一体化系统｜2025.06 -- 至今

- 原有教辅录题依赖 Excel + 多工具协作，流程碎片化、效率低，需要一套打通上传、录题、质检、入库全链路的一体化系统。
- 基于 Vue 3 + TypeScript、Fastify + Prisma + PostgreSQL + Redis 开发录题工作台及后端服务，支持 PDF 双栏浏览、框选 OCR、公式渲染、草稿自动保存，打通教材上传到结果入库的完整链路。
- 主导自动切题模块，基于 MinerU 的块级坐标解析与目录分段能力将教材 PDF 转换为结构化内容，结合大模型完成题号、题干、选项、答案、解析抽取，基于"章节标题 + 题号"实现题答配对，生成标注结果回写工作台。
- 基于 BullMQ 承接 PDF 拆页、MinerU 解析与自动切题等长任务，避免重任务阻塞工作台；补充队列监控告警、草稿缓存与历史恢复、提交前校验等机制。

## 竞赛与获奖

### KDD Cup 2024 学术论文问答检索挑战赛｜2024.05 -- 2024.07

- 从海量论文中检索与问题最匹配的候选论文。设计"问题精炼 + 向量召回 + 重排序"方案——Llama3 精炼 Query，BGE M3 向量化 + FAISS 召回 Top100，BGE-Reranker 精排 Top20；通过 Hard Example Mining 微调 Reranker，检索分数从 0.138 提升至 0.166（+20.3%）。

### "天翼云息壤杯" AI 大赛：大语言模型数学推理能力提升｜2024.11 -- 2025.07

- 基于 Qwen2.5-Math-72B 蒸馏生成 10 万条 CoT 样本，结合 Math-Verify 与 Qwen2.5-72B 做答案校验；在 8 卡昇腾 910B 环境完成 Qwen3-8B 的 LoRA 微调，数学题正确率从 38% 提升至 62%。

### 获奖

- ICCV 2025 感知测试挑战赛：多项选择题视频问答 — 冠军
- WACV 2025 目标距离估计挑战赛 — 冠军
- CVPR 2025 少样本目标检测 — 三等奖

## 专业技能

- **Agent 开发**：熟悉 LangGraph 状态图编排（StateGraph、条件路由、ToolNode、Checkpointer），能独立设计带工具调用、多轮循环的 Tool-use Agent；有记忆管理、上下文裁剪、工具安全校验、JSONL 事件审计等工程化实践；熟悉 MCP 协议与技能插件机制，有多模型提供商适配经验。
- **LLM 应用**：熟悉 LangChain 工具链与 Prompt 设计，具备 RAG（向量召回 + 重排序）、知识蒸馏、LoRA 微调及 GRPO 强化学习实践经验；了解 vLLM 推理部署。
- **后端开发**：熟悉 Python（FastAPI/Django）和 Node.js（Fastify）后端开发，掌握 RESTful API、SSE 流式接口、PostgreSQL/MySQL、Redis、Celery/BullMQ 消息队列及 JWT 认证。
- **前端开发**：熟悉 Vue 3 + TypeScript + Vite，了解 React/Next.js；掌握 Element Plus、Pinia、动态路由等工程化实践。
- **DevOps**：熟悉 Docker/Docker Compose 容器化部署，掌握 MinIO 对象存储、Prometheus + Grafana 监控体系及 Nginx 反向代理。
