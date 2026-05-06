# 面试题汇总 — LangChain & LangGraph

---

## Part 1：LangChain 基础

**Q：LangChain 的核心抽象有哪些？**
> - **Model**：LLM / ChatModel 的统一接口，屏蔽不同提供商的 API 差异
> - **Prompt**：PromptTemplate / ChatPromptTemplate，管理提示词模板
> - **Chain**：把多个组件串联成流水线（LCEL 表达式 `|` 操作符）
> - **Tool**：用 `@tool` 装饰器定义，供 Agent 调用
> - **Memory**：对话历史管理（已逐渐被 LangGraph Checkpointer 替代）
> - **Retriever**：文档检索接口，统一向量库、BM25 等检索方式

**Q：LCEL（LangChain Expression Language）是什么？**
> LCEL 是 LangChain 的链式组合语法，用 `|` 把 Runnable 组件串联：
> ```python
> chain = prompt | llm | output_parser
> result = chain.invoke({"question": "..."})
> ```
> 优势：自动支持流式输出（`.stream()`）、批量处理（`.batch()`）、异步（`.ainvoke()`），以及 LangSmith 追踪。

**Q：`@tool` 装饰器做了什么？**
> 把普通 Python 函数包装成 LangChain `StructuredTool`：
> - 从函数签名和类型注解自动生成 JSON Schema（供 LLM 的 function calling 使用）
> - 从 docstring 提取工具描述
> - 统一 `.invoke()` / `.ainvoke()` 接口
> 
> LLM 看到的是 JSON Schema，应用层调用的是原始 Python 函数。

**Q：LangChain 的 Runnable 接口有哪些方法？**
> 所有 LangChain 组件都实现 `Runnable` 接口：
> - `.invoke(input)` — 同步单次调用
> - `.ainvoke(input)` — 异步单次调用
> - `.stream(input)` — 同步流式输出
> - `.astream(input)` — 异步流式输出
> - `.batch(inputs)` — 批量并行调用

**Q：LangChain 的 Memory 和 LangGraph Checkpointer 有什么区别？**
> LangChain Memory 是早期方案，把对话历史存在内存里，进程重启即丢失，且线程不安全。LangGraph Checkpointer 把完整 State 序列化持久化（SQLite/PostgreSQL），支持跨会话恢复、多线程并发、按 `thread_id` 隔离不同会话，是更现代的方案。

---

## Part 2：LangGraph 核心概念

**Q：LangGraph 的核心数据结构是什么？**
> `StateGraph`：有向图，节点（Node）是函数，边（Edge）是状态转移规则。
> - **State**：用 TypedDict 定义，每个字段可以指定 reducer 函数
> - **Node**：接收当前 State，返回 State 的部分更新（dict）
> - **Edge**：普通边（固定跳转）或条件边（根据 State 动态路由）

**Q：LangGraph 的 reducer 是什么？**
> reducer 定义了如何把节点的返回值合并到 State 中。
> ```python
> class AgentState(TypedDict):
>     messages: Annotated[list, add_messages]  # add_messages 是 reducer
>     count: int  # 无 reducer，直接覆盖
> ```
> `add_messages` reducer 把新消息追加到列表，而不是覆盖。自定义 reducer 可以实现任意合并逻辑。

**Q：条件边（conditional_edge）怎么用？**
> ```python
> graph.add_conditional_edges(
>     "agent",                    # 从哪个节点出发
>     tools_condition,            # 路由函数，返回下一个节点名
>     {"tools": "tools", END: END}  # 路由映射
> )
> ```
> 路由函数接收当前 State，返回字符串（下一个节点名）。`tools_condition` 是内置函数，检查最后一条消息是否有 `tool_calls`。

