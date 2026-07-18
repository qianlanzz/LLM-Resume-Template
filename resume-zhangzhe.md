# 张哲

- 手机：13160006317
- 邮箱：3220857484@qq.com
- 学校：南京理工大学
- 学历：软件工程 · 硕士在读

**关键词**：Agent 框架, LangGraph, MCP, Tool-use, RAG, Memory, 智能运维, Django, Vue3

---

南京理工大学软件工程硕士在读，主要做 LLM Agent 和 AI 应用工程。独立开发 AgentBot，覆盖 LangGraph 工具调用、长对话记忆、沙盒执行和审计日志；主导教辅录题审核系统，把教材 PDF 解析、OCR 标注、质检和入库串成一套工作流。实习中参与航天领域模型管理平台，负责语义检索、推荐和监控模块；技术栈以 Python/Node.js 后端、Vue 3 前端、Redis/队列和 Docker 部署为主。

---

## 教育背景

- **南京理工大学** · 软件工程 · 硕士在读（2024.09 -- 至今）
- **南通大学** · 软件工程 · 本科（2019.09 -- 2023.06）

## 实习经历

### 中国电子科技集团公司第二十八研究所（智能模型资源管理平台研发实习）｜2025.09 -- 至今

- 参与航天领域 AI 模型全生命周期管理平台开发，项目覆盖模型、算法、数据集、镜像和部署监控等功能。
- 基于 Qwen3-Embedding-0.6B + FAISS 实现语义检索；将模型名称、分类、描述、关联数据集和版本说明拼成检索文档，支持用自然语言任务描述召回候选模型。
- 实现模型资源推荐模块，将用户历史行为、资源类型偏好和近期热度合并排序；对高频推荐结果做用户级 Redis 缓存，减少重复计算和接口抖动。
- 补充系统运行监控和智能分析能力，基于 psutil 采集 CPU、内存、磁盘、网络等指标，用 DBSCAN 标出异常时段，并用线性回归给出 7 天资源趋势，辅助运维提前发现容量风险。

## 项目经历

### AgentBot · 大模型智能体工具｜2025.10 -- 至今

- 用 LangGraph StateGraph 实现 CLI Agent 主循环，将 LLM 决策节点和 ToolNode 显式拆开，通过条件边判断是否继续调用工具；Provider 层统一适配 OpenAI、Anthropic、阿里云及 OpenAI 兼容接口，运行时可切换模型。
- 设计两层记忆：长期记忆用 Markdown 保存用户偏好，由 LLM 通过工具调用主动更新；短期对话通过 SQLite Checkpointer 按 thread 保存，超过 40 轮后压缩旧上下文并保留最近 10 轮，长对话 Token 用量降低约 60%。
- 内置时间、计算器、任务调度、文件读写、Shell 等 12 个工具，统一走 LangChain @tool 接口；文件和 Shell 操作限定在沙盒目录内，路径解析后做前缀校验，并拦截高风险命令和超时进程，减少 LLM 误操作。
- 开发 WikiAgent 子智能体，支持摄入 PDF、URL 和文章，抽取实体、交叉引用并写入 Markdown 知识库；按摄入、查询、健康检查拆分职责，可扫描死链、孤立页面和矛盾内容，知识库可直接用 Obsidian 查看关系图。
- 为 Agent 执行过程增加 JSONL 审计日志，记录模型输入、工具调用、工具返回、最终回复和系统动作；配合 Rich 终端输出，可回放一次任务的完整决策链，定位工具参数错误和上下文污染问题。

### 教辅录题审核一体化系统｜2025.06 -- 至今

- 面向教辅录题从 Excel、多工具协作迁移到统一平台的需求，基于 Vue 3 + TypeScript、Fastify + Prisma + PostgreSQL + Redis 开发录题工作台和后端服务，覆盖教材上传、PDF 浏览、框选 OCR、人工校对、质检和入库。
- 开发自动切题工作流模块，利用 MinerU 输出的块级坐标和目录结构解析教材 PDF，再用大模型抽取题号、题干、选项、答案和解析；通过“章节标题 + 题号”做题答配对，解决题目页和答案页分离的问题。
- 基于 BullMQ 承接 PDF 拆页、MinerU 解析、OCR 调用和自动切题等长任务，接口只返回 jobId，Worker 异步消费；前端通过 SSE 获取处理进度，避免几分钟级任务阻塞 HTTP 请求。
- 开发 ops-monitor 服务接入 Prometheus + Grafana，补充 API 延迟、队列积压、任务等待/执行时长 P95、Worker 心跳等指标，并配置 Worker 离线、API P95 超阈值和机器高 CPU 告警。

## 竞赛与获奖

### ICCV 2025 感知测试挑战赛：多项选择题视频问答｜冠军

- 基于 Qwen2.5-VL-7B 做视频问答指令微调，针对细粒度视觉辨别、遮挡推理和选项位置偏置设计困难题型挖掘、课程学习和选项乱序增强；推理阶段结合题型专用 Prompt、TTA 与多模型按题型集成，最终测试集准确率 80.9%。

### “天翼云息壤杯” AI 大赛：大语言模型数学推理｜三等奖

- 基于 Qwen2.5-Math-72B 蒸馏生成 10 万条 CoT 样本，并用 Math-Verify 与 Qwen2.5-72B 做答案校验；在 8 卡昇腾 910B 环境对 Qwen3-8B 做 LoRA 微调，使数学题正确率从 38% 提升至 62%。

### 其他获奖

- WACV 2025 目标距离估计挑战赛｜冠军
- CVPR 2025 少样本目标检测｜三等奖

## 专业技能

- **Agent / LLM**：LangGraph、LangChain Tool、MCP、RAG、长上下文压缩、工具安全、JSONL 审计、LoRA 微调、知识蒸馏、vLLM 推理部署。
- **后端**：Python（FastAPI / Django）、Node.js（Fastify）、RESTful API、SSE、PostgreSQL / MySQL、Redis、Celery / BullMQ、JWT。
- **前端**：Vue 3、TypeScript、Vite、Element Plus、Pinia、动态路由；了解 React / Next.js。
- **工程化**：Docker / Docker Compose、MinIO、Prometheus + Grafana、Nginx、Linux 常用部署与排障。
