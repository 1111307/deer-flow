# Sub-Agent 多智能体委派 —— 面试问题链

> 本文档按"大厂面试追问链"组织:每条问题链从基础问题开始,层层追问到实现细节、设计权衡、异常边界。配套深读笔记:[05-subagent-delegation.md](05-subagent-delegation.md)(深读笔记讲"怎么实现",本文档讲"怎么被问、怎么答")。
> 代码引用格式:[文件名.py:起始行-结束行](../backend/仓库相对路径#L起始行-L结束行)。只引用你实际读过的行,禁止编造行号。

## 问题链 1:task tool 委派全流程

**Q1.1(基础)** 主 agent 调用 `task` 工具委派一个子任务,从 tool_call 发起到结果返回,整个链路是怎么走的?

**参考回答**:入口是 `task_tool`,一个 `@tool("task", parse_docstring=True)` 装饰的 async 函数,签名收 `description / prompt / subagent_type` 三个业务参数外加注入的 `runtime` 和 `tool_call_id`,见 [task_tool.py:187-194](../backend/packages/harness/deerflow/tools/builtins/task_tool.py#L187-L194)。流程分五步:(1) 用 `get_subagent_config` 解析子 agent 配置,未知类型直接返回 `"Error: Unknown subagent type '...'. Available: ..."` 并把可用名单拼进错误文案,让 LLM 下一轮自我纠正 [task_tool.py:235-238](../backend/packages/harness/deerflow/tools/builtins/task_tool.py#L235-L238);(2) 从 runtime 抽取父级上下文——sandbox_state、thread_data、thread_id、parent_model、trace_id、user_id [task_tool.py:260-275](../backend/packages/harness/deerflow/tools/builtins/task_tool.py#L260-L275);(3) 构造 `SubagentExecutor` 并调 `execute_async(prompt, task_id=tool_call_id)` 后台启动 [task_tool.py:318-322](../backend/packages/harness/deerflow/tools/builtins/task_tool.py#L318-L322);(4) 主循环每 5 秒轮询一次 `_background_tasks`,把新产生的 AI message 通过 stream writer 推成 `task_running` 事件,索引用 `last_message_count` 做增量水位 [task_tool.py:353-369](../backend/packages/harness/deerflow/tools/builtins/task_tool.py#L353-L369);(5) 检测到四个终态之一(COMPLETED/FAILED/CANCELLED/TIMED_OUT)后写对应事件、`cleanup_background_task` 清理,并返回形如 `"Task Succeeded. Result: ..."` 的字符串作为 ToolMessage 内容 [task_tool.py:373-400](../backend/packages/harness/deerflow/tools/builtins/task_tool.py#L373-L400)。注意所有失败都走"返回错误字符串"而非抛异常,这样 lead agent 能把失败纳入推理继续规划,而不是整个 graph 崩掉。

**链路解析**:

```
Lead Agent LLM
   | tool_call: task(subagent_type="general-purpose", prompt=...)
   v
task_tool (async, 主 event loop)
   |-- get_subagent_config()  --> registry 三层解析(内置/custom_agents/agents override)
   |-- 抽取 runtime: sandbox_state / thread_data / thread_id / parent_model / trace_id
   |-- SubagentExecutor(...).execute_async(prompt, task_id=tool_call_id)
   |        |-- _background_tasks[task_id] = SubagentResult(PENDING)
   |        |-- _scheduler_pool.submit(run_task)        # ThreadPoolExecutor, 3 workers
   |                 |-- run_coroutine_threadsafe(_aexecute, isolated_loop)
   v
writer({"type": "task_started", task_id, description})   # SSE 事件①
   v
while True:                                        # 每 5s 轮询一次
   result = get_background_task_result(task_id)
   |-- result is None -> "Error: Task ... disappeared" + task_failed 事件
   |-- 新增 ai_messages -> writer({"type": "task_running",
   |                               message_index: i+1, total_messages: N})   # SSE 事件②
   |-- COMPLETED -> writer(task_completed) + "Task Succeeded. Result: ..."
   |-- FAILED    -> writer(task_failed)    + "Task failed. Error: ..."
   |-- CANCELLED -> writer(task_cancelled) + "Task cancelled by user."
   |-- TIMED_OUT -> writer(task_timed_out) + "Task timed out. Error: ..."
   |        (四个终态分支都做三件事: _cache_subagent_usage /
   |         _report_subagent_usage -> RunJournal / cleanup_background_task)
   v
cleanup_background_task(task_id)     # 只删终态条目,防与后台线程竞态
   v
ToolMessage(content=结果字符串, additional_kwargs.subagent_status=...)
   v
回到 Lead Agent,继续下一轮推理
```

**Q1.2(深挖)** 为什么要"后台执行 + 主循环轮询",而不是直接在 tool 函数里 `await` 子 agent 跑完?

**参考回答**:因为子 agent 可能跑满 30 分钟(全局默认 `timeout_seconds=1800` [subagents_config.py:74-78](../backend/packages/harness/deerflow/config/subagents_config.py#L74-L78)),直接 await 意味着这期间主 agent 协程被独占,前端看不到任何中间进展,也拿不到 `task_running` 流式事件。现在的结构把执行扔到 `_scheduler_pool`(3 个 worker)+ 持久隔离 event loop 上,tool 协程每 5 秒查一次共享的 `_background_tasks` 字典,把 `result.ai_messages` 里新增的条目逐条推给前端 [task_tool.py:353-369](../backend/packages/harness/deerflow/tools/builtins/task_tool.py#L353-L369)。轮询本身有上限:`max_poll_count = (timeout_seconds + 60) // 5`,即执行超时再加 60 秒 buffer,按 5 秒间隔换算成轮询次数 [task_tool.py:329](../backend/packages/harness/deerflow/tools/builtins/task_tool.py#L329)。这套设计把长任务对 LLM 伪装成一次同步工具调用——LLM 不需要自己写"查状态"的循环,编排复杂度全部藏在工具内部。边界情况也处理了:轮询中如果 `get_background_task_result` 返回 None(条目被意外清掉),会记 error 日志、向前端写 `task_failed` 事件并兜底 cleanup,再返回错误字符串而不是让协程死循环 [task_tool.py:341-345](../backend/packages/harness/deerflow/tools/builtins/task_tool.py#L341-L345)。

**Q1.3(深挖)** 我注意到 `execute_async(prompt, task_id=tool_call_id)` 直接拿 tool_call_id 当 task_id,这不是偷懒吗?有什么好处?

**参考回答**:这是刻意的可观测性设计,一次委派全链路一个 ID。`task_id` 会被写进 `SubagentResult` 并存入全局 `_background_tasks` 字典 [executor.py:804-818](../backend/packages/harness/deerflow/subagents/executor.py#L804-L818),它等于 tool_call_id 意味着:(1) 前端收到的 `task_started / task_running / task_completed` 事件里的 task_id 可以直接和消息流里的 tool_call 对上,卡片渲染无需任何映射表 [task_tool.py:333-335](../backend/packages/harness/deerflow/tools/builtins/task_tool.py#L333-L335);(2) token 用量缓存 `_subagent_usage_cache` 也以 tool_call_id 为 key,后面 `TokenUsageMiddleware` 用 `pop_cached_subagent_usage(tool_msg.tool_call_id)` 精确找回这次委派的用量并合并回发起它的 AIMessage [task_tool.py:31-51](../backend/packages/harness/deerflow/tools/builtins/task_tool.py#L31-L51);(3) 用户取消时 `request_cancel_background_task(task_id)` 同样直接命中,不需要维护"前端任务 ID → 内部任务 ID"的转换层。如果另造一个随机 ID,这三处每处都要多一张关联表,还要处理表的生命周期。

---

## 问题链 2:并发执行架构 —— 调度线程池与持久隔离 event loop

**Q2.1(基础)** 子 agent 实际跑在哪个线程、哪个 event loop 上?这套线程模型怎么画?

**参考回答**:`execute_async` 先把任务以 PENDING 状态塞进全局 `_background_tasks` 字典(加锁),再 `copy_context()` 抓父级 ContextVar 快照,最后把 `run_task` 提交给 `_scheduler_pool` [executor.py:794-855](../backend/packages/harness/deerflow/subagents/executor.py#L794-L855)。三层分工:(1) `_scheduler_pool = ThreadPoolExecutor(max_workers=3, thread_name_prefix="subagent-scheduler-")`,负责承载 `run_task` 调度函数——它本身不跑 agent,只是标记 RUNNING、等待 future 和处理超时 [executor.py:143](../backend/packages/harness/deerflow/subagents/executor.py#L143) 和 [executor.py:822-827](../backend/packages/harness/deerflow/subagents/executor.py#L822-L827);(2) 一个**全进程唯一**的持久 event loop,跑在名为 `subagent-persistent-loop` 的 daemon 线程里,由 `_get_isolated_subagent_loop` 加锁懒启动,启动有 5 秒超时保护,失败会清理现场并抛 `RuntimeError` [executor.py:204-232](../backend/packages/harness/deerflow/subagents/executor.py#L204-L232);(3) 子 agent 的 `_aexecute` 协程通过 `asyncio.run_coroutine_threadsafe` 提交到这个持久 loop 上执行,多个并发子 agent 的协程共享这同一个 loop [executor.py:832-835](../backend/packages/harness/deerflow/subagents/executor.py#L832-L835)。需要纠正一个常见误解:早期文档说"双线程池 `_scheduler_pool` + `_execution_pool` 各 3 worker",但当前代码里 `_execution_pool` 已不存在(grep 全仓库无匹配)——执行层已重构为持久 loop 线程,调度池只负责阻塞等待与超时治理,真实并发由 asyncio 协程提供。

**链路解析**:

```
主 event loop (LangGraph ASGI, lead agent 所在)
   | task_tool: execute_async(prompt, task_id=tool_call_id)
   |   1) _background_tasks[task_id] = SubagentResult(PENDING)   # 加锁
   |   2) parent_context = copy_context()
   |   3) _scheduler_pool.submit(run_task)
   v
+------------------------+     run_coroutine_threadsafe     +-------------------------------+
| _scheduler_pool        |  (在 parent_context 里创建协程)   | subagent-persistent-loop      |
| 3 worker threads       |  ----------------------------->  | 1 daemon thread + 1 event loop|
| run_task():            |                                  | _run_isolated_subagent_loop:  |
|  - 标记 RUNNING/起时间   |   future.result(timeout=1800s)   |   asyncio.set_event_loop(loop)|
|  - 等 future            |  <-----------------------------  |   loop.run_forever()          |
|  - 超时: cancel_event   |        SubagentResult           | _aexecute():                  |
|    + TIMED_OUT + cancel |                                 |   agent.astream(stream_mode=  |
+------------------------+                                   |     "values") 多协程并发共享  |
   ^                                                          +-------------------------------+
   | atexit: _shutdown_isolated_subagent_loop
   |   call_soon_threadsafe(loop.stop) -> thread.join(timeout=1s)
   |   线程停稳且 loop 不 running 才 loop.close(),否则只 warning
   |
   | 模块热重载防护: 重新 import 时先注销旧 shutdown 钩子并执行之(executor.py:43-46)
```

**Q2.2(深挖)** 为什么不给每个子 agent 新建一个 event loop 跑 `asyncio.run`?搞一个全局持久 loop 是为了什么?

**参考回答**:`_execute_in_isolated_loop` 的 docstring 说得很直白:复用长生命周期 loop 是为了避免"共享 async 资源绑定到短生命周期 loop 然后被关闭"的问题 [executor.py:714-722](../backend/packages/harness/deerflow/subagents/executor.py#L714-L722)。子 agent 里会用到 MCP 工具、httpx client 这类 async 资源,它们内部缓存 connection pool,而 pool 里的 transport 在创建时绑定当前 loop;如果每个任务 `asyncio.run` 新建 loop 用完即关,下一个任务复用这些 client 时就会拿到挂在已关闭 loop 上的 transport,抛 `RuntimeError: Event loop is closed`。**不这样设计会怎样**:每委派一次新建/销毁一个 loop,第二个子 agent 起随机报 loop closed,报错点埋在共享 client 内部极难排查;而且 `execute()` 是同步 API,在已有运行中 loop 的父协程里直接 `asyncio.run` 本身就是非法的,代码里专门先 `asyncio.get_running_loop()` 探测,有运行中 loop 才走 isolated loop 路径,没有才走 `asyncio.run` 标准路径 [executor.py:768-779](../backend/packages/harness/deerflow/subagents/executor.py#L768-L779)。持久 loop 只在进程退出时由 `atexit` 注册的 `_shutdown_isolated_subagent_loop` 关闭,且关闭很谨慎:先 `call_soon_threadsafe(loop.stop)`,join 线程最多 1 秒,确认线程停稳且 loop 不再 running 才 `loop.close()`,否则只打 warning 跳过,避免在还有 pending callback 的 loop 上强关造成二次崩溃 [executor.py:167-201](../backend/packages/harness/deerflow/subagents/executor.py#L167-L201)。模块顶部还有一段热重载防护:模块被重新加载时先注销并执行旧版 shutdown 钩子,防止旧 loop 泄漏 [executor.py:43-46](../backend/packages/harness/deerflow/subagents/executor.py#L43-L46)。

**Q2.3(深挖)** 提交协程时为什么要 `copy_context()`?丢了这个会怎样?

**参考回答**:`_submit_to_isolated_loop_in_context` 的实现是 `context.run(lambda: asyncio.run_coroutine_threadsafe(coro_factory(), loop))` [executor.py:235-245](../backend/packages/harness/deerflow/subagents/executor.py#L235-L245)。关键点在于 `run_coroutine_threadsafe` 的参数是一个**协程对象**,它是在调用现场(父线程)实例化的——协程对象创建时会捕获当前 ContextVar 快照,所以必须在 `copy_context()` 得到的父级上下文里创建,才能让协程体后续读到父级的 ContextVar(tracing 回调、用户上下文、LangChain run tree 等)。`execute_async` 里同样先 `parent_context = copy_context()` 再传下去 [executor.py:820](../backend/packages/harness/deerflow/subagents/executor.py#L820)。**不这样设计会怎样**:协程在 loop 线程以空上下文运行,子 agent 的 Langfuse trace 挂不到父 trace 下,`inject_langfuse_metadata` 注入的 session/user 链路断裂 [executor.py:550-559](../backend/packages/harness/deerflow/subagents/executor.py#L550-L559),排障时父子日志对不上,计费归因也会丢失用户维度。

---

## 问题链 3:MAX_CONCURRENT_SUBAGENTS=3 的强制截断

**Q3.1(基础)** 系统说最多并发 3 个子 agent,这个限制具体在哪一层、怎么实现的?

**参考回答**:常量在 [executor.py:858](../backend/packages/harness/deerflow/subagents/executor.py#L858) 定义为 `MAX_CONCURRENT_SUBAGENTS = 3`,但真正的执行者是 lead agent 侧的 `SubagentLimitMiddleware`,挂在 `after_model` 钩子上,同步异步两个入口 `after_model / aafter_model` 都指向同一个 `_truncate_task_calls` [subagent_limit_middleware.py:70-76](../backend/packages/harness/deerflow/agents/middlewares/subagent_limit_middleware.py#L70-L76)。注册点在 lead agent 的 middleware 装配处:只有 `subagent_enabled` 为真时才 append,上限值从 runtime config 读、默认 3 [agent.py:351-355](../backend/packages/harness/deerflow/agents/lead_agent/agent.py#L351-L355)。每次模型返回后,它取最后一条消息,如果是 AIMessage 且带 tool_calls,就数出 `tc.get("name") == "task"` 的下标;超过 `max_concurrent` 的部分按"保留前 N 个"策略砍掉,用 `clone_ai_message_with_tool_calls` 生成截断后的新 AIMessage 返回给 state(同 id 触发 LangGraph 的 replace 语义)[subagent_limit_middleware.py:41-68](../backend/packages/harness/deerflow/agents/middlewares/subagent_limit_middleware.py#L41-L68)。所以限制发生在"模型输出之后、工具执行之前",被砍的 tool_call 根本不会进入 ToolNode,只留一条 warning 日志。另外构造函数里 `_clamp_subagent_limit` 把配置值钳制在 `[2, 4]`——传 10 得 4,传 1 得 2,防止运维把红线调飞 [subagent_limit_middleware.py:16-22](../backend/packages/harness/deerflow/agents/middlewares/subagent_limit_middleware.py#L16-L22)。注意它只数 `name == "task"` 的调用,同一条响应里混着的普通工具调用全部保留且位置不变。

**链路解析**:

```
LLM 返回 AIMessage(tool_calls=[task#1, task#2, task#3, task#4, task#5])
   v
after_model: SubagentLimitMiddleware._truncate_task_calls
   |-- 末条消息不是 AIMessage / 无 tool_calls? -> return None(不干预)
   |-- task_indices = [0,1,2,3,4], len=5 > max_concurrent=3
   |-- indices_to_drop = {3, 4}            # 保留前 3 个,砍尾部
   |-- 非 task 的 tool_call(若有)全部保留,位置不变
   |-- clone_ai_message_with_tool_calls(原 msg, 截断后列表)
   |       |-- kept_ids = 保留 call 的 id 集合
   |       |-- 同步过滤 additional_kwargs.tool_calls 里的 raw payload
   |       |-- model_copy(update=...) 保持 id 不变 -> LangGraph replace
   v
state 更新: AIMessage(tool_calls=[task#1, task#2, task#3])
   v
ToolNode 只看到 3 个 task call -> 3 个后台子 agent
(logger.warning: "Truncated 2 excess task tool call(s) from model response (limit: 3)")

配置侧: 构造时 _clamp_subagent_limit(value) -> [2, 4]
   传 1 -> 2(并行能力下限), 传 10 -> 4(成本红线)
```

**Q3.2(深挖)** 为什么不在 system prompt 里写"最多并行 3 个 task"来约束?middleware 截断比 prompt 约束强在哪?

**参考回答**:middleware 的 docstring 一句话点破:"This is more reliable than prompt-based limits" [subagent_limit_middleware.py:26-30](../backend/packages/harness/deerflow/agents/middlewares/subagent_limit_middleware.py#L26-L30)。prompt 是软约束,模型在"帮我并行调研 8 个方向"这类任务上极易一次吐出 5-10 个 task call,而每个子 agent 都是完整的 LLM 循环加工具调用,成本是真金白银。**不这样设计会怎样**:没有截断时,一次失控的并行委派可能同时起 8-10 个子 agent,每个最长跑 1800 秒、最多 150 轮模型调用,token 成本和下游 API 限流瞬间爆炸;超时治理、取消传播、token 归账这些机制都是按"少量并发任务"设计的,并发无上限时 `_scheduler_pool` 的 3 个 worker 排队积压,轮询协程越挂越多,主 loop 也会被拖慢。截断是确定性代码,不依赖模型自觉——这是"工程约束"与"行为约束"的本质区别,面试里可以把这句话直接说出来。

**Q3.3(深挖)** 截断时直接改 `tool_calls` 列表不就行了,为什么还要走 `clone_ai_message_with_tool_calls` 这么麻烦?

**参考回答**:因为 LangChain 的 AIMessage 有**两份** tool_call 表示:解析后的 `tool_calls` 字段,和 `additional_kwargs` 里 provider 原始的 `tool_calls` raw payload。只改前者会让两者不一致——某些 provider 在下一轮请求里读 raw payload,被砍掉的 tool_call 会"复活"。`clone_ai_message_with_tool_calls` 先算出保留的 id 集合,再同步过滤 `additional_kwargs["tool_calls"]` 中不在集合里的 raw 条目;过滤完为空就把 key 删掉;tool_calls 全空时还顺手清掉遗留的 `function_call` 字段、并把 `response_metadata.finish_reason` 从 `"tool_calls"` 改回 `"stop"`,防止下游按 finish_reason 误判 [tool_call_metadata.py:18-50](../backend/packages/harness/deerflow/agents/middlewares/tool_call_metadata.py#L18-L50)。另外 LangGraph 的消息合并是"同 id 替换",所以必须 `model_copy(update=...)` 产出一个同 id 的新消息对象返回,而不是原地改老消息 [subagent_limit_middleware.py:66-68](../backend/packages/harness/deerflow/agents/middlewares/subagent_limit_middleware.py#L66-L68)。同一个 helper 也被 SafetyFinishReasonMiddleware 和 summarization 复用,说明"改 tool_calls 必须双写同步"在这个仓库里是共识。

---

## 问题链 4:status_contract —— 前后端状态契约

**Q4.1(基础)** `status_contract.py` 这个文件解决的是什么问题?前端直接用结果文本判断状态不行吗?

**参考回答**:模块 docstring 直接引用了 issue #3146:前端以前靠**字符串匹配** task 工具返回文本的前缀来推断子任务卡片状态,后端只要改一个文案,前端卡片生命周期就悄悄坏掉,#3107 BUG-007 / #3131 的 review 历史表明这事反复发生 [status_contract.py:1-21](../backend/packages/harness/deerflow/subagents/status_contract.py#L1-L21)。解决方案是把契约结构化:`ToolMessage.additional_kwargs` 里塞 `subagent_status`(限定 5 个枚举值:completed / failed / cancelled / timed_out / polling_timed_out)加可选 `subagent_error`,前端读结构化字段而不是猜文本 [status_contract.py:27-47](../backend/packages/harness/deerflow/subagents/status_contract.py#L27-L47)。更关键的是治理手段:共享 fixture `contracts/subagent_status_contract.json` 是单一事实源,前后端测试都加载它做断言,文案或枚举漂移会在两侧测试里同时爆炸,而不是在线上悄悄降级。还要答出"打标在哪发生":不在 task_tool 的 5 个正常返回分支里,而是集中在 `ToolErrorHandlingMiddleware._stamp_task_subagent_status`——docstring 明说选这里是因为它是"the one place every task tool result flows through",集中化防的就是"加了新返回路径忘了打标"的漂移模式 [tool_error_handling_middleware.py:29-56](../backend/packages/harness/deerflow/agents/middlewares/tool_error_handling_middleware.py#L29-L56)。连工具异常包装出来的 `"Error: Tool 'task' failed ..."`(detail 超 500 字符还会截断成 497+"...")也会走同一个函数打上 failed 标 [tool_error_handling_middleware.py:62-82](../backend/packages/harness/deerflow/agents/middlewares/tool_error_handling_middleware.py#L62-L82)。

**链路解析**:

```
task_tool 返回文本                    ToolErrorHandlingMiddleware
"Task Succeeded. Result: ..."     ->  _stamp_task_subagent_status
"Task failed. Error: ..."         ->       | 唯一打标点(所有路径汇聚处)
"Task timed out. Error: ..."      ->       v
"Task polling timed out ..."      ->  extract_subagent_status(content)
"Task cancelled by user."         ->       | 按 _PREFIX_TO_STATUS 顺序匹配
"Error: Unknown subagent ..."     ->       | "Task polling timed out" 先于 "Task timed out"
"Error: Tool 'task' failed ..."   ->       | "Error" 兜底最后
   (异常包装, detail>500 字符截断)          v
                                      make_subagent_additional_kwargs(status, error=...)
                                           | 非法 status 直接 ValueError
                                           | error 空白 -> 字段丢弃
                                           v
                              ToolMessage.additional_kwargs = {
                                  "subagent_status": "timed_out",
                                  "subagent_error": "Execution timed out after 1800 seconds"
                              }
                                           v
                              前端卡片: 读结构化字段(不再匹配文本)
                              非终态 chunk -> status=None -> 不打标 -> 卡片保持 in-progress
                              共享 fixture contracts/subagent_status_contract.json
                              前后端测试共同 pin 住枚举与映射
```

**Q4.2(深挖)** 前缀匹配表 `_PREFIX_TO_STATUS` 的顺序有什么讲究?"Error" 为什么放最后?

**参考回答**:代码注释明说"ordered most-specific-first because some prefixes are substrings of others" [status_contract.py:49-62](../backend/packages/harness/deerflow/subagents/status_contract.py#L49-L62)。具体有两对子串陷阱:`"Task timed out"` 是 `"Task polling timed out"` 的前缀——顺序反了的话,轮询超时会被错标成执行超时,前端卡片状态就错了;`"Task failed."` 带句号也是为了避免误配其他以 "Task failed" 开头的文本。`"Error"` 兜底放最后,因为它要同时接住 task_tool 的 3 种执行前错误返回(如未知 subagent 类型)和 `ToolErrorHandlingMiddleware` 包装出的 `"Error: Tool 'task' failed ..."` 异常文案 [status_contract.py:52-62](../backend/packages/harness/deerflow/subagents/status_contract.py#L52-L62)。匹配逻辑就是 trim 之后遍历表、`startswith` 命中即返,全部不命中返回 `None` [status_contract.py:65-78](../backend/packages/harness/deerflow/subagents/status_contract.py#L65-L78)。这张表是后端 stamper 和前端 fallback parser 唯一必须达成一致的东西,所以被 fixture 钉死。

**Q4.3(边界/异常)** 流式中间帧文本不匹配任何前缀,或者有人给 `make_subagent_additional_kwargs` 传了个拼错的 status,分别会发生什么?

**参考回答**:两个方向分别设计成"显式静默"和"显式爆炸"。`extract_subagent_status` 对非终态 chunk 返回 `None`,middleware 此时**不打** `subagent_status`,前端卡片保持 in-progress 占位,等真正的终态帧——注释里明确说这是 by design [status_contract.py:65-73](../backend/packages/harness/deerflow/subagents/status_contract.py#L65-L73)。反过来,`make_subagent_additional_kwargs` 对不在 `SUBAGENT_STATUS_VALUES` 里的 status 直接 `raise ValueError`,注释解释了为什么:接受任意字符串的话,一个 typo 会静默漏到前端并降级回脆弱的旧前缀匹配,宁可 loudly fail [status_contract.py:91-98](../backend/packages/harness/deerflow/subagents/status_contract.py#L91-L98)。另外 error 为空白时字段会被丢弃,避免线上出现误导性的 `"subagent_error": ""` [status_contract.py:99-102](../backend/packages/harness/deerflow/subagents/status_contract.py#L99-L102)。这套"静默忽略中间态、响亮拒绝脏数据"的组合是契约设计的标准范式。

---

## 问题链 5:token 收集与按位置合并回 dispatching AIMessage

**Q5.1(基础)** 子 agent 消耗的 token 是怎么统计出来、又怎么归到主 agent 账上的?

**参考回答**:每个子 agent 执行时创建自己的 `SubagentTokenCollector`(一个 `BaseCallbackHandler`),caller 标记为 `"subagent:{name}"`,挂进 `run_config["callbacks"]` 和 tags [executor.py:522-530](../backend/packages/harness/deerflow/subagents/executor.py#L522-L530)。每次 LLM 调用结束触发 `on_llm_end`,先按 `run_id` 去重(`_counted_run_ids` 集合,同一 run 重复触发直接 return),再从 `gen.message.usage_metadata` 里抽 `input_tokens / output_tokens / total_tokens`,total 缺失或为零时用 input+output 兜底,仍是 0 就跳过不记——宁可少记也不记假数据;同时从 `response_metadata` 取真实产出响应的 `model_name`,让父级账本能按真实模型分桶而不是记到 lead agent 的模型头上 [token_collector.py:25-68](../backend/packages/harness/deerflow/subagents/token_collector.py#L25-L68)。子 agent 终态时 records 挂到 `SubagentResult.token_usage_records`,task_tool 通过 `_report_subagent_usage` 在 runtime callbacks 里找带 `record_external_llm_usage_records` 方法的 RunJournal 完成上报,`usage_reported` 标志保证只报一次 [task_tool.py:146-164](../backend/packages/harness/deerflow/tools/builtins/task_tool.py#L146-L164)。

**链路解析**:

```
子 agent LLM 调用 -> SubagentTokenCollector.on_llm_end(run_id=R)
   |-- rid 已在 _counted_run_ids? -> 直接 return(去重)
   |-- usage_metadata -> {input_tokens, output_tokens, total_tokens}
   |-- total <= 0 时用 input+output 兜底;仍为 0 -> 跳过不记
   |-- response_metadata.model_name -> 真实产出模型(不是 lead 的模型)
   |-- record = {source_run_id, caller: "subagent:general-purpose",
   |             model_name, input/output/total_tokens}
   v
SubagentResult.token_usage_records (终态时随 try_set_terminal 一次性写入)
   |
   |-- task_tool 终态分支:
   |     _summarize_usage(records) -> {input, output, total} 汇总
   |     _cache_subagent_usage(tool_call_id, usage)   # 按 tool_call_id 缓存
   |     _report_subagent_usage -> RunJournal.record_external_llm_usage_records
   |                               (usage_reported 标志保证只报一次)
   v
TokenUsageMiddleware.after_model (主 agent 侧):
   从 messages[-2] 倒序扫连续 ToolMessage
   |-- pop_cached_subagent_usage(tool_msg.tool_call_id)  # pop = 只消费一次
   |-- 继续向前找 dispatching AIMessage(_has_tool_call 判定)
   |-- state_updates[dispatch_idx] 累加 input/output/total(多子任务合并)
   v
dispatching AIMessage.usage_metadata = 模型自身 token + 全部子 agent token
```

**Q5.2(深挖)** "按位置合并回 dispatching AIMessage"具体怎么做的?为什么不能假设 ToolMessage 的前一条就是发起它的 AIMessage?

**参考回答**:`TokenUsageMiddleware._apply` 从 `messages[-2]` 开始**倒序**遍历连续的 ToolMessage(遇到非 ToolMessage 就 break);每发现一条有缓存用量的 ToolMessage,就从它的位置继续向前扫描,找到第一条满足 `isinstance(candidate, AIMessage) and _has_tool_call(candidate, tool_msg.tool_call_id)` 的消息——也就是真正发起这个 tool_call 的那条 AIMessage [token_usage_middleware.py:281-313](../backend/packages/harness/deerflow/agents/middlewares/token_usage_middleware.py#L281-L313)。注释给了不能假设固定偏移的原因:"A single model response can dispatch multiple task tool calls"——3 个并发 task 会产生 3 条连续 ToolMessage,它们共享**同一条** dispatching AIMessage,固定偏移 -1 只对第一条成立。合并用 `state_updates: dict[int, AIMessage]` 以消息下标为 key 累积:同一个 AIMessage 被多个子任务命中时,在已有 update 的 usage_metadata 上继续累加 input/output/total 三项,而不是互相覆盖,最后一次性 `model_copy` 回写 [token_usage_middleware.py:303-311](../backend/packages/harness/deerflow/agents/middlewares/token_usage_middleware.py#L303-L311)。这样前端看那条发起委派的 AIMessage,就能看到"模型自身 token + 3 个子 agent token"的总账。缓存入口侧还有个开关:只有 `token_usage.enabled` 为真才写 `_subagent_usage_cache` [task_tool.py:36-47](../backend/packages/harness/deerflow/tools/builtins/task_tool.py#L36-L47),读取侧用 `pop` 保证每条缓存只被消费一次,消息重放也不会重复计账。

**Q5.3(边界/异常)** 上报环节有哪些防御?runtime 里找不到账本、callbacks 形状不对、上报抛异常,分别会怎样?

**参考回答**:三层防御。第一,`_find_usage_recorder` 兼容 LangChain 传 callbacks 的三种形状:`None`、普通 `list[BaseCallbackHandler]`、`BaseCallbackManager`(取 `.handlers` 再迭代);其他形状(比如单个裸 handler)一律视为"没有 recorder"而不是 raise [task_tool.py:103-132](../backend/packages/harness/deerflow/tools/builtins/task_tool.py#L103-L132)。第二,找不到 recorder 只打 debug 日志跳过,不阻塞工具返回 [task_tool.py:156-159](../backend/packages/harness/deerflow/tools/builtins/task_tool.py#L156-L159)。第三,`record_external_llm_usage_records` 调用本身包了 try/except,失败只 warning [task_tool.py:160-164](../backend/packages/harness/deerflow/tools/builtins/task_tool.py#L160-L164)。还有一条兜底:用户取消导致的 CancelledError 路径里,task_tool 会 `asyncio.shield` 住 `_await_subagent_terminal` 等子 agent 到终态,把最后的 token 快照报给账本后才 re-raise,保证取消场景账也不丢 [task_tool.py:422-444](../backend/packages/harness/deerflow/tools/builtins/task_tool.py#L422-L444)。核心原则:计量是旁路,永远不能反过来影响主流程的成败。

---

## 问题链 6:checkpointer=False 与状态隔离

**Q6.1(基础)** 创建子 agent 时 `checkpointer=False` 是为什么?主 agent 不是都开 checkpoint 吗?

**参考回答**:见 [executor.py:354-361](../backend/packages/harness/deerflow/subagents/executor.py#L354-L361),`create_agent(..., state_schema=ThreadState, checkpointer=False)`。原因有三:(1) 子 agent 是**一次性**执行单元,跑完结果以字符串形式经 ToolMessage 回到父对话,没有"恢复子 agent 会话"的产品需求;(2) 开 checkpoint 就要提供 checkpointer 和 thread_id,而 `run_config["configurable"]["thread_id"]` 复用的是父线程 id [executor.py:561-564](../backend/packages/harness/deerflow/subagents/executor.py#L561-L564),3 个并发子 agent 共用同一 thread_id 写 checkpoint 会互相覆盖、状态串台;(3) 父 agent 的对话历史已由父级 checkpoint 持久化,子 agent 最多 150 轮的中间消息再写进去,checkpoint 存储会膨胀一个数量级。**不这样设计会怎样**:checkpoint 表被短命子任务灌爆;并发子 agent 互相读到对方的中间 state,恢复时出现"A 子 agent 的半成品消息混进 B 子 agent"的串台 bug;而且子 agent 图没有 interrupt 机制,checkpoint 买来的恢复能力根本用不上,是纯成本。

**链路解析**:

```
父对话 (thread_id=T, 有 checkpointer, 历史已持久化)
   |
   |-- task#1 -> 子 agent A (checkpointer=False, 内存态 state)
   |-- task#2 -> 子 agent B (checkpointer=False, 内存态 state)
   |-- task#3 -> 子 agent C (checkpointer=False, 内存态 state)
   |        每个子 agent 的 run_config["configurable"]["thread_id"] 都是 T
   |        若开 checkpoint: A/B/C 同时写 thread T 的 checkpoint -> 互相覆盖
   v
子 agent state(一次性, 手工构造):
   state = {
       "messages": [SystemMessage(system_prompt + skills + deferred 提示),
                    HumanMessage(task)],
       "sandbox": 父级透传, "thread_data": 父级透传,
   }
   |-- astream 跑完 -> 提取最后一条 AIMessage 的文本
   |-- 中间 150 轮消息随 SubagentResult 一起被 cleanup -> 不落盘
   v
唯一回传物: 结果字符串 -> ToolMessage -> 进入父对话(由父级 checkpoint 持久化)
```

**Q6.2(深挖)** 不开 checkpoint,那子 agent 的初始 state 怎么构造?system prompt 又是怎么注入的?

**参考回答**:`_build_initial_state` 手工拼 state dict:先加载 skills(按 `config.skills` 白名单,`None` 继承全部启用、`[]` 显式为空),做工具政策过滤和 deferred MCP 工具装配;然后把 system_prompt、每条 skill 的 `<skill name="...">` 内容、deferred tools 提示段合并成**一条** SystemMessage,再追加一条 HumanMessage(task),最后透传父级的 `sandbox` 和 `thread_data` [executor.py:425-487](../backend/packages/harness/deerflow/subagents/executor.py#L425-L487)。合并成单条 SystemMessage 是刻意的——注释写明有些 LLM API 拒绝多条 SystemMessage,报 "System message must be at the beginning" [executor.py:456-458](../backend/packages/harness/deerflow/subagents/executor.py#L456-L458),所以 `_create_agent` 里 `system_prompt=None`,prompt 只走 messages 通道 [executor.py:352-358](../backend/packages/harness/deerflow/subagents/executor.py#L352-L358)。skills 走的是 Codex 模式:每个子 agent 按自己的白名单独立加载 SKILL.md 全文注入,而不是从父级继承文本;加载过程中的磁盘 IO 全部用 `asyncio.to_thread` 卸载,避免阻塞 event loop(注释标明这是 LangGraph ASGI 的要求)[executor.py:363-389](../backend/packages/harness/deerflow/subagents/executor.py#L363-L389) 和 [executor.py:394-423](../backend/packages/harness/deerflow/subagents/executor.py#L394-L423)。父级的 skill 白名单还会和子 agent 配置做交集 `_merge_skill_allowlists`,父级没开的 skill 子 agent 也拿不到 [task_tool.py:176-184](../backend/packages/harness/deerflow/tools/builtins/task_tool.py#L176-L184)。

**Q6.3(深挖)** 子 agent 的模型和中间件栈跟主 agent 是什么关系?全新一套吗?

**参考回答**:模型默认继承父级:`model="inherit"` 时 `resolve_subagent_model_name` 按 config.model → parent_model → app_config 第一个模型的顺序解析 [config.py:49-61](../backend/packages/harness/deerflow/subagents/config.py#L49-L61);创建时 `create_chat_model(..., thinking_enabled=False, attach_tracing=False)`——关 thinking 省 token,关模型级 tracing 是为了配合 graph 级 tracing 避免双重计数,注释里明确写了这个配对关系 [executor.py:345](../backend/packages/harness/deerflow/subagents/executor.py#L345) 和 [executor.py:532-539](../backend/packages/harness/deerflow/subagents/executor.py#L532-L539)。中间件则**复用**主 agent 的运行时栈:`build_subagent_runtime_middlewares` 与 lead 共用底层 `_build_runtime_middlewares`(只是关掉 uploads 等 lead 专属件),再按需追加 ViewImageMiddleware、DeferredToolFilterMiddleware、SafetyFinishReasonMiddleware [tool_error_handling_middleware.py:200-249](../backend/packages/harness/deerflow/agents/middlewares/tool_error_handling_middleware.py#L200-L249)。值得注意:子 agent 栈里**没有** SubagentLimitMiddleware——子 agent 根本拿不到 task 工具,不存在超额委派问题,挂了也是死代码。

---

## 问题链 7:超时 1800s、max_turns=150 与取消竞态

**Q7.1(基础)** 子 agent 的超时控制在哪几层落地?1800 秒这个数从哪来?

**参考回答**:全局默认值在 `SubagentsAppConfig.timeout_seconds = 1800`(30 分钟)[subagents_config.py:74-78](../backend/packages/harness/deerflow/config/subagents_config.py#L74-L78),`SubagentConfig` 字段上的 900 只是"没有全局值时的兜底",docstring 里明说内置 agent 的生效值由 registry 层叠上全局值 [config.py:26-40](../backend/packages/harness/deerflow/subagents/config.py#L26-L40)。落地有三层:(1) 执行层,scheduler worker 里 `execution_future.result(timeout=self.config.timeout_seconds)`,超时后 set `cancel_event`、`try_set_terminal(TIMED_OUT)`、`execution_future.cancel()` [executor.py:838-847](../backend/packages/harness/deerflow/subagents/executor.py#L838-L847);(2) 轮询层,task_tool 的 `max_poll_count = (timeout_seconds + 60) // 5` 作安全网,兜住"后台卡死但 future 超时没生效"的边角,按默认算是 (1800+60)//5 = 372 次轮询 [task_tool.py:329](../backend/packages/harness/deerflow/tools/builtins/task_tool.py#L329) 和 [task_tool.py:409-421](../backend/packages/harness/deerflow/tools/builtins/task_tool.py#L409-L421);(3) 轮数层,`recursion_limit = max_turns` 限制 LangGraph 循环轮数 [executor.py:526-527](../backend/packages/harness/deerflow/subagents/executor.py#L526-L527)。

**链路解析**:

```
时间治理三层防线(以默认配置为例)
+----------------------------------+----------------------------------------+
| 层1 执行超时(主防线)            | future.result(timeout=1800s)           |
|   scheduler worker 线程内触发     | 超时 -> cancel_event.set()             |
|   error = "Execution timed out   |        + try_set_terminal(TIMED_OUT)   |
|          after 1800 seconds"     |        + execution_future.cancel()     |
+----------------------------------+----------------------------------------+
| 层2 轮询安全网(task_tool)       | max_poll_count = (1800+60)//5 = 372 次 |
|   每 5s 一次轮询                  | 超限 -> request_cancel_background_task |
|   "Task polling timed out after  |        + _schedule_deferred_subagent_  |
|    30 minutes..."                |          cleanup(延迟清理)            |
+----------------------------------+----------------------------------------+
| 层3 轮数上限                     | run_config["recursion_limit"]          |
|   general-purpose=150            |        = config.max_turns              |
|   bash=60, dataclass 默认=50     | 超限 -> GraphRecursionError            |
|   custom agent 不受全局默认影响   |        -> except -> FAILED             |
+----------------------------------+----------------------------------------+

状态收敛: 四条终态路径 -> try_set_terminal(first terminal wins)
   COMPLETED(正常)/ FAILED(异常)/ TIMED_OUT(层1)/ CANCELLED(cancel_event)
```

**Q7.2(深挖)** `max_turns=150` 是怎么变成 LangGraph 的 `recursion_limit` 的?general-purpose 和 bash 为什么数值不一样?

**参考回答**:`SubagentConfig.max_turns` 在 dataclass 里默认 50,两个内置 agent 显式覆盖:general-purpose=150 [general_purpose.py:60](../backend/packages/harness/deerflow/subagents/builtins/general_purpose.py#L60),bash=60 [bash_agent.py:49](../backend/packages/harness/deerflow/subagents/builtins/bash_agent.py#L49)。执行时直接塞进 `run_config["recursion_limit"]` [executor.py:526-527](../backend/packages/harness/deerflow/subagents/executor.py#L526-L527),LangGraph 每跑一个 super-step 计数一次,超限抛 `GraphRecursionError`,被 `_aexecute` 的通用 except 捕获并标 FAILED [executor.py:704-710](../backend/packages/harness/deerflow/subagents/executor.py#L704-L710)。数值差异反映任务形态:general-purpose 要"探索+修改+多步依赖",150 轮给足工具往返空间;bash 是命令执行专家,60 轮够用,且失控 shell 循环的破坏半径更大,轮数给低是风控。registry 还有条微妙规则:全局 `subagents.max_turns` 只覆盖**内置** agent,custom agent 自己定义的值不受全局影响,只有 per-agent override 才行——注释说这是为了不让全局默认值践踏自定义 agent 的自有配置 [registry.py:72-99](../backend/packages/harness/deerflow/subagents/registry.py#L72-L99)。

**Q7.3(深挖)** 超时路径和正常完成路径同时到达,结果状态会不会被写花?

**参考回答**:不会,`SubagentResult.try_set_terminal` 专门解决这个竞态,docstring 写明:"Background timeout/cancellation and the execution worker can race on the same result holder. The first terminal transition wins; late terminal writes must not change status or payload fields" [executor.py:102-117](../backend/packages/harness/deerflow/subagents/executor.py#L102-L117)。实现上用 `_state_lock` 保护,进入临界区先查 `self.status.is_terminal`,已终态直接返回 False,后到者的 result/error/ai_messages/token_usage_records 全部丢弃 [executor.py:121-135](../backend/packages/harness/deerflow/subagents/executor.py#L121-L135)。所有终态写入都收敛到这一个入口:正常完成 [executor.py:698-702](../backend/packages/harness/deerflow/subagents/executor.py#L698-L702)、异常失败 [executor.py:704-710](../backend/packages/harness/deerflow/subagents/executor.py#L704-L710)、超时 [executor.py:843-846](../backend/packages/harness/deerflow/subagents/executor.py#L843-L846)、取消 [executor.py:589-596](../backend/packages/harness/deerflow/subagents/executor.py#L589-L596)。`is_terminal` 判定是枚举集合成员测试:COMPLETED / FAILED / CANCELLED / TIMED_OUT [executor.py:59-66](../backend/packages/harness/deerflow/subagents/executor.py#L59-L66)。**不这样设计会怎样**:超时 worker 把状态写成 TIMED_OUT 后,执行协程恰好跑完又覆盖成 COMPLETED,task_tool 轮询到哪个状态全凭时序,同一个 task_id 可能先报 timed_out 事件又被改成 succeeded,前端卡片状态机直接错乱。

**Q7.4(边界/异常)** 用户中途取消,而子 agent 正在跑一个 5 分钟的 bash 命令,能立刻杀掉吗?

**参考回答**:不能立刻杀,是**协作式**取消。`request_cancel_background_task` 只是对共享的 `cancel_event` 置位 [executor.py:861-876](../backend/packages/harness/deerflow/subagents/executor.py#L861-L876),`_aexecute` 在 `agent.astream()` 的**每个迭代边界**检查这个 event,命中才 `try_set_terminal(CANCELLED)` 退出;流开始前还有一次 pre-check,避免已取消任务空转 [executor.py:574-596](../backend/packages/harness/deerflow/subagents/executor.py#L574-L596)。代码注释坦承局限:"long-running tool calls within a single iteration will not be interrupted until the next chunk is yielded"——单个迭代内的长工具调用打不断,因为线程无法被 `Future.cancel()` 强杀 [executor.py:586-588](../backend/packages/harness/deerflow/subagents/executor.py#L586-L588)。父级协程被取消(CancelledError 传播进 task_tool)时还有一串善后:shield 住 `_await_subagent_terminal` 等子 agent 到终态,把最后的 token 用量报给 RunJournal,能清理就 `cleanup_background_task`,清理不了就 `_schedule_deferred_subagent_cleanup` 延迟清理 [task_tool.py:422-444](../backend/packages/harness/deerflow/tools/builtins/task_tool.py#L422-L444)。清理本身也谨慎:只删"已终态或 completed_at 非空"的条目,避免和还在写状态的后台线程竞态 [executor.py:914-931](../backend/packages/harness/deerflow/subagents/executor.py#L914-L931)。

**链路解析**:

```
用户点击停止 / 父 graph 被取消
   v
两条入口:
   A) request_cancel_background_task(task_id)   # 显式取消 API
   B) task_tool 捕获 asyncio.CancelledError     # 父协程取消传播
   v
result.cancel_event.set()          # 只是置位,不强杀线程
   v
_aexecute 的 astream 迭代边界检查:
   |-- 下一个 chunk 到达时发现置位 -> try_set_terminal(CANCELLED) 退出
   |-- 单迭代内的长工具调用(如 5 分钟 bash)-> 打不断,等到迭代边界
   v
task_tool 善后(CancelledError 分支):
   |-- asyncio.shield(_await_subagent_terminal(task_id, max_polls))
   |-- 终态后 _report_subagent_usage(取消场景 token 账也不丢)
   |-- 已终态 -> cleanup_background_task;未终态 -> 延迟清理任务
   |-- _subagent_usage_cache.pop(tool_call_id) -> re-raise
```

---

## 问题链 8:配置解析、防递归与安全门控

**Q8.1(基础)** `get_subagent_config("general-purpose")` 背后有几层配置?优先级怎么排?

**参考回答**:三层,顺序写在 docstring 里 [registry.py:50-64](../backend/packages/harness/deerflow/subagents/registry.py#L50-L64):(1) 内置 `BUILTIN_SUBAGENTS`(general-purpose / bash)[builtins/__init__.py:12-15](../backend/packages/harness/deerflow/subagents/builtins/__init__.py#L12-L15);(2) config.yaml 的 `custom_agents` 段,内置查不到才查它 [registry.py:66-68](../backend/packages/harness/deerflow/subagents/registry.py#L66-L68);(3) config.yaml 的 `agents` 段 per-agent override(timeout / max_turns / model / skills),外加只作用于内置 agent 的全局 `timeout_seconds` / `max_turns` 默认值 [registry.py:72-114](../backend/packages/harness/deerflow/subagents/registry.py#L72-L114)。后一层有个容易答错的点:全局默认值**故意不**覆盖 custom agent——注释写明 custom agents 在 custom_agents 段里自带默认值,全局默认只该管内置 agent,per-agent override 才对两者通吃 [registry.py:72-76](../backend/packages/harness/deerflow/subagents/registry.py#L72-L76)。最终用 `dataclasses.replace(config, **overrides)` 生成新实例返回,不原地改全局内置对象,避免不同请求间的配置污染 [registry.py:113-114](../backend/packages/harness/deerflow/subagents/registry.py#L113-L114)。这个"内置 < custom < per-agent override"的分层是对齐 Codex 配置层叠惯例的,docstring 里有注明。

**链路解析**:

```
get_subagent_config(name)
   |
   |-- (1) BUILTIN_SUBAGENTS.get(name)            # general-purpose / bash
   |         |-- miss -> _build_custom_subagent_config(config.yaml custom_agents)
   |         |-- 都 miss -> None -> task_tool 返回 "Unknown subagent type '...'"
   v
   |-- (2) agents[name] per-agent override?
   |         timeout_seconds / max_turns / model / skills
   |-- (3) 内置专属: 全局 timeout(默认 1800s)/ max_turns 默认值
   |         (is_builtin 判定: name in BUILTIN_SUBAGENTS)
   v
dataclasses.replace(config, **overrides)   # 不可变更新,内置单例不被污染

bash 类型的两道安全门:
   名单门: get_available_subagent_names
   |-- is_host_bash_allowed() == False -> 从名单剔除 "bash"
   |-- 判定本身抛异常 -> 暴露全部(可用性优先)
   执行门: task_tool 内再查一次
   |-- 不允许 -> "Error: {LOCAL_BASH_SUBAGENT_DISABLED_MESSAGE}"
   分工: 名单门防"模型主动选", 执行门防"模型幻觉编造类型名"
```

**Q8.2(深挖)** 子 agent 能不能再委派子 agent(递归嵌套)?代码在哪几层堵住的?

**参考回答**:堵了三道。第一道在配置默认值:`SubagentConfig.disallowed_tools` 默认就是 `["task"]` [config.py:36](../backend/packages/harness/deerflow/subagents/config.py#L36),两个内置 agent 也都显式把 task 列进 disallowed [general_purpose.py:58](../backend/packages/harness/deerflow/subagents/builtins/general_purpose.py#L58);第二道在工具装配:task_tool 调 `get_available_tools(..., subagent_enabled=False)` 拿给子 agent 的工具集,从源头就不生成 task 工具 [task_tool.py:295-303](../backend/packages/harness/deerflow/tools/builtins/task_tool.py#L295-L303);第三道在 executor 构造时 `_filter_tools` 应用 allow/deny 双名单兜底 [executor.py:326-331](../backend/packages/harness/deerflow/subagents/executor.py#L326-L331) 和 [executor.py:248-275](../backend/packages/harness/deerflow/subagents/executor.py#L248-L275)。**不这样设计会怎样**:子 agent 拿到 task 工具就能无限递归委派,每个分支起独立 LLM 循环,深度不受控、并发也不受 SubagentLimitMiddleware 约束(它只挂在 lead agent 上),token 成本指数爆炸;取消信号只覆盖第一层,深层子 agent 变成孤儿任务耗到 1800 秒超时。同样被禁的还有 `ask_clarification`(子 agent 无人可问,配置了也没人答)和 `present_files`(产物统一由父级呈现)[general_purpose.py:58](../backend/packages/harness/deerflow/subagents/builtins/general_purpose.py#L58)。

**Q8.3(边界/异常)** `subagent_type="bash"` 在任何部署环境下都能用吗?

**参考回答**:不能,有两道门。registry 侧:`get_available_subagent_names` 调 `is_host_bash_allowed()`,host bash 不被允许时直接从暴露名单里剔除 "bash",模型在工具描述里就看不到这个类型;判定本身抛异常时选择"暴露全部"而非全禁,可用性优先 [registry.py:150-165](../backend/packages/harness/deerflow/subagents/registry.py#L150-L165)。执行侧:task_tool 里再查一次,不允许就返回 `Error: {LOCAL_BASH_SUBAGENT_DISABLED_MESSAGE}` [task_tool.py:239-242](../backend/packages/harness/deerflow/tools/builtins/task_tool.py#L239-L242)。双保险的分工:名单过滤防"模型主动选",执行期检查防"模型幻觉出已隐藏的类型名"——LLM 完全可能编造一个它在描述里没见过的 subagent_type,所以执行期这道必须独立存在。bash 子 agent 的工具集也被限到沙箱五件套 `["bash", "ls", "read_file", "write_file", "str_replace"]` [bash_agent.py:46](../backend/packages/harness/deerflow/subagents/builtins/bash_agent.py#L46),与 general-purpose 的 `tools=None`(继承父级全部)[general_purpose.py:57](../backend/packages/harness/deerflow/subagents/builtins/general_purpose.py#L57) 形成鲜明对比——能力越大越要显式收缩。

---

## 面试官最爱追问的 3 个点

1. **"并发上限 3 到底在哪强制?模型不听话一次发 8 个 task call 呢?"** —— 应答策略:先答 prompt 是软约束不可信,再精确到 `SubagentLimitMiddleware._truncate_task_calls` 在 after_model 钩子里按下标截断、用 `clone_ai_message_with_tool_calls` 同步 raw payload 双写,顺带提 `_clamp_subagent_limit` 把配置钳到 [2,4] 的防呆 [subagent_limit_middleware.py:37-39](../backend/packages/harness/deerflow/agents/middlewares/subagent_limit_middleware.py#L37-L39)。
2. **"超时/取消和正常完成撞在一起,状态会不会错乱?"** —— 应答策略:祭出 `try_set_terminal` 的 "first terminal wins" 语义加 `_state_lock`,强调完成/失败/超时/取消四条路径全部收敛到这一个入口;取消是协作式的,只在 astream 迭代边界生效,单迭代内的长工具调用打不断是已知取舍 [executor.py:102-135](../backend/packages/harness/deerflow/subagents/executor.py#L102-L135)。
3. **"为什么子 agent 不开 checkpoint?开了会怎样?"** —— 应答策略:一次性执行单元无需恢复;复用父 thread_id 时并发子 agent 会互相串台;checkpoint 会被 150 轮 × N 个子任务的中间状态灌爆。再补一刀:不开 checkpoint 不等于没状态,初始 state 由 `_build_initial_state` 手工构造,sandbox/thread_data 从父级透传 [executor.py:481-485](../backend/packages/harness/deerflow/subagents/executor.py#L481-L485)。
