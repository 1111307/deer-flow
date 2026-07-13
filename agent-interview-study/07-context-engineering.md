# 第七部分：Context Engineering —— 摘要与预算控制

deer-flow 用两套完全独立的中间件应对"上下文爆炸"这个 Agent 系统的核心难题：`DeerFlowSummarizationMiddleware`(整体对话历史做摘要压缩)和 `ToolOutputBudgetMiddleware`(单个工具返回结果做体积管控)。它们运行在不同的时间点、不同的粒度上，但都遵循同一个设计哲学：**先把"大内容"挪到别处，只在上下文里留一个指针/摘要**。

## 1. 两套机制的分工与配置骨架

先看配置,理解它们各自在管什么。

**`SummarizationConfig`** ([summarization_config.py:21-72](../backend/packages/harness/deerflow/config/summarization_config.py#L21-L72)) 管的是"整段对话历史"：

```python
trigger: ContextSize | list[ContextSize] | None   # messages/tokens/fraction 任一达标就触发
keep: ContextSize                                    # 摘要后保留多少
preserve_recent_skill_count / tokens / tokens_per_skill  # skill 文件救援预算
```

`ContextSize.to_tuple()` ([summarization_config.py:16-18](../backend/packages/harness/deerflow/config/summarization_config.py#L16-L18)) 把 pydantic 模型转成 LangChain 原生 `SummarizationMiddleware` 期望的 `(type, value)` 元组格式——这是配置层到框架层的一层适配,配置本身用结构化的 pydantic 字段（能做 `ge=0` 校验、能生成 JSON Schema），但底层库吃的是元组。

**`ToolOutputConfig`** ([tool_output_config.py:8-63](../backend/packages/harness/deerflow/config/tool_output_config.py#L8-L63)) 管的是"单条工具结果"：

```python
externalize_min_chars: int = 12_000   # 超过就外部化到磁盘
preview_head_chars / preview_tail_chars = 2_000 / 1_000   # 外部化后留多少预览
fallback_max_chars: int = 30_000      # 外部化不可用时的兜底硬上限
exempt_tools = ["read_file", "read_file_tool"]   # 防止"外部化→读回→再外部化"死循环
tool_overrides: dict[str, int]        # 按工具名覆盖阈值
```

两者的关系：`SummarizationMiddleware` 解决的是"对话轮次太多，历史太长"；`ToolOutputBudgetMiddleware` 解决的是"单次工具调用返回了一个巨大的东西"（比如 `bash` 跑出来几万行日志）。后者甚至不需要等到对话变长——第一轮工具调用就可能触发。

## 2. TAG_NOSTREAM：摘要模型调用为什么要单独打标签

`DeerFlowSummarizationMiddleware.__init__` 最后几行是本模块第一个精巧设计：

[summarization_middleware.py:120-131](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L120-L131)
```python
existing_tags = list((getattr(self.model, "config", None) or {}).get("tags") or [])
merged_tags = [*existing_tags, TAG_NOSTREAM] if TAG_NOSTREAM not in existing_tags else existing_tags
self._summary_model = self.model.with_config(tags=merged_tags)
```

问题背景：摘要生成的 LLM 调用是在 `before_model` 钩子内部发生的（见下节），不是一次正常的对话轮次。但 LangGraph 的流式回调是按"token 流"广播的，不区分调用来源——如果不做标记，前端会看到一条突然出现的"幽灵 AI 消息"，用户会一脸问号地看着屏幕上多出一段不知道从哪来的文字。`TAG_NOSTREAM` 让流式处理器知道"这条调用不要广播"。

三个值得注意的细节：

1. **为什么新建 `_summary_model` 而不是直接改 `self.model`**：中间件实例是被缓存并在多个并发请求间复用的。如果在某次摘要调用期间临时把 `self.model` 换成打标签的版本，`await` 让出控制权的瞬间，另一个协程可能正好也在用这个中间件实例读 `self.model`，读到的就是被污染的版本——而且父类的 `profile`/`_get_ls_params` 逻辑也依赖 `self.model` 是原始未加工的版本。所以只能"新建一个绑定副本"，绝不做原地替换。
2. **为什么要手动合并 tags 而不是直接 `with_config(tags=[TAG_NOSTREAM])`**：`self.model` 在 [agent.py:110](../backend/packages/harness/deerflow/agents/lead_agent/agent.py#L110) 已经被打上了 `"middleware:summarize"` 标签（供 `RunJournal` 做归因，区分这条 LLM 调用是中间件发的还是 lead_agent 本体发的）。`RunnableBinding.with_config` 对 `tags` 字段做的是浅层覆盖而不是合并，直接传新列表会把已有的 `"middleware:summarize"` 标签整个覆盖掉——所以必须先读出已有 tags，再拼接。
3. `_summarize_with`/`_asummarize_with` ([summarization_middleware.py:141-177](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L141-L177)) 覆盖了父类的 `_create_summary`/`_acreate_summary`，唯一的差异就是把 `self.model.invoke` 换成 `self._summary_model.invoke`——其余提示词构建、异常兜底（`except Exception: return f"Error generating summary: {e!s}"`，摘要失败不应该让整个对话崩掉）逻辑原样保留。

## 3. `_maybe_summarize`：一次摘要触发的完整流程

[summarization_middleware.py:195-219](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L195-L219) 是同步版本（异步版本 `_amaybe_summarize` 逻辑完全对称）：

```python
total_tokens = self.token_counter(messages)
if not self._should_summarize(messages, total_tokens):
    return None
cutoff_index = self._determine_cutoff_index(messages)
if cutoff_index <= 0:
    return None
messages_to_summarize, preserved_messages = self._partition_with_skill_rescue(messages, cutoff_index)
messages_to_summarize, preserved_messages = self._preserve_dynamic_context_reminders(messages_to_summarize, preserved_messages)
self._fire_hooks(messages_to_summarize, preserved_messages, runtime)
summary = self._create_summary(messages_to_summarize)
new_messages = self._build_new_messages(summary)
return {"messages": [RemoveMessage(id=REMOVE_ALL_MESSAGES), *new_messages, *preserved_messages]}
```

`_should_summarize`/`_determine_cutoff_index`/`_partition_messages` 都是父类 `SummarizationMiddleware` 提供的——判断"是否达到 trigger 阈值"和"从哪条消息切开"的核心逻辑住在 LangChain 库里，deer-flow 不重新发明这一层，只在切分完成之后的两步插入自己的救援逻辑：**skill bundle 救援** 和 **动态上下文提醒救援**。这是一个很典型的"继承 + 钩子式扩展"模式：父类定义骨架，子类在关键节点插自己的逻辑，而不是整个重写。

最后的返回值是 LangGraph 状态更新的标准写法——`RemoveMessage(id=REMOVE_ALL_MESSAGES)` 清空当前的 messages 状态，再用 `*new_messages, *preserved_messages` 重建。注意顺序：摘要消息在前，救援消息在后——这样模型看到的历史顺序是"过去的摘要 → 被救回来的近期重要内容"，符合时间线直觉。

## 4. Skill bundle 救援：为什么摘要会"误杀"刚加载的技能文件

这是本模块最复杂的一块逻辑，分三个方法接力完成。

**背景动机**：`SkillActivationMiddleware`（第 11 部分要讲的模块）会在需要时让模型读取 `/mnt/skills/xxx/SKILL.md` 之类的技能说明文件，这些说明文件里的指令后续很多轮都要用。但摘要机制是按"消息新旧"一刀切的——如果技能文件恰好是在被切掉的那段窗口里读的，摘要会把它压缩成一句模糊的话，模型就丢失了具体的操作指令。

**第一步，找出所有"技能读取包"** —— `_find_skill_bundles` ([summarization_middleware.py:320-380](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L320-L380))：扫描 `to_summarize` 列表，找到每一个"带 tool_calls 的 AIMessage + 紧随其后的一串 ToolMessage"，从中筛出调用名在 `skill_file_read_tool_names`（默认 `read_file`/`read`/`view`/`cat`）且路径落在 `skills_container_path`（默认 `/mnt/skills`）下的那些调用，打包成一个 `_SkillBundle`：

```python
@dataclass
class _SkillBundle:
    ai_index: int
    skill_tool_indices: tuple[int, ...]
    skill_tool_call_ids: frozenset[str]
    skill_tool_tokens: int
    skill_key: str   # 排序后拼接的路径列表,用于去重
```

路径判定在 `_is_skill_tool_call` ([summarization_middleware.py:410-419](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L410-L419))，用的是 `path == root or path.startswith(root + "/")`——和第六部分讲过的沙箱虚拟路径校验是同一个"前缀匹配防越界"写法。

**第二步，在预算内挑选要救的包** —— `_select_bundles_to_rescue` ([summarization_middleware.py:382-408](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L382-L408))：从**最新往最旧**遍历（`reversed(bundles)`），受三个独立预算共同约束：

```python
for bundle in reversed(bundles):
    if kept >= self._preserve_recent_skill_count: break        # 最多救 N 个包(默认5)
    if bundle.skill_key in seen_skill_keys: continue            # 同一技能文件读了两次,只留最新那次
    if bundle.skill_tool_tokens > self._preserve_recent_skill_tokens_per_skill: continue  # 单包不能太大(默认5000 token)
    if total_tokens + bundle.skill_tool_tokens > self._preserve_recent_skill_tokens: continue  # 累计预算(默认25000 token)
    selected.append(bundle); total_tokens += ...; kept += 1; seen_skill_keys.add(...)
```

三个约束缺一不可：只看数量，一个巨大的技能文件也会被无脑救回来撑爆上下文；只看总 token，可能所有预算被一个包吃光，导致"救 1 个不救 5 个"；只看单包上限不看总量，5 个刚好卡线的包加起来仍然可能很大。三层预算叠加才是真正安全的设计。

**第三步，把选中的包从"待摘要"里摘出来** —— `_partition_with_skill_rescue` ([summarization_middleware.py:272-318](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L272-L318))。这里有个容易被忽略但很重要的细节：**AIMessage 里的 tool_calls 要按救援结果拆开，而不是整条消息一救或不救**：

```python
rescued_tool_calls = [tc for tc in msg.tool_calls if tc.get("id") in bundle.skill_tool_call_ids]
remaining_tool_calls = [tc for tc in msg.tool_calls if tc.get("id") not in bundle.skill_tool_call_ids]
if rescued_tool_calls:
    rescued.append(_clone_ai_message(msg, rescued_tool_calls, content=""))
if remaining_tool_calls or msg.content:
    remaining.append(_clone_ai_message(msg, remaining_tool_calls))
```

原因：一条 AIMessage 完全可能同时调用了一个技能读取工具和一个不相关的工具（比如同时 `read_file(skill.md)` 和 `web_search(...)`）。如果整条消息一起救援，不相关的调用也被拖进"保留区"，污染了摘要该压缩的范围；如果整条消息一起放弃，该救的技能内容也丢了。所以必须在 tool_calls 粒度上拆分并各自克隆成新的 AIMessage（`_clone_ai_message` 复用了第 3 部分讲过的 `clone_ai_message_with_tool_calls`，专门处理"替换 tool_calls 但保留其余字段"这种需要保持 provider 特定字段完整性的克隆操作）。

**失败模式是 fail-open**：`_partition_with_skill_rescue` 把整个查找过程包在 `try/except`，一旦救援逻辑本身抛异常，直接 `logger.exception(...)` 然后退回父类默认切分 ([summarization_middleware.py:283-287](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L283-L287))。这和第六部分沙箱模块里"路径校验失败必须 fail-closed（拒绝执行）"是相反的取向——因为这里的救援是一个"锦上添花"的优化，不救援只是让摘要略微损失一些细节，而不是安全边界被击穿；宁可退化到默认行为,也不能因为一个优化路径的 bug 让整条对话直接崩掉。

## 5. 动态上下文提醒救援：一个更小但同样精巧的坑

[summarization_middleware.py:254-270](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L254-L270)：

```python
def _preserve_dynamic_context_reminders(self, messages_to_summarize, preserved_messages):
    reminders = [msg for msg in messages_to_summarize if is_dynamic_context_reminder(msg)]
    if not reminders:
        return messages_to_summarize, preserved_messages
    remaining = [msg for msg in messages_to_summarize if not is_dynamic_context_reminder(msg)]
    return remaining, reminders + preserved_messages
```

`DynamicContextMiddleware` 会往对话里插一些隐藏消息（当前日期、长期记忆片段之类），注入逻辑很可能依赖"这是不是对话里的第一条 HumanMessage"之类的位置判断。如果摘要把这些提醒消息当普通历史压缩掉，摘要产生的新 `HumanMessage`（也就是下一节要讲的那条摘要消息）就会被 `DynamicContextMiddleware` 误认成"用户的第一条消息"，导致提醒被插到错误的位置——本该在最前面的日期提醒,可能被插到摘要消息后面变成一条突兀的追加内容。解法很直接：这类提醒消息不管在切分窗口的哪一侧，一律强制归入"保留"区，不参与摘要压缩,注意顺序是 `reminders + preserved_messages`（提醒消息排在其他保留消息前面，维持"提醒总是在最前"的位置约定）。

这一步在 `_maybe_summarize` 里紧跟在 skill 救援之后执行（[summarization_middleware.py:208](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L208)），说明两次救援是叠加的、独立的——一次处理"业务内容不能丢"，一次处理"系统消息的位置语义不能乱"。

## 6. `_build_new_messages`：用 `name="summary"` 隐藏摘要消息

[summarization_middleware.py:247-252](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L247-L252)：

```python
def _build_new_messages(self, summary: str) -> list[HumanMessage]:
    return [HumanMessage(content=f"Here is a summary of the conversation to date:\n\n{summary}", name="summary")]
```

覆盖父类版本,唯一区别是加了 `name="summary"`。这个特殊名字是前端和后端之间的一个隐式契约——前端渲染消息列表时会按 `name` 字段过滤掉这类消息，用户看到的对话历史里不会突然多出一条"以下是之前对话的摘要:……"，但这条消息依然完整地留在 LangGraph 的 `messages` 状态里，模型下一轮调用时能正常读到。这是"对模型可见,对用户不可见"的一个轻量实现方式,不需要额外的状态字段或消息类型,只用一个约定好的 `name` 值。

## 7. `_fire_hooks`：跨模块的钩子分发点

[summarization_middleware.py:421-444](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L421-L444)：

```python
def _fire_hooks(self, messages_to_summarize, preserved_messages, runtime) -> None:
    if not self._before_summarization_hooks:
        return
    event = SummarizationEvent(
        messages_to_summarize=tuple(messages_to_summarize),
        preserved_messages=tuple(preserved_messages),
        thread_id=_resolve_thread_id(runtime),
        agent_name=_resolve_agent_name(runtime),
        runtime=runtime,
    )
    for hook in self._before_summarization_hooks:
        try:
            hook(event)
        except Exception:
            hook_name = getattr(hook, "__name__", None) or type(hook).__name__
            logger.exception("before_summarization hook %s failed", hook_name)
```

这一步发生在"决定摘要哪些消息"之后、"真正调用 LLM 生成摘要"之前——也就是消息即将被压缩但还没被压缩的那个时间窗口。`SummarizationEvent` 是个 `frozen=True` 的 dataclass（不可变,钩子拿到手不能改动它,只能读取），把 `messages_to_summarize`（即将消失的原始消息）连同 `thread_id`/`agent_name`/`runtime` 一起传给每一个注册的钩子。

`_resolve_thread_id`/`_resolve_agent_name` ([summarization_middleware.py:42-63](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L42-L63)) 都是"先查 `runtime.context`，查不到再退回 `get_config()` 读 LangGraph 的 `configurable`"的双通道解析——这种双通道模式在这个仓库里反复出现,说明同一份信息（thread_id、agent_name）在不同调用路径下可能来自 `Runtime.context` 或者 `RunnableConfig.configurable` 两个不同的地方,取决于这次调用是从哪个入口进来的。

单个钩子失败被 `try/except` 隔离——一个钩子炸了不该连累其他钩子,也不该阻断摘要本身继续往下执行生成摘要文本。这和 skill 救援的 fail-open 是同一种"优化/增强逻辑允许失败,核心流程不能被拖累"的设计取向。

**当前唯一注册的钩子** 是 `memory_flush_hook` ([summarization_hook.py:12-34](../backend/packages/harness/deerflow/agents/memory/summarization_hook.py#L12-L34))，在 [agent.py:125-127](../backend/packages/harness/deerflow/agents/lead_agent/agent.py#L125-L127) 里根据 `resolved_app_config.memory.enabled` 条件注册：

```python
def memory_flush_hook(event: SummarizationEvent) -> None:
    if not get_memory_config().enabled or not event.thread_id:
        return
    filtered_messages = filter_messages_for_memory(list(event.messages_to_summarize))
    user_messages = [m for m in filtered_messages if getattr(m, "type", None) == "human"]
    assistant_messages = [m for m in filtered_messages if getattr(m, "type", None) == "ai"]
    if not user_messages or not assistant_messages:
        return
    correction_detected = detect_correction(filtered_messages)
    reinforcement_detected = not correction_detected and detect_reinforcement(filtered_messages)
    user_id = resolve_runtime_user_id(event.runtime)
    get_memory_queue().add_nowait(
        thread_id=event.thread_id, messages=filtered_messages, agent_name=event.agent_name,
        user_id=user_id, correction_detected=correction_detected, reinforcement_detected=reinforcement_detected,
    )
```

这是本模块和"长期记忆系统"（第 9 部分,还没讲）之间的关键连接点：**即将被摘要压缩、丢失细节的原始消息，在丢失之前会先被塞进一个异步队列（`get_memory_queue().add_nowait`），供记忆系统提炼成长期记忆。** 摘要是有损压缩,给模型当前上下文用;记忆队列是另一条独立的持久化路径,给未来的对话用——两者都消费同一批"即将消失"的消息，但服务不同的时间尺度。

两个检测逻辑的优先级值得注意：`correction_detected` 先判断，`reinforcement_detected` 只在**没有**检测到纠正时才判断 (`not correction_detected and detect_reinforcement(...)`)。也就是"用户纠正了 AI 的错误"这个信号的优先级高于"用户认可/强化了 AI 的做法"——同一段对话不太可能同时是纠正又是强化，如果纠正信号已经命中，就不必再浪费一次强化检测。这里遵循了"短路求值,同时编码优先级语义"的常见写法。

## 8. `ToolOutputBudgetMiddleware`：单条工具结果的体积治理

如果 `SummarizationMiddleware` 管的是"整段历史太长"，`ToolOutputBudgetMiddleware` 管的是"这一条工具结果本身就太大"——哪怕对话才刚开始，一次 `bash` 命令跑出五万行日志，也必须在它进入消息历史之前被处理掉。

核心决策逻辑在 `_budget_content` ([tool_output_budget_middleware.py:325-415](../backend/packages/harness/deerflow/agents/middlewares/tool_output_budget_middleware.py#L325-L415))，本质是一个两级降级策略：

```
超过 externalize_min_chars？
  → 尝试外部化到磁盘/沙箱，换成"预览 + 文件路径"
  → 外部化失败或阈值被设为 0？
    → 尝试 fallback 截断（head+tail），保证不超过 fallback_max_chars 硬上限
      → 都不满足触发条件？原样返回
```

外部化优先于截断,是因为外部化不丢信息——完整内容还在磁盘上,模型可以用 `read_file` 配合 `start_line`/`end_line` 按需读取任意片段;截断是真正丢信息的最后手段,只有在没有地方可写的时候才使用。

## 9. 双重外部化路径：host 磁盘 vs 沙箱内部写入

这是本模块第二个精巧设计,处理"沙箱到底能不能直接看到宿主机的输出目录"这个环境差异问题。

`_budget_content` 里的分支逻辑（[tool_output_budget_middleware.py:348-374](../backend/packages/harness/deerflow/agents/middlewares/tool_output_budget_middleware.py#L348-L374)）：

```python
if sandbox is not None:
    provider = get_sandbox_provider()
    if provider is not None and getattr(provider, "uses_thread_data_mounts", False):
        # 宿主机的 outputs 目录本身就被 bind-mount 进沙箱同一虚拟路径下,直接写宿主机磁盘等价于写进沙箱
        virtual_path = _externalize(content, ..., outputs_path=outputs_path, ...)
    else:
        # 远程/无挂载沙箱(如 AIO sandbox),宿主机写的文件沙箱内看不到,必须直接写进沙箱文件系统
        virtual_path = _externalize_to_sandbox(content, ..., sandbox=sandbox)
elif outputs_path:
    # 压根没有沙箱(遗留路径/非沙箱工具),直写宿主机
    virtual_path = _externalize(content, ..., outputs_path=outputs_path, ...)
```

`_externalize`（宿主机路径,[tool_output_budget_middleware.py:120-149](../backend/packages/harness/deerflow/agents/middlewares/tool_output_budget_middleware.py#L120-L149))用的是标准文件写入，配合第六部分讲过的"解析前后双重校验"防路径穿越：

```python
if os.path.isabs(storage_subdir) or ".." in storage_subdir:
    return None
...
if not os.path.abspath(filepath).startswith(os.path.abspath(storage_dir)):
    return None
```

`_externalize_to_sandbox`（沙箱内部路径,[tool_output_budget_middleware.py:152-198](../backend/packages/harness/deerflow/agents/middlewares/tool_output_budget_middleware.py#L152-L198)）则要绕开 AIO sandbox 后端的两个限制：`write_file` 不会自动创建父目录（所以先手动 `mkdir -p`），`execute_command` 失败时不抛异常而是把 `"Error: ..."` 当作正常字符串输出返回（所以不能靠 try/except 判断成败，必须显式检查返回值）。因此写完之后还要额外跑一次验证：

```python
check = sandbox.execute_command(f"test -s {shlex.quote(virtual_path)} && echo OK || echo MISSING")
if not isinstance(check, str) or check.strip() != "OK":
    return None
```

宁可外部化"假失败"（返回 None，退化成截断），也不能把一个实际写入失败、模型读不到的虚假路径塞给模型——那样模型拿着一个 `read_file` 永远读不到东西的路径，反而比直接截断更糟。这是"验证优于信任"在 I/O 场景下的具体体现，和第六部分 `SandboxAuditMiddleware` 的安全校验是同一种谨慎态度。

`shlex.quote` 出现在两处命令拼接里——这是命令注入防护的标准写法，`virtual_dir`/`virtual_path` 虽然是内部生成的文件名（`uuid.uuid4().hex[:12]`），理论上不会包含恶意字符，但工具名（`tool_name`）会经过 `_sanitize_tool_name` ([tool_output_budget_middleware.py:101-105](../backend/packages/harness/deerflow/agents/middlewares/tool_output_budget_middleware.py#L101-L105)) 先去掉路径分隔符和 `..`，双重防御叠加。

## 10. 便宜的预检查：避免小输出也走一遍线程池调度

`_needs_budget`/`_tool_message_over_budget`/`_effective_trigger` 三个函数（[tool_output_budget_middleware.py:457-493](../backend/packages/harness/deerflow/agents/middlewares/tool_output_budget_middleware.py#L457-L493)）构成一个轻量级的"值不值得处理"预判：

```python
def _effective_trigger(tool_name, config) -> int:
    candidates = []
    externalize = config.tool_overrides.get(tool_name, config.externalize_min_chars)
    if externalize > 0: candidates.append(externalize)
    if config.fallback_max_chars > 0: candidates.append(config.fallback_max_chars)
    return min(candidates) if candidates else -1

def _tool_message_over_budget(msg, config) -> bool:
    if (msg.name or "") in config.exempt_tools: return False
    trigger = _effective_trigger(msg.name or "", config)
    if trigger < 0: return False
    text = _message_text(msg.content)
    return text is not None and len(text) > trigger
```

`awrap_tool_call` ([tool_output_budget_middleware.py:596-613](../backend/packages/harness/deerflow/agents/middlewares/tool_output_budget_middleware.py#L596-L613)) 里能看到这个预检查存在的价值：

```python
result = await handler(request)
if not self._config.enabled: return result
if not _needs_budget(result, self._config): return result   # 便宜检查,不满足直接短路
outputs_path = _resolve_outputs_path(request)
sandbox = _resolve_sandbox(request)
return await asyncio.to_thread(_patch_result, result, self._config, outputs_path, sandbox)
```

真正的 `_patch_result` 被 `asyncio.to_thread` 扔进线程池执行（因为里面有磁盘 I/O，甚至沙箱的 `execute_command` 网络调用，不能占用事件循环）。但线程池调度本身是有开销的（哪怕开销不大，每次工具调用都要付一次也不值得）——绝大多数工具输出根本不大（一次简单的文件读取可能只有几百字符），如果每次都无条件地过一遍"判断是否需要外部化→分配新字符串→线程调度"的完整流程，是给 99% 的正常调用平白增加延迟。`_needs_budget` 用最轻量的字符串长度比较，在真正的重逻辑之前先把"明显不需要处理"的情况过滤掉。

注意 `_resolve_sandbox` 特意留在事件循环上同步调用而不是丢进线程池（[tool_output_budget_middleware.py:608-611](../backend/packages/harness/deerflow/agents/middlewares/tool_output_budget_middleware.py#L608-L611) 的注释解释了原因）：它只读 `runtime.state` 和 provider 的内存态注册表,没有真正的 I/O,丢进线程池反而多一次线程切换的开销。真正的沙箱 I/O（`mkdir`/`write_file`/`test -s`）都发生在 `_patch_result` 内部，那部分才被 `to_thread` 包起来。这是"分辨哪些步骤真的阻塞、哪些只是看起来像阻塞"的细致考量。

## 11. `_patch_model_messages`：模型调用时刻的历史消息补丁

`wrap_tool_call`/`awrap_tool_call` 只能处理"这一次刚产生的工具结果"。但如果历史消息列表里混进了一条从未被这个中间件处理过的超大 ToolMessage（比如中间件是后来才启用的，或者某条消息是通过别的路径注入的），它会一直待在 `messages` 状态里，每次模型调用都要重新发送这个巨大内容。`wrap_model_call`/`awrap_model_call` ([tool_output_budget_middleware.py:617-643](../backend/packages/harness/deerflow/agents/middlewares/tool_output_budget_middleware.py#L617-L643)) 补上这道防线：

```python
def wrap_model_call(self, request, handler):
    if self._config.enabled:
        messages = getattr(request, "messages", None)
        if isinstance(messages, list):
            patched = _patch_model_messages(messages, self._config)
            if patched is not None:
                request = request.override(messages=patched)
    return handler(request)
```

`_patch_model_messages` ([tool_output_budget_middleware.py:531-557](../backend/packages/harness/deerflow/agents/middlewares/tool_output_budget_middleware.py#L531-L557)) 本身也套了一层预检查（`if not any(... for msg in messages): return None`），避免在长对话历史上无谓地重建列表。真正要修补时，注意它调用 `_patch_tool_message(msg, config, outputs_path=None)`——**不传 `sandbox` 参数，也就是彻底放弃外部化，只用兜底截断**：

> "历史消息里如果还有超预算的内容，说明它在产生的那一刻就该被外部化过而没有被外部化成功（或者中间件是后来才启用的）；此时再重新触发一次外部化写盘意义不大，唯一要做的是保证它不会把模型上下文撑爆。"

这是一个明确的职责边界：**外部化只在工具刚产生结果的那一刻发生一次；历史扫描永远只做兜底截断，不重复做昂贵的 I/O。**

## 12. "先执行、后处理"的统一模式

`wrap_tool_call`/`awrap_tool_call` 都是无条件先调 `handler(request)` 拿到完整结果，再决定要不要处理：

```python
result = handler(request)          # 工具永远先跑完
if not self._config.enabled: return result
if not _needs_budget(result, self._config): return result
...
return _patch_result(result, self._config, outputs_path, sandbox)
```

预算控制是纯粹的**后处理**（post-processing），从不在工具执行前做拦截或限流——工具该跑多久跑多久，该产生多大的结果就产生多大的结果，中间件只负责在结果诞生之后决定"这段内容要不要挪个地方存放"。这和第六部分 `SandboxAuditMiddleware` 的"先执行、事后审计"是同一个模式在不同场景下的复用：执行阶段保持简单直接，把复杂的策略判断都挪到执行完成之后。

## 13. 中间件装配位置：一处文档与代码的落差

上一部分我们确认了 `SummarizationMiddleware` 在 `backend/CLAUDE.md` 文档里排在 lead agent 19 项中间件链的第 10 位。但读完实际装配代码后发现，`ToolOutputBudgetMiddleware` 的真实位置比这更靠前——它在 `_build_runtime_middlewares` ([tool_error_handling_middleware.py:129-146](../backend/packages/harness/deerflow/agents/middlewares/tool_error_handling_middleware.py#L129-L146)) 里是列表的**第一个元素**：

```python
middlewares: list[AgentMiddleware] = [
    ToolOutputBudgetMiddleware.from_app_config(app_config),
    ThreadDataMiddleware(lazy_init=lazy_init),
    SandboxMiddleware(lazy_init=lazy_init),
]
```

而 `build_lead_runtime_middlewares` ([tool_error_handling_middleware.py:190-197](../backend/packages/harness/deerflow/agents/middlewares/tool_error_handling_middleware.py#L190-L197)) 只是这个共享基础列表加了几个 lead-only 的追加项；`lead_agent/agent.py:300` 拿到这份列表后，才依次 `append` 后续的 `DynamicContextMiddleware`、`SkillActivationMiddleware`、`SummarizationMiddleware`（[agent.py:316-318](../backend/packages/harness/deerflow/agents/lead_agent/agent.py#L316-L318)）等等。

也就是说：**`ToolOutputBudgetMiddleware` 排在整条中间件链的最前面，比 `SummarizationMiddleware` 早了差不多 10 个位置**，而 `backend/CLAUDE.md` 列出的 19 项中间件清单里完全没提到它。这不是无关紧要的顺序问题——顺序本身就是设计意图的体现：单条工具结果的体积治理必须发生在最早期（工具调用刚返回的那一刻），远早于"整段对话历史是否要摘要"这种更宏观的判断。文档里漏掉这一项，读者容易误以为体积治理只有摘要这一层，从而忽略"单条超大结果"这个更早、更直接的爆炸点。这是继第 5 部分之后，这个仓库里又一处文档滞后于代码的例子。

## 14. 全局脉络小结

| 维度 | `DeerFlowSummarizationMiddleware` | `ToolOutputBudgetMiddleware` |
|---|---|---|
| 治理粒度 | 整段对话历史 | 单条工具结果 |
| 触发时机 | `before_model`（消息数/token数/占比达标） | 每次 `wrap_tool_call` 结果产生后 + 每次 `wrap_model_call` 历史扫描 |
| 装配位置 | lead agent 链第 10 位（较靠后） | 共享基础中间件链第 1 位（最靠前） |
| 主策略 | LLM 生成摘要,替换为一条隐藏 HumanMessage | 外部化到磁盘/沙箱,替换为预览+路径引用 |
| 兜底策略 | 摘要失败返回错误字符串（fail-open,不阻断对话） | 外部化不可用退化为 head+tail 截断（硬上限保证） |
| 特殊救援 | skill bundle 三重预算救援 + 动态上下文提醒救援 | 无（`exempt_tools` 是预防性排除,不是救援） |
| 跨模块联动 | `memory_flush_hook` → 长期记忆队列（第 9 部分） | 无直接联动,产物由 `read_file` 工具读回 |
| 并发/性能考量 | 独立 `_summary_model` 避免污染共享实例 + `TAG_NOSTREAM` 避免幽灵消息流式广播 | 便宜预检查避免线程池滥用 + 区分"读 state"和"真 I/O"决定是否 `to_thread` |

两套机制看似独立，实际共享同一套底层哲学：**detect early, externalize/compress, leave a pointer**——尽量早地识别"这块内容会撑爆上下文"，把真正的大块内容挪到别处持久化，只在消息流里留下一个可以按需展开的引用（摘要文本 or 文件路径）。这也是几乎所有长上下文 Agent 系统工程实践的共同套路，在面试被问到"如何处理超长上下文/超大工具输出"时，这两个真实实现是很扎实的参考案例。
