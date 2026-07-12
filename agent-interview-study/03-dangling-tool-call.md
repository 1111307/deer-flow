# Dangling Tool Call 消息补偿：DanglingToolCallMiddleware

> 本文件延续 [01-state-and-middleware.md](01-state-and-middleware.md) / [02-loop-detection-and-safety.md](02-loop-detection-and-safety.md) 的"逐行代码精读"格式。

这个中间件跟上一模块的主题一脉相承——都是在跟"assistant tool_calls 后面必须紧跟对应 ToolMessage"这条跨 provider 的硬约束打交道，但触发原因完全不同：不是循环、不是安全过滤，而是**用户中断/请求取消**导致工具还没跑完，历史记录里就出现了"半截"的调用。

## 1. 问题定义

模块 docstring（[dangling_tool_call_middleware.py:1-14](../backend/packages/harness/deerflow/agents/middlewares/dangling_tool_call_middleware.py#L1-L14)）：

> A dangling tool call occurs when an AIMessage contains tool_calls but there are no corresponding ToolMessages in the history (e.g., due to user interruption or request cancellation). This causes LLM errors due to incomplete message format.

场景很具体：用户在 agent 调用工具的过程中把请求取消了（比如前端点了"停止"），工具还没来得及返回 `ToolMessage`，但这条带 `tool_calls` 的 `AIMessage` 已经进了历史。下一次用户发消息、graph 重新跑起来时，这条历史被原样发给模型——OpenAI/Moonshot 之类严格校验的 provider 一看到"assistant 说要调用工具，但后面没跟对应结果"，直接拒绝这次请求。这个中间件的任务就是在**发给模型之前**把这些缺口补上。

## 2. 三种"tool call"来源的归一化：`_message_tool_calls`

[dangling_tool_call_middleware.py:43-101](../backend/packages/harness/deerflow/agents/middlewares/dangling_tool_call_middleware.py#L43-L101)。这里有个容易被忽略的知识点：一条 `AIMessage` 携带的"工具调用信息"其实有**三种可能来源**，这个函数把它们统一成同一种字典结构：

1. **`msg.tool_calls`**——LangChain 解析成功的标准结构化调用，最常见的情况。
2. **`msg.additional_kwargs["tool_calls"]`**——某些 provider 适配器会把原始 payload 原样存一份在这里；只有当 `tool_calls`（第一种）为空时才去看这个字段，避免同一个调用被数两次。
3. **`msg.invalid_tool_calls`**——LangChain 自己的概念：provider 返回了一个工具调用，但参数没能解析成合法 JSON，LangChain 把它归类成"invalid"，不会被执行。**但即便不执行，provider 侧的适配器仍然可能把这个调用的 id/name 序列化进下一次请求**，严格的 OpenAI 兼容校验器一样会要求它有匹配的 `ToolMessage`。

第三种是这段代码里最容易被漏掉的细节——如果只处理前两种，遇到"参数解析失败但 id 还留在 payload 里"的情况仍然会触发同样的 400 错误。把这类调用也当作"dangling"来处理，让模型看到的是一条**可恢复的工具错误**，而不是又一次 provider 400。

## 3. 补偿消息的内容：`_synthetic_tool_message_content`

[dangling_tool_call_middleware.py:103-126](../backend/packages/harness/deerflow/agents/middlewares/dangling_tool_call_middleware.py#L103-L126)。这里区分了两种性质完全不同的"缺失"：

- **`invalid`**（解析失败）→ 告诉模型"这次调用的参数不是合法 JSON，工具没有执行"；
- **其余情况**（用户中断）→ 告诉模型"这次调用被中断了，没有拿到结果"。

针对 `invalid` 里的 `write_file` 又单独细化了一条很长的提示文案（issue #2894 的具体修复）：

```python
if name == "write_file":
    ...
    return (
        "[write_file failed before execution: the tool-call arguments were not valid JSON, "
        "so no file was written. This often happens when the model tries to write a very "
        "large Markdown file in a single tool call... Do not retry the same "
        "large `write_file` payload for this artifact; provide the report/content directly "
        "as normal assistant text in your next response..."
    )
```

**为什么单独给 `write_file` 写这么详细的引导**：这是一个真实观察到的失败模式——模型试图在一次调用里塞一个很大的 Markdown 文件，`content` 参数里的引号/反斜杠/代码块没有正确转义，导致整个 JSON 参数解析失败。如果只给一句"参数无效"的通用错误，模型很可能**原样重试同一个畸形的大 payload**，再次失败，陷入和第二模块讨论过的"循环"一样的死循环——只是这次的根因是格式错误而不是内容重复（`LoopDetectionMiddleware` 的 hash 判重甚至可能因为参数太大/编码方式不同而检测不到这种重复）。所以这里选择直接告诉模型"别重试这个大 payload，用普通文本回答，或者分段写"，把问题从"防止死循环"提前变成"从根源引导模型换一种做法"。

还有一处防御性截断（[dangling_tool_call_middleware.py:29-32, 108](../backend/packages/harness/deerflow/agents/middlewares/dangling_tool_call_middleware.py#L29-L32)）：

```python
_MAX_RECOVERY_ERROR_DETAIL_LEN = 500
...
error_text = error[:_MAX_RECOVERY_ERROR_DETAIL_LEN] if isinstance(error, str) and error else ""
```

畸形的 `write_file` 调用本身可能携带巨大的 Markdown payload，如果错误详情里原样把这段内容回显给模型，等于把这个已经出问题的大 payload 又送回了下一轮上下文——不仅浪费 token，还可能让模型继续"看到"这段畸形内容而误以为需要继续处理它。截断到 500 字符，只保留足够诊断用的错误摘要。这跟第二模块 `SafetyFinishReasonMiddleware` "故意不记录被过滤的参数"是同一种"错误恢复机制本身不能重新引入原问题"的设计直觉，只是这里的风险是"上下文膨胀/污染"而不是"绕过安全过滤"。

## 4. 核心算法：`_build_patched_messages`

[dangling_tool_call_middleware.py:128-183](../backend/packages/harness/deerflow/agents/middlewares/dangling_tool_call_middleware.py#L128-L183)。这不只是"检测缺口再补一条"，而是把整个消息列表**按 AIMessage → 对应 ToolMessage 的顺序重新分组**：

```python
tool_messages_by_id: dict[str, deque[ToolMessage]] = defaultdict(deque)
for msg in messages:
    if isinstance(msg, ToolMessage):
        tool_messages_by_id[msg.tool_call_id].append(msg)
```
第一步：把所有已存在的 `ToolMessage` 按 `tool_call_id` 分桶，存进 `deque`（用 deque 是为了在下面按顺序 `popleft()`，处理同一个 id 理论上出现多条消息的边界情况，保持先进先出）。

```python
patched: list = []
for msg in messages:
    if isinstance(msg, ToolMessage) and msg.tool_call_id in tool_call_ids:
        continue
    patched.append(msg)
    if getattr(msg, "type", None) != "ai":
        continue
    for tc in self._message_tool_calls(msg):
        ...
        existing_tool_msg = tool_msg_queue.popleft() if tool_msg_queue else None
        if existing_tool_msg is not None:
            patched.append(existing_tool_msg)
        else:
            patched.append(ToolMessage(...))  # 合成占位符
```
第二步：重新走一遍消息列表——**先把所有原本就存在的 `ToolMessage` 从原位置剔除**（第一个 `continue`），然后每遇到一条 `AIMessage`，紧跟着把它的每个 `tool_call` 对应的结果**重新插入**：如果这个 `tool_call_id` 在第一步的分桶里能找到已有的 `ToolMessage`，就把它挪到这个位置；找不到，就合成一条占位符。

**这解决的问题比"补缺口"更广**：即便所有 `ToolMessage` 都存在、一个没漏，如果由于某些历史原因它们在消息列表里的相对顺序跟对应的 `AIMessage` 没有严格挨在一起（比如中间插入了别的消息），这个重新分组过程也会把它们捋顺。也就是说，这个中间件同时做了"补缺口"和"强制排序"两件事，用一次遍历完成。

**幂等性检查**：
```python
if patched == messages:
    return None
```
如果重新分组之后跟原列表完全一样（说明历史本来就是合法有序的），直接返回 `None`，不去构造新的 `request.override(messages=patched)`。这不只是性能优化——避免了在"什么都不需要改"的正常情况下，每次模型调用都创建一份新的消息列表对象，减少不必要的对象churn。

## 5. Hook 选择：`wrap_model_call`，回顾第一模块的 reducer 知识

Docstring 里给出的理由（[dangling_tool_call_middleware.py:11-13](../backend/packages/harness/deerflow/agents/middlewares/dangling_tool_call_middleware.py#L11-L13)）：

> Uses `wrap_model_call` instead of `before_model` to ensure patches are inserted at the correct positions (immediately after each dangling AIMessage), not appended to the end of the message list as `before_model` + `add_messages` reducer would do.

这里直接呼应了**第一模块**讲过的 reducer 机制：`ThreadState.messages` 继承自 LangChain `AgentState`，默认用的是 `add_messages` reducer——语义是"追加"（按消息 id 去重/更新，但新消息始终排在已有消息之后）。如果这个中间件用 `before_model` 返回一个 `{"messages": [synthetic_tool_msg]}` 的更新，这个合成消息会经过 `add_messages` reducer，**被扔到整个列表的最后**，而不是插入到那条 dangling `AIMessage` 的正后方——顺序错了，问题完全没解决，可能比原来的报错更难排查。

`wrap_model_call` 不一样：它操作的是 `ModelRequest.messages`，这是**即将发给模型的请求消息列表**（一个普通的 list），不经过任何 state reducer。所以可以用 `request.override(messages=patched)` 直接把重新排好序的完整列表塞进去，插入位置完全由代码自己控制，不受 reducer 追加语义的限制。

**这里还有一个更深的架构选择，值得跟第二模块的 Loop/Safety 对比**：`DanglingToolCallMiddleware` 的两个 hook（`wrap_model_call`/`awrap_model_call`）都**没有向 state 写任何更新**——它只改写"即将发给模型的这一次请求"，不改写、也不持久化到 checkpoint 里的 `ThreadState.messages`。也就是说，这个补丁是**每次模型调用都临时重新计算一次**，而不是"修一次，永久生效"。

**为什么不直接把补丁写回 state**（这是一个很好的反向思考题）：如果把合成的占位 `ToolMessage` 持久化进 state，它就会永久出现在这个 thread 的历史里，被 `MemoryMiddleware` 提取事实、被 `TitleMiddleware` 用来生成标题、被前端渲染成一条真实的工具结果——但它其实是框架编造出来的"补丁"，不是真实发生过的工具执行结果。这跟第二模块 `LoopDetectionMiddleware` 选择"排队到 `wrap_model_call` 再注入警告、而不是直接改 `AIMessage`"是同一种顾虑：**框架自己造出来的补偿性内容，应该只在"发给模型看"这个层面存在，不应该污染需要长期保留的、面向用户/其他消费者的持久历史**。所以即使中断确实在 state 里留下了一个真实的缺口，这个缺口本身被允许一直留在 checkpoint 里，只在每次真正要调用模型之前被临时"整平"一次——修复成本很低（一次线性扫描），换来的是持久历史保持"诚实"。

## 总结表

| 概念 | 关键位置 | 一句话记忆点 |
|---|---|---|
| 三种 tool-call 来源归一化 | [dangling_tool_call_middleware.py:43-101](../backend/packages/harness/deerflow/agents/middlewares/dangling_tool_call_middleware.py#L43-L101) | 除了标准 `tool_calls`，还要处理 provider 原始 payload 和 LangChain 的 `invalid_tool_calls`，否则漏掉解析失败但 id 仍被序列化的情况 |
| write_file 专属恢复提示 | [dangling_tool_call_middleware.py:112-122](../backend/packages/harness/deerflow/agents/middlewares/dangling_tool_call_middleware.py#L112-L122) | 针对已知失败模式（大 Markdown payload 转义失败）直接引导模型换做法，而不是让它原样重试 |
| 错误详情截断 500 字符 | [dangling_tool_call_middleware.py:29-32](../backend/packages/harness/deerflow/agents/middlewares/dangling_tool_call_middleware.py#L29-L32) | 错误恢复机制不能把已经出问题的大 payload 重新塞回上下文 |
| 重新分组 + 补占位符 | [dangling_tool_call_middleware.py:128-183](../backend/packages/harness/deerflow/agents/middlewares/dangling_tool_call_middleware.py#L128-L183) | 不只补缺口，顺带把 ToolMessage 强制排到对应 AIMessage 正后方 |
| `patched == messages` 短路 | [dangling_tool_call_middleware.py:178-179](../backend/packages/harness/deerflow/agents/middlewares/dangling_tool_call_middleware.py#L178-L179) | 历史本就合法时不创建新列表，避免无意义的对象churn |
| `wrap_model_call` 而非 `before_model` | [dangling_tool_call_middleware.py:11-13](../backend/packages/harness/deerflow/agents/middlewares/dangling_tool_call_middleware.py#L11-L13) | `before_model` 走 `add_messages` reducer 只能追加到末尾；`wrap_model_call` 直接操作请求消息列表，插入位置可控 |
| 只补"发给模型的请求"，不写回 state | 两个 hook 都不返回 state 更新 | 合成的补偿消息是框架编造的，不该污染持久历史（呼应 Loop 的排队注入设计） |
