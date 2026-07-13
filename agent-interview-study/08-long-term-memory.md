# 第八部分：长期记忆系统（Long-term Memory）

长期记忆系统解决的问题：`SummarizationMiddleware`（Module 7）只保证**单个会话内**的上下文不爆炸，但用户下次开一个新 thread 时，agent 应该还记得"这个人是谁、之前聊过什么、纠正过什么"。这就是记忆系统的职责——把对话异步蒸馏成结构化事实，持久化到磁盘，下次对话再注入回 system prompt。

## 1. 模块概览：四个角色分工

- **`MemoryMiddleware`**（[memory_middleware.py](backend/packages/harness/deerflow/agents/middlewares/memory_middleware.py)）—— 每轮对话结束后的"收集器"，决定要不要把这轮对话送去记忆更新。
- **`MemoryUpdateQueue`**（[queue.py](backend/packages/harness/deerflow/agents/memory/queue.py)）—— 防抖队列，避免用户连续发消息时每轮都触发一次昂贵的 LLM 调用。
- **`MemoryUpdater`**（[updater.py](backend/packages/harness/deerflow/agents/memory/updater.py)）—— 真正调 LLM 把对话压缩成结构化 JSON（用户画像 + 历史 + 事实列表）。
- **`FileMemoryStorage`**（[storage.py](backend/packages/harness/deerflow/agents/memory/storage.py)）—— 落盘，按用户隔离。

四者串联：`MemoryMiddleware.after_agent` → `queue.add()` → 30 秒防抖计时器到期 → `MemoryUpdater.update_memory()` → LLM 抽取 → `FileMemoryStorage.save()`。下次对话开始时，`DynamicContextMiddleware` 再把存好的记忆读出来塞进 system-reminder。

## 2. `MemoryMiddleware.after_agent`：一轮对话结束后发生了什么

