# 安全中间件(循环检测/兜底/消息补偿/错误处理)—— 面试问题链

> 本文档按"大厂面试追问链"组织:每条问题链从基础问题开始,层层追问到实现细节、设计权衡、异常边界。配套深读笔记:[02-loop-detection-and-safety.md](02-loop-detection-and-safety.md)、[03-dangling-tool-call.md](03-dangling-tool-call.md)(深读笔记讲"怎么实现",本文档讲"怎么被问、怎么答")。
> 代码引用格式:[文件名.py:起始行-结束行](../backend/仓库相对路径#L起始行-L结束行)。只引用你实际读过的行,禁止编造行号。

涉及的核心文件(均在 `backend/packages/harness/deerflow/agents/middlewares/` 下):

- `loop_detection_middleware.py`:循环检测,hash + frequency 两层策略,先警告后强制收尾
- `safety_finish_reason_middleware.py` + `safety_termination_detectors.py`:provider 安全截断时抑制半截 tool_calls
- `dangling_tool_call_middleware.py`:用户中断后补偿缺失的 ToolMessage
- `llm_error_handling_middleware.py`:LLM 错误分类、重试、熔断、归一化为兜底 AIMessage
- `tool_error_handling_middleware.py`:工具异常转 error ToolMessage,保证 run 继续
- `clarification_middleware.py`:`ask_clarification` 拦截 + `Command(goto=END)` 中断
- `tool_call_metadata.py`:`clone_ai_message_with_tool_calls` 同步结构化与原始 tool_calls 元数据

阅读前提(深读笔记已覆盖、本文不再展开的点):

- LangChain `AgentMiddleware` 的三组 hook:`before_agent/before_model/after_model/after_agent` 是"通知+返回 state 更新",`wrap_model_call/wrap_tool_call` 是"洋葱包裹 handler,可改 request 或短路返回"。
- `add_messages` reducer:按消息 id 替换、无 id 追加;这决定了为什么"插到中间"必须走 `wrap_model_call` 的 `request.override`。
- provider 配对校验:OpenAI/Moonshot 要求 assistant 的 tool_calls 紧随其后必须是对应 ToolMessage,否则请求级 400。
- `Command(goto=END)`:LangGraph 的控制流对象,既可携带 state 更新(update)又可指定跳转目标(goto),工具中间件借此实现"带结果的中断"。

面试前建议按顺序自测:能否不看代码讲清"警告为何延迟注入"、"Safety 与 Loop 的先后"、"硬停的三处清理"、"dangling 补丁为何不落 state"、"LLM 错误与工具错误为何分流"、"Clarification 为何殿后"——这六问是本文的骨架。

全景链路(一次模型-工具往返中各安全件的出手点):

```
        ┌────────────────────── agent loop ──────────────────────┐
        │                                                        │
 history ─► DanglingToolCall(wrap_model_call 前修补配对)           │
        ─► LoopDetection(wrap_model_call 注入排队警告)             │
        ─► LLMErrorHandling(wrap_model_call 重试/熔断/兜底)        │
        ─► model                                                 │
        ─► after_model 逆序链: SafetyFinishReason(剥截断调用)      │
              → LoopDetection(记账/排队警告/硬停)                  │
              → SubagentLimit(截断超额 task 调用)                  │
        ─► ToolNode: ToolErrorHandling(异常→ToolMessage)           │
              → Clarification(ask_clarification→goto=END)         │
        └────────────────────────────────────────────────────────┘
```

## 问题链 1:循环检测的两层策略

**Q1.1(基础)** 我看你们 agent 里有个 LoopDetectionMiddleware,它解决什么问题?基本思路是什么?

**参考回答**:这是 P0 级安全网,防止 agent 用相同参数无限调用同一个工具、直到 recursion limit 把整个 run 打死。思路是:每次模型响应后,把 tool_calls(name + args)算成一个 hash,放进滑动窗口里计数;同一 hash 出现次数达到 `warn_threshold`(默认 3 次)就注入一条"你在重复自己,请收尾"的警告;达到 `hard_limit`(默认 5 次)就直接剥掉 tool_calls,强制模型产出纯文本最终答案。默认值定义在 [loop_detection_middleware.py:64-70](../backend/packages/harness/deerflow/agents/middlewares/loop_detection_middleware.py#L64-L70),检测入口在 `_track_and_check` [loop_detection_middleware.py:322-344](../backend/packages/harness/deerflow/agents/middlewares/loop_detection_middleware.py#L322-L344)。整套设计(检测、延迟注入、硬停)在模块 docstring 里有完整动机说明,见 [loop_detection_middleware.py:1-17](../backend/packages/harness/deerflow/agents/middlewares/loop_detection_middleware.py#L1-L17)。

**链路解析**:

```
model response (AIMessage with tool_calls)
        │
        ▼
after_model → _track_and_check()
        │  hash = md5(sorted(name:stable_key))[:12]
        │  history[thread_id].append(hash)   # window=20,超出截尾
        ▼
┌─ Layer 1: hash 层(完全相同的调用集合)─────────────┐
│ count >= hard_limit(5)? ──yes──► return (HARD_STOP, True)   │
│ count >= warn_threshold(3)? ──yes──► return (WARNING, False)│
└──────────────────────────────────────────────────┘
        │ 未命中
        ▼
┌─ Layer 2: frequency 层(同工具名累计,不看参数)────┐
│ freq[name] += 1                                   │
│ count >= eff_hard(默认50,可被 override)? ──► 强停  │
│ count >= eff_warn(默认30)? ──► 频率警告(每工具一次)│
└──────────────────────────────────────────────────┘
        │
        ▼
   放行,进入 tools 节点
```

**Q1.2(深挖)** 这个 hash 具体怎么算的?直接 `json.dumps(args)` 不就行了,为什么还要搞 `_stable_tool_key` 这么复杂?

**参考回答**:直接全参数 hash 会有两个方向的问题:`read_file` 这类工具,模型每次微调 `start_line`/`end_line` 就会产生不同 hash,漏报"反复读同一文件同一区域"的循环;而 `write_file`/`str_replace` 是内容敏感的,同一路径不同内容是正常迭代,只看 path 又会误报。所以 `_stable_tool_key` 对 `read_file` 用 path + 200 行的 bucket 区间做键(`bucket_size = 200`,行号还被归一化排序、非法值兜底为 1),对 `write_file`/`str_replace` 保留全参数 hash,其余工具只取 `path/url/query/command/pattern/glob/cmd` 这些 salient 字段,见 [loop_detection_middleware.py:99-139](../backend/packages/harness/deerflow/agents/middlewares/loop_detection_middleware.py#L99-L139)。整个 multiset 排序后取 md5 前 12 位,保证并行 tool_calls 的顺序不影响结果,见 [loop_detection_middleware.py:142-160](../backend/packages/harness/deerflow/agents/middlewares/loop_detection_middleware.py#L142-L160)。参数本身还有一层防御:有的 provider 把 args 序列化成 JSON 字符串,`_normalize_tool_call_args` 会先尝试解析,失败则保留稳定 fallback key,见 [loop_detection_middleware.py:73-96](../backend/packages/harness/deerflow/agents/middlewares/loop_detection_middleware.py#L73-L96)。

**Q1.3(深挖)** 有了 hash 层为什么还要 frequency 层?两层各自的盲区是什么?

**参考回答**:hash 层只能抓"完全相同的一组调用",抓不到"同一个工具换着参数调"的循环——典型场景是模型依次 `read_file` 40 个不同文件,每次 hash 都不同,hash 层永远沉默。frequency 层按工具名(不看参数)累计每线程调用次数,默认 `tool_freq_warn=30` 次警告、`tool_freq_hard_limit=50` 次强停,实现见 [loop_detection_middleware.py:399-436](../backend/packages/harness/deerflow/agents/middlewares/loop_detection_middleware.py#L399-L436),这个动机在构造函数 docstring 里写得很直白("Catches cross-file read loops that hash-based detection misses"),见 [loop_detection_middleware.py:189-194](../backend/packages/harness/deerflow/agents/middlewares/loop_detection_middleware.py#L189-L194)。frequency 层自己的盲区是误伤合法的高频工具(比如批处理流水线里 bash 调几百次很正常),所以留了 `tool_freq_overrides` 按工具名单独抬阈值,见 [loop_detection_middleware.py:408-411](../backend/packages/harness/deerflow/agents/middlewares/loop_detection_middleware.py#L408-L411),配置侧用 Pydantic 校验 `hard_limit >= warn`,见 [loop_detection_config.py:66-73](../backend/packages/harness/deerflow/config/loop_detection_config.py#L66-L73)。

**Q1.4(边界/异常)** 这个中间件是有状态的,多线程并发、长跑服务场景下状态怎么管?会不会内存涨爆或者串线程?

**参考回答**:所有共享状态都收在 `threading.Lock` 内,按 `thread_id` 分桶:`_history` 是 `OrderedDict` 做 LRU,超过 `max_tracked_threads`(默认 100)就淘汰最旧线程并连带清掉它的 warned/freq 状态,见 [loop_detection_middleware.py:266-279](../backend/packages/harness/deerflow/agents/middlewares/loop_detection_middleware.py#L266-L279)。待注入警告按 `(thread_id, run_id)` 二元组隔离,单 run 最多攒 4 条(`_MAX_PENDING_WARNINGS_PER_RUN = 4`,超出从头部丢弃),pending key 总数上限是 `max(1, max_tracked_threads * 2) = 200`,见 [loop_detection_middleware.py:70](../backend/packages/harness/deerflow/agents/middlewares/loop_detection_middleware.py#L70)、[loop_detection_middleware.py:231-233](../backend/packages/harness/deerflow/agents/middlewares/loop_detection_middleware.py#L231-L233) 和 [loop_detection_middleware.py:310-320](../backend/packages/harness/deerflow/agents/middlewares/loop_detection_middleware.py#L310-L320)。同线程新 run 启动时 `before_agent` 会清掉旧 run 的残留警告,run 结束 `after_agent` 清当前 run,见 [loop_detection_middleware.py:524-550](../backend/packages/harness/deerflow/agents/middlewares/loop_detection_middleware.py#L524-L550)。测试/运维侧还留了 `reset(thread_id)` 全量或按线程清理的出口,见 [loop_detection_middleware.py:595-612](../backend/packages/harness/deerflow/agents/middlewares/loop_detection_middleware.py#L595-L612)。

## 问题链 2:优雅降级——警告为什么必须延迟注入

**Q2.1(基础)** 检测到循环之后你们的处理是"先警告后强停",这个警告是怎么发给模型的?

**参考回答**:警告不是改历史,而是在下一次模型调用前,由 `wrap_model_call` 把排队中的警告合并成一条 `HumanMessage`(name=`loop_warning`)追加到请求消息列表末尾,见 [loop_detection_middleware.py:560-577](../backend/packages/harness/deerflow/agents/middlewares/loop_detection_middleware.py#L560-L577)。检测发生在 `after_model`,但 `after_model` 里只把警告推入 `(thread_id, run_id)` 作用域的队列就返回 `None`,不动任何已有消息,见 [loop_detection_middleware.py:491-500](../backend/packages/harness/deerflow/agents/middlewares/loop_detection_middleware.py#L491-L500)。这样模型下一轮看到的序列是:AIMessage(tool_calls) → ToolMessage×N → loop_warning,配对完整;同一 hash 只警告一次(`_warned` 集合记账,且随窗口滑出自动清理),见 [loop_detection_middleware.py:362-366](../backend/packages/harness/deerflow/agents/middlewares/loop_detection_middleware.py#L362-L366) 和 [loop_detection_middleware.py:384-397](../backend/packages/harness/deerflow/agents/middlewares/loop_detection_middleware.py#L384-L397)。

**链路解析**:

```
轮次 N:  after_model 检测到 count>=3
            │  (此时 tools 节点还没跑,历史里没有 ToolMessage)
            ▼
        _queue_pending_warning() ── 入队,不改消息
            │
轮次 N+1: wrap_model_call
            │  _drain_pending_warnings() 取出并清空
            ▼
 [... AIMessage(tool_calls), ToolMessage, ToolMessage, HumanMessage(loop_warning)]
            │
            ▼
          LLM
```

**Q2.2(深挖)** 为什么不能在 `after_model` 里直接把警告消息塞进 state?那样不是更简单吗?

**参考回答**:不行,这是这个模块最核心的设计约束。`after_model` 触发时模型刚产出带 tool_calls 的 AIMessage,tools 节点还没执行,历史里不存在对应的 ToolMessage;此时往里塞任何非 ToolMessage,都会落在 assistant 的 tool_calls 和它们的响应之间,下一次请求发给 OpenAI/Moonshot 会被校验器拒绝:`"tool_call_ids did not have response messages"`;Anthropic 那边也不允许 mid-stream 的 SystemMessage。这段权衡写在模块 docstring [loop_detection_middleware.py:18-32](../backend/packages/harness/deerflow/agents/middlewares/loop_detection_middleware.py#L18-L32)。**不这样设计会怎样**:如果图省事在 `after_model` 直接 append 一条 HumanMessage,第一次触发警告的那一轮请求直接 400,循环没打破反而引入新的确定性失败;如果改成篡改那条 AIMessage 的 content,又等于"把框架的话塞进模型嘴里",会污染 MemoryMiddleware 这类下游消费者,见 [loop_detection_middleware.py:491-498](../backend/packages/harness/deerflow/agents/middlewares/loop_detection_middleware.py#L491-L498)。

**Q2.3(深挖)** 警告不听怎么办?硬停止具体怎么实现的,光把 `tool_calls` 置空就够了吗?

**参考回答**:不够,要同时清三处。`_build_hard_stop_update` 除了把结构化 `tool_calls` 置空,还要从 `additional_kwargs` 里删掉原始的 `tool_calls` 和 `function_call` 字段(provider 适配器序列化时会读这些原始载荷,不清等于白剥),再把 `response_metadata.finish_reason` 从 `"tool_calls"` 改成 `"stop"`,见 [loop_detection_middleware.py:457-475](../backend/packages/harness/deerflow/agents/middlewares/loop_detection_middleware.py#L457-L475)。content 也不能简单字符串拼接:Anthropic thinking 模式下 content 是 block 列表,`_append_text` 会追加一个 `{"type": "text"}` block 而不是 `list + str` 触发 TypeError,见 [loop_detection_middleware.py:440-455](../backend/packages/harness/deerflow/agents/middlewares/loop_detection_middleware.py#L440-L455)。因为 tool_calls 已剥光,这条 AIMessage 不再需要配对 ToolMessage,原地改写是安全的,这点在 `_apply` 的注释里写明,见 [loop_detection_middleware.py:480-489](../backend/packages/harness/deerflow/agents/middlewares/loop_detection_middleware.py#L480-L489)。**不这样设计会怎样**:只清结构化字段、留着 `additional_kwargs.tool_calls`,OpenAI 适配器下次仍会按原始载荷发出 tool_calls,硬停形同虚设;finish_reason 不改,下游 SSE/转换器会继续把这条消息当成"待执行工具"渲染。

**链路解析**(硬停止的消息重写):

```
原 AIMessage:
  content            = "..."
  tool_calls         = [tc1, tc2]                 ← 置 []
  additional_kwargs  = {tool_calls: [...],        ← pop
                        function_call: {...}}     ← pop
  response_metadata  = {finish_reason: "tool_calls"} ← 改 "stop"
        │
        ▼  model_copy(update=_build_hard_stop_update(...))
新 AIMessage: 纯文本助手消息,无需配对 ToolMessage
        │
        ▼  after_model 返回 {"messages": [stripped]}
add_messages 按同 id 替换原消息 → tools 节点无调用可执行 → agent 收尾
```

**Q2.4(边界/异常)** 如果警告刚入队,这个 run 就结束了(用户点了停止),警告会带到下一轮甚至别的 run 吗?

**参考回答**:不会。pending 警告按 `(thread_id, run_id)` 双键作用域:同线程新 run 的 `before_agent` 会清掉该线程所有非当前 run 的残留(`_clear_other_run_pending_warnings`),当前 run 结束时 `after_agent` 清掉自己的,见 [loop_detection_middleware.py:504-516](../backend/packages/harness/deerflow/agents/middlewares/loop_detection_middleware.py#L504-L516) 和 [loop_detection_middleware.py:542-550](../backend/packages/harness/deerflow/agents/middlewares/loop_detection_middleware.py#L542-L550)。设计意图在 docstring:"Queued warnings are intentionally transient",见 [loop_detection_middleware.py:34-38](../backend/packages/harness/deerflow/agents/middlewares/loop_detection_middleware.py#L34-L38)。注意 hash 历史本身是按 thread 保留的,不随 run 清——持续复读的线程跨 run 仍会撞上 hard_limit,这是有意的。

## 问题链 3:safety finish reason 兜底

**Q3.1(基础)** provider 因为安全原因截断响应,会发生什么?你们怎么处理?

**参考回答**:OpenAI 的 `finish_reason='content_filter'`、Anthropic 的 `stop_reason='refusal'`、Gemini 的 `SAFETY` 这类信号,会在流式生成中途掐断输出,但 LangChain 仍可能把半截的 `tool_calls` 解析出来;tool router 只看 `tool_calls` 非空就去执行,于是一个写到一半的 `write_file` 被当成完整调用执行,agent 看到截断文件想修,又被过滤,形成循环(issue #3028)。`SafetyFinishReasonMiddleware` 在 `after_model` 检测:若 detector 命中且 AIMessage 带 tool_calls,就剥掉 tool calls、追加一段用户可读的说明、并在 `additional_kwargs.safety_termination` 里留下观测记录,见 [safety_finish_reason_middleware.py:1-19](../backend/packages/harness/deerflow/agents/middlewares/safety_finish_reason_middleware.py#L1-L19) 和 [safety_finish_reason_middleware.py:266-307](../backend/packages/harness/deerflow/agents/middlewares/safety_finish_reason_middleware.py#L266-L307)。

**链路解析**:

```
provider stream ──finish_reason=content_filter + 半截 tool_calls──► AIMessage
        │
        ▼  after_model (逆序: Safety 最先看到原始响应)
SafetyFinishReasonMiddleware._detect() 命中?
        │yes
        ▼
clone_ai_message_with_tool_calls(msg, [], content=原content+说明)
        │  剥 tool_calls / additional_kwargs.tool_calls / function_call
        ▼
additional_kwargs["safety_termination"] = {detector, reason, suppressed_names...}
        │
        ├─► _emit_event(): SSE 通知前端撤掉 "tool starting..." 占位
        └─► _record_audit_event(): RunJournal 落审计(不记录 args!)
```

**Q3.2(深挖)** 各家 provider 的信号字段不一样,你们的检测器体系怎么设计的?新增一个 provider 要改核心代码吗?

**参考回答**:不用改核心代码。检测抽象成 `SafetyTerminationDetector` Protocol(一个 `name` 属性 + 一个 `detect(message)` 方法),命中返回 frozen dataclass `SafetyTermination`,见 [safety_termination_detectors.py:23-58](../backend/packages/harness/deerflow/agents/middlewares/safety_termination_detectors.py#L23-L58)。内置三个:OpenAI 兼容系(默认匹配 `content_filter`,可扩展 `sensitive`/`violation` 等国产网关 token)、Anthropic `refusal`、Gemini 一组大写枚举(默认 8 个:SAFETY/BLOCKLIST/PROHIBITED_CONTENT/SPII/RECITATION 加 3 个图像类),见 [safety_termination_detectors.py:82-119](../backend/packages/harness/deerflow/agents/middlewares/safety_termination_detectors.py#L82-L119)、[safety_termination_detectors.py:122-144](../backend/packages/harness/deerflow/agents/middlewares/safety_termination_detectors.py#L122-L144) 和 [safety_termination_detectors.py:183-194](../backend/packages/harness/deerflow/agents/middlewares/safety_termination_detectors.py#L183-L194)。读取时同时查 `response_metadata` 和 `additional_kwargs` 两个容器、只接受 str 值,容忍各家适配器的不一致,见 [safety_termination_detectors.py:61-79](../backend/packages/harness/deerflow/agents/middlewares/safety_termination_detectors.py#L61-L79)。自定义 detector 通过 `config.yaml` 的 `safety_finish_reason.detectors` 用类路径反射加载,和 guardrails 同一套 `resolve_variable` 机制,见 [safety_finish_reason_middleware.py:90-100](../backend/packages/harness/deerflow/agents/middlewares/safety_finish_reason_middleware.py#L90-L100) 和 [safety_finish_reason_config.py:26-47](../backend/packages/harness/deerflow/config/safety_finish_reason_config.py#L26-L47)。Gemini 默认集刻意排除 `MAX_TOKENS`、`MALFORMED_FUNCTION_CALL` 等——它们不是安全过滤,归类进来会污染观测口径,取舍写在类 docstring,见 [safety_termination_detectors.py:158-178](../backend/packages/harness/deerflow/agents/middlewares/safety_termination_detectors.py#L158-L178)。

**链路解析**(detector 决策流):

```
AIMessage ──► _get_metadata_value(msg, field)
              │  先 response_metadata 后 additional_kwargs,只认 str
              ▼
┌─ OpenAICompatibleContentFilterDetector ─────────────┐
│ finish_reason.lower() ∈ {"content_filter", ...}?     │──命中──► SafetyTermination
└──────────────────────────────────────────────────────┘   (+ extras.content_filter_results)
              │未命中
              ▼
┌─ AnthropicRefusalDetector ───────────────────────────┐
│ stop_reason.lower() ∈ {"refusal"}?                   │──命中──► SafetyTermination
└──────────────────────────────────────────────────────┘
              │未命中
              ▼
┌─ GeminiSafetyDetector ───────────────────────────────┐
│ finish_reason.upper() ∈ 8 个默认安全枚举?             │──命中──► SafetyTermination
└──────────────────────────────────────────────────────┘   (+ extras.safety_ratings)
              │全部未命中
              ▼
            None → 原样放行
```

**Q3.3(深挖)** 文档里写"注册在 LoopDetectionMiddleware 之后",但两个都是 `after_model`,后注册不是后执行吗?

**参考回答**:恰恰相反——LangChain factory 给 `after_model` 拉边是逆序的:列表里**最后**注册的中间件**最先**看到模型输出,docstring 里引用了 factory 的 `add_edge` 逆序逻辑,见 [safety_finish_reason_middleware.py:26-32](../backend/packages/harness/deerflow/agents/middlewares/safety_finish_reason_middleware.py#L26-L32)。所以注册顺序 Safety 在 Loop 之后,执行顺序是 Safety 先:Safety 先把被过滤的半截 tool_calls 剥干净,Loop 再基于清洗后的消息记账,不会对一次注定不执行的调用误计一次循环。agent 组装处的注释也强调了这点,见 [agent.py:366-373](../backend/packages/harness/deerflow/agents/lead_agent/agent.py#L366-L373)。**不这样设计会怎样**:若 Loop 先执行,被 Safety 抑制的每一轮都会被 Loop 记进 hash 窗口,模型反复撞 content_filter 时会被误判为"复读循环",警告消息和 Safety 说明互相打架,甚至提前触发 hard stop。这也解释了为什么两者共用"剥 tool_calls"的 mechanic 但触发器不同、且都实现了 sync/async 双 hook,见 [safety_finish_reason_middleware.py:21-24](../backend/packages/harness/deerflow/agents/middlewares/safety_finish_reason_middleware.py#L21-L24)。

**Q3.4(边界/异常)** detector 本身出 bug 抛异常怎么办?配置成空列表想关掉检测行不行?content_filter 但没带 tool_calls 呢?

**参考回答**:三种都有明确答案。detector 异常被逐条 try/except 吞掉、记 exception 日志、当作未命中继续下一个——"never let a buggy detector break the agent run",见 [safety_finish_reason_middleware.py:104-113](../backend/packages/harness/deerflow/agents/middlewares/safety_finish_reason_middleware.py#L104-L113)。显式空列表直接 `ValueError` 拒绝启动,因为它会"让中间件留在链里却静默关闭检测,是最差的两头不占",想关应该用 `enabled: false`,见 [safety_finish_reason_middleware.py:76-88](../backend/packages/harness/deerflow/agents/middlewares/safety_finish_reason_middleware.py#L76-L88)。content_filter 但没有 tool_calls 的响应原样放行——此时没有需要抑制的东西,部分文本仍可自然到达用户,见 [safety_finish_reason_middleware.py:275-280](../backend/packages/harness/deerflow/agents/middlewares/safety_finish_reason_middleware.py#L275-L280)。另外审计事件刻意不记录 tool 参数——那正是被过滤的内容,落盘就违背了过滤的意义,只记 names/count/ids,见 [safety_finish_reason_middleware.py:225-228](../backend/packages/harness/deerflow/agents/middlewares/safety_finish_reason_middleware.py#L225-L228) 和 [safety_finish_reason_middleware.py:238-250](../backend/packages/harness/deerflow/agents/middlewares/safety_finish_reason_middleware.py#L238-L250)。

## 问题链 4:dangling tool call 补偿

**Q4.1(基础)** 用户在工具执行中途点了取消,下一轮对话会遇到什么问题?怎么修?

**参考回答**:中断时历史里会留下一条带 `tool_calls` 的 AIMessage,但没有对应的 ToolMessage,这就是 dangling tool call;下一次请求发给 OpenAI 系 provider 会因消息格式不完整被 400 拒掉("tool_call_ids did not have response messages")。`DanglingToolCallMiddleware` 在每次模型调用前扫描历史,给每个没有响应的 tool_call 合成一条 `status="error"` 的 ToolMessage(内容是"[Tool call was interrupted and did not return a result.]"),紧跟在发起它的 AIMessage 后面插入,见 [dangling_tool_call_middleware.py:1-13](../backend/packages/harness/deerflow/agents/middlewares/dangling_tool_call_middleware.py#L1-L13) 和 [dangling_tool_call_middleware.py:150-176](../backend/packages/harness/deerflow/agents/middlewares/dangling_tool_call_middleware.py#L150-L176)。它在运行时中间件装配里由 `include_dangling_tool_call_patch` 开关控制,lead agent 与 subagent 都默认开启,见 [tool_error_handling_middleware.py:153-156](../backend/packages/harness/deerflow/agents/middlewares/tool_error_handling_middleware.py#L153-L156)。

**链路解析**:

```
中断后的历史:
  HumanMessage → AIMessage(tool_calls=[A,B]) → (没有 ToolMessage)
        │
        ▼ wrap_model_call → _build_patched_messages()
  1) 收集所有 ToolMessage,按 tool_call_id 建队列
  2) 重放消息流:AIMessage 之后立刻按序补出每个 tc 的 ToolMessage
        │  有真的用真的,没有就合成 error ToolMessage
        ▼
  HumanMessage → AIMessage(tc A,B) → TM(A) → TM(B, synthetic error) → LLM
```

**Q4.2(深挖)** 为什么用 `wrap_model_call` 而不是 `before_model`?后者不是更常规吗?

**参考回答**:关键是插入位置。`before_model` 返回 `{"messages": [...]}` 要走 `add_messages` reducer,新消息只能**追加到列表末尾**;而补丁 ToolMessage 必须**紧跟在发起它的那条 AIMessage 后面**——历史里可能有多轮 dangling AIMessage,追加到末尾既破坏配对顺序,也还是被 provider 校验拒绝。`wrap_model_call` 拿到的是完整 request,可以在任意位置重排后再 `request.override(messages=patched)` 交给 handler,这个理由写在文件 docstring,见 [dangling_tool_call_middleware.py:10-13](../backend/packages/harness/deerflow/agents/middlewares/dangling_tool_call_middleware.py#L10-L13)。另外补丁只作用于发给模型的 request,不写回 graph state,持久化历史保持原样、不污染审计,见 [dangling_tool_call_middleware.py:185-194](../backend/packages/harness/deerflow/agents/middlewares/dangling_tool_call_middleware.py#L185-L194)。**不这样设计会怎样**:用 `before_model` + reducer 追加,补丁全部堆到历史末尾,离它们该配对的 AIMessage 隔着十万八千里,provider 照样 400;若改成把补丁写进 state,等于伪造"工具已执行且报错"的持久事实,后续 summarization、memory、前端回放都会看到从未发生的工具结果。

**链路解析**(两种 hook 的落点对比):

```
历史:  H → AI(tc1) → TM(tc1) → AI(tc2,tc3) → (中断,无 TM)

方案 A: before_model + add_messages  ✗
  H → AI(tc1) → TM(tc1) → AI(tc2,tc3) → [TM'(tc2), TM'(tc3)]  ← 只能追加到末尾
  provider 视角: AI(tc2,tc3) 与 TM'(tc2) 之间隔着整个尾部…仍判配对失败?不,
  更糟的是若后面还有新 HumanMessage,补丁会落在它之后,配对彻底乱序。

方案 B: wrap_model_call + request.override  ✓
  H → AI(tc1) → TM(tc1) → AI(tc2,tc3) → TM'(tc2) → TM'(tc3) → (后续消息)
  补丁紧贴各自 AIMessage;只改 request,graph state 里的历史原封不动。
```

**Q4.3(深挖)** LangChain 还有 `invalid_tool_calls`(解析失败的调用),也 dangling 吗?`write_file` 有特判是为什么?

**参考回答**:`invalid_tool_calls` 不会被执行,但 provider 适配器下次序列化时仍可能把 call id/name 带回请求,严格校验器照样要求配对 ToolMessage,所以一并按 dangling 处理,合成"[Tool call could not be executed because its arguments were invalid]",见 [dangling_tool_call_middleware.py:43-53](../backend/packages/harness/deerflow/agents/middlewares/dangling_tool_call_middleware.py#L43-L53) 和 [dangling_tool_call_middleware.py:88-99](../backend/packages/harness/deerflow/agents/middlewares/dangling_tool_call_middleware.py#L88-L99)。`write_file` 特判源于 issue #2894:模型试图一次写入超大 Markdown 时 JSON 参数常常非法,合成消息会明确指导"不要再重试同样的大 payload,直接在正文里给内容;必须写文件就拆小段",并且错误详情截断到 500 字符(`_MAX_RECOVERY_ERROR_DETAIL_LEN = 500`),避免把巨大非法参数回声给模型,见 [dangling_tool_call_middleware.py:29-32](../backend/packages/harness/deerflow/agents/middlewares/dangling_tool_call_middleware.py#L29-L32) 和 [dangling_tool_call_middleware.py:104-126](../backend/packages/harness/deerflow/agents/middlewares/dangling_tool_call_middleware.py#L104-L126)。

**Q4.4(边界/异常)** 有些 provider 把 `args` 序列化成 JSON 字符串而不是 dict,或者 raw payload 只有 `function.arguments`,扫描会不会崩?

**参考回答**:不会,`_message_tool_calls` 对 raw `additional_kwargs.tool_calls` 做了防御性归一化:名字优先取 `raw_tc.name`,没有再取 `function.name`;args 优先取 `raw_tc.args`,没有就把 `function.arguments` 字符串 `json.loads`,解析失败一律退成空 dict,见 [dangling_tool_call_middleware.py:59-86](../backend/packages/harness/deerflow/agents/middlewares/dangling_tool_call_middleware.py#L59-L86)。循环检测那边也有同款防御(`_normalize_tool_call_args`),非 dict 载荷保留稳定 fallback key,保证检测不崩且键稳定,见 [loop_detection_middleware.py:73-96](../backend/packages/harness/deerflow/agents/middlewares/loop_detection_middleware.py#L73-L96)。另外 `_build_patched_messages` 里已配对的真 ToolMessage 会按队列顺序重新紧跟其 AIMessage 排放,等于顺带做了一次因果序归一化;已经合法的 transcript 原样返回 `None` 零开销,见 [dangling_tool_call_middleware.py:128-183](../backend/packages/harness/deerflow/agents/middlewares/dangling_tool_call_middleware.py#L128-L183)。

## 问题链 5:LLM 错误归一化 vs 工具错误转 ToolMessage

**Q5.1(基础)** LLM 调用失败和工具调用失败,你们的处理策略完全不同,为什么不统一?

**参考回答**:因为两者的"恢复主体"不同。工具失败是**局部**的:把异常转成一条 `status="error"` 的 ToolMessage("Continue with available context, or choose an alternative tool."),让模型自己看到错误并换路,run 可以继续,见 [tool_error_handling_middleware.py:62-82](../backend/packages/harness/deerflow/agents/middlewares/tool_error_handling_middleware.py#L62-L82)。LLM 失败是**全局**的:模型这一跳根本没产出,没有东西可以让模型"看着办",所以 `LLMErrorHandlingMiddleware` 做分类重试 + 熔断,最终归一化成一条带 `deerflow_error_fallback` 标记的兜底 AIMessage,把控制权还给用户,见 [llm_error_handling_middleware.py:228-244](../backend/packages/harness/deerflow/agents/middlewares/llm_error_handling_middleware.py#L228-L244)。**不这样设计会怎样**:若工具异常也向上抛,整个 run 会因为一个 `grep` 的正则写错而全灭;若 LLM 错误也塞进 ToolMessage,消息里没有对应的 tool_call 可配,直接破坏配对校验。

**链路解析**:

```
工具异常:  wrap_tool_call ──handler 抛异常──► _build_error_message()
           └─► ToolMessage(status="error", "Error: Tool 'x' failed ...") ──► run 继续

LLM 异常:  wrap_model_call ──handler 抛异常──► _classify_error()
           ├─ quota/auth      → 不重试,直接兜底 AIMessage
           ├─ transient/busy  → 指数退避重试(≤3 次) → 仍败 → _record_failure + 兜底
           └─ GraphBubbleUp   → 原样 re-raise(interrupt 控制流不可吞)
```

**Q5.2(深挖)** 重试策略具体怎么做的?所有异常都重试 3 次吗?

**参考回答**:不是。默认 `retry_max_attempts=3`、`retry_base_delay_ms=1000`、指数退避封顶 `retry_cap_delay_ms=8000`,见 [llm_error_handling_middleware.py:104-106](../backend/packages/harness/deerflow/agents/middlewares/llm_error_handling_middleware.py#L104-L106)。分类先行:quota/auth 类(匹配"余额不足/欠费/invalid api key"等中英双语 pattern)不可重试,直接给用户可操作的文案;transient 类按异常类名(APITimeoutError/ReadError/RemoteProtocolError 等)+ HTTP 状态码集合 {408, 409, 425, 429, 500, 502, 503, 504} 判定,见 [llm_error_handling_middleware.py:27](../backend/packages/harness/deerflow/agents/middlewares/llm_error_handling_middleware.py#L27) 和 [llm_error_handling_middleware.py:185-211](../backend/packages/harness/deerflow/agents/middlewares/llm_error_handling_middleware.py#L185-L211)。延迟优先尊重 provider 的 `Retry-After`/`retry-after-ms` 头(支持秒、毫秒、HTTP-date 三种格式),见 [llm_error_handling_middleware.py:432-458](../backend/packages/harness/deerflow/agents/middlewares/llm_error_handling_middleware.py#L432-L458)。还有按异常名的预算表:`StreamChunkTimeoutError` 只给 2 次尝试,因为它本身已在上游卡了 120-240 秒,满配 3 次会堆出 6-12 分钟死寂,见 [llm_error_handling_middleware.py:66-83](../backend/packages/harness/deerflow/agents/middlewares/llm_error_handling_middleware.py#L66-L83)。键用异常类**名字符串**而非类对象,避免对 langchain-openai 产生 import 期耦合,见 [llm_error_handling_middleware.py:76-80](../backend/packages/harness/deerflow/agents/middlewares/llm_error_handling_middleware.py#L76-L80)。每次重试还会通过 stream writer 发 `llm_retry` 事件给前端,见 [llm_error_handling_middleware.py:278-294](../backend/packages/harness/deerflow/agents/middlewares/llm_error_handling_middleware.py#L278-L294)。

**Q5.3(深挖)** 熔断器怎么实现的?half-open 状态并发下会不会所有请求都变成探针?

**参考回答**:标准三态:closed 累计连续失败,达到 `failure_threshold`(默认 5 次)跳 open,`recovery_timeout_sec`(默认 60 秒)内所有请求快速失败、直接返回熔断文案,见 [app_config.py:54-55](../backend/packages/harness/deerflow/config/app_config.py#L54-L55) 和 [llm_error_handling_middleware.py:302-308](../backend/packages/harness/deerflow/agents/middlewares/llm_error_handling_middleware.py#L302-L308)。防"探针风暴"靠 `_circuit_probe_in_flight` 标志:open 到期转 half_open 时只放第一个请求当探针,并发后续请求在 `_check_circuit` 里看到已有探针在飞就直接快速失败,见 [llm_error_handling_middleware.py:133-150](../backend/packages/harness/deerflow/agents/middlewares/llm_error_handling_middleware.py#L133-L150)。探针成功则 `_record_success` 全量复位;探针失败立刻重新 open 并重新计时 60 秒,见 [llm_error_handling_middleware.py:161-171](../backend/packages/harness/deerflow/agents/middlewares/llm_error_handling_middleware.py#L161-L171)。所有状态读写都在 `_circuit_lock` 内,sync/async 两条路径共用,见 [llm_error_handling_middleware.py:115-119](../backend/packages/harness/deerflow/agents/middlewares/llm_error_handling_middleware.py#L115-L119)。注意熔断是实例级的——中间件实例随 agent 缓存复用,所以全进程共享一个熔断视图,这与"保护同一个 provider"的语义一致。

**Q5.4(边界/异常)** 重试循环里有两个细节:为什么要单独 catch `GraphBubbleUp` 再 re-raise?兜底 AIMessage 会不会又被 LoopDetection 当成循环?

**参考回答**:`GraphBubbleUp` 是 LangGraph 的 interrupt/pause/resume 控制流信号,技术上也是 Exception,若被通用 `except Exception` 吞掉,人工审批之类的中断语义会被静默吃掉,所以必须原样透传;half-open 下透传前还要把探针标志复位,否则探针名额永久泄漏,见 [llm_error_handling_middleware.py:316-321](../backend/packages/harness/deerflow/agents/middlewares/llm_error_handling_middleware.py#L316-L321)——工具侧中间件也有同款透传,见 [tool_error_handling_middleware.py:104-106](../backend/packages/harness/deerflow/agents/middlewares/tool_error_handling_middleware.py#L104-L106)。兜底 AIMessage 是纯文本、无 tool_calls,而 LoopDetection 的检测前置条件就是"末条消息为 ai 且 tool_calls 非空",不满足直接返回,见 [loop_detection_middleware.py:338-344](../backend/packages/harness/deerflow/agents/middlewares/loop_detection_middleware.py#L338-L344),所以不会互相干扰。这条兜底消息带 `deerflow_error_fallback: True` 和 error_type/error_reason,前端可以渲染成专属错误卡片;针对 stream 中断类异常还会给出"拆分/缩短输出"的具体建议而非笼统的"稍后重试",见 [llm_error_handling_middleware.py:228-244](../backend/packages/harness/deerflow/agents/middlewares/llm_error_handling_middleware.py#L228-L244) 和 [llm_error_handling_middleware.py:252-267](../backend/packages/harness/deerflow/agents/middlewares/llm_error_handling_middleware.py#L252-L267)。

**链路解析**(熔断器状态机):

```
            连续失败 >= failure_threshold (5)
 closed ──────────────────────────────────► open
   ▲                                        │
   │ 探针成功 (_record_success)              │ 60s (recovery_timeout_sec)
   │                                        ▼
   │                                     half_open
   │                                        │
   │            探针失败 (_record_failure)    │ 只放一个探针
   └────────────────────────────────────────┘ (_circuit_probe_in_flight
                                               挡住并发请求→快速失败)
```

## 问题链 6:ClarificationMiddleware 为什么必须放最后

**Q6.1(基础)** `ask_clarification` 这个工具为什么不走正常执行?`Command(goto=END)` 做了什么?

**参考回答**:澄清的本质是"打断当前 run、把问题抛给用户、等用户回答后再继续",而不是在 agent 循环内产出一个工具结果。所以 `ClarificationMiddleware` 在 `wrap_tool_call` 里拦截名为 `ask_clarification` 的调用:不执行 handler,而是把问题格式化成一条 ToolMessage(带类型图标、编号选项,中文语境自动识别),返回 `Command(update={"messages": [tool_message]}, goto=END)` 直接跳到图终点,见 [clarification_middleware.py:117-156](../backend/packages/harness/deerflow/agents/middlewares/clarification_middleware.py#L117-L156)。这条 ToolMessage 的 `tool_call_id` 与发起它的 tool_call 配对,历史格式依然合法;前端识别 `ask_clarification` 的 ToolMessage 渲染成问题卡片,用户的下一条输入自然开启新 run——代码注释明确说不额外加 AIMessage,由前端直接检测 tool message,见 [clarification_middleware.py:148-156](../backend/packages/harness/deerflow/agents/middlewares/clarification_middleware.py#L148-L156)。其他名字的 tool call 原样 `handler(request)` 放行,见 [clarification_middleware.py:158-178](../backend/packages/harness/deerflow/agents/middlewares/clarification_middleware.py#L158-L178)。

**链路解析**:

```
AIMessage(tool_calls=[ask_clarification, ...])
        │
        ▼ ToolNode 逐个执行,wrap_tool_call 洋葱:外层 → 内层
 ToolOutputBudget → ThreadData → ... → ToolErrorHandling → 【Clarification(最内)】
                                                              │ name == ask_clarification
                                                              ▼  不调 handler
                                          Command(update=ToolMessage, goto=END)
                                                              │
                                                              ▼
                                            图终止,等待用户下一轮输入
```

**Q6.2(深挖)** 为什么这个中间件必须注册在最后?放前面会怎样?

**参考回答**:`wrap_tool_call` 是洋葱模型——列表里越靠前的中间件越在外层。Clarification 拦截时是**短路返回、不调 handler** 的:它内侧(注册在它之后)的任何中间件对这次调用完全不会执行。把它放最后(最内层),能保证审计、错误转换、输出预算等所有中间件在请求路径上都已先过一遍;返回路径上它们收到的也都是自己能处理的类型——`ToolErrorHandlingMiddleware._maybe_stamp` 显式跳过 Command,`ToolOutputBudgetMiddleware._patch_result` 能钻进 `Command.update.messages` 处理,见 [tool_error_handling_middleware.py:84-94](../backend/packages/harness/deerflow/agents/middlewares/tool_error_handling_middleware.py#L84-L94)。agent 组装处一串注释把约束写死了:"ToolErrorHandlingMiddleware should be before ClarificationMiddleware to convert tool exceptions to ToolMessages / ClarificationMiddleware should be last",见 [agent.py:260-269](../backend/packages/harness/deerflow/agents/lead_agent/agent.py#L260-L269) 和 [agent.py:375-376](../backend/packages/harness/deerflow/agents/lead_agent/agent.py#L375-L376)。**不这样设计会怎样**:若 Clarification 放在中间,它后面的中间件对 `ask_clarification` 完全失聪;更糟的是若某个内层中间件假设"每个 tool call 都会真执行"并为此预分配资源,短路返回会让它的后半逻辑永远不配对,出现难以排查的状态泄漏。

**链路解析**(洋葱顺序与短路半径):

```
列表顺序: [..., SandboxAudit, ToolErrorHandling, Clarification]  ← 越后越内

普通工具(bash):
  请求 → Audit(分类/审计) → ErrHandling(try) → Clarification(放行) → handler
  返回 ← Audit(加警告)   ← ErrHandling(盖章) ← Clarification      ← ToolMessage

ask_clarification:
  请求 → Audit → ErrHandling → Clarification ──╳ 不调 handler,直接 Command(goto=END)
  若 Clarification 后面还有中间件 X:X 的请求/返回两半边对这次调用都不会执行,
  X 若做了"调用前记账、返回后销账",账就永远挂在那里。
```

**Q6.3(边界/异常)** 重试场景下这条澄清 ToolMessage 会不会重复追加?`options` 参数被模型序列化成 JSON 字符串怎么办?

**参考回答**:不会重复。消息 ID 是确定性的:有 tool_call_id 就是 `clarification:{tool_call_id}`,没有就取格式化文本的 sha256 前 16 位;`add_messages` reducer 按 id 替换而非追加,重试只会覆盖同一条,见 [clarification_middleware.py:40-45](../backend/packages/harness/deerflow/agents/middlewares/clarification_middleware.py#L40-L45)。`options` 的防御与前面几处一脉相承:Qwen3-Max 等模型会把数组参数序列化成 JSON 字符串,先尝试 `json.loads`,失败则包成单元素列表,保证渲染逻辑拿到的永远是 list,见 [clarification_middleware.py:75-84](../backend/packages/harness/deerflow/agents/middlewares/clarification_middleware.py#L75-L84)。

## 问题链 7:横切设计——共享助手与"为什么放在中间件层"

**Q7.1(深挖)** Safety、SubagentLimit、Summarization 都要改写 AIMessage 的 tool_calls,这个逻辑是各自实现还是共享的?共享助手解决了什么坑?

**参考回答**:共享在 `tool_call_metadata.py` 的 `clone_ai_message_with_tool_calls`:它不仅替换结构化 `tool_calls`,还按保留的 call id 集合同步过滤 `additional_kwargs.tool_calls` 里的原始 provider 载荷(一条都不剩就连 key 一起删),清空时顺带删 `function_call`,并在 tool_calls 被清空且原 `finish_reason` 是 `"tool_calls"` 时改写成 `"stop"`,见 [tool_call_metadata.py:18-50](../backend/packages/harness/deerflow/agents/middlewares/tool_call_metadata.py#L18-L50)。SafetyFinishReason 剥截断调用、SubagentLimit 截断超额 task 调用、Summarization 的 skill rescue 拆分 AIMessage,三处都用它,见 [safety_finish_reason_middleware.py:146-151](../backend/packages/harness/deerflow/agents/middlewares/safety_finish_reason_middleware.py#L146-L151) 和 [subagent_limit_middleware.py:66-68](../backend/packages/harness/deerflow/agents/middlewares/subagent_limit_middleware.py#L66-L68)。**不这样设计会怎样**:三处各自手写拷贝,迟早有一处忘记同步 raw 载荷或 finish_reason,出现"结构化字段空了但 provider 仍发出 tool_calls"的幽灵调用;集中在一个 50 行的助手里,语义只有一份,改 bug 只改一处。

**链路解析**(一个助手,三个客户):

```
                clone_ai_message_with_tool_calls(msg, kept_calls)
                ├─ tool_calls ← kept
                ├─ additional_kwargs.tool_calls ← 按 kept_ids 过滤(空则删 key)
                ├─ additional_kwargs.function_call ← kept 为空时删
                └─ response_metadata.finish_reason "tool_calls"→"stop"(仅清空时)
                          │
        ┌─────────────────┼────────────────────────┐
        ▼                 ▼                        ▼
 SafetyFinishReason  SubagentLimit           Summarization
 (剥截断 tool_calls)  (task 调用超 3 截断)    (skill rescue 拆 AIMessage)
```

**Q7.2(设计权衡)** 这些安全逻辑为什么不写进 agent 核心循环,而是全部做成中间件?中间件层给了什么核心循环给不了的东西?

**参考回答**:三点。第一是**hook 语义**:循环检测要的是"after_model 记账 + wrap_model_call 注入"两个不同出手点,dangling 补丁要的是"改 request 不改 state",这些细粒度位置控制只有中间件协议(before/after/wrap 三组 hook)能表达,写进核心循环就是一坨 if。第二是**顺序组合**:`after_model` 逆序、`wrap_*` 洋葱的顺序语义让 Safety-before-Loop、Clarification-最内这类关键约束可以用"注册顺序"声明式表达,组装处一目了然,见 [agent.py:357-376](../backend/packages/harness/deerflow/agents/lead_agent/agent.py#L357-L376)。第三是**可配置降级**:每个安全件都有独立 `enabled` 开关和 Pydantic 校验的配置(阈值、detector 列表、per-tool override),出问题可以单件关闭而不动核心,见 [loop_detection_config.py:24-64](../backend/packages/harness/deerflow/config/loop_detection_config.py#L24-L64)。**不这样设计会怎样**:写进核心循环意味着每个新 provider 的 safety 信号、每类新循环模式都要改框架主路径,回归面从"一个中间件文件"扩大成"整个 agent loop",且无法按部署关闭——这正是该仓库把 P0 安全网也做成可插拔件的根本原因。

## 关键数字速查(面试时脱口而出)

| 数字 | 含义 | 出处 |
|---|---|---|
| 3 / 5 | 循环检测 hash 层:相同调用 3 次警告、5 次强停 | [loop_detection_middleware.py:64-65](../backend/packages/harness/deerflow/agents/middlewares/loop_detection_middleware.py#L64-L65) |
| 20 / 100 | 滑窗 20 条 hash;LRU 最多 100 个线程 | [loop_detection_middleware.py:66-67](../backend/packages/harness/deerflow/agents/middlewares/loop_detection_middleware.py#L66-L67) |
| 30 / 50 | frequency 层:同工具 30 次警告、50 次强停(可 per-tool override) | [loop_detection_middleware.py:68-69](../backend/packages/harness/deerflow/agents/middlewares/loop_detection_middleware.py#L68-L69) |
| 4 / 200 | 单 run 最多 4 条待注入警告;pending key 上限 200 | [loop_detection_middleware.py:70](../backend/packages/harness/deerflow/agents/middlewares/loop_detection_middleware.py#L70)、[loop_detection_middleware.py:233](../backend/packages/harness/deerflow/agents/middlewares/loop_detection_middleware.py#L233) |
| 200 行 / 12 位 | read_file 分桶粒度;md5 取前 12 位 | [loop_detection_middleware.py:106](../backend/packages/harness/deerflow/agents/middlewares/loop_detection_middleware.py#L106)、[loop_detection_middleware.py:160](../backend/packages/harness/deerflow/agents/middlewares/loop_detection_middleware.py#L160) |
| 3 次 / 1s / 8s | LLM 重试:最多 3 次,退避 1s 起步、8s 封顶 | [llm_error_handling_middleware.py:104-106](../backend/packages/harness/deerflow/agents/middlewares/llm_error_handling_middleware.py#L104-L106) |
| 5 次 / 60s | 熔断:连续 5 次失败跳闸,60s 后单探针试探 | [app_config.py:54-55](../backend/packages/harness/deerflow/config/app_config.py#L54-L55) |
| 500 字符 | 工具错误详情 / dangling 恢复详情统一截断长度 | [tool_error_handling_middleware.py:66-67](../backend/packages/harness/deerflow/agents/middlewares/tool_error_handling_middleware.py#L66-L67)、[dangling_tool_call_middleware.py:32](../backend/packages/harness/deerflow/agents/middlewares/dangling_tool_call_middleware.py#L32) |
| 8 个 | Gemini 默认安全 finish_reason 集合大小 | [safety_termination_detectors.py:183-194](../backend/packages/harness/deerflow/agents/middlewares/safety_termination_detectors.py#L183-L194) |

## 面试官最爱追问的 3 个点

1. **"警告为什么不在 after_model 直接注入?"** —— 应答策略:一句话点破 pairing 约束——after_model 时 ToolMessage 尚未产生,任何插入都会让 OpenAI/Moonshot 报 `tool_call_ids did not have response messages`;正确姿势是 `after_model` 入队 + `wrap_model_call` 在请求末尾以 HumanMessage 追加,引用 [loop_detection_middleware.py:18-32](../backend/packages/harness/deerflow/agents/middlewares/loop_detection_middleware.py#L18-L32)。
2. **"after_model 链的执行顺序和注册顺序是什么关系?Safety 和 Loop 谁先谁后?"** —— 应答策略:逆序执行——最后注册的最先看到模型输出;所以 Safety 注册在 Loop 之后、执行在 Loop 之前,先清洗被 provider 截断的 tool_calls,Loop 再记账,避免误计循环,引用 [safety_finish_reason_middleware.py:26-32](../backend/packages/harness/deerflow/agents/middlewares/safety_finish_reason_middleware.py#L26-L32)。
3. **"硬停止只把 tool_calls 置空,够吗?"** —— 应答策略:不够,三处同步:结构化 `tool_calls`、`additional_kwargs` 里的原始 `tool_calls`/`function_call`、`response_metadata.finish_reason` 从 `"tool_calls"` 改 `"stop"`;否则 provider 适配器按原始载荷序列化,等于没剥,引用 [loop_detection_middleware.py:457-475](../backend/packages/harness/deerflow/agents/middlewares/loop_detection_middleware.py#L457-L475) 和公共助手 [tool_call_metadata.py:18-50](../backend/packages/harness/deerflow/agents/middlewares/tool_call_metadata.py#L18-L50)。