**Q：LangGraph 的 Checkpointer 怎么工作？**
> 每次图执行完一个节点后，Checkpointer 把当前完整 State 序列化存储，关联到 `thread_id`（会话 ID）和 `checkpoint_id`（时间戳）。下次调用时传入相同 `thread_id`，图从最新 Checkpoint 恢复 State 继续执行。
> ```python
> config = {"configurable": {"thread_id": "user_123"}}
> graph.invoke(input, config=config)
> ```

**Q：LangGraph 如何实现 Human-in-the-loop？**
> 在需要人工确认的节点前设置 `interrupt_before`：
> ```python
> graph = builder.compile(
>     checkpointer=checkpointer,
>     interrupt_before=["dangerous_action"]
> )
> ```
> 图执行到该节点前暂停，State 持久化。人工审核后调用 `graph.invoke(None, config)` 继续执行。

**Q：LangGraph 的 `Send` API 是什么？**
> 用于动态并行：在条件边中返回 `Send` 对象列表，可以同时向多个节点发送不同的 State，实现 Map-Reduce 模式。
> ```python
> def route(state):
>     return [Send("worker", {"task": t}) for t in state["tasks"]]
> ```

---

## Part 3：LangGraph 工程实践

**Q：LangGraph 图的 `compile()` 做了什么？**
> 验证图结构（无孤立节点、有 START/END）、绑定 Checkpointer 和中断点，生成可执行的 `CompiledGraph` 对象。`compile()` 之后图结构不可修改，但可以通过 `config` 在运行时传入动态参数。

**Q：如何在 LangGraph 中实现流式输出？**
> ```python
> for chunk in graph.stream(input, config, stream_mode="values"):
>     print(chunk)  # 每个节点执行后的完整 State
>
> for chunk in graph.stream(input, config, stream_mode="updates"):
>     print(chunk)  # 每个节点的增量更新
> ```
> `stream_mode="messages"` 可以流式输出 LLM token（需要 LLM 支持流式）。

**Q：LangGraph 中如何处理工具调用错误？**
> `ToolNode` 默认会捕获工具执行异常，把错误信息包装成 `ToolMessage` 返回给 LLM，让 LLM 根据错误信息修正参数重试。可以通过 `handle_tool_errors=True/False` 控制。

**Q：多个 Agent 之间怎么通信？**
> 两种方式：
> 1. **子图（Subgraph）**：把一个 Agent 的 `CompiledGraph` 作为另一个图的节点，State 通过共享字段传递
> 2. **消息传递**：主 Agent 把任务描述写入消息，子 Agent 读取消息执行，结果写回消息列表

**Q：LangGraph 的 StateGraph 和 MessageGraph 有什么区别？**
> `MessageGraph` 是 `StateGraph` 的特化版本，State 固定为消息列表，适合简单的对话 Agent。`StateGraph` 更通用，State 可以包含任意字段，适合复杂的有状态工作流。实际项目推荐直接用 `StateGraph`，灵活性更高。

---

## Part 4：常见坑与调试

**Q：LangGraph 图执行卡住了怎么排查？**
> 1. 检查是否有无限循环：条件边的路由函数是否有终止条件
> 2. 检查 `tools_condition`：LLM 是否一直输出 `tool_calls` 而工具一直失败
> 3. 设置 `recursion_limit`（默认 25）防止无限递归：
>    ```python
>    graph.invoke(input, config={"recursion_limit": 10})
>    ```

**Q：工具调用参数类型错误怎么处理？**
> LangChain `@tool` 会用 Pydantic 做参数校验，类型不匹配时抛出 `ValidationError`。`ToolNode` 会捕获这个错误并返回给 LLM。解决方式：在工具 docstring 中明确说明参数格式，或在工具内部做类型转换兜底。

**Q：如何调试 LangGraph 的执行过程？**
> 1. `stream_mode="updates"` 打印每个节点的输出
> 2. 集成 LangSmith：设置 `LANGCHAIN_TRACING_V2=true`，自动追踪每次 LLM 调用和工具调用
> 3. 在节点函数里加日志，打印收到的 State