[memory_middleware.py:52-110](backend/packages/harness/deerflow/agents/middlewares/memory_middleware.py#L52-L110)

这是一个 `after_agent` 钩子，而不是前面模块常见的 `before_model`/`wrap_model_call`——它在**整个 agent 运行完毕**（可能包含多轮工具调用）之后才触发一次，而不是每次模型调用都触发。

关键步骤：
1. **拿 thread_id**（[L68-71](backend/packages/harness/deerflow/agents/middlewares/memory_middleware.py#L68-L71)）：优先从 `runtime.context`，兜底走 LangGraph 的 `get_config()["configurable"]`。拿不到就直接跳过（[L72-74](backend/packages/harness/deerflow/agents/middlewares/memory_middleware.py#L72-L74)）。
2. **过滤消息**（[L83](backend/packages/harness/deerflow/agents/middlewares/memory_middleware.py#L83)）：调用 `filter_messages_for_memory`，只留用户输入和最终 AI 文本回复。
3. **最小门槛检查**（[L87-91](backend/packages/harness/deerflow/agents/middlewares/memory_middleware.py#L87-L91)）：过滤后必须至少有一条 human 消息和一条 ai 消息，否则不值得触发一次 LLM 调用去更新记忆。
4. **检测信号 + 捕获 user_id + 入队**（[L94-108](backend/packages/harness/deerflow/agents/middlewares/memory_middleware.py#L94-L108)）。

## 3. `filter_messages_for_memory`：为什么要过滤

[message_processing.py:56-85](backend/packages/harness/deerflow/agents/memory/message_processing.py#L56-L85)

只保留 `human` 消息和**没有 tool_calls 的** `ai` 消息（[L77-79](backend/packages/harness/deerflow/agents/memory/message_processing.py#L77-L79)）——中间那些"调用了 bash / write_file"的 AI 消息不算，因为记忆抽取只关心"用户说了什么、agent 最终答复了什么"，工具调用细节没有长期价值还会污染 LLM 抽取的 prompt。

还有一个精细处理：如果某条 human 消息全是 `<uploaded_files>` 标签（上传文件通知），剥离后内容为空，就整条跳过，并且设置 `skip_next_ai = True`（[L65-69](backend/packages/harness/deerflow/agents/memory/message_processing.py#L65-L69)），连带把它后面紧跟的那条 AI 回复（通常是"收到文件"之类的应答）也一起丢弃（[L80-82](backend/packages/harness/deerflow/agents/memory/message_processing.py#L80-L82)）——否则记忆里会留下"用户上传了 xxx.pdf"这种事实，但上传文件是会话级、临时的，下次对话文件已经不存在了，agent 却会去找一个不存在的文件。

`detect_correction` / `detect_reinforcement`（[L88-109](backend/packages/harness/deerflow/agents/memory/message_processing.py#L88-L109)）只扫**最近 6 条**消息中的 human 消息，用中英双语正则（[L10-37](backend/packages/harness/deerflow/agents/memory/message_processing.py#L10-L37)）匹配"那不对/you misunderstood/重试"或者"完全正确/perfect/继续保持"之类的信号。这两个信号会在下一步影响 LLM 抽取 prompt 的措辞。

## 4. `MemoryUpdateQueue`：防抖 + OR 合并语义

[queue.py:28-116](backend/packages/harness/deerflow/agents/memory/queue.py#L28-L116)

队列的 key 是 `(thread_id, user_id, agent_name)` 三元组（[L43-50](backend/packages/harness/deerflow/agents/memory/queue.py#L43-L50)）。`add()`（[L52-88](backend/packages/harness/deerflow/agents/memory/queue.py#L52-L88)）走标准防抖路径：入队后重置一个 `debounce_seconds`（默认 30s，[memory_config.py:33-38](backend/packages/harness/deerflow/config/memory_config.py#L33-L38)）的计时器；用户在窗口内连续发消息，计时器不断被重置，只有真正安静下来才会触发处理——避免用户输入三句话触发三次 LLM 调用。

`_enqueue_locked`（[L117-144](backend/packages/harness/deerflow/agents/memory/queue.py#L117-L144)）是这里最值得精读的部分：

```python
merged_correction_detected = correction_detected or (existing_context.correction_detected if existing_context is not None else False)
```

（[L132-133](backend/packages/harness/deerflow/agents/memory/queue.py#L132-L133)）—— 如果同一个 key 在防抖窗口内被多次入队（比如同一个 thread 连续两轮对话都触发了 `after_agent`），新的 `ConversationContext` 会完全**替换**旧的（[L143-144](backend/packages/harness/deerflow/agents/memory/queue.py#L143-L144)，因为要用最新的完整 messages 列表），但 `correction_detected`/`reinforcement_detected` 这两个布尔标志用 **OR** 语义合并，不会被替换丢失。也就是说：如果窗口内第一轮检测到了"用户纠正"信号，但第二轮的最后 6 条消息里已经看不到那句纠正话了，这个信号依然不会丢——只要窗口期内任意一次入队检测到过,就一直保留到这批真正被处理。

`_process_queue`（[L166-214](backend/packages/harness/deerflow/agents/memory/queue.py#L166-L214)）有一个并发保护：`self._processing` 标志位（[L172-175](backend/packages/harness/deerflow/agents/memory/queue.py#L172-L175)）——如果计时器触发时上一批还在处理中，不会静默丢弃这批新数据，而是重新调度一次立即执行（`_schedule_timer(0)`），保证"立即刷新"语义不被并发处理吞掉。

## 5. `add()` vs `add_nowait()`：两条入队路径

对比 [queue.py:52-88](backend/packages/harness/deerflow/agents/memory/queue.py#L52-L88)（`add`，`_reset_timer()` 用配置的 30s 延迟）和 [queue.py:90-115](backend/packages/harness/deerflow/agents/memory/queue.py#L90-L115)（`add_nowait`，直接 `_schedule_timer(0)` 立即调度）。

- `MemoryMiddleware.after_agent` 走 `add()`（[memory_middleware.py:101](backend/packages/harness/deerflow/agents/middlewares/memory_middleware.py#L101)）——正常对话结束，可以等 30 秒攒批。
- Module 7 讲过的 `memory_flush_hook`（[summarization_hook.py](backend/packages/harness/deerflow/agents/memory/summarization_hook.py)）走 `add_nowait()`——因为这个钩子是在**摘要即将吞掉旧消息**之前触发的，如果还傻等 30 秒，等摘要真的把消息压缩掉了，这批对话细节就永久丢失了，必须立即处理。

这就是 Module 7 和 Module 8 之间的直接联动：摘要中间件在丢弃消息前，会先给记忆系统一次"抢救"机会。

## 6. `user_id` 为什么必须在入队时捕获（ContextVar 陷阱）

[memory_middleware.py:96-99](backend/packages/harness/deerflow/agents/middlewares/memory_middleware.py#L96-L99)：

```python
# Capture user_id at enqueue time while the request context is still alive.
# threading.Timer fires on a different thread where ContextVar values are not
# propagated, so we must store user_id explicitly in ConversationContext.
user_id = get_effective_user_id()
```

`get_effective_user_id()` 底层是基于 Python `ContextVar` 实现的请求级上下文变量。`ContextVar` 只在**同一条 asyncio 任务链**或显式 `contextvars.copy_context()` 场景下传播，而 `threading.Timer`（[queue.py:159-164](backend/packages/harness/deerflow/agents/memory/queue.py#L159-L164)）是开一条全新的裸线程执行回调——这条线程里再调用 `get_effective_user_id()` 只会拿到默认值或者报错。所以必须在还处于原始请求上下文的时候就把 `user_id` 取出来，存进 `ConversationContext`（[queue.py:16-25](backend/packages/harness/deerflow/agents/memory/queue.py#L16-L25)）这个 dataclass 里作为一个显式字段，让它跨越线程边界存活下来。

这是一个非常典型的"上下文传播陷阱"，在任何用裸线程/线程池做异步分发的系统里都会遇到。

## 7. `MemoryUpdater.update_memory`：同步/异步分发，规避 issue #2615

[updater.py:539-598](backend/packages/harness/deerflow/agents/memory/updater.py#L539-L598)

```python
try:
    loop = asyncio.get_running_loop()
except RuntimeError:
    loop = None

if loop is not None and loop.is_running():
    future = _SYNC_MEMORY_UPDATER_EXECUTOR.submit(self._do_update_memory_sync, ...)
    return future.result()

return self._do_update_memory_sync(...)
```

`_process_queue` 是在 `threading.Timer` 开的裸线程里跑的，所以理论上永远不会有 running event loop，`update_memory` 走的应该总是最后那个直接同步调用分支。但这个方法也可能被其他调用方（比如某些同步包装场景）在 async 上下文里调用，所以还是做了防御性判断。

真正的技术细节在 [L29-38](backend/packages/harness/deerflow/agents/memory/updater.py#L29-L38) 的模块级注释和线程池定义：

```python
_SYNC_MEMORY_UPDATER_EXECUTOR = concurrent.futures.ThreadPoolExecutor(
    max_workers=4, thread_name_prefix="memory-updater-sync",
)
```

如果在 running loop 里，**不是**用 `asyncio.run()` 开一个新事件循环去跑异步版本，而是把纯**同步**的 `_do_update_memory_sync`（内部用 `model.invoke()`，同步 HTTP 调用）扔进一个专门的线程池同步执行。为什么？因为很多 LangChain provider 的异步客户端用 `@lru_cache` 全局缓存了一个 httpx `AsyncClient` 连接池，这个池子是绑定在创建它的那个 event loop 上的。如果在记忆更新流程里意外开一个新 event loop 再用异步客户端，就会跨事件循环复用同一个连接池对象，触发 httpx 底层的连接复用 bug（"Attempted to send request, but the connection has been closed"之类）——这就是文档里提到的 issue #2615。用纯同步的 `model.invoke()`，连接池对象是完全独立的（同步客户端走不同的连接池实现），从根源上避免了跨循环复用的可能性。

`aupdate_memory()`（[L467-492](backend/packages/harness/deerflow/agents/memory/updater.py#L467-L492)）是给真正需要 await 的调用方准备的异步包装，内部走 `asyncio.to_thread`——同样是"把同步调用扔到线程"这个思路，只是用标准库的 `to_thread` 而不是自建线程池。

## 8. Prompt 设计：`MEMORY_UPDATE_PROMPT` 与纠正/强化提示

[prompt.py:22-138](backend/packages/harness/deerflow/agents/memory/prompt.py#L22-L138) 是喂给 LLM 的记忆更新提示词模板，`{correction_hint}` 是一个占位符（[L48](backend/packages/harness/deerflow/agents/memory/prompt.py#L48)），由 `_build_correction_hint`（[updater.py:397-420](backend/packages/harness/deerflow/agents/memory/updater.py#L397-L420)）动态填充：

- 如果 `correction_detected=True`：注入一段"重点关注 agent 哪里做错了、用户纠正了什么，记录为 `category="correction"`、置信度 ≥0.95 的事实"。
- 如果 `reinforcement_detected=True`：注入"用户明确确认了某种做法是对的，记录为 `preference`/`behavior` 类别、置信度 ≥0.9 的事实"。

这两类事实分别对应"避免重复犯错"和"保留已验证有效的做法"——这其实和 CLAUDE.md 定义的"feedback memory"（纠正 vs 确认两种）思路是一模一样的模式,只是这里是系统自动做,而不是靠人手动说"记住"。

`newFacts` 每条还带一个可选的 `sourceError` 字段（[L127](backend/packages/harness/deerflow/agents/memory/prompt.py#L127) prompt 规则），只有 `category="correction"` 且原始错误在对话里被明确提到时才填，用来在未来注入时展示"避免:xxx"提示（回看第 15 节 `format_memory_for_injection` 里 `(avoid: ...)` 的拼接逻辑）。

## 9. `_parse_memory_update_response`：从 LLM 输出里稳健挖 JSON

[updater.py:313-331](backend/packages/harness/deerflow/agents/memory/updater.py#L313-L331)

即使 prompt 里明确要求"Return ONLY valid JSON"，很多模型还是会在 JSON 外面包一层思考过程、markdown 代码块或者解释性文字。这个函数的策略很朴素但很稳健：

```python
for match in re.finditer(r"\{", response_text):
    try:
        parsed, _end = decoder.raw_decode(response_text[match.start():])
    except json.JSONDecodeError:
        continue
    if isinstance(parsed, dict) and _REQUIRED_MEMORY_UPDATE_TOP_LEVEL_KEYS.issubset(parsed):
        return _normalize_memory_update_data(parsed)
```

扫描文本里**每一个** `{` 出现的位置，尝试用 `json.JSONDecoder().raw_decode` 从那个位置开始解析——`raw_decode` 的特性是只要开头是合法 JSON 就会成功返回,并告诉你结束位置,不要求整个字符串都是 JSON（这正是为什么不能直接 `json.loads(response_text)`）。第一个解析成功**且**包含四个必需顶层 key（`user`/`history`/`newFacts`/`factsToRemove`，[L230](backend/packages/harness/deerflow/agents/memory/updater.py#L230)）的对象就是答案。这样即使模型输出是"让我分析一下...{"user": ...} 希望有帮助！"这种格式,也能正确提取出中间的 JSON,而不会被开头的思考文字或结尾的客套话干扰。

## 10. `_normalize_memory_update_data`：fail-closed 的"unsafe partial memory update"防御

[updater.py:281-310](backend/packages/harness/deerflow/agents/memory/updater.py#L281-L310)

```python
if normalized_facts_to_remove and dropped_new_fact:
    raise json.JSONDecodeError(
        "Unsafe partial memory update: factsToRemove with malformed newFacts", ...
    )
```

这是本模块里最值得记住的防御性设计。场景：LLM 返回的 JSON 里 `factsToRemove` 是合法的（要删除旧事实 A），但 `newFacts` 里混进了一条格式错误的记录（比如 `confidence` 是个非法字符串），被 `_normalize_memory_update_fact`（[L233-278](backend/packages/harness/deerflow/agents/memory/updater.py#L233-L278)）判定为无效并丢弃（`dropped_new_fact = True`）。

如果这时候放任流程继续,`_apply_updates` 会正常执行"删除事实 A",但本该同时写入的替代事实（比如 LLM 打算用一条新事实取代旧的 A）却因为解析失败而没有写进去——净效果是**信息净损失**：删了旧的,没补上新的。这个函数选择直接抛出异常,让整次更新失败（`_do_update_memory_sync` 的 `except json.JSONDecodeError` 分支会捕获并返回 `False`,[updater.py:532-534](backend/packages/harness/deerflow/agents/memory/updater.py#L532-L534)），宁可这一批更新完全不生效,也不要在部分失败的情况下静默地丢事实。这是一个经典的"fail-closed 优于 fail-open"设计——尤其是对"记忆"这种一旦丢失很难注意到、也很难追溯的数据。

## 11. `_apply_updates`：去重、置信度门槛、上限截断

[updater.py:600-684](backend/packages/harness/deerflow/agents/memory/updater.py#L600-L684)

三层过滤,按顺序发生在把新事实写入 `current_memory["facts"]` 之前（[L644-673](backend/packages/harness/deerflow/agents/memory/updater.py#L644-L673)）：

1. **置信度门槛**（[L649](backend/packages/harness/deerflow/agents/memory/updater.py#L649)）：`confidence >= config.fact_confidence_threshold`（默认 0.7），低于阈值的事实直接跳过,不写入。
2. **内容去重**（[L645, L654-656](backend/packages/harness/deerflow/agents/memory/updater.py#L645-L656)）：`_fact_content_key`（[L371-377](backend/packages/harness/deerflow/agents/memory/updater.py#L371-L377)）对内容做 `.strip().casefold()`,构建已存在事实的 key 集合,新事实如果 casefold 后内容重复就跳过——避免"用户喜欢 Python"和"用户喜欢 python"被存成两条。
3. **总量上限截断**（[L676-682](backend/packages/harness/deerflow/agents/memory/updater.py#L676-L682)）：所有新事实追加完之后,如果总数超过 `config.max_facts`（默认 100）,按 `confidence` 降序排序只保留前 `max_facts` 条——低置信度的旧事实会被自然淘汰。

## 12. `FileMemoryStorage`：mtime 缓存 + 原子写 + 路径穿越防御

[storage.py:62-189](backend/packages/harness/deerflow/agents/memory/storage.py#L62-L189)

**缓存**（`load`，[L123-143](backend/packages/harness/deerflow/agents/memory/storage.py#L123-L143)）：key 是 `(user_id, agent_name)` 元组（[L119-121](backend/packages/harness/deerflow/agents/memory/storage.py#L119-L121)），value 是 `(memory_data, file_mtime)`。每次 `load()` 先 `stat()` 拿当前 mtime,和缓存里记的 mtime 比对,一致就直接返回缓存内容,不重新读文件、不重新解析 JSON——这对"每次系统提示词构建都要读一次记忆文件"这种高频路径很关键。文件被外部修改（比如另一个进程、或 `/api/memory/reload`）会自然让 mtime 变化,触发缓存失效。

**原子写**（`save`，[L160-189](backend/packages/harness/deerflow/agents/memory/storage.py#L160-L189)）：
```python
temp_path = file_path.with_suffix(f".{uuid.uuid4().hex}.tmp")
with open(temp_path, "w", encoding="utf-8") as f:
    json.dump(memory_data, f, ...)
temp_path.replace(file_path)
```
先写一个带随机 UUID 后缀的临时文件,写完整后再用 `Path.replace()` 做原子重命名覆盖正式文件。这样即使写入过程中进程崩溃/断电,正式的 `memory.json` 要么是写入前的完整旧版本,要么是写入后的完整新版本,不会出现"写了一半的半截 JSON"——这和 Module 7 里 `_externalize` 的临时文件模式是同一套思路,在这个代码库里是个反复出现的落盘约定。

**路径穿越防御**（`_validate_agent_name`，[L73-82](backend/packages/harness/deerflow/agents/memory/storage.py#L73-L82)）：`agent_name` 会被拼进文件路径（[L84-102](backend/packages/harness/deerflow/agents/memory/storage.py#L84-L102) `_get_memory_file_path`），如果不校验,一个包含 `../../etc/passwd` 之类内容的 `agent_name` 就能让保存路径逃逸出预期目录。用仓库统一的 `AGENT_NAME_PATTERN` 白名单正则做校验,和其他模块（比如 skills、自定义 agent 目录)用的是同一套模式,保持一致性。

## 13. `get_memory_storage`：反射加载存储类

[storage.py:196-231](backend/packages/harness/deerflow/agents/memory/storage.py#L196-L231)

`config.storage_class` 默认是字符串 `"deerflow.agents.memory.storage.FileMemoryStorage"`（[memory_config.py:29-32](backend/packages/harness/deerflow/config/memory_config.py#L29-L32)），通过 `importlib.import_module` + `getattr` 动态加载（[L210-214](backend/packages/harness/deerflow/agents/memory/storage.py#L210-L214)），加载后**必须**校验 `issubclass(storage_class, MemoryStorage)`（[L219-220](backend/packages/harness/deerflow/agents/memory/storage.py#L219-L220)）——如果配置指向一个根本不是 `MemoryStorage` 子类的东西,直接 `TypeError`,并且外层 `except Exception` 会 catch 住,自动回退到 `FileMemoryStorage()`（[L222-229](backend/packages/harness/deerflow/agents/memory/storage.py#L222-L229)）。这和其他地方提到的 model factory 反射模式是同一个套路：允许通过配置文件替换实现,但加一层类型安全校验,加载失败时优雅降级而不是直接让整个应用起不来。

## 14. 记忆注入回系统提示词：完整链路

`_get_memory_context`（[prompt.py:563-604](backend/packages/harness/deerflow/agents/lead_agent/prompt.py#L563-L604)）：

```python
memory_data = get_memory_data(agent_name, user_id=get_effective_user_id())
memory_content = format_memory_for_injection(
    memory_data, max_tokens=config.max_injection_tokens,
    use_tiktoken=(config.token_counting == "tiktoken"),
)
return f"<memory>\n{memory_content}\n</memory>\n"
```

调用方是 `DynamicContextMiddleware._build_full_reminder`（[dynamic_context_middleware.py:111-126](backend/packages/harness/deerflow/agents/middlewares/dynamic_context_middleware.py#L111-L126)），只在**第一轮**对话（`last_date is None`，[L177-190](backend/packages/harness/deerflow/agents/middlewares/dynamic_context_middleware.py#L177-L190)）把 `<memory>...</memory>` 和 `<current_date>` 一起拼进一个隐藏的 `<system-reminder>`，塞进第一条 HumanMessage 前面，之后这条消息的内容被"冻结"、永不再变（module 开头的注释解释这是为了让 system prompt 前缀能被 KV cache 命中，[L1-27](backend/packages/harness/deerflow/agents/middlewares/dynamic_context_middleware.py#L1-L27)）。

这正是 Module 7 里 `_preserve_dynamic_context_reminders` 要专门保护、不让摘要吞掉的那条隐藏消息——两个模块在这里精确对接：记忆内容被注入进这条 reminder，而 reminder 本身的存活由 Module 7 的摘要中间件负责。

还有一个容错细节：`abefore_agent`（[L209-232](backend/packages/harness/deerflow/agents/middlewares/dynamic_context_middleware.py#L209-L232)）把同步的 `_inject`（内含记忆文件 I/O 和可能的 tiktoken BPE 下载）扔进 `asyncio.to_thread`，并且设了 5 秒超时（`_INJECT_TIMEOUT_SECONDS`，[L51](backend/packages/harness/deerflow/agents/middlewares/dynamic_context_middleware.py#L51)）。超时就直接跳过这次注入（[L227-231](backend/packages/harness/deerflow/agents/middlewares/dynamic_context_middleware.py#L227-L231)），保证一次冷启动的 tiktoken 网络下载不会拖死整条请求——这和 `prompt.py` 里 tiktoken 加载失败缓存冷却时间的设计是同一个"网络受限环境下优雅降级"主题的两处体现。

## 15. `format_memory_for_injection`：真正的截断逻辑（纠正一处文档误传）

[prompt.py:319-439](backend/packages/harness/deerflow/agents/memory/prompt.py#L319-L439)

`learn.md` 现有摘要和 `backend/CLAUDE.md` 都写着"下次对话注入 **top 15 facts**"——读完实际代码后，这个说法是不准确的。真实逻辑（[L378-419](backend/packages/harness/deerflow/agents/memory/prompt.py#L378-L419)）：

1. 所有事实按 `confidence` **降序排序**（[L380-384](backend/packages/harness/deerflow/agents/memory/prompt.py#L380-L384)），没有任何"取前 15 条"的硬编码数字。
2. 先算出 `user`/`history` 两个 section 已经占用的 token 数（`base_tokens`，[L388-389](backend/packages/harness/deerflow/agents/memory/prompt.py#L388-L389)）。
3. 逐条把排序后的事实拼成一行（`- [category | confidence] content`，带 `sourceError` 就加 `(avoid: ...)` 后缀,[L403-409](backend/packages/harness/deerflow/agents/memory/prompt.py#L403-L409)），**逐行累加 token 数**，一旦加上这一行会超过 `max_tokens`（默认 2000，[memory_config.py:59-64](backend/packages/harness/deerflow/config/memory_config.py#L59-L64)）就立刻 `break`（[L415-419](backend/packages/harness/deerflow/agents/memory/prompt.py#L415-L419)），不再继续尝试后面置信度更低的事实。

也就是说实际注入的事实数量是**由 token 预算和每条事实的长度动态决定的**，可能是 5 条也可能是 40 条，跟"15"这个数字没有任何关系——"15"很可能是过去某个版本的实现残留，或者是撰写文档时的估算误传，代码里根本搜不到这个数字。这是继 Module 7 发现 `ToolOutputBudgetMiddleware` 中间件装配位置与 CLAUDE.md 文档不符之后，本次学习中第二处"文档 vs 实际代码"的落差，说明读代码永远要优先于读文档摘要。

最后还有个兜底截断（[L429-437](backend/packages/harness/deerflow/agents/memory/prompt.py#L429-L437)）：即使前面逐行累加已经很谨慎，最终拼出的整个字符串还是可能因为 token 估算误差略微超出预算，这里再做一次基于"字符/token 比例"的整体裁剪，乘 0.95 留安全余量。

## 16. 小结：与 Module 7 的联动关系图

```
一轮对话结束
   ├─ MemoryMiddleware.after_agent ──(debounced add())──┐
   │                                                     ├─→ MemoryUpdateQueue (30s 防抖, OR合并信号)
Summarization 即将丢弃旧消息                              │        │
   └─ memory_flush_hook ──(immediate add_nowait())──────┘        │
                                                                   ▼
                                                     MemoryUpdater.update_memory()
                                                     (同步 model.invoke，规避 #2615)
                                                                   │
                                            LLM 抽取 → _parse_memory_update_response
                                            → _normalize_memory_update_data (fail-closed)
                                                                   │
                                                     _apply_updates (去重/置信度/上限)
                                                                   │
                                                     FileMemoryStorage.save (原子写)
                                                                   │
                                                          （下次对话开始）
                                                                   ▼
                                    DynamicContextMiddleware._build_full_reminder
                                      → _get_memory_context → format_memory_for_injection
                                      → <memory>...</memory> 塞进隐藏 system-reminder
                                                                   │
                                    （该 reminder 由 Module 7 的
                                     _preserve_dynamic_context_reminders 保护，不被摘要吞掉）
```

Module 7 负责"会话内不丢重要上下文"，Module 8 负责"跨会话把重要上下文变成可持久化的结构化记忆"——两者共享同一条隐藏 reminder 消息作为交汇点。
