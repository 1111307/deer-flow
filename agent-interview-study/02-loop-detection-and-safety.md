# 循环检测与安全兜底：LoopDetectionMiddleware + SafetyFinishReasonMiddleware

> 本文件延续 [01-state-and-middleware.md](01-state-and-middleware.md) 的"逐行代码精读"格式：打开真实源码，一行一行讲为什么这么写，换一种写法会在哪里炸。

这两个中间件都挂在 `after_model`，都是"防止 agent 失控"的最后一道闸门，但触发条件完全不同——一个防"自己瞎转圈"，一个防"provider 安全过滤截断到一半"。

## 第一部分：LoopDetectionMiddleware —— 循环检测

### 1.1 问题定义

模块开头的 docstring（[loop_detection_middleware.py:1-16](../backend/packages/harness/deerflow/agents/middlewares/loop_detection_middleware.py#L1-L16)）说得很直白：

> P0 safety: prevents the agent from calling the same tool with the same arguments indefinitely until the recursion limit kills the run.

没有这道保护，agent 在推理链里卡死重复调用同一个工具是完全可能发生的（模型对同一个失败结果反复重试同一个查询、反复读同一个文件段落等），最终结果只是被 LangGraph 的 `recursion_limit` 粗暴杀掉——没有任何总结，用户体验很差。这个中间件的目标是**优雅降级**：先警告，警告无效就强制模型收尾。

两层检测策略：
1. **Hash-based**：同一个 tool_calls 集合（名字+参数）重复出现达到阈值 → 说明模型在原地打转。
2. **Frequency-based**：同一个工具类型被调用了很多次，但参数每次都不同（比如连续读了 40 个不同文件）→ hash 检测抓不到，但同样是浪费资源的信号，需要单独统计。

### 1.2 参数规范化：`_normalize_tool_call_args`

[loop_detection_middleware.py:73-96](../backend/packages/harness/deerflow/agents/middlewares/loop_detection_middleware.py#L73-L96)：

```python
def _normalize_tool_call_args(raw_args: object) -> tuple[dict, str | None]:
    if isinstance(raw_args, dict):
        return raw_args, None
    if isinstance(raw_args, str):
        try:
            parsed = json.loads(raw_args)
        except (TypeError, ValueError, json.JSONDecodeError):
            return {}, raw_args
        if isinstance(parsed, dict):
            return parsed, None
        return {}, json.dumps(parsed, sort_keys=True, default=str)
    if raw_args is None:
        return {}, None
    return {}, json.dumps(raw_args, sort_keys=True, default=str)
```

**为什么要这么写**：不同 provider 对 `tool_call["args"]` 的序列化不统一——多数是 dict，但某些 provider/某些 SDK 版本会把它序列化成 JSON 字符串。如果直接假设是 dict 去 `.get()`，遇到字符串就会抛异常，而这是**安全兜底中间件自己不能崩**的地方（如果 loop detection 自己挂了，反而更危险）。所以这里返回一个 `(dict, fallback_key)` 二元组：能解析成 dict 就正常处理；解析不了就退化成一个字符串 `fallback_key`，后面直接拿这个字符串当稳定 key 用，保证无论如何都能算出一个 hash，而不是抛异常。

**如果去掉这层防御会怎样**：某个 provider 偶尔返回非标准 args 格式时，整个 `after_model` hook 抛异常，中间件链路中断，agent run 直接失败——比"没有循环检测"更糟。

### 1.3 稳定 key：`_stable_tool_key`

[loop_detection_middleware.py:99-139](../backend/packages/harness/deerflow/agents/middlewares/loop_detection_middleware.py#L99-L139)。这是整个 hash 检测能不能"抓准"的核心，针对不同工具类型用了不同策略：

**`read_file`**——分桶（bucket）而不是精确匹配：
```python
bucket_size = 200
...
bucket_start = (bucket_start - 1) // bucket_size
bucket_end = (bucket_end - 1) // bucket_size
return f"{path}:{bucket_start}-{bucket_end}"
```
为什么不能用精确的 `start_line`/`end_line` 做 key？因为模型探索同一个文件时经常会用略微不同的行号范围（1-50、1-60、10-70……），这些调用语义上是"重复看这个区域"，但如果按精确参数 hash，每次都是不同 hash，Loop 检测完全抓不到。分桶把"同一个 200 行区间内的任意子范围"都归并成同一个 key，这样即使参数不完全一样也能识别出"在同一片区域来回读"的模式。

**`write_file` / `str_replace`**——反过来，故意用全参数 hash，不做任何"稳定化"：
```python
if name in {"write_file", "str_replace"}:
    if fallback_key is not None:
        return fallback_key
    return json.dumps(args, sort_keys=True, default=str)
```
注释写得很清楚：这两个工具是内容敏感的，同一个路径可能在迭代过程中被写入不同的内容（比如反复修正同一个文件的不同版本）。如果只用 `path` 当 key，会把"合理的多次迭代修改"误判成循环。所以这里反其道而行之：只有当**参数完全一致**（包括写入内容）时才算作重复。

**其他工具**——用"显著字段"白名单：
```python
salient_fields = ("path", "url", "query", "command", "pattern", "glob", "cmd")
stable_args = {field: args[field] for field in salient_fields if args.get(field) is not None}
```
只取这几个跟"调用意图"强相关的字段，忽略掉噪声字段（比如某些工具会带 timestamp、request_id 之类每次都不同但语义无关的参数），避免这些噪声字段把本该判定为重复的调用识破。

这是一个很好的面试点：**loop 检测不是无脑对 args 做 hash**，而是针对每类工具的语义单独设计"什么才算重复"。这种领域知识注入到通用中间件里，是工程实践里常见但容易被忽视的细节。

### 1.4 顺序无关 hash：`_hash_tool_calls`

[loop_detection_middleware.py:142-160](../backend/packages/harness/deerflow/agents/middlewares/loop_detection_middleware.py#L142-L160)：

```python
normalized = [f"{name}:{key}" for tc in tool_calls ...]
normalized.sort()
blob = json.dumps(normalized, sort_keys=True, default=str)
return hashlib.md5(blob.encode()).hexdigest()[:12]
```

关键是 `normalized.sort()`——模型在一次响应里可能并行发起多个 tool_calls，调用顺序本身不携带语义信息（`[read_file(a), read_file(b)]` 和 `[read_file(b), read_file(a)]` 是同一件事）。排序后再 hash，保证同一个"调用集合"无论顺序如何都产生相同 hash。如果不排序，模型只是因为随机性把两个调用的顺序换了一下，就会被误判成"不是重复"，循环检测直接失效。

### 1.5 滑动窗口 + 按线程 LRU

`_track_and_check`（[loop_detection_middleware.py:322-438](../backend/packages/harness/deerflow/agents/middlewares/loop_detection_middleware.py#L322-L438)）维护的核心数据结构：

- `self._history: OrderedDict[thread_id, list[hash]]`——每个线程一个滑动窗口（`window_size=20`），只看最近 20 次调用；
- `self._evict_if_needed`——当 tracked 的线程数超过 `max_tracked_threads=100` 就按 LRU 淘汰最久未使用的线程记录，防止这个中间件是个全局单例、长期运行下内存无限增长。

这里有个很容易漏掉但很重要的细节（[loop_detection_middleware.py:362-366](../backend/packages/harness/deerflow/agents/middlewares/loop_detection_middleware.py#L362-L366)）：

```python
warned_hashes = self._warned.get(thread_id)
if warned_hashes is not None:
    warned_hashes.intersection_update(history)
    if not warned_hashes:
        self._warned.pop(thread_id, None)
```

`self._warned` 记录"这个 hash 已经警告过了，不用重复警告"。但滑动窗口会把旧的 hash 挤出去——如果某个 hash 已经滑出窗口，说明"最近的历史"里已经不再包含这次重复了，应该允许它在未来重新触发警告（万一之后又开始重复同一个调用）。所以每次都要用当前窗口内容对 `warned_hashes` 做交集清理，避免"警告状态"永久生效导致未来真正的新一轮循环反而不被警告。

**两层检测的判定逻辑**（[loop_detection_middleware.py:371-436](../backend/packages/harness/deerflow/agents/middlewares/loop_detection_middleware.py#L371-L436)）：

- Layer 1（hash）：`count >= hard_limit(5)` → 直接返回 hard stop；`count >= warn_threshold(3)` 且这个 hash 还没警告过 → 返回警告，并标记为已警告（避免连续 3、4、5... 次都触发一遍警告消息）。
- Layer 2（frequency）：对**每一个** tool_call 按 `name` 累加全局计数（不管参数），支持 `tool_freq_overrides` 按工具单独设置阈值（比如 `bash` 在批处理场景本来就该被调用很多次，可以单独放宽）。

### 1.6 待注入警告队列——为什么不能在 `after_model` 直接改消息

这是本模块**最值得记的架构决策**，也是第一模块结尾我们讨论 middleware 消息配对约束时埋下的伏笔的直接应用。docstring 写得非常清楚（[loop_detection_middleware.py:18-32](../backend/packages/harness/deerflow/agents/middlewares/loop_detection_middleware.py#L18-L32)）：

> `after_model` fires immediately after the model emits an `AIMessage` that may carry `tool_calls`. The tools node has not run yet, so no matching `ToolMessage` exists in the history. Any message we add here lands *between* the assistant's tool_calls and their responses. OpenAI/Moonshot reject the next request with `"tool_call_ids did not have response messages"`... Anthropic also disallows mid-stream `SystemMessage`.

翻译一下：`after_model` 触发的时刻，模型刚说"我要调用这些工具"，但工具还没跑、`ToolMessage` 还不存在。如果这时候往消息列表末尾插一条新消息（不管是 warning 还是别的），这条消息就插在了"assistant 说要调用工具"和"工具真正返回结果"之间——这破坏了 OpenAI/Moonshot 要求的"tool_calls 后必须紧跟对应 ToolMessage"的强约束，也会撞上 Anthropic 对"流中间不能出现 SystemMessage"的限制。

解法是**延迟注入**：`after_model`（`_apply`，[loop_detection_middleware.py:477-502](../backend/packages/harness/deerflow/agents/middlewares/loop_detection_middleware.py#L477-L502)）检测到需要警告时，不碰当前这条 AIMessage，而是把警告文本塞进一个按 `(thread_id, run_id)` 分类的待处理队列（`_queue_pending_warning`）。真正的注入发生在**下一次**模型调用前的 `wrap_model_call`（`_augment_request`，[loop_detection_middleware.py:560-577](../backend/packages/harness/deerflow/agents/middlewares/loop_detection_middleware.py#L560-L577)）：

```python
new_messages = [
    *request.messages,
    HumanMessage(content=self._format_warning_message(warnings), name="loop_warning"),
]
```

到了这个时间点，工具已经执行完，所有 `ToolMessage` 都已经在消息列表里了，往列表最后追加一条 `HumanMessage` 完全不违反配对约束——而且用 `HumanMessage` 而不是 `SystemMessage`，是为了绕开 Anthropic 的限制。

这个队列的完整生命周期（四个 hook 各司其职）：
- `before_agent`（[loop_detection_middleware.py:524-527](../backend/packages/harness/deerflow/agents/middlewares/loop_detection_middleware.py#L524-L527)）：清掉同一线程里**其他 run** 遗留的过期警告——同一个 thread 可能被中断后用新的 run_id 重新发起，旧 run 排队的警告不该串到新 run 里。
- `after_model`：检测到重复就 `_queue_pending_warning` 入队。
- `wrap_model_call`：`_drain_pending_warnings` 取出并清空队列，拼进请求消息里。
- `after_agent`（[loop_detection_middleware.py:543-545](../backend/packages/harness/deerflow/agents/middlewares/loop_detection_middleware.py#L543-L545)）：run 结束时清空**当前 run** 的残留队列——如果 run 在警告被消费之前就结束了（比如中途被打断），不应该把这条警告带到未来的调用里，因为语境已经变了。

这个"排队 → 延迟到安全时机再注入"的模式，跟纯粹"能不能在 after_model 改 AIMessage"是两个问题：AIMessage 本身**可以**改（hard stop 就是直接改的，见下一节），但**插入一条新消息**不行，因为新消息的位置正好卡在配对约束的裂缝上。

### 1.7 Hard stop：直接改写 AIMessage

`_build_hard_stop_update`（[loop_detection_middleware.py:457-475](../backend/packages/harness/deerflow/agents/middlewares/loop_detection_middleware.py#L457-L475)）：

```python
update = {"tool_calls": [], "content": content}
additional_kwargs = dict(...)
for key in ("tool_calls", "function_call"):
    additional_kwargs.pop(key, None)
update["additional_kwargs"] = additional_kwargs

response_metadata = deepcopy(...)
if response_metadata.get("finish_reason") == "tool_calls":
    response_metadata["finish_reason"] = "stop"
```

这里为什么**可以**直接原地改这条 AIMessage（`_apply` 里的注释说得很直接，[loop_detection_middleware.py:480-484](../backend/packages/harness/deerflow/agents/middlewares/loop_detection_middleware.py#L480-L484)）：一旦把 `tool_calls` 清空，这条 AIMessage 就不再要求任何 `ToolMessage` 配对了——它现在纯粹是一条文本回答。所以在 hard stop 这条路径上直接 `model_copy` 改写是安全的，不需要走"排队延迟"那一套。同时把 `finish_reason` 从 `"tool_calls"` 改写成 `"stop"`，是为了让下游（前端渲染、日志分析）看到的 finish_reason 跟实际内容（一条纯文本回答）保持一致，不会出现"finish_reason 说要调用工具，但 tool_calls 是空的"这种自相矛盾的状态。

**记住这个改写细节**——下一节会看到 `SafetyFinishReasonMiddleware` 故意**不**这么做，这个对比本身就是一个很好的面试问题。

## 第二部分：SafetyFinishReasonMiddleware —— provider 安全截断兜底

### 2.1 问题定义

[safety_finish_reason_middleware.py:1-32](../backend/packages/harness/deerflow/agents/middlewares/safety_finish_reason_middleware.py#L1-L32) 的开场把场景讲得很具体：

> Some providers (OpenAI `finish_reason='content_filter'`, Anthropic `stop_reason='refusal'`, Gemini `finish_reason='SAFETY'` ...) can stop generation mid-stream while still returning partially-formed `tool_calls`.

问题链条是：provider 的安全过滤器在生成过程中触发，截断了输出——但截断发生的时候，模型可能已经"说出"了一部分结构化的 `tool_calls`（比如一个 `write_file` 调用，`content` 参数写到一半被截断）。LangChain 的 tool router 只看 `AIMessage.tool_calls` 是否非空，看到非空就无条件执行——于是一个**参数不完整/被审查截断**的调用被当成合法调用送去执行了。文档里给的例子很形象：写文件写到一半被截断，agent 看到truncated 结果，尝试去修复，又触发一次过滤，陷入循环——这跟第一部分的"循环"是两个完全不同的成因，但表现可能相似。

这也是为什么 `LoopDetectionMiddleware` 和 `SafetyFinishReasonMiddleware` 要放在同一个 `after_model` 责任链里：**它们共享"清空 tool_calls 阻止执行"这同一个动作**，但触发条件不同（一个看重复模式，一个看 provider 的安全信号）。

### 2.2 可插拔检测器模式

构造函数（[safety_finish_reason_middleware.py:70-73](../backend/packages/harness/deerflow/agents/middlewares/safety_finish_reason_middleware.py#L70-L73)）接收一个 `SafetyTerminationDetector` 列表，默认用 `default_detectors()`（内置对 OpenAI/Anthropic/Gemini 几种 provider 的安全信号做识别）。`from_config`（[safety_finish_reason_middleware.py:75-100](../backend/packages/harness/deerflow/agents/middlewares/safety_finish_reason_middleware.py#L75-L100)）支持通过配置文件用 `resolve_variable` 反射加载自定义检测器——跟第一模块见过的 `GuardrailMiddleware` provider 加载是同一套反射机制（`deerflow/reflection/`）。

一个很值得注意的防御性设计：

```python
if not config.detectors:
    raise ValueError("... use enabled=false to disable the middleware entirely.")
```

如果配置里显式传了一个**空列表**，直接抛异常拒绝构造，而不是"静默地什么都不检测"。为什么？因为一个空检测器列表的 `SafetyFinishReasonMiddleware` 会被挂载在 middleware 链里、日志里显示"已启用"，但实际上永远不会触发任何检测——这是运维上最危险的一种状态：看起来配置正确，实际完全没有防护。强制用户要么显式提供至少一个检测器，要么用 `enabled: false` 彻底禁用，不允许"启用了但形同虚设"这种中间状态存在。

### 2.3 检测本身要能容错

`_detect`（[safety_finish_reason_middleware.py:104-113](../backend/packages/harness/deerflow/agents/middlewares/safety_finish_reason_middleware.py#L104-L113)）：

```python
for detector in self._detectors:
    try:
        hit = detector.detect(message)
    except Exception:
        logger.exception("SafetyTerminationDetector %r raised; treating as no-match", ...)
        continue
    if hit is not None:
        return hit
```

每个检测器独立 try/except——一个检测器实现有 bug 抛异常，只是被当作"这个检测器没命中"处理并记日志，不会让整个 `after_model` 崩掉、进而搞垮整个 agent run。这跟第一模块 `ToolErrorHandlingMiddleware` 把工具异常转成 `ToolMessage` 而不是让异常向上传播是同一种"防止插件级别的故障扩散成系统级故障"的思路，在这里应用到了反射加载的第三方检测器插件上。

### 2.4 消息重写：`_build_suppressed_message`

[safety_finish_reason_middleware.py:133-166](../backend/packages/harness/deerflow/agents/middlewares/safety_finish_reason_middleware.py#L133-L166)，核心一行：

```python
cleared = clone_ai_message_with_tool_calls(message, [], content=new_content)
```

`clone_ai_message_with_tool_calls` 是个共享 helper（在 `tool_call_metadata.py` 里），同时清理结构化 `tool_calls`、`additional_kwargs.tool_calls`（某些 provider 的原始 payload 会额外存一份在这里）、以及 `function_call`（旧式 OpenAI function calling 字段）——这正是上一模块讨论过的 `SubagentLimitMiddleware._truncate_task_calls`（[subagent_limit_middleware.py:67](../backend/packages/harness/deerflow/agents/middlewares/subagent_limit_middleware.py#L67)）复用的同一个 helper，只不过 `SubagentLimitMiddleware` 是截断成"保留前 N 个"，这里是清空成"一个都不留"。三个中间件（Loop 的 hard stop、Safety、SubagentLimit）在"改写 AIMessage 的 tool_calls"这件事上没有各写各的实现，而是收敛到同一个 helper——这是"发现重复模式就提取共享工具函数"的教科书案例。

**关键差异点、也是最好的面试对比题**：注释里明确写了（[safety_finish_reason_middleware.py:148-150](../backend/packages/harness/deerflow/agents/middlewares/safety_finish_reason_middleware.py#L148-L150)）：

> It only rewrites `finish_reason` when the old value was `"tool_calls"`, which is not our case — `content_filter` / `refusal` / `SAFETY` stay put so downstream SSE / converters keep seeing the real provider reason.

对比第一部分的 Loop hard stop：Loop 会把 `finish_reason` 从 `"tool_calls"` **改写**成 `"stop"`，因为它清空 tool_calls 之后这条消息确实变成了一条普通的文本终止；而 Safety 这边，`finish_reason` 本来就是 `content_filter`/`refusal`/`SAFETY` 这些值（不是 `"tool_calls"`），所以 `clone_ai_message_with_tool_calls` 的判断条件不成立，**真实的 provider 原因被完整保留**。这是故意的：下游的日志系统、SSE 事件、前端渲染都需要知道"这条回复是因为安全过滤被截断的"，而不是被抹平成一个通用的 `"stop"`。**是否要暴露真实终止原因**取决于这个原因对下游有没有诊断价值——循环检测的 `"stop"` 只是"模型确实说完了"，而安全过滤的原因本身就是需要审计的信号。

### 2.5 可观测性的两条腿：SSE vs 持久化审计

`_apply`（[safety_finish_reason_middleware.py:266-307](../backend/packages/harness/deerflow/agents/middlewares/safety_finish_reason_middleware.py#L266-L307)）在改写消息后，同时做两件独立的事：

- `_emit_event`（[safety_finish_reason_middleware.py:170-205](../backend/packages/harness/deerflow/agents/middlewares/safety_finish_reason_middleware.py#L170-L205)）：通过 `get_stream_writer()` 发一个 `"safety_termination"` 自定义 SSE 事件，让前端能够实时"撤销"已经渲染出来的"工具执行中..."占位符。这是给**当前正在观看的用户**用的，run 结束后这个事件就不存在了。整个函数包在 try/except 里，任何失败都只记 debug 日志——这是一个 best-effort 的通知，不该因为拿不到 stream writer（比如非流式调用场景）就影响主流程。
- `_record_audit_event`（[safety_finish_reason_middleware.py:207-262](../backend/packages/harness/deerflow/agents/middlewares/safety_finish_reason_middleware.py#L207-L262)）：往 `RunJournal`（通过 `runtime.context["__run_journal"]` 拿到，在 unit test/subagent 等场景可能不存在，直接跳过）写一条持久化记录，供事后"今天有哪些 run 被安全过滤"这类审计查询使用。

**两者分工不同，缺一不可**：SSE 是给正在看屏幕的人看的即时反馈，`RunJournal` 是给以后想复盘的人用的持久记录，两者的失败模式也不同（一个是连接层面的最佳努力，一个是数据库/存储层面的最佳努力），所以分成两个独立函数、独立 try/except，一个挂了不影响另一个。

**故意不做的事**（[safety_finish_reason_middleware.py:225-228](../backend/packages/harness/deerflow/agents/middlewares/safety_finish_reason_middleware.py#L225-L228)）：

> Tool **arguments** are deliberately **not** recorded — those are the very content the provider filtered; persisting them would defeat the purpose of the safety filter.

只记录被抑制的工具**名字**、**数量**、**id**，绝不记录**参数**本身——因为参数正是被 provider 判定为不安全/被截断的那部分内容，如果为了"审计方便"把它完整存进日志/数据库，等于绕过了安全过滤器本来要阻止的事情。这是一个"可观测性设计也要考虑安全边界"的好例子：不是"记录得越详细越好"，而是要问"这条记录本身会不会重新引入被过滤的风险"。

### 2.6 为什么用 `after_model` 而不是 `wrap_model_call`

[safety_finish_reason_middleware.py:21-24](../backend/packages/harness/deerflow/agents/middlewares/safety_finish_reason_middleware.py#L21-L24)：

> Hook choice: `after_model` (not `wrap_model_call`) because the response is a *normal* return — not an exception — and we want to participate in the same after-model chain as `LoopDetectionMiddleware`.

这里的对比对象不是"消息配对约束"（那是 Loop 选择延迟到 `wrap_model_call` 注入警告的原因），而是"这次调用本身是不是一次正常返回"。`wrap_model_call` 通常用来包裹**请求前/请求后**的逻辑，或者处理**异常**（比如 `LLMErrorHandlingMiddleware` 捕获调用失败）。而 provider 安全截断产生的是一次**正常返回**的 `AIMessage`（只是内容不完整），不是异常——用 `after_model` 去检查这条已经生成的消息本身，语义上更直接，而且能跟 Loop 共享同一个"事后检查 AIMessage、按需清空 tool_calls"的责任链位置。

## 第三部分：注册顺序——一段可以直接引用 LangChain 源码的机制

### 3.1 反转执行顺序的确切依据

`SafetyFinishReasonMiddleware` 的模块 docstring 里直接引用了 LangChain 内部实现（[safety_finish_reason_middleware.py:26-32](../backend/packages/harness/deerflow/agents/middlewares/safety_finish_reason_middleware.py#L26-L32)）：

> LangChain factory wires `after_model` edges in reverse list order (`langchain/agents/factory.py:add_edge("model", middleware_w_after_model[-1])`, then walks `range(len-1, 0, -1)`), so the *last* registered middleware is the *first* to observe the model output.

也就是说：`before_model`/`wrap_model_call` 按 middleware **注册顺序（正序）**执行；但 `after_model` 是**反着**执行的——最后一个 `append` 进 middleware 列表的中间件，第一个看到模型的原始输出。这不是 DeerFlow 自己发明的规则，是 LangChain agent 工厂的既有行为，DeerFlow 的中间件排列必须顺着它设计。

### 3.2 实际的 append 顺序

`build_middlewares`（[agent.py:351-373](../backend/packages/harness/deerflow/agents/lead_agent/agent.py#L351-L373)）里，跟本节相关的三个中间件按这个顺序被 append：

```python
middlewares.append(SubagentLimitMiddleware(...))              # 第351-355行
middlewares.append(LoopDetectionMiddleware.from_config(...))  # 第357-360行
# ...custom_middlewares 插在中间...
middlewares.append(SafetyFinishReasonMiddleware.from_config(...))  # 第366-373行
middlewares.append(ClarificationMiddleware())                 # 最后
```

按"最后 append 的最先执行"这条规则反推，`after_model` 的**实际执行顺序**是：

**Safety → Loop → SubagentLimit**

### 3.3 为什么这个顺序是对的

- **Safety 先跑**：如果 provider 安全过滤触发了，Safety 把 `tool_calls` 整个清空。这个"清理"发生在 Loop 统计之前，所以 Loop 永远不会把"被安全过滤截断的半成品调用"计入它的 hash/frequency 统计——这正是 Safety 自己文档里写的"Loop then accounts against the cleaned message"。如果反过来（Loop 先跑），Loop 会先对着一个**将被清空**的、可能参数残缺的 tool_calls 做统计，这个统计毫无意义，甚至可能因为多次触发安全过滤而误判成"循环"，用错误的原因（循环）掩盖了真正的原因（安全过滤）。

- **Loop 其次**：在 Safety 已经排除了"provider 主动截断"这种情况之后，Loop 看到的都是模型**真实想发起**的完整 tool_calls 集合，可以放心地做 hash 判重和频率统计。

- **SubagentLimit 最后跑**：有了它的源码（[subagent_limit_middleware.py:41-68](../backend/packages/harness/deerflow/agents/middlewares/subagent_limit_middleware.py#L41-L68)），可以确切回答第一模块留下的思考题——"如果对调 SubagentLimit 和 Loop 的 append 顺序会怎样"。当前顺序下 SubagentLimit **最后**执行，意味着 Loop 统计 hash 时看到的是模型**原始的、未截断**的 task 调用集合（哪怕模型一次发起了 5 个并行 task，超过 `max_concurrent=3`）。如果对调过来（让 SubagentLimit 先于 Loop 执行），Loop 看到的就是已经被砍到只剩前 3 个的 task 调用——这时候如果模型每次都倾向于把同样的前几个子任务排在前面（截断规则是"保留前 N 个，丢弃剩下的"，[subagent_limit_middleware.py:60-61](../backend/packages/harness/deerflow/agents/middlewares/subagent_limit_middleware.py#L60-L61)），Loop 可能会对着"每次都截断成相同前 3 个"这种由截断规则本身造成的假象误判为循环，而真实情况可能是模型每次都在请求不同的 5 个任务、只是被并发上限反复砍掉了后两个。**顺序错了会把"并发限制的副作用"误诊断成"agent 在打转"。**

一句话总结这条设计链：**清理类中间件（Safety、SubagentLimit）不应该在诊断类中间件（Loop）之前"污染"它要分析的原始信号**——除非清理的正是诊断类中间件本该忽略的噪声本身（这里是安全过滤的截断），此时清理必须先于诊断。

## 总结表

| 概念 | 关键文件 | 一句话记忆点 |
|---|---|---|
| `_stable_tool_key` 分桶/白名单 | [loop_detection_middleware.py:99-139](../backend/packages/harness/deerflow/agents/middlewares/loop_detection_middleware.py#L99-L139) | 不同工具"什么算重复"语义不同：read_file 分桶模糊匹配，write_file/str_replace 全参数精确匹配 |
| 顺序无关 hash | [loop_detection_middleware.py:142-160](../backend/packages/harness/deerflow/agents/middlewares/loop_detection_middleware.py#L142-L160) | 先排序再 hash，避免并行调用顺序的随机性破坏判重 |
| 两层检测 | [loop_detection_middleware.py:322-438](../backend/packages/harness/deerflow/agents/middlewares/loop_detection_middleware.py#L322-L438) | hash 抓"完全一样的调用"，frequency 抓"同工具不同参数的滥用" |
| 待注入警告队列 | [loop_detection_middleware.py:229-233](../backend/packages/harness/deerflow/agents/middlewares/loop_detection_middleware.py#L229-L233) + `_augment_request` | after_model 检测但不改消息；wrap_model_call 时消息列表已完整，才安全注入 HumanMessage |
| Hard stop 改写 finish_reason | [loop_detection_middleware.py:457-475](../backend/packages/harness/deerflow/agents/middlewares/loop_detection_middleware.py#L457-L475) | tool_calls 清空后消息语义已变成纯文本，此时可以安全原地重写 |
| 可插拔安全检测器 | [safety_finish_reason_middleware.py:70-100](../backend/packages/harness/deerflow/agents/middlewares/safety_finish_reason_middleware.py#L70-L100) | 反射加载 + 拒绝空 list（防止"启用但形同虚设"） |
| 保留真实 finish_reason | [safety_finish_reason_middleware.py:148-150](../backend/packages/harness/deerflow/agents/middlewares/safety_finish_reason_middleware.py#L148-L150) | 与 Loop 的 hard stop 相反：只在旧值是 tool_calls 时才重写，安全过滤原因要原样透传给下游 |
| SSE vs RunJournal 双轨审计 | [safety_finish_reason_middleware.py:170-262](../backend/packages/harness/deerflow/agents/middlewares/safety_finish_reason_middleware.py#L170-L262) | 实时通知 vs 持久审计分离；且故意不记录被过滤的参数本身 |
| after_model 反序执行 | [safety_finish_reason_middleware.py:26-32](../backend/packages/harness/deerflow/agents/middlewares/safety_finish_reason_middleware.py#L26-L32) | LangChain 源码级机制：最后 append 的中间件最先执行 after_model |
| 三方注册顺序契约 | [agent.py:351-373](../backend/packages/harness/deerflow/agents/lead_agent/agent.py#L351-L373) | 实际执行序 Safety → Loop → SubagentLimit：清理噪声必须先于诊断，诊断必须先于并发截断 |
