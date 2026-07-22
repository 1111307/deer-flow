# Gateway 与运行时(HTTP/SSE/RunManager)—— 面试问题链

> 本文档按"大厂面试追问链"组织:每条问题链从基础问题开始,层层追问到实现细节、设计权衡、异常边界。配套深读笔记:无(本模块尚无深读笔记,本文档是第一份)(深读笔记讲"怎么实现",本文档讲"怎么被问、怎么答")。
> 代码引用格式:[文件名.py:起始行-结束行](../backend/仓库相对路径#L起始行-L结束行)。只引用你实际读过的行,禁止编造行号。

## 问题链 1:完整请求链路 —— 从 POST /runs/stream 到 SSE 事件流

**Q1.1(基础)** 假设前端调 `POST /api/threads/{thread_id}/runs/stream`,你给我讲讲这个请求在 Gateway 里完整走一遍会经过哪些组件?

**参考回答**:入口在 [thread_runs.py:150-175](../backend/app/gateway/routers/thread_runs.py#L150-L175) 的 `stream_run`:先经 `@require_permission("runs", "create", owner_check=True)` 做线程归属校验,然后委托服务层 `start_run` 创建 RunRecord 并启动后台任务,最后返回一个 `StreamingResponse`,其生成器是 `sse_consumer`。`start_run`([services.py:294-439](../backend/app/gateway/services.py#L294-L439))做四件事:线程归属二次校验(404 防枚举)、`run_mgr.create_or_reject` 原子创建 run、upsert thread_meta、`asyncio.create_task(run_agent(...))` 启动 LangGraph 后台执行。`run_agent`([worker.py:124-437](../backend/packages/harness/deerflow/runtime/runs/worker.py#L124-L437))驱动 `agent.astream()` 把每个 chunk 通过 `bridge.publish` 投进 StreamBridge,`sse_consumer` 从 bridge 订阅并格式化成 SSE 帧吐给客户端。

路由挂载关系也值得知道:`thread_runs.router` 挂在 `/api/threads` 前缀下,提供 runs 生命周期(create/stream/wait/cancel/join);`runs.router` 挂在 `/api/runs` 前缀下,提供 stateless stream/wait;两者在 `create_app` 里注册([app.py:403-407](../backend/app/gateway/app.py#L403-L407))。路由器本身很薄,业务逻辑集中在 `services.py`——这是刻意的分层:router 只做 HTTP 语义(状态码、响应头、Pydantic 模型),service 层可以被 IM channel 等非 HTTP 入口复用,`start_run` 的 docstring 也明说 "Router modules are thin HTTP handlers that delegate here"([services.py:1-6](../backend/app/gateway/services.py#L1-L6))。

**链路解析**:

```
浏览器 POST /api/threads/{tid}/runs/stream
  │ (cookie + CSRF header)
  ▼
CORSMiddleware → CSRFMiddleware → AuthMiddleware   [app.py:341-358]
  (Starlette: 后 add 的在最外层,所以注册顺序 Auth→CSRF→CORS
   对应执行顺序 CORS→CSRF→Auth;preflight OPTIONS 由 CORS 直接应答)
  ▼ request.state.user 已盖章
@require_permission("runs","create",owner_check)  [thread_runs.py:151]
  ▼
stream_run() ──► start_run()                      [services.py:294]
  │                ├─ create_or_reject() → RunRecord(pending)   [manager.py:543]
  │                ├─ thread_store upsert(running)
  │                └─ asyncio.create_task(run_agent) ──┐
  ▼                                                   │
StreamingResponse(sse_consumer(...))                  │ 后台任务
  │ subscribe(run_id) ◄── StreamBridge ◄── publish ◄──┤ agent.astream()
  │                                                   │ (LangGraph + checkpointer)
  ▼ yield format_sse(...)                             │
SSE: event: metadata / values / messages / end  ◄─────┘ publish_end
```

**Q1.2(深挖)** `start_run` 里 `create_or_reject` 和 `asyncio.create_task` 是两个独立动作,中间如果客户端断连或者创建任务失败,会不会出现一个永远没有执行体的 run 记录?

**参考回答**:不会泄漏"永远 pending"的记录。`create_or_reject`([manager.py:543-629](../backend/packages/harness/deerflow/runtime/runs/manager.py#L543-L629))在同一个 `asyncio.Lock` 里完成"检查 inflight + 插入内存 + 持久化"三步,持久化失败会回滚内存记录并抛异常([manager.py:603-614](../backend/packages/harness/deerflow/runtime/runs/manager.py#L603-L614))。之后 `start_run` 把 `asyncio.create_task(run_agent(...))` 挂到 `record.task`([services.py:415-430](../backend/app/gateway/services.py#L415-L430));`run_agent` 自身的 try/except/finally 兜底所有路径,即使抛异常也会 `set_status(error)` 并 `publish_end`([worker.py:387-398](../backend/packages/harness/deerflow/runtime/runs/worker.py#L387-L398),[worker.py:436-437](../backend/packages/harness/deerflow/runtime/runs/worker.py#L436-L437))。最坏情况(进程崩溃)留下的 pending/running 行,sqlite 后端会在下次启动时被 `reconcile_orphaned_inflight_runs` 标成 error([deps.py:219-223](../backend/app/gateway/deps.py#L219-L223))。

还要注意 `start_run` 是**两条入口的汇合点**:threaded 端点 `/api/threads/{tid}/runs/stream` 靠装饰器做归属校验,而 stateless 端点 `/api/runs/stream`([runs.py:35-57](../backend/app/gateway/routers/runs.py#L35-L57))的 thread_id 藏在 body 的 `config.configurable.thread_id` 里(没带就 `uuid4()` 新建临时线程,[runs.py:27-32](../backend/app/gateway/routers/runs.py#L27-L32)),装饰器够不到——所以 `start_run` 内部自己做 `thread_store.check_access(thread_id, user.id)`,不通过返回 **404 而不是 403**,和 threaded 路径一致的反枚举行为;内部 channel 调用还要按 owner 头再查一次,注释明说"泄露的内部 token 不能授予跨用户线程访问"([services.py:334-356](../backend/app/gateway/services.py#L334-L356))。创建 run 前还有一道 model allowlist 校验:`context.model_name` 不在配置名单里直接 400([services.py:324-332](../backend/app/gateway/services.py#L324-L332))。

**Q1.3(边界/异常)** SSE 帧的 wire format 具体长什么样?响应头里那个 `Content-Location` 是干什么的,去掉会怎样?

**参考回答**:`format_sse`([services.py:50-63](../backend/app/gateway/services.py#L50-L63))按 `event:` → `data:` → 可选 `id:` → 空行拼帧,对齐 LangGraph Platform 协议,`useStream` React hook 和 `langgraph-sdk` 的 SSE 解码器原样可消费。`Content-Location: /api/threads/{tid}/runs/{run_id}`([thread_runs.py:166-174](../backend/app/gateway/routers/thread_runs.py#L166-L174))是 Platform 协议的一部分,SDK 用贪婪正则从这个路径里提取 run_id,所以必须指向规范资源路径、不能带后缀。去掉它,`useStream` 拿不到 run 元数据,后续的 cancel/joinStream 都拼不出 URL。另外 `X-Accel-Buffering: no` 是告诉 nginx 不要缓冲 SSE,否则事件会被攒批,前端看到的就不是实时的。

事件类型上,第一帧固定是 `metadata`(携带 `run_id` 和 `thread_id`,[worker.py:203-211](../backend/packages/harness/deerflow/runtime/runs/worker.py#L203-L211))——`useStream` 两个值都要;之后是按 stream_mode 映射的事件流(`values`/`updates`/`messages` 等,`_lg_mode_to_sse_event` 目前 1:1 映射,[worker.py:554-563](../backend/packages/harness/deerflow/runtime/runs/worker.py#L554-L563));异常时发 `error` 帧([worker.py:391-398](../backend/packages/harness/deerflow/runtime/runs/worker.py#L391-L398));终帧是 `publish_end` 触发的 `end` 事件([worker.py:436](../backend/packages/harness/deerflow/runtime/runs/worker.py#L436),[services.py:464-466](../backend/app/gateway/services.py#L464-L466))。一个容易踩的坑:LangGraph 的 `events` stream_mode 在网关里**不支持**——它需要 `astream_events`,和 `astream` 的 values 快照不能同时产出,worker 里显式跳过并记日志([worker.py:158-162](../backend/packages/harness/deerflow/runtime/runs/worker.py#L158-L162),[worker.py:1-14](../backend/packages/harness/deerflow/runtime/runs/worker.py#L1-L14));`messages-tuple` 则映射为 LangGraph 的 `messages` 模式([worker.py:286-288](../backend/packages/harness/deerflow/runtime/runs/worker.py#L286-L288))。

**Q1.4(深挖)** `body.config` 里的 `configurable` 和 `context` 两个字段都往 LangGraph 传,它们什么关系?客户端两个都传会怎样?

**参考回答**:这是 LangGraph 版本演进留下的兼容层。LangGraph >= 0.6 引入 `context` 作为线程级数据的推荐通道,并且**拒绝同时带 `configurable` 和 `context` 的请求**;LangGraph >= 1.1.9 又不再让 `ToolRuntime.context` 回退读 `configurable`。`build_run_config` 的策略([services.py:204-286](../backend/app/gateway/services.py#L204-L286)):客户端传了 `context` 就尊重它、跳过自建 `configurable`(同时告警,[services.py:239-245](../backend/app/gateway/services.py#L239-L245));没传就按老方式建 `configurable={"thread_id": ...}`。而 DeerFlow 自己的扩展键(`model_name`、`thinking_enabled`、`agent_name` 等白名单,[_CONTEXT_CONFIGURABLE_KEYS, services.py:128-140](../backend/app/gateway/services.py#L128-L140))由 `merge_run_context_overrides` 用 `setdefault` **同时写进两边**([services.py:143-166](../backend/app/gateway/services.py#L143-L166))—— legacy configurable 读者和新 ToolRuntime.context 读者都能看到。另一个具体数字:lead agent 的 `recursion_limit=100` 个 super-step 在这里硬编码([services.py:233](../backend/app/gateway/services.py#L233)),注释明确说它和 subagent 的 `max_turns` 是两本账,不许混淆。另外 `assistant_id` 若不是 `lead_agent`,会被规范化(小写、下划线转连字符、正则 `[a-z0-9-]+` 校验)后作为 `agent_name` 注入,自定义 agent 本质是"lead_agent + agent_name 路由"([services.py:267-283](../backend/app/gateway/services.py#L267-L283),[services.py:190-201](../backend/app/gateway/services.py#L190-L201))。

## 问题链 2:StreamBridge 与断连语义

**Q2.1(基础)** 为什么不直接让 `run_agent` 往 SSE 响应里写,而是中间隔一层 StreamBridge?

**参考回答**:这是生产者-消费者解耦。StreamBridge 是抽象基类([base.py:37-72](../backend/packages/harness/deerflow/runtime/stream_bridge/base.py#L37-L72)),定义了 `publish / publish_end / subscribe / cleanup` 四个方法。有了这层:`/runs/stream`(创建并消费)、`/runs/{id}/join`(纯消费)、`/wait`(消费但不吐 SSE)三种端点共享同一个生产者;`join` 端点可以让第二个客户端订阅同一个 run([thread_runs.py:262-282](../backend/app/gateway/routers/thread_runs.py#L262-L282))。默认实现是 `MemoryStreamBridge`,工厂在 [async_provider.py:29-55](../backend/packages/harness/deerflow/runtime/stream_bridge/async_provider.py#L29-L55),配置里预留了 redis 类型但抛 `NotImplementedError("Redis stream bridge planned for Phase 2")`——这就是横向扩展的接缝。

协议层还有两个设计点。一是事件模型 `StreamEvent(id, event, data)`([base.py:16-31](../backend/packages/harness/deerflow/runtime/stream_bridge/base.py#L16-L31)),id 是单调递增的,直接用作 SSE 的 `id:` 字段支撑 `Last-Event-ID` 重连。二是两个哨兵:`HEARTBEAT_SENTINEL` 和 `END_SENTINEL` 是模块级单例([base.py:33-34](../backend/packages/harness/deerflow/runtime/stream_bridge/base.py#L33-L34)),消费者用 `entry is HEARTBEAT_SENTINEL` 身份比较而不是字符串比较,杜绝了和真实事件撞名的可能。`cleanup(run_id, delay=N)` 的 delay 语义是给迟到订阅者留排空窗口——run 结束后不是立刻删流,而是 `delay=60` 秒后清([worker.py:437](../backend/packages/harness/deerflow/runtime/runs/worker.py#L437))。

**链路解析**:

```
run_agent (生产者, 每 run 一个 asyncio.Task)
  │ bridge.publish(run_id, event, serialize(chunk))
  ▼
MemoryStreamBridge._streams[run_id] = _RunStream{
    events: list[StreamEvent]   # 最多保留 queue_maxsize=256 条 [memory.py:32]
    condition: asyncio.Condition
    ended: bool
}
  ▲ subscribe(run_id, last_event_id, heartbeat_interval=15.0)
  │
  ├─ sse_consumer   → /runs/stream 与 /join (SSE 客户端)
  └─ wait_for_run_completion → /wait (阻塞 HTTP 客户端)
```

**Q2.2(深挖)** `on_disconnect` 有 `cancel` 和 `continue` 两种,具体是在哪一行生效的?如果客户端在 run 已经 success 之后断开,会误杀吗?

**参考回答**:生效点在 `sse_consumer` 的 `finally`([services.py:470-473](../backend/app/gateway/services.py#L470-L473)):只有当 `record.status` 仍是 `pending/running` 且 `on_disconnect == DisconnectMode.cancel` 时才调 `run_mgr.cancel`。状态判断在前,所以 run 已到终态(success/error/interrupted)时 finally 什么都不做,不会误杀。默认值上,`RunCreateRequest.on_disconnect` 默认 `"cancel"`([thread_runs.py:52](../backend/app/gateway/routers/thread_runs.py#L52))——浏览器场景用户关了页面通常就是希望停;`continue` 留给"提交后关页面、稍后回来 join"的场景。检测断连靠每个事件循环里轮询 `request.is_disconnected()`([services.py:456-458](../backend/app/gateway/services.py#L456-L458)),心跳哨兵保证即使 agent 长时间不吐事件,循环也至少每 15 秒醒一次来检测断连。

心跳的实现值得展开:`subscribe` 的等待是 `await asyncio.wait_for(stream.condition.wait(), timeout=heartbeat_interval)`(`heartbeat_interval=15.0`,[memory.py:90](../backend/packages/harness/deerflow/runtime/stream_bridge/memory.py#L90),[memory.py:113-116](../backend/packages/harness/deerflow/runtime/stream_bridge/memory.py#L113-L116)),超时就把 `HEARTBEAT_SENTINEL` 递给消费者,`sse_consumer` 把它翻译成 SSE 注释行 `": heartbeat\n\n"`([services.py:460-462](../backend/app/gateway/services.py#L460-L462))——注释行对 SDK 是透明的,但能让浏览器/EventSource 和中间代理不把连接当死连接掐掉。一个容易被问的边界:`is_disconnected()` 轮询只发生在"有事件醒来"时,如果没有心跳机制,agent 静默 10 分钟期间客户端断开,服务端要到下一个事件才发现——这 10 分钟的孤儿执行就是白烧的 LLM token,所以心跳在这里不是保活装饰,而是断连检测的计时器。

**Q2.3(边界/异常)** 客户端网络抖动断开后用 `Last-Event-ID` 重连,事件会不会丢?buffer 满了会怎样?

**参考回答**:`MemoryStreamBridge` 每个 run 保留有界事件日志,`queue_maxsize=256`([memory.py:32-33](../backend/packages/harness/deerflow/runtime/stream_bridge/memory.py#L32-L33)),超出时从头截断并推进 `start_offset`([memory.py:71-77](../backend/packages/harness/deerflow/runtime/stream_bridge/memory.py#L71-L77))。重连时 `sse_consumer` 把请求头 `Last-Event-ID` 传给 `subscribe`([services.py:454-456](../backend/app/gateway/services.py#L454-L456)),`_resolve_start_offset` 在 retained buffer 里找到该 id 并从下一条开始重放;找不到就告警并从最早保留事件重放([memory.py:51-64](../backend/packages/harness/deerflow/runtime/stream_bridge/memory.py#L51-L64))。所以:断开期间事件数 ≤256 条时不丢;超过则丢最老的,订阅方还会收到"fell behind"告警日志。事件 id 是 `{毫秒时间戳}-{序号}`([memory.py:45-49](../backend/packages/harness/deerflow/runtime/stream_bridge/memory.py#L45-L49))。run 结束后 60 秒清理整个流(`bridge.cleanup(run_id, delay=60)`,[worker.py:437](../backend/packages/harness/deerflow/runtime/runs/worker.py#L437))。

订阅循环本身是一个"偏移量游标 + condition 变量"模型([memory.py:85-123](../backend/packages/harness/deerflow/runtime/stream_bridge/memory.py#L85-L123)):每次迭代先在锁内把 `next_offset` 换算成 buffer 局部下标,有未读事件就直接取;没有且流已 ended 就递 `END_SENTINEL`;否则 `condition.wait()` 挂起,被 `publish`/`publish_end` 的 `notify_all` 唤醒或被 15 秒心跳超时唤醒。注意它不是 asyncio.Queue——Queue 是单消费者语义,事件取走就没了,无法支撑多订阅者(join)和 Last-Event-ID 重放;事件日志 + 每订阅者独立偏移量才是正确抽象。如果订阅者消费太慢,偏移量落到 `start_offset` 之后,循环里会强制对齐到 `start_offset` 并记告警([memory.py:98-104](../backend/packages/harness/deerflow/runtime/stream_bridge/memory.py#L98-L104))——慢消费者丢事件但不会被卡死,这是有界 buffer 的必然取舍。

## 问题链 3:cancel 与 multitask_strategy

**Q3.1(基础)** `POST /runs/{id}/cancel?action=interrupt` 和 `action=rollback` 有什么区别?底层是怎么停掉一个正在跑 LangGraph 的任务的?

**参考回答**:`cancel_run` 端点在 [thread_runs.py:227-259](../backend/app/gateway/routers/thread_runs.py#L227-L259)。`RunManager.cancel`([manager.py:512-541](../backend/packages/harness/deerflow/runtime/runs/manager.py#L512-L541))做三件事:把 `abort_action` 记到 record 上、`abort_event.set()`、`record.task.cancel()` 给 asyncio 任务发 CancelledError,然后把状态置为 `interrupted` 并持久化。双通道设计:`abort_event` 让 `run_agent` 的流式循环在每个 chunk 边界自查并 break([worker.py:313-315](../backend/packages/harness/deerflow/runtime/runs/worker.py#L313-L315)),`task.cancel()` 兜底打断阻塞中的 await。interrupt 保留当前 checkpoint(可 resume);rollback 额外把线程状态回滚到 run 启动前的快照。

响应语义也值得一提:`wait=false`(默认)立即返回 202(取消已受理、不保证已停);`wait=true` 会 `await record.task` 吞掉 `CancelledError` 后返回 204([thread_runs.py:252-259](../backend/app/gateway/routers/thread_runs.py#L252-L259))。`cancel` 是幂等的:对已经 `interrupted` 的 run 再次 cancel 返回 True 而不是报错([manager.py:529-530](../backend/packages/harness/deerflow/runtime/runs/manager.py#L529-L530));但对 `success/error` 等其它终态返回 False → 409,detail 文案按状态区分("not active on this worker" vs "not cancellable")([thread_runs.py:109-112](../backend/app/gateway/routers/thread_runs.py#L109-L112))。前端停止按钮走的是 `POST /runs/{id}/stream?action=interrupt`([thread_runs.py:289-334](../backend/app/gateway/routers/thread_runs.py#L289-L334)):先 cancel,再把 buffer 里剩余事件流完,让客户端观察到干净的关停。

**链路解析**:

```
cancel_run(action=rollback)
  ▼
RunManager.cancel(run_id, action)
  ├─ record.abort_action = "rollback"
  ├─ record.abort_event.set()      ──► run_agent 流循环下一 chunk 自查 break
  ├─ record.task.cancel()          ──► CancelledError 注入阻塞点
  └─ status = interrupted; persist
              ▼
run_agent 收尾 (两条路径汇合):
  break 路径 [worker.py:340-355] / CancelledError 路径 [worker.py:367-385]
  │ action == "rollback"
  ▼
_rollback_to_pre_run_checkpoint(...)   [worker.py:456-546]
  ├─ aput(旧的 checkpoint 深拷贝, 新 id/ts)
  └─ aput_writes(恢复的 pending_writes)
```

**Q3.2(深挖)** rollback 具体怎么恢复到"run 之前"的状态?LangGraph 的 checkpointer 没有提供"回滚"原语吧?

**参考回答**:对,是应用层自己实现的。`run_agent` 在标 running 之后、启动图之前,先 `aget_tuple` 拿当前 checkpoint 并 `copy.deepcopy` 存下 `pre_run_checkpoint_id` 和 `pre_run_snapshot`(含 checkpoint、metadata、pending_writes)([worker.py:186-201](../backend/packages/harness/deerflow/runtime/runs/worker.py#L186-L201))。回滚时 `_rollback_to_pre_run_checkpoint` 给旧快照换上新的 `id/ts`(通过 `empty_checkpoint()` 生成的新 marker,[worker.py:492-497](../backend/packages/harness/deerflow/runtime/runs/worker.py#L492-L497))再 `aput` 写回——相当于把"过去的状态"作为"最新的 checkpoint"重新插入,利用 LangGraph "latest = max(checkpoint_id)" 的规则让旧状态重新成为最新。最后按 task_id 分组 `aput_writes` 恢复 pending_writes([worker.py:525-546](../backend/packages/harness/deerflow/runtime/runs/worker.py#L525-L546))。边界:若启动前线程没有 checkpoint(`pre_run_snapshot is None`),rollback 退化为 `adelete_thread` 清空([worker.py:474-477](../backend/packages/harness/deerflow/runtime/runs/worker.py#L474-L477));快照捕获失败则直接跳过回滚只告警。

实现里还有两个面试可以加分的细节。一是 `_call_checkpointer_method`([worker.py:445-453](../backend/packages/harness/deerflow/runtime/runs/worker.py#L445-L453)):对每个 checkpointer 调用先取异步名再取同步名,返回值是 awaitable 就 await——这让 rollback 逻辑对 InMemorySaver(同步 API)和 AsyncSqliteSaver/AsyncPostgresSaver(异步 API)都成立。二是防御性校验:pending_write 不是 3 元组、channel 不是字符串、恢复后没返回 checkpoint_id,都直接 `raise RuntimeError`([worker.py:516-536](../backend/packages/harness/deerflow/runtime/runs/worker.py#L516-L536))——回滚是"灾后恢复"路径,宁可失败暴露也不能写出一个悄悄损坏的 checkpoint。另外注意回滚后 run 的状态是 `error` 配 "Rolled back by user"([worker.py:342-343](../backend/packages/harness/deerflow/runtime/runs/worker.py#L342-L343)),不是 interrupted:从用户视角这是一次"撤销",从审计视角它确实没有成功产出。

**Q3.3(边界/异常)** 我用一个重启前创建的 run_id 去调 cancel,返回 409 "is not active on this worker",为什么?内存里明明能查到这个 run 啊?

**参考回答**:你查到的是从 RunStore **水合**出来的只读记录。`RunManager.get` 在内存 miss 时会 fallback 到 store 查询并构造 `store_only=True` 的 RunRecord([manager.py:267-300](../backend/packages/harness/deerflow/runtime/runs/manager.py#L267-L300),[manager.py:402-432](../backend/packages/harness/deerflow/runtime/runs/manager.py#L402-L432))——它有元数据但没有 `task` 和 `abort_event`,因为这两个是进程内对象,进程重启后就没了,执行体可能早死了,也可能活在另一个 worker 上。`cancel` 只操作内存 `_runs` dict,水合记录不在其中,返回 False,路由层转成 409([thread_runs.py:248-250](../backend/app/gateway/routers/thread_runs.py#L248-L250));`join`/`stream` 对 `store_only` 记录也直接 409([thread_runs.py:270-271](../backend/app/gateway/routers/thread_runs.py#L270-L271))。这是诚实的失败:与其假装 cancel 成功,不如告诉客户端"这个执行体不归我管"。配套地,sqlite 后端启动时 `reconcile_orphaned_inflight_runs` 会把这些孤儿标成 error([manager.py:631-683](../backend/packages/harness/deerflow/runtime/runs/manager.py#L631-L683)),把歧义态收敛成显式终态。

`get` 的水合路径还有一个精巧的并发处理:store 查询是 await,await 期间另一个请求可能恰好在本 worker 上创建了同名 run 的内存记录,所以拿到 store 行之后要**再持锁复查一次**内存,命中就返回内存记录而不是水合记录([manager.py:420-425](../backend/packages/harness/deerflow/runtime/runs/manager.py#L420-L425))——内存记录永远优先,因为它带着活的 task 句柄。水合记录默认值也有讲究:老行缺 `status`/`on_disconnect` 列时默认 `pending` 和 `cancel`([manager.py:274-279](../backend/packages/harness/deerflow/runtime/runs/manager.py#L274-L279)),这是 schema 演进期写入的历史数据的兼容策略。`list_by_thread` 同理是内存 + store 合并后按 `created_at` 倒序、默认截断 100 条([manager.py:441-471](../backend/packages/harness/deerflow/runtime/runs/manager.py#L441-L471)),同一 run_id 内存优先。

**Q3.4(深挖)** `multitask_strategy` 有 reject/interrupt/rollback,为什么把"检查有没有 inflight"和"创建新 run"放进同一把锁?分两步检查-then-act 不是更简单吗?

**参考回答**:那是经典的 TOCTOU 竞态。两个并发请求各自 `has_inflight()` 都看到 False,然后各自 `create()`,同一个 thread 就跑起了两个 run。`create_or_reject` 在 `async with self._lock` 内完成"查 inflight → 按策略 reject(抛 `ConflictError` → 路由转 409)/ interrupt/rollback(取消旧的)→ 插入新记录 + 持久化"全过程([manager.py:570-624](../backend/packages/harness/deerflow/runtime/runs/manager.py#L570-L624))。**不这样设计会怎样**:并发双击发送按钮会产生两个并发 run 同时写同一线程的 checkpoint,checkpoint 历史交错,messages 流出现双份 AI 回复。注意 `enqueue` 策略在 schema 里允许但运行时抛 `UnsupportedStrategyError` → 501([manager.py:567-572](../backend/packages/harness/deerflow/runtime/runs/manager.py#L567-L572),[services.py:373-374](../backend/app/gateway/services.py#L373-L374))。

interrupt/rollback 策略下"取消旧 run"也在这把锁里做:给旧 record 设 `abort_action = multitask_strategy`、`abort_event.set()`、`task.cancel()`、置 interrupted([manager.py:616-624](../backend/packages/harness/deerflow/runtime/runs/manager.py#L616-L624)),锁释放后再逐条持久化旧 run 的 interrupted 状态([manager.py:626-627](../backend/packages/harness/deerflow/runtime/runs/manager.py#L626-L627))——持久化是 IO,故意移出锁外,锁内只做内存状态切换,持锁时间最小化。这里 rollback 策略的语义是"新 run 取代旧 run,旧 run 按 rollback 收尾":旧 run 的 worker 循环在下一个 chunk 看到 abort_event,走 Q3.2 的快照恢复路径。还要提醒一点:这个原子性只在**单进程**内成立(见问题链 7),多 worker 下两个请求落到不同进程,各自锁各自的,`reject` 也会失效——这再次解释了 GATEWAY_WORKERS=1 的默认值。

## 问题链 4:POST /wait 为什么用 wait_for_run_completion 而不是裸 await task

**Q4.1(基础)** `/runs/wait` 和 `/runs/stream` 的区别是什么?wait 内部是怎么"等"的?

**参考回答**:`/stream` 返回 SSE 流,`/wait` 阻塞到 run 终态后直接返回最终 checkpoint 的 channel_values([thread_runs.py:178-202](../backend/app/gateway/routers/thread_runs.py#L178-L202))。wait 不是 `await record.task`,而是调 `wait_for_run_completion`([services.py:476-521](../backend/app/gateway/services.py#L476-L521))——它和 `sse_consumer` 消费同一个 StreamBridge,循环里等 `END_SENTINEL`,每轮醒来自查 `request.is_disconnected()`。拿到 END 后用 `checkpointer.aget_tuple` 读最终 checkpoint,经 `serialize_channel_values_for_api` 序列化返回([thread_runs.py:191-198](../backend/app/gateway/routers/thread_runs.py#L191-L198))。

有个防御性细节:`END_SENTINEL` 分支在 `is_disconnected()` 检查**之前**([services.py:508-515](../backend/app/gateway/services.py#L508-L515)),注释说"honour it even if the client just disconnected so the caller still serializes the real final checkpoint"——run 刚好在客户端断开瞬间完成时,返回的是真实终态的完整 checkpoint,而不是被断连逻辑降级成半成品。读 checkpoint 失败(比如 checkpointer 恰好异常)也不 500,只记日志然后落到 `{"status", "error"}` 兜底返回([thread_runs.py:199-202](../backend/app/gateway/routers/thread_runs.py#L199-L202))。序列化侧 `serialize_channel_values_for_api` 除了剥 `__pregel_*` 内部键,还会把 `hide_from_ui` 消息里的 base64 `data:` 图片块剥掉([serialization.py:110-121](../backend/packages/harness/deerflow/runtime/serialization.py#L110-L121))——内部模型上下文绝不上 wire。

**链路解析**:

```
POST /runs/wait
  ▼
start_run → record + background task (同 /stream)
  ▼
wait_for_run_completion(bridge, record, request, run_mgr)
  │ subscribe(run_id)
  │  loop:
  │   END_SENTINEL? → completed=True, return True ──► aget_tuple → serialize → 200
  │   is_disconnected()? → break → return False ────► {"status":..., "error":...}
  │   heartbeat/事件 → 继续等
  ▼ finally
  未 completed 且 pending/running 且 on_disconnect=cancel → run_mgr.cancel
```

**Q4.2(深挖)** 为什么不用裸的 `await record.task`?反正 task 结束 run 就结束了,等 task 不是更直接?那客户端中途断开时,/wait 到底返回什么?

**参考回答**:这正是 issue #3265 修掉的 bug,`wait_for_run_completion` 的 docstring 写得很清楚([services.py:483-503](../backend/app/gateway/services.py#L483-L503))。**不这样设计会怎样**:客户端(或中间代理)在长工具调用(比如 `pip install`)期间超时断开,HTTP handler 的 await 被取消,handler 吞下 `CancelledError` 后把"当时恰好存在的半个 checkpoint"序列化返回——一个跑了一半的 run 被伪装成正常完成响应。用 bridge 等有三个好处:一是和 SSE 路径共享同一套断连语义,断连时按 `on_disconnect` 决定是否 cancel 后台 run;二是 bridge 的 15 秒心跳哨兵([base.py:53-55](../backend/packages/harness/deerflow/runtime/stream_bridge/base.py#L53-L55))保证 agent 静默期也能定期醒来自查断连;三是 END_SENTINEL 优先于断连判断([services.py:508-513](../backend/app/gateway/services.py#L508-L513)),run 刚好在断连瞬间完成时仍返回真实终态。

断连场景下(`completed=False`)`/wait` 返回的是 `{"status": record.status.value, "error": record.error}`([thread_runs.py:202](../backend/app/gateway/routers/thread_runs.py#L202)),**不返回 checkpoint**。docstring 明确要求"Callers must skip checkpoint serialization on False so a partial checkpoint is not returned as a normal response"([services.py:499-503](../backend/app/gateway/services.py#L499-L503))——此刻 checkpoint 很可能是半成品,序列化出去就是把歧义态伪装成结果。其实返回体客户端大概率也收不到(已断连),这个分支的真正价值是 finally 里按 `on_disconnect=cancel` 取消后台 run([services.py:518-521](../backend/app/gateway/services.py#L518-L521)),不让孤儿任务白烧 token。

## 问题链 5:checkpointer 与线程恢复

**Q5.1(基础)** checkpointer 支持哪几种后端?启动时怎么装配,生命周期谁管?

**参考回答**:三种:memory(InMemorySaver)、sqlite(AsyncSqliteSaver)、postgres(AsyncPostgresSaver),工厂是 `make_checkpointer`([async_provider.py:167-202](../backend/packages/harness/deerflow/runtime/checkpointer/async_provider.py#L167-L202)),优先级:legacy `checkpointer:` 配置段 > 统一 `database:` 段 > 默认 InMemory。它是个 async context manager,在 lifespan 里由 `langgraph_runtime` 的 `AsyncExitStack` 接管([deps.py:174-236](../backend/app/gateway/deps.py#L174-L236)),退出时统一关闭。postgres 连接池带 TCP keepalive(`keepalives_idle=60`、`keepalives_interval=10`、`keepalives_count=6`)和 `check_connection`([async_provider.py:50-67](../backend/packages/harness/deerflow/runtime/checkpointer/async_provider.py#L50-L67))。注意装配顺序:`init_engine_from_config` 先于 checkpointer([deps.py:179-183](../backend/app/gateway/deps.py#L179-L183)),因为 postgres 的 auto-create-database 逻辑要先跑。

**链路解析**:

```
lifespan(app)                                    [app.py:162-245]
  ▼
langgraph_runtime(app, startup_config)           [deps.py:144-236]
  │ AsyncExitStack(逆序拆解)
  ├─ 1. make_stream_bridge(config)  → app.state.stream_bridge (memory, maxsize=256)
  ├─ 2. init_engine_from_config     → SQL session factory(先于 checkpointer!)
  ├─ 3. make_checkpointer(config)   → app.state.checkpointer
  │       ├─ memory   → InMemorySaver
  │       ├─ sqlite   → AsyncSqliteSaver.from_conn_string + setup()
  │       └─ postgres → AsyncConnectionPool(keepalives_idle=60) + AsyncPostgresSaver
  ├─ 4. make_store(config)          → app.state.store (LangGraph store)
  ├─ 5. RunRepository / MemoryRunStore → app.state.run_store
  ├─ 6. make_run_event_store        → app.state.run_event_store
  └─ 7. RunManager(store=run_store) → app.state.run_manager
        └─ sqlite 时: reconcile_orphaned_inflight_runs() 一次性恢复
  ▼ yield(服务期)
  finally: _drain_inflight_runs(5s) → close_engine() → ExitStack 逆序关闭
```

这些单例全部通过 `app.state` 暴露,路由侧用 `_require` 工厂生成的依赖函数逐个取([deps.py:244-262](../backend/app/gateway/deps.py#L244-L262)):取不到就 503,语义是"网关没有可用配置/组件就不能服务"。注意 `AppConfig` **故意不缓存**在 `app.state` 上——`get_config` 每次请求走 `get_app_config()` 的 mtime 热重载([deps.py:108-141](../backend/app/gateway/deps.py#L108-L141)),而引擎类单例绑定启动快照,这是"配置热重载边界":改 `config.yaml` 里的模型参数下一个请求就生效,改数据库后端则必须重启。

与 checkpointer 相关的另一个考点是 `POST /api/threads` 为什么创建线程时顺手写一个空 checkpoint:为了让 state 类端点(`GET /state`、history)在没有任何 run 的情况下也能立即工作——否则 `aget_tuple` 返回 None,`GET /state` 只能 404([threads.py:462-463](../backend/app/gateway/routers/threads.py#L462-L463))。创建流程先写 thread_meta(让它立刻出现在 `/threads/search`),再 `aput(config, empty_checkpoint(), metadata, {})`,metadata 带 `step=-1, source="input"`([threads.py:295-309](../backend/app/gateway/routers/threads.py#L295-L309));幂等性上 `thread_id` 已存在则直接返回已有记录不重复写([threads.py:267-281](../backend/app/gateway/routers/threads.py#L267-L281))。

**Q5.2(深挖)** Gateway 进程重启后,数据库里还躺着 status=running 的 run 记录,UI 上会一直显示"运行中",这个怎么处理?

**参考回答**:sqlite 后端启动时跑一次性恢复:`langgraph_runtime` 里调 `reconcile_orphaned_inflight_runs(error="Gateway restarted before this run reached a durable final state.")`([deps.py:214-223](../backend/app/gateway/deps.py#L214-L223))。实现([manager.py:631-683](../backend/packages/harness/deerflow/runtime/runs/manager.py#L631-L683)):`store.list_inflight()` 捞出所有持久化的 pending/running 行,逐条检查内存里有没有活着的同名任务(防御:正常关闭路径不会留下活跃行,但万一有就跳过),没有的标成 `error` 并持久化。之后 `_mark_latest_recovered_threads_error` 把"最新 run 恰好是被恢复的那个"的 thread 也标 error([deps.py:84-105](../backend/app/gateway/deps.py#L84-L105))。设计依据写在 docstring:run 的 asyncio task 和 abort_event 是进程本地的,行是持久的,重启后持久化的 active 行必然没有本地执行体,把歧义态变成显式 error 而不是让 UI 无限转圈。

**链路解析**:

```
sqlite 后端启动 (langgraph_runtime, deps.py:214-223)
  ▼
reconcile_orphaned_inflight_runs(before=now_iso())
  │ store.list_inflight() → 持久化的 pending/running 行
  ▼ 逐行:
  ├─ 内存 _runs 有活跃同名任务? → 跳过(正常关闭路径的防御)
  ├─ 否则: status=error, error="Gateway restarted before..." → _persist_status
  │     └─ 持久化失败 → 跳过该行(不把内存态和磁盘态搞分裂)
  ▼ 返回 recovered 列表
_mark_latest_recovered_threads_error(...)
  │ 按 thread 分组,只当"该 thread 最新 run"在被恢复集合里
  ▼
thread_store.update_status(thread_id, "error")
```

注意两个克制之处:一是只有**最新 run** 被恢复的 thread 才标 error——历史里有失败 run 但后来又跑成功的 thread 不被误伤([deps.py:94-101](../backend/app/gateway/deps.py#L94-L101));二是这个恢复只在 sqlite 后端跑([deps.py:214](../backend/app/gateway/deps.py#L214)),因为 memory 后端重启后本来就什么都没有,postgres 多实例场景"别的 worker 可能还活着"的假设不成立,不能把所有 inflight 行一刀切。

**Q5.3(边界/异常)** `POST /threads/{id}/state`(HITL 改状态)写新 checkpoint 时,为什么用 `uuid6()` 而不是 `uuid4()`?为什么不直接复用读出来的 config 做 `aput`?

**参考回答**:[threads.py:549-569](../backend/app/gateway/routers/threads.py#L549-L569) 的注释把两条都回答了。用 uuid6(时间有序)而不是 uuid4(随机),因为 LangGraph 的 checkpointer 用 `max(checkpoint_id)` 字符串序判断"最新",新 id 必须在字典序上大于旧 id,uuid6 的 epoch 前缀天然满足。config 里故意**不带**旧 checkpoint_id,让 write 以新 checkpoint payload 为键——如果带旧 id,`aput` 会对同一行做原地 REPLACE。**不这样设计会怎样**:原地覆盖会把 checkpoint 历史悄悄抹掉,`/threads/{id}/history` 只能看到一条记录,rollback、时间旅行调试、HITL 分支全部失效。该行为有回归测试锁定(`test_update_thread_state_inserts_new_checkpoint_each_call`,见 [threads.py:560-563](../backend/app/gateway/routers/threads.py#L560-L563) 注释)。

**Q5.4(深挖)** Human-in-the-loop 的"审批后继续"是怎么实现的?前端拿到 interrupt 之后怎么让 run 从断点继续?

**参考回答**:两条 API 配合。读侧:`GET /threads/{id}/state` 从 checkpointer 读 `tasks`(pending interrupts)和 `next` 返回([threads.py:445-494](../backend/app/gateway/routers/threads.py#L445-L494));`__interrupt__` 键在序列化时被故意保留(只剥 `__pregel_*`,[serialization.py:59-71](../backend/packages/harness/deerflow/runtime/serialization.py#L59-L71)),好让 SDK 从 values chunk 里识别 interrupt。写侧:前端再发一次 `POST /runs/stream`,body 里带 `command: {"resume": <审批值>}`;`start_run` 检测到 `command.resume is not None` 就把 graph_input 换成 `Command(resume=command["resume"])` 而不是普通 input([services.py:399-403](../backend/app/gateway/services.py#L399-L403))。LangGraph 拿到 `Command(resume=...)` 会从该 thread 最新 checkpoint 的 interrupt 点继续执行——所以恢复语义完全由 checkpointer 的持久状态支撑,run 本身是一个全新的 RunRecord。`interrupt_before`/`interrupt_after` 则是在 `run_agent` 里挂到 `agent.interrupt_before_nodes` 上([worker.py:277-280](../backend/packages/harness/deerflow/runtime/runs/worker.py#L277-L280))。

另一个相关路径是 `POST /threads/{id}/state` 直接改状态([threads.py:497-596](../backend/app/gateway/routers/threads.py#L497-L596)):读取最新 checkpoint → 在内存副本上 merge `body.values` → 换新 uuid6 id 写回(见 Q5.3 的 uuid6 分析)。带 `as_node` 时 metadata 里 `source="update"`、`step+1`、`writes={as_node: values}`,语义是"伪装成某个节点产出了这些写入",让图认为该节点已执行([threads.py:544-547](../backend/app/gateway/routers/threads.py#L544-L547))。如果 values 里带 `title`,还会同步到 `threads_meta.display_name` 让 `/threads/search` 立刻可见([threads.py:582-588](../backend/app/gateway/routers/threads.py#L582-L588));run 结束时 worker 的 finally 也会从 checkpoint 反向同步一次 title([worker.py:415-426](../backend/packages/harness/deerflow/runtime/runs/worker.py#L415-L426))——两个方向都通了,列表页标题才不会滞后。

## 问题链 6:auth / CSRF / internal auth 三层防线

**Q6.1(基础)** 一个请求进到 Gateway,认证这块有几层?每层防什么?

**参考回答**:三层。第一层 `AuthMiddleware`([auth_middleware.py:59-143](../backend/app/gateway/auth_middleware.py#L59-L143)):fail-closed 全局闸口,非 public 路径必须有合法 session cookie(或内部 token),JWT 严格校验失败直接 401,顺带把 user 盖章到 `request.state.user` 和 user_context contextvar。第二层 `CSRFMiddleware`:对 POST/PUT/DELETE/PATCH 做 Double Submit Cookie 校验。第三层在路由上:`@require_permission(..., owner_check=True)` 做资源级归属授权,防"用户 A 猜 URL 读用户 B 的 thread"。public 路径只有 `/health`、`/docs`、`/openapi.json` 前缀和 5 个 auth 精确路径([auth_middleware.py:32-49](../backend/app/gateway/auth_middleware.py#L32-L49))。

补充一个容易被追问的细节:AuthMiddleware 校验通过后会调 `set_current_user(user)` 写 contextvar,并在 `finally` 里 `reset_current_user(token)` 恢复([auth_middleware.py:139-143](../backend/app/gateway/auth_middleware.py#L139-L143))。这个 contextvar 是仓储层 owner 过滤的自动数据源:`resolve_user_id` 有三态语义——`AUTO` 哨兵(默认,读 contextvar,没有就 raise)、显式字符串(覆盖,测试/管理用)、显式 `None`(不加 WHERE,仅限迁移脚本)([user_context.py:166-195](../backend/packages/harness/deerflow/runtime/user_context.py#L166-L195))。因为 ContextVar 在 asyncio 里是 task-local 的,每个请求一个 task,天然隔离;`asyncio.create_task` 会继承父 task 上下文,所以 `run_agent` 后台任务也能看到发起请求的 user。工具层还有 `resolve_runtime_user_id` 三级回退:`runtime.context["user_id"]` > contextvar > `"default"`([user_context.py:112-137](../backend/packages/harness/deerflow/runtime/user_context.py#L112-L137))——第一级就是为"后台任务可能丢 contextvar"准备的(对应 `inject_authenticated_user_context` 把 user_id 盖进 config["context"],[services.py:169-187](../backend/app/gateway/services.py#L169-L187))。

**链路解析**:

```
请求(执行顺序: Starlette 后注册的中间件在最外层)
 ├─ path ∈ public? ──是──► 直通路由(/health、/docs、login 等 5 个精确路径)
 ▼ 否
[1] CORSMiddleware(仅当 GATEWAY_CORS_ORIGINS 非空才挂载)
    preflight OPTIONS 在此直接应答 —— 必须在最外层,
    否则无凭据的 preflight 会被下游 Auth 401 掉
 ▼
[2] CSRFMiddleware: 状态变更方法(POST/PUT/DELETE/PATCH)?
    auth 端点 → 只查 Origin(同源或 GATEWAY_CORS_ORIGINS 显式允许)
    其他端点 → cookie csrf_token vs header X-CSRF-Token, compare_digest
    不符 → 403
 ▼
[3] AuthMiddleware: internal token? → cookie JWT 严格校验?
    否 → 401 (NOT_AUTHENTICATED / TOKEN_INVALID / USER_NOT_FOUND)
    是 → request.state.user + set_current_user(contextvar)
 ▼
[4] 路由 @require_permission(resource, action, owner_check=True)
    thread_store.check_access(thread_id, user.id) → 404(防枚举)
```

追问点:为什么 CSRF 在 Auth **外**层?因为 CSRF 防的是"浏览器在已登录状态下被诱导发请求",校验材料(cookie 里的 csrf_token 和 header)本身不依赖认证结果,放外层可以尽早 403 掉伪造请求、省一次 JWT 解码和 DB 查询;而 Auth 放内层的好处是 401 响应能带上细分错误码(token_expired / user_not_found 等),且 `request.state.user` 盖章后路由层直接用。三层防线里 CORS 与 CSRF 共享 `get_configured_cors_origins()` 数据源([app.py:347-358](../backend/app/gateway/app.py#L347-L358)),Auth 与 internal auth 共享 token 校验([auth_middleware.py:86-88](../backend/app/gateway/auth_middleware.py#L86-L88))。

**Q6.2(深挖)** CSRF 用的是 Double Submit Cookie。登录接口本身没有 CSRF token 可用,它怎么防 login CSRF?

**参考回答**:登录/注册/initialize 在 `_AUTH_EXEMPT_PATHS` 里豁免 token 校验([csrf_middleware.py:53-60](../backend/app/gateway/csrf_middleware.py#L53-L60)),但豁免不等于裸奔:`is_allowed_auth_origin`([csrf_middleware.py:158-176](../backend/app/gateway/csrf_middleware.py#L158-L176))要求带 Origin 的浏览器请求的 Origin 必须等于请求自身的 origin(经 Forwarded/X-Forwarded-* 还原,[csrf_middleware.py:140-155](../backend/app/gateway/csrf_middleware.py#L140-L155))或落在 `GATEWAY_CORS_ORIGINS` 显式白名单里,否则 403 "Cross-site auth request denied";无 Origin 的非浏览器客户端(curl)放行。这是防 login CSRF / session fixation。token 本身 `secrets.token_urlsafe(64)` 生成([csrf_middleware.py:21-31](../backend/app/gateway/csrf_middleware.py#L21-L31)),比对用 `secrets.compare_digest` 防时序侧信道([csrf_middleware.py:204-208](../backend/app/gateway/csrf_middleware.py#L204-L208)),登录成功后 Set-Cookie 下发,`httponly=False`(JS 必须能读出来放进 header,这是 Double Submit 的本质)、`samesite=strict`([csrf_middleware.py:213-223](../backend/app/gateway/csrf_middleware.py#L213-L223))。

Origin 规范化本身也值得追问:`_normalize_origin`([csrf_middleware.py:82-98](../backend/app/gateway/csrf_middleware.py#L82-L98))只接受 `http/https` scheme,拒绝带用户名密码、path、query、fragment 的"URL 形"输入——因为浏览器 Origin 头严格只有 scheme://host:port,接受更宽的格式就是给绕过留口子。默认端口(80/443)会被剥掉再比较([csrf_middleware.py:71-79](../backend/app/gateway/csrf_middleware.py#L71-L79)),避免 `https://a.com` 和 `https://a.com:443` 被当成两个 origin。请求侧 origin 还原依次读 RFC 7239 `Forwarded`、`X-Forwarded-Proto/Host/Port`、最后才是原始 Host([csrf_middleware.py:140-155](../backend/app/gateway/csrf_middleware.py#L140-L155))——这层还原的信任前提是入口 nginx 会覆盖这些头,这也是部署文档要求所有流量经统一 nginx 入口的原因之一。CORS 白名单和 CSRF origin 检查共享同一个 `get_configured_cors_origins()` 数据源([app.py:347-358](../backend/app/gateway/app.py#L347-L358)),避免两处配置漂移出不一致的安全判断。

**Q6.3(边界/异常)** 内部调用(比如 IM channel worker 回调 Gateway)走的是什么认证?`X-DeerFlow-Owner-User-Id` 这个头如果我作为普通浏览器用户伪造一个,能冒充别人吗?

**参考回答**:不能。内部认证用 `X-DeerFlow-Internal-Token` 头,token 来自环境变量 `DEER_FLOW_INTERNAL_AUTH_TOKEN`,未设置时进程启动随机生成 32 字节([internal_auth.py:18-25](../backend/app/gateway/internal_auth.py#L18-L25)),比对用 `compare_digest`([internal_auth.py:36-38](../backend/app/gateway/internal_auth.py#L36-L38))。AuthMiddleware 校验通过后盖章一个合成用户 `SimpleNamespace(id="default", system_role="internal")`([internal_auth.py:41-43](../backend/app/gateway/internal_auth.py#L41-L43))。关键在 `get_trusted_internal_owner_user_id`([internal_auth.py:46-61](../backend/app/gateway/internal_auth.py#L46-L61)):它先检查 `request.state.user.system_role == "internal"`,不满足直接返回 None——owner 头只在内部 token 认证通过后才被尊重,浏览器伪造的 owner 头被静默忽略。而且 `inject_authenticated_user_context` 对 internal 角色直接 return、不往 run context 里盖 user_id([services.py:182-187](../backend/app/gateway/services.py#L182-L187)),服务端认证的身份永远优先于客户端 context(`setdefault`,[services.py:143-166](../backend/app/gateway/services.py#L143-L166))。配套的线程访问上,内部调用也按 owner 头做 `check_access`,注释明说"泄露的内部 token 不能授予跨用户线程访问"([services.py:334-356](../backend/app/gateway/services.py#L334-L356))。

**Q6.4(边界/异常)** thread 的 metadata 是客户端传的 dict,我在里面塞一个 `{"user_id": "victim"}` 能把线程挂到别人名下吗?

**参考回答**:不能,有两层防。第一层在 API 边界:`_SERVER_RESERVED_METADATA_KEYS = frozenset({"owner_id", "user_id"})`([threads.py:42](../backend/app/gateway/routers/threads.py#L42)),`ThreadCreateRequest`/`ThreadPatchRequest` 的 `@field_validator("metadata")` 会把这两个键剥掉([threads.py:45-49](../backend/app/gateway/routers/threads.py#L45-L49),[threads.py:83](../backend/app/gateway/routers/threads.py#L83))——恶意客户端不能通过 metadata blob 反射伪造 owner 身份。第二层是行级不变量:`threads_meta.user_id` 只从 auth contextvar 填,从不读客户端输入。注释把第一层称为 "defense-in-depth",因为真正的不变量在第二层。这个模式和 `inject_authenticated_user_context` 里"服务端身份永远覆盖客户端 context"是同一个安全原则:身份只信服务端认证结果。

## 问题链 7:为什么 GATEWAY_WORKERS 默认 1,以及优雅关闭

**Q7.1(基础)** docker-compose 里 gateway 默认只起 1 个 uvicorn worker,是性能上不想扩还是不能扩?

**参考回答**:是不能(暂时)。compose 注释写明:RunManager 和 StreamBridge 是 in-process singleton,run 状态活在 worker 内存里;worker >1 且没有 nginx sticky session 时,run cancel、SSE 重连、请求去重、per-worker IM channel 服务全部跨 worker 失效,要等共享的(如 redis)stream bridge 落地——而 `make_stream_bridge` 里 redis 分支还是 `NotImplementedError`([async_provider.py:52-53](../backend/packages/harness/deerflow/runtime/stream_bridge/async_provider.py#L52-L53))。默认 `${GATEWAY_WORKERS:-1}` 有测试锁定([test_compose_default_workers.py:35-45](../backend/tests/test_compose_default_workers.py#L35-L45))。**不这样设计会怎样**:客户端在 worker A 上创建的 run,SSE 重连被负载均衡到 worker B,B 的内存里查无此 run(只能水合出 `store_only` 记录),join/cancel 全部 409(见问题链 3.3);AB 两个 worker 还可能对同一 thread 各自起 run,因为 `create_or_reject` 的锁是进程内的 asyncio.Lock,跨进程不互斥。

"状态在进程内"在代码里具体是这些东西:`RunManager._runs`(run_id → RunRecord)加二级索引 `_runs_by_thread`(thread_id → 有序 run_id 集合),两者在同一把 `asyncio.Lock` 里 lockstep 维护,按 thread 查询是 O(该 thread 的 run 数) 而不是全表扫([manager.py:123-161](../backend/packages/harness/deerflow/runtime/runs/manager.py#L123-L161));`MemoryStreamBridge._streams`(run_id → 事件日志 + `asyncio.Condition`)([memory.py:32-43](../backend/packages/harness/deerflow/runtime/stream_bridge/memory.py#L32-L43));以及 RunRecord 上的 `task: asyncio.Task` 和 `abort_event: asyncio.Event`([manager.py:89-91](../backend/packages/harness/deerflow/runtime/runs/manager.py#L89-L91))——asyncio 原语本质上无法跨进程共享,这是"必须单 worker"的最硬约束,比"状态同步麻烦"更底层。`asyncio.Condition` 只能在创建它的那个 event loop 上使用,多 worker 各自有独立 loop,subscribe 和 publish 跨进程根本无法唤醒对方。

**链路解析**:

```
单 worker 现状:
  uvicorn --workers 1
    └─ app.state: { stream_bridge(Memory), run_manager(内存 _runs + asyncio.Lock),
                    checkpointer, store, run_store(SQL) }
       └─ 内存态(task/abort_event)与持久态(run row)同进程 → cancel/join 自洽

多 worker 反例:
  client ──► nginx LB ──► worker A: create run (task 在 A 内存)
                     └─► worker B: cancel/join → get() 水合 store_only 记录
                                   → 409 "not active on this worker"
  解锁条件: Redis StreamBridge(跨进程事件总线) + 跨进程 run 锁/归属路由
```

**Q7.2(深挖)** 关闭时为什么要先 drain 在飞的 run 再关 checkpointer?顺序反了会怎样?如果 drain 自己又被第二次 SIGINT 取消呢?

**参考回答**:反了就是 issue #3373 的 `psycopg_pool.PoolClosed`。`langgraph_runtime` 的 finally 先 `_drain_inflight_runs(run_manager)` 再 `close_engine()`、再由 AsyncExitStack 拆 checkpointer 连接池([deps.py:227-236](../backend/app/gateway/deps.py#L227-L236))。原因写在 `RunManager.shutdown` 的 docstring([manager.py:700-726](../backend/packages/harness/deerflow/runtime/runs/manager.py#L700-L726)):run 任务在后台 asyncio task 里跑,langgraph 内部的 `_checkpointer_put_after_previous` 会在 `finally` 里向 checkpointer 写 checkpoint;如果连接池先关,这个内部 task 的 `aput` 打到已关闭的池上,而它不在 `run_agent` 的调用栈上,worker 捕不到,变成 asyncio.run 关闭期的未处理异常。drain 的预算是 `_RUN_DRAIN_TIMEOUT_SECONDS = 5.0` 秒([deps.py:45](../backend/app/gateway/deps.py#L45)):先 cancel 所有 inflight 任务,`asyncio.wait(tasks, timeout=5)`,超时未安定的才标 interrupted,安定的不覆盖其真实终态([manager.py:727-790](../backend/packages/harness/deerflow/runtime/runs/manager.py#L727-L790)),连最后的状态持久化也用剩余预算 `wait_for` 封顶。

drain 自身被信号打断的情况也防了:`_drain_inflight_runs` 用双重 `asyncio.shield`([deps.py:48-71](../backend/app/gateway/deps.py#L48-L71))——drain 包成独立 task 后 shield;若 lifespan 协程自己收到 CancelledError(第二次 SIGINT 或服务器 graceful-shutdown 超时,即 #3373 的"信号风暴"),catch 后**再 shield 一次**等这个有界 drain 收尾,然后 re-raise 传播取消;因为 drain 有 5 秒上限,这样等不会挂死。channel 服务停止也有独立的 5 秒钩子上限 `_SHUTDOWN_HOOK_TIMEOUT_SECONDS = 5.0`([app.py:47-50](../backend/app/gateway/app.py#L47-L50),[app.py:229-243](../backend/app/gateway/app.py#L229-L243)),两个预算管独立的拆解步骤。启动期同理:tiktoken 预热用 `asyncio.wait_for(..., timeout=5)` 封顶,失败降级为字符计数,不阻塞启动([app.py:191-208](../backend/app/gateway/app.py#L191-L208))。设计原则一句话:任何可能触碰外部资源或等待任务收尾的启动/关闭步骤都必须有超时和降级路径,否则 uvicorn 的 reload supervisor 会反复向卡死的 worker 发信号。

## 问题链 8:RunJournal、token 统计与事件持久化

**Q8.1(基础)** 一个 run 的 token 用量、消息数这些统计是怎么采集的?为什么不从 SSE 流里解析,而是单独搞一套?

**参考回答**:走 LangChain 回调机制。`run_agent` 启动时创建 `RunJournal`(一个 `BaseCallbackHandler`),塞进 `config["callbacks"]`([worker.py:234-235](../backend/packages/harness/deerflow/runtime/runs/worker.py#L234-L235)),同时通过 `runtime_ctx["__run_journal"]` 哨兵键暴露给 middleware 写审计事件([worker.py:222-227](../backend/packages/harness/deerflow/runtime/runs/worker.py#L222-L227))。`on_llm_end` 从 `message.usage_metadata` 累加 input/output/total tokens,按 langchain run_id 去重防止回调重复触发导致 double-count([journal.py:311-340](../backend/packages/harness/deerflow/runtime/journal.py#L311-L340));按 caller tag 分桶(lead_agent / subagent:* / middleware:*)([journal.py:326-331](../backend/packages/harness/deerflow/runtime/journal.py#L326-L331))。从 SSE 流解析是不可靠的:流是给客户端的视图,可能被截断、被跳过,而 callback 挂在执行引擎上,是事实源。run 结束时 worker 的 finally 里 `journal.get_completion_data()` 落到 RunStore([worker.py:408-413](../backend/packages/harness/deerflow/runtime/runs/worker.py#L408-L413))。

几个值得展开的设计决策:一是 `on_llm_new_token` **刻意不实现**,只在 `on_llm_end` 统计完整消息([journal.py:8-9](../backend/packages/harness/deerflow/runtime/journal.py#L8-L9))——逐 token 回调频率太高,统计价值和写入成本不成比例。二是首条 human 消息从 `on_chat_model_start` 提取而不是 `on_chain_start`,因为前者拿到的是完全结构化的 messages、且只在真实 LLM 调用时触发([journal.py:196-210](../backend/packages/harness/deerflow/runtime/journal.py#L196-L210))。三是 caller 识别靠 tag 注入,主 agent 图不打 tag,`_identify_caller` 默认归到 `lead_agent`([journal.py:438-446](../backend/packages/harness/deerflow/runtime/journal.py#L438-L446));`last_ai_message` 只接受 lead_agent 的文本,subagent/middleware 的模型调用和纯 tool-call 的空 AI 消息不会覆盖用户可见答案([journal.py:135-146](../backend/packages/harness/deerflow/runtime/journal.py#L135-L146))。事件落库走 `RunEventStore` 接口,`memory/db/jsonl` 三实现由 `make_run_event_store` 按配置选择([__init__.py:5-23](../backend/packages/harness/deerflow/runtime/events/store/__init__.py#L5-L23))。

**链路解析**:

```
LangGraph 执行引擎
  │ (LangChain callback 机制)
  ▼
RunJournal(BaseCallbackHandler)              [journal.py:38-100]
  ├─ on_chat_model_start → llm.human.input 事件 + 首条 human 消息
  ├─ on_llm_end          → llm.ai.response 事件 + token 累加(按 run_id 去重)
  ├─ on_tool_end         → llm.tool.result 事件
  └─ _put(...) → _buffer(list)
       │ len(buffer) >= flush_threshold=20      [journal.py:48]
       ▼
     _flush_sync → loop.create_task(_flush_async) → event_store.put_batch
       │ 失败 → batch 放回 buffer 首部,下次重试  [journal.py:417-428]
       ▼
RunEventStore(memory / db / jsonl 三实现,make_run_event_store 选择)
```

**Q8.2(深挖)** callback 是同步函数,写库是异步的,这个矛盾怎么解决?并发 flush 会不会把 SQLite 打爆?运行途中的 token 进度也走这条路吗?

**参考回答**:`_flush_sync`([journal.py:392-415](../backend/packages/harness/deerflow/runtime/journal.py#L392-L415))是桥接点:先查 `_pending_flush_tasks` 非空就直接跳过——同一时刻只允许一个 flush 任务在飞,注释明说就是避免多个 fire-and-forget 任务并发写同一个 SQLite 文件;然后 `asyncio.get_running_loop()` 拿不到 loop(纯同步上下文)时把事件留在 buffer 里等 worker finally 的异步 `flush()` 兜底;拿到 loop 就 `loop.create_task(self._flush_async(batch))` 并注册 done 回调。`_flush_async` 失败会把 batch 放回 buffer 头部等下次重试([journal.py:417-428](../backend/packages/harness/deerflow/runtime/journal.py#L417-428))。缓冲阈值 `flush_threshold=20` 条事件([journal.py:48](../backend/packages/harness/deerflow/runtime/journal.py#L48))。最终的 `flush()`([journal.py:550-570](../backend/packages/harness/deerflow/runtime/journal.py#L550-L570))在 worker finally 里先 gather 所有在飞任务,再按阈值分批清空剩余 buffer。

运行中的 token 进度(前端实时显示)走**独立通道**且严格节流:journal 每次累加 token 后 `_schedule_progress_flush`,间隔 `progress_flush_interval=5.0` 秒([journal.py:50](../backend/packages/harness/deerflow/runtime/journal.py#L50),[journal.py:572-621](../backend/packages/harness/deerflow/runtime/journal.py#L572-621)),经 `progress_reporter` 回调到 `RunManager.update_run_progress`([worker.py:179](../backend/packages/harness/deerflow/runtime/runs/worker.py#L179))。关键保护在 [manager.py:339-355](../backend/packages/harness/deerflow/runtime/runs/manager.py#L339-L355):只有 `record.status == RunStatus.running` 时才更新内存并持久化——run 已到终态后迟到的进度快照会被丢弃,防止把终态数据回写成进行中的样子。completion 写入(`update_run_completion`)则不同:它带"行恢复"逻辑,如果 `update_run_completion` 返回 False(行不存在),先用内存快照 `put` 重建行再重试([manager.py:302-337](../backend/packages/harness/deerflow/runtime/runs/manager.py#L302-337))。另外所有短存储写都套了 `PersistenceRetryPolicy`:`max_attempts=5`、`initial_delay=0.05s`、`max_delay=1.0s`、指数退避,只对 "database is locked" 类瞬时 SQLite 错误重试([manager.py:64-71](../backend/packages/harness/deerflow/runtime/runs/manager.py#L64-L71),[manager.py:180-208](../backend/packages/harness/deerflow/runtime/runs/manager.py#L180-208))。

## 面试官最爱追问的 3 个点

1. **"断连了到底会发生什么?"** —— 应答策略:分清三条路径:SSE 流(`sse_consumer` finally + `on_disconnect`,默认 cancel)、/wait(共享 bridge 断连语义,#3265 的反例先讲)、进程重启(孤儿行由 `reconcile_orphaned_inflight_runs` 收敛为 error)。核心词:heartbeat 15s 自查、`store_only` 409、显式终态优于歧义态。
2. **"为什么不能横向扩 worker?"** —— 应答策略:一句话"状态在进程内":RunManager 的 asyncio.Lock、StreamBridge 的内存事件日志、record.task/abort_event 都是 in-process;解锁路径是 Redis StreamBridge(代码里已留 `NotImplementedError` 接缝)加跨进程 run 归属。顺手给出默认 1 有 compose 注释和测试双锁定。
3. **"cancel 的 interrupt 和 rollback 在 checkpoint 层面差在哪?"** —— 应答策略:interrupt 只是停执行、保留最新 checkpoint(可 resume);rollback 是应用层实现的"时间旅行"——启动前深拷贝快照,回滚时换 uuid6 新 id 重新 `aput` 并恢复 pending_writes,利用 LangGraph "latest = max(id)" 规则让旧状态重新生效。提到 pre-run 无 checkpoint 时退化为 `adelete_thread` 这个边界,基本就答满了。

这三个点的共同主线其实是一句话:**这套运行时把"进程内易失状态"和"持久化状态"分得很清**——易失的(task、abort_event、事件 buffer)绝不假装能跨进程/跨重启恢复,持久的(run 行、checkpoint、事件)保证重启后可水合、可 reconcile;所有 409、409-store_only、error 终态都是这条主线上的诚实表达。面试时把这条主线点出来,再按上面三条展开细节,比零散背 API 更能体现系统理解。
