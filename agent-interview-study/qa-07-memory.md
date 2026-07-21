# 长期记忆系统 —— 面试问题链

> 本文档按"大厂面试追问链"组织:每条问题链从基础问题开始,层层追问到实现细节、设计权衡、异常边界。配套深读笔记:[08-long-term-memory.md](08-long-term-memory.md)(深读笔记讲"怎么实现",本文档讲"怎么被问、怎么答")。
> 代码引用格式:[文件名.py:起始行-结束行](../backend/仓库相对路径#L起始行-L结束行)。只引用你实际读过的行,禁止编造行号。

## 全局链路总览

```
                        ┌──────────────────────────────────────────────────┐
                        │              写入路径(后台, best-effort)          │
                        └──────────────────────────────────────────────────┘
agent 执行完成 / summarization 即将裁剪消息
   │                                    │
   │ MemoryMiddleware.after_agent()     │ memory_flush_hook(SummarizationEvent)
   │ (防抖 add, 30s)                    │ (立即 add_nowait, 不等防抖)
   ▼                                    ▼
filter_messages_for_memory() ── 剥 <uploaded_files>、丢 tool_calls 消息
   ▼
detect_correction / detect_reinforcement ── 最近 6 条 human 消息正则匹配
   ▼
MemoryUpdateQueue ── (thread_id, user_id, agent_name) 去重合并, threading.Timer 防抖
   ▼ (Timer 线程触发; user_id 已在入队时物化, 不依赖 ContextVar)
MemoryUpdater.update_memory()
   ├─ 有 running loop → _SYNC_MEMORY_UPDATER_EXECUTOR(max_workers=4) 里跑
   └─ _do_update_memory_sync: 同步 model.invoke()(不碰全局 async httpx 池)
   ▼
_parse_memory_update_response → _normalize_memory_update_data(fail-closed)
   ▼
_apply_updates: 阈值 0.7 → casefold 去重 → max_facts=100 截断
   ▼
_strip_upload_mentions_from_memory → deepcopy 后 mutation
   ▼
FileMemoryStorage.save: {uuid}.tmp → 原子 replace → mtime 缓存刷新

                        ┌──────────────────────────────────────────────────┐
                        │              读取路径(每次组 prompt, fail-open)    │
                        └──────────────────────────────────────────────────┘
lead_agent 组 system prompt
   ▼
get_memory_data(user_id=get_effective_user_id())  # mtime 校验缓存
   ▼
format_memory_for_injection(max_tokens=2000)
   ├─ tiktoken 计数(600s 失败冷却 + char/CJK 估算兜底)
   └─ Facts 按 confidence 降序试装, 超预算 break, 兜底 95% 截断
   ▼
<memory>...</memory> 注入; 任何异常 → 返回 "" 不影响主对话
```

## 关键配置速览(均见 memory_config.py)

| 配置项 | 默认值 | 边界 | 作用 |
|---|---|---|---|
| `enabled` | `True` | — | 记忆机制总开关 |
| `debounce_seconds` | `30` | 1-300 | 防抖窗口秒数 |
| `max_facts` | `100` | 10-500 | fact 容量上限 |
| `fact_confidence_threshold` | `0.7` | 0.0-1.0 | fact 入库置信度阈值 |
| `injection_enabled` | `True` | — | 注入开关(独立于 enabled) |
| `max_injection_tokens` | `2000` | 100-8000 | 注入 token 预算 |
| `token_counting` | `"tiktoken"` | tiktoken/char | token 计数策略 |
| `storage_class` | `FileMemoryStorage` | 类路径 | 可插拔存储后端 |

代码引用:[memory_config.py:8-75](../backend/packages/harness/deerflow/config/memory_config.py#L8-L75)。

## 问题链索引

| 链 | 主题 | 层数 | 核心记忆点 |
|---|---|---|---|
| 1 | 防抖队列与并发控制 | 4 | 30s debounce、三元组去重、`_processing` 让位重试 |
| 2 | ContextVar 跨线程丢失与 user_id 捕获 | 3 | 入队时物化 user_id、三级解析、default 兜底 |
| 3 | LLM 更新流水线与同步调用路径 | 3 | sync invoke 避开全局 async httpx 池(issue #2615) |
| 4 | 响应解析、归一化与 fail-closed | 3 | raw_decode 扫描、脏 confidence 清洗、部分更新作废 |
| 5 | 纠错/强化信号检测与消息过滤 | 2 | 最近 6 条 human 正则、hint 注入而非程序化直写 |
| 6 | 事实去重、置信度阈值与容量控制 | 3 | 0.7 阈值、casefold 精确去重、100 条封顶截断 |
| 7 | 持久化与注入 | 4 | temp+rename 原子写、deepcopy、2000 token 试装、tiktoken 600s 冷却 |
| 8 | per-user 隔离、上传清洗与迁移 | 3 | 路径分桶、三道上传防线、冲突不覆盖迁移 |

## 问题链 1:防抖队列与并发控制

**Q1.1(基础)** 对话结束后要更新长期记忆,但你们没有每轮对话都立刻调 LLM 写记忆,而是搞了个"防抖队列"。这个防抖具体怎么实现的?

**参考回答**:核心是 `MemoryUpdateQueue`,每次 `add()` 时把对话上下文入队,然后调用 `_reset_timer()` 把 `threading.Timer` 重置为 `config.debounce_seconds`(默认 30 秒,配置边界 ge=1/le=300);30 秒内再有新对话进来就取消旧 timer 重新计时,直到静默满 30 秒才触发 `_process_queue()` 批量处理,见 [queue.py:146-164](../backend/packages/harness/deerflow/agents/memory/queue.py#L146-L164) 和 [memory_config.py:33-38](../backend/packages/harness/deerflow/config/memory_config.py#L33-L38)。timer 设为 daemon 线程,进程退出不阻塞。所有队列操作都在 `self._lock` 内完成,timer 回调与 `add()` 的竞态由同一把锁串行化,见 [queue.py:77-86](../backend/packages/harness/deerflow/agents/memory/queue.py#L77-L86)。队列本体是进程级单例,`get_memory_queue()` 双重检查锁保证全进程共享一个实例,见 [queue.py:260-275](../backend/packages/harness/deerflow/agents/memory/queue.py#L260-L275)。

**链路解析**:
```
agent 执行完成
   │ MemoryMiddleware.after_agent()
   ▼
queue.add(thread_id, messages, user_id, ...)
   │  with self._lock:
   │      _enqueue_locked()      # 去重合并
   │      _reset_timer()         # cancel 旧 timer, 新开 30s timer
   ▼
(30s 静默期过去)
threading.Timer 线程触发 _process_queue()
   │  copy 队列 → 清空 → 逐条调 MemoryUpdater.update_memory()
   ▼
LLM 抽取 → 写 memory.json
```

**Q1.2(深挖)** 同一个 thread 在防抖窗口内连续来了 3 次对话,队列里会有 3 条记录吗?去重 key 是什么?

**参考回答**:不会,只有 1 条。`_enqueue_locked()` 用 `(thread_id, user_id, agent_name)` 三元组作为去重 key,同 key 的旧记录会被新记录整体替换(messages 用最新的),但 `correction_detected` / `reinforcement_detected` 两个信号标志用 OR 合并,保证"用户纠正过"这个信号不会因为消息被覆盖而丢失,见 [queue.py:43-50](../backend/packages/harness/deerflow/agents/memory/queue.py#L43-L50) 和 [queue.py:127-144](../backend/packages/harness/deerflow/agents/memory/queue.py#L127-L144)。注意 key 里包含 user_id 和 agent_name,所以同一 thread_id 不同用户/不同 agent 的记忆更新不会互相覆盖。另外 `ConversationContext.timestamp` 每次入队都刷新(`default_factory=lambda: datetime.now(UTC)`),即防抖窗口内保留的是最后一次对话的时间戳,可用于观测入队延迟,见 [queue.py:15-25](../backend/packages/harness/deerflow/agents/memory/queue.py#L15-L25)。

**Q1.3(边界/异常)** 如果 30 秒到了、timer 触发时上一次处理还没跑完(`_processing=True`),会发生什么?会丢消息吗?

**参考回答**:不会丢,但会"让位重试"。`_process_queue()` 开头检查 `_processing`,若已有 worker 在处理,就直接 `_schedule_timer(0)` —— 立刻再排一个 0 延迟的 timer,然后 return;等前一个 worker 在 finally 里把 `_processing` 置回 False 后,新 timer 马上接管队列,见 [queue.py:166-183](../backend/packages/harness/deerflow/agents/memory/queue.py#L166-L183)。这一支主要是为 `flush_nowait()`(summarization hook 用的立即处理路径)保留语义。另外批量处理时若队列里多于 1 条,每条之间 `time.sleep(0.5)` 防止打爆 LLM 限流,见 [queue.py:208-210](../backend/packages/harness/deerflow/agents/memory/queue.py#L208-L210)。真正的丢失风险在进程退出:timer 是 daemon 线程,`flush_nowait` 的注释也明说"queued messages may be lost if the process exits",这是 best-effort 语义下的刻意取舍,见 [queue.py:228-233](../backend/packages/harness/deerflow/agents/memory/queue.py#L228-L233)。

**Q1.4(反例)** 不这样设计会怎样?—— 为什么不用"每条消息立刻更新"或者"纯异步任务队列(如 Celery)"?

**参考回答**:两个方向都被否决过。立刻更新的问题是成本与写冲突:一轮对话可能触发多次 middleware 回调,每次都调一次 LLM + 写文件,token 成本翻倍且 `memory.json` 写放大会加剧并发写竞争;防抖把 N 次触发压缩成 1 次 LLM 调用。用 Celery 之类外部队列则是部署复杂度问题:这是一个本地文件存储的 best-effort 功能,引入 broker 违背了"单进程即可运行"的定位,`threading.Timer` + 进程内 list 已经足够;代价就是 Q1.3 说的进程退出可能丢最后 30 秒的更新,但对"长期记忆"这种下次对话还能重建的信号是可接受的。

---

## 问题链 2:ContextVar 跨线程丢失与 user_id 捕获

**Q2.1(基础)** 我注意到 `ConversationContext` 里显式存了 `user_id` 字段,而代码里明明有 `get_effective_user_id()` 可以从 ContextVar 拿。为什么要多此一举在入队时捕获?

**参考回答**:因为 `threading.Timer` 在**另一个线程**触发回调,而 ContextVar 不会跨裸线程传播——等到 30 秒后 timer 线程执行 `_process_queue()` 时,请求上下文早已销毁,`get_effective_user_id()` 只会返回兜底值 `"default"`,所有用户的记忆会全写到同一个 default 桶里串号。所以中间件在请求线程里、上下文还活着时就把 user_id 物化进 `ConversationContext`,代码注释写得很直白:"threading.Timer fires on a different thread where ContextVar values are not propagated",见 [memory_middleware.py:96-99](../backend/packages/harness/deerflow/agents/middlewares/memory_middleware.py#L96-L99) 和 queue 侧对应注释 [queue.py:66-69](../backend/packages/harness/deerflow/agents/memory/queue.py#L66-L69)。不这样做的后果是数据泄露级事故:用户 A 的职业、偏好会写进 default 桶再注入用户 B 的 prompt,且单用户环境永远测不出来。

**链路解析**:
```
请求线程 (ContextVar 有效)                Timer 线程 (ContextVar 丢失)
┌─────────────────────────────┐        ┌──────────────────────────┐
│ auth middleware:            │        │ _process_queue()         │
│   set_current_user(u123)    │        │   updater.update_memory( │
│ ...                         │  30s   │     user_id=ctx.user_id) │
│ after_agent():              │ ─────► │        ▲                 │
│   user_id = get_effective_  │        │        │ 从入队时物化的    │
│     user_id()  → "u123"     │        │        │ ConversationContext│
│   queue.add(user_id="u123") │        │        │ 取, 而非再读      │
└─────────────────────────────┘        │        │ ContextVar       │
                                       └──────────────────────────┘
```

**Q2.2(深挖)** summarization hook 那条路径拿 user_id 的方式和 middleware 不一样,用的是 `resolve_runtime_user_id(event.runtime)`,为什么?

**参考回答**:因为 hook 触发时(summarization 即将删除消息前)已经处于更不可信的上下文——可能是不 copy_context 的 worker pool 或后台任务,ContextVar 可能已经丢了。`resolve_runtime_user_id` 做三级解析:优先 `runtime.context["user_id"]`(gateway 在 `inject_authenticated_user_context` 里显式注入的、经过 auth 校验的值,唯一能跨越上下文丢失边界的通道)→ 其次 `_current_user` ContextVar → 最后兜底 `DEFAULT_USER_ID = "default"`,见 [user_context.py:112-137](../backend/packages/harness/deerflow/runtime/user_context.py#L112-L137) 和 [summarization_hook.py:25](../backend/packages/harness/deerflow/agents/memory/summarization_hook.py#L25)。middleware 路径还跑在请求任务里,用 `get_effective_user_id()`(不抛异常、unset 时回 "default")就够,见 [user_context.py:100-109](../backend/packages/harness/deerflow/runtime/user_context.py#L100-L109)。同样思路也体现在 thread_id 获取上:优先 `runtime.context["thread_id"]`,拿不到才 fallback 到 LangGraph 的 `get_config()`,见 [memory_middleware.py:67-74](../backend/packages/harness/deerflow/agents/middlewares/memory_middleware.py#L67-L74)。

**Q2.3(边界/异常)** 如果哪儿都没设置 user_id(CLI、测试、迁移脚本),记忆写到哪里去?会不会抛异常?

**参考回答**:不会抛异常,会落到 `"default"` 用户桶:`get_effective_user_id()` 设计上 never raises,ContextVar unset 时返回 `DEFAULT_USER_ID`,文件写到 `{base_dir}/users/default/memory.json`,见 [user_context.py:97-109](../backend/packages/harness/deerflow/runtime/user_context.py#L97-L109) 和 [paths.py:210-212](../backend/packages/harness/deerflow/config/paths.py#L210-L212)。这是刻意的 fail-open:记忆是增强功能,不能因为上下文缺失把主对话流程打挂。对比 persistence 层的 `require_current_user()` 是 fail-closed(直接 raise),两个层级的语义是分开的——文件系统路径解析"必须总有一个合法桶"。

---

## 问题链 3:LLM 更新流水线与同步调用路径

**Q3.1(基础)** `aupdate_memory` 是 async 接口,但里面并没有 `await model.ainvoke()`,而是 `asyncio.to_thread(self._do_update_memory_sync, ...)` 跑**同步** invoke。为什么 async 接口要包一层同步实现?

**参考回答**:这是为修复 issue #2615 的跨事件循环连接复用 bug 而刻意设计的。langchain provider 的 async httpx `AsyncClient` 连接池是 `@lru_cache` 全局缓存的、被 lead agent 共享;如果在记忆更新里创建第二个 event loop 去 `asyncio.run()` 或触碰这个池,就会出现连接被错误的 loop 复用。改成纯同步 `model.invoke()` 后,同步 HTTP 走的是完全独立的连接池,全程不碰 async client,从根上消除了跨 loop 复用;`asyncio.to_thread` 只是把阻塞调用挪出事件循环,见 [updater.py:467-492](../backend/packages/harness/deerflow/agents/memory/updater.py#L467-L492) 和 [updater.py:494-510](../backend/packages/harness/deerflow/agents/memory/updater.py#L494-L510) 的注释。

**链路解析**:
```
调用方(两种入口)
├─ aupdate_memory() ──► asyncio.to_thread(_do_update_memory_sync)
└─ update_memory() ───► 检测到 running loop?
                          ├─ 是 → _SYNC_MEMORY_UPDATER_EXECUTOR.submit() → future.result()
                          └─ 否 → 直接调 _do_update_memory_sync
                                       │
                                       ▼
                          model.invoke(prompt, config={"run_name": "memory_agent"})
                                       │ (同步 HTTP, 独立连接池)
                                       ▼
                          _finalize_update(): 解析 → 应用 → save()
```

**Q3.2(深挖)** 同步 `update_memory()` 在已有事件循环里被调用时(比如 LangGraph node 里),怎么处理?直接调不就阻塞 loop 了吗?

**参考回答**:`update_memory()` 先 `asyncio.get_running_loop()` 探测,如果处在运行中的 loop 里,就把 `_do_update_memory_sync` 提交到模块级的 `_SYNC_MEMORY_UPDATER_EXECUTOR`——一个 `max_workers=4`、`thread_name_prefix="memory-updater-sync"` 的 ThreadPoolExecutor,然后 `future.result()` 同步等待;不在 loop 里就直接调用,见 [updater.py:570-598](../backend/packages/harness/deerflow/agents/memory/updater.py#L570-L598)。这个 executor 在模块加载时创建、通过 `atexit` 注册 `shutdown(wait=False)` 清理,见 [updater.py:34-38](../backend/packages/harness/deerflow/agents/memory/updater.py#L34-L38)。值得注意的是这里依然是**阻塞等待** future 结果的——它解决的是"不能碰 async 连接池"而不是"不能阻塞",所以真正的后台化是靠上游的 queue/timer 线程完成的;`run_name="memory_agent"` 让这次 LLM 调用在 tracing 里可辨识,见 [updater.py:524](../backend/packages/harness/deerflow/agents/memory/updater.py#L524)。

**Q3.3(边界/异常)** LLM 返回了解析不了的 JSON,或者模型调用本身抛异常,记忆更新会怎样?会影响主对话吗?

**参考回答**:完全不影响,返回 False 完事。`_do_update_memory_sync` 把 `json.JSONDecodeError` 单独 catch 记 warning,其他所有 Exception 走 `logger.exception` 后也返回 False,见 [updater.py:532-537](../backend/packages/harness/deerflow/agents/memory/updater.py#L532-L537)。queue 侧逐条处理时也有 per-context 的 try/except,一条失败不影响同批其他 thread,见 [queue.py:190-206](../backend/packages/harness/deerflow/agents/memory/queue.py#L190-L206)。整条链路是 best-effort:失败只损失这一次记忆更新,主对话、后续批次都不受牵连;连"记忆未启用或消息为空"也只是静默返回 False,见 [updater.py:430-438](../backend/packages/harness/deerflow/agents/memory/updater.py#L430-L438)。

---

## 问题链 4:LLM 响应解析、归一化与 fail-closed 部分更新

**Q4.1(基础)** LLM 被要求"只返回 JSON",但实际经常裹着 thinking 内容、markdown fence 或废话。你们怎么从响应里抠出 JSON?

**参考回答**:`_parse_memory_update_response` 不做"修复",只做"安全提取":先把 content 统一成文本(处理 str / content-block list 两种形态),然后用正则找每一个 `{` 的位置,逐个尝试 `json.JSONDecoder().raw_decode()`,第一个解析成功**且**包含全部四个必需顶层键(`user`/`history`/`newFacts`/`factsToRemove`)的 dict 被采纳,之后进 `_normalize_memory_update_data` 归一化;全都失败就抛 `JSONDecodeError`,见 [updater.py:313-331](../backend/packages/harness/deerflow/agents/memory/updater.py#L313-L331)、必需键定义 [updater.py:230](../backend/packages/harness/deerflow/agents/memory/updater.py#L230) 和文本抽取 [updater.py:193-227](../backend/packages/harness/deerflow/agents/memory/updater.py#L193-L227)。raw_decode 的好处是只消费"从该位置起的一个完整 JSON 值",fence 后面的解释文字不会干扰;注释也明确"does not repair truncated or malformed JSON"——截断的 JSON 直接判失败等下一轮,而不是猜测修补。

**链路解析**:
```
LLM response.content (str 或 content blocks)
   │ _extract_text()            # 拼 text block, 丢弃非文本块
   ▼
纯文本 → re.finditer(r"\{") 逐个起点试 raw_decode
   │ 解析成功且 ⊇ {user,history,newFacts,factsToRemove}?
   ├─ 是 → _normalize_memory_update_data()
   │         ├─ factsToRemove: 只留 str 元素
   │         └─ newFacts: 逐条 _normalize_memory_update_fact()
   ▼
返回规整 update_data → _apply_updates()
   全部失败 → raise JSONDecodeError → 上层记 warning, return False
```

**Q4.2(深挖)** 模型输出的 fact 里 `confidence` 可能是字符串 `"0.9"`、可能是 `true`、可能是 `NaN`。归一化怎么处理这些脏数据?

**参考回答**:`_normalize_memory_update_fact` 是一条严格的清洗管线:bool 直接拒绝(`isinstance(raw_confidence, bool)` 先于数值判断,因为 Python 里 bool 是 int 子类);字符串 strip 后尝试 `float()`,失败则丢弃整条 fact;int/float 转 float 后还要求 `math.isfinite`,NaN/inf 被丢弃;content 必须是非空字符串,category 缺省回 `"context"`,见 [updater.py:233-278](../backend/packages/harness/deerflow/agents/memory/updater.py#L233-L278)。持久化侧入口(手动 API)`_validate_confidence` 更严格——非有限或越界 [0,1] 直接 raise ValueError,见 [updater.py:89-93](../backend/packages/harness/deerflow/agents/memory/updater.py#L89-L93)。注入侧排序再用 `_coerce_confidence` 做最后一道 clamp 到 [0,1]、非有限值回落默认值,防止 NaN 在排序里"胜出",见 [prompt.py:303-316](../backend/packages/harness/deerflow/agents/memory/prompt.py#L303-L316)。三道防线各司其职:写入前归一化、API 校验、读取时钳制。

**Q4.3(边界/异常)** 设想模型返回了合法的 `factsToRemove`(要删 3 条旧 fact),但 `newFacts` 里有一条畸形的 fact 被归一化丢弃了。直接应用会发生什么?你们怎么防?

**参考回答**:直接应用就是灾难——旧 fact 被删了、新 fact 没加上,等于**静默丢数据**。代码对此 fail-closed:`_normalize_memory_update_data` 里只要 `factsToRemove` 非空且任何一个 newFact 被丢弃(`dropped_new_fact=True`),就主动抛 `JSONDecodeError("Unsafe partial memory update: factsToRemove with malformed newFacts")`,整个更新作废、memory 保持原样,见 [updater.py:287-303](../backend/packages/harness/deerflow/agents/memory/updater.py#L287-L303)。反过来,如果只是 newFacts 整体缺失/为空、没有删除操作,属于"无害的部分更新",可以安全放行。这是典型的"删除+新增必须原子可见,否则宁可不做"的保守策略——反例就是不做这个检查:模型一次抽风输出就能把用户积累的高价值 fact 清掉,而且不可恢复。

---

## 问题链 5:纠错/强化信号检测与消息过滤

**Q5.1(基础)** 系统会检测"用户在纠正 agent"和"用户在表扬 agent"两种信号,这个检测具体怎么做的?为什么只看最近几条消息?

**参考回答**:`detect_correction` / `detect_reinforcement` 用预编译正则对**最近 6 条**(`messages[-6:]`)human 消息做匹配,双语覆盖:英文如 `that's wrong`、`you misunderstood`、`try again`,中文如 `不对`、`你理解错了`、`重新来`;强化侧如 `yes, exactly`、`完全正确`、`就是这个意思`,见 [message_processing.py:10-37](../backend/packages/harness/deerflow/agents/memory/message_processing.py#L10-L37) 和 [message_processing.py:88-109](../backend/packages/harness/deerflow/agents/memory/message_processing.py#L88-L109)。只看最近 6 条是信噪比取舍:纠正信号的时效性强,太旧的消息里匹配到"重试"字样多半是历史噪音;只看 human 是因为纠正语义只可能来自用户。检测之前消息已经过了 `filter_messages_for_memory`:保留 human 和**无 tool_calls** 的最终 AI 回复,中间工具调用过程全部丢弃——长期记忆记的是"关于用户的事实",不是执行轨迹,见 [message_processing.py:56-85](../backend/packages/harness/deerflow/agents/memory/message_processing.py#L56-L85)。剥上传块用的 `_UPLOAD_BLOCK_RE` 是 `[\s\S]*?` 非贪婪 + `re.IGNORECASE`,只吞最小闭合块,不会误伤标签之后的正文,见 [message_processing.py:9](../backend/packages/harness/deerflow/agents/memory/message_processing.py#L9)。

**链路解析**:
```
filtered_messages (已剥上传块/丢 tool_calls)
   ▼
messages[-6:] 中的 human 消息
   ├─ 命中 _CORRECTION_PATTERNS? → correction_detected=True
   │        └─ 是 → 跳过 reinforcement 检测(互斥)
   └─ 否则命中 _REINFORCEMENT_PATTERNS? → reinforcement_detected=True
   ▼
随 ConversationContext 入队(去重合并时 OR 保留, 见问题链 1)
   ▼
MemoryUpdater._build_correction_hint() 生成 IMPORTANT 提示注入 prompt
   ▼
correction → 要求 LLM 产出 category="correction", confidence >= 0.95 的 fact
reinforcement → category="preference"/"behavior", confidence >= 0.9
```

**Q5.2(深挖)** 检测到信号之后,它是怎么影响最终写入的记忆的?为什么不直接程序化写一条 fact?

**参考回答**:信号本身不写 fact,而是作为 `correction_hint` 注入 MEMORY_UPDATE_PROMPT,提示模型"这段对话里检测到了显式纠正,请把正确做法记成 category=correction、confidence >= 0.95 的 fact",强化信号对应 preference/behavior、>= 0.9,见 [updater.py:397-420](../backend/packages/harness/deerflow/agents/memory/updater.py#L397-L420)。prompt 模板里对 correction 还有额外要求:错误明确时才附 `sourceError` 字段记录"之前错在哪",见 [prompt.py:126-127](../backend/packages/harness/deerflow/agents/memory/prompt.py#L126-L127)。不程序化直写的原因:正则只能告诉你"有纠正",告诉不了你"纠正的内容是什么"——从对话里提炼正确做法本质上是理解任务,必须交给 LLM;正则只做廉价的触发器。另外两者互斥(`reinforcement_detected = not correction_detected and ...`),纠正优先,避免同一段对话既记"做错了"又记"做得好",见 [memory_middleware.py:94-95](../backend/packages/harness/deerflow/agents/middlewares/memory_middleware.py#L94-L95)。正则误报是可接受的:代价只是 prompt 里多一段 hint,LLM 仍会基于对话内容判断是否真的有纠正;漏报才会丢高价值信号,所以模式宁可宽一点(连 `换一种`、`改用` 这种弱信号也收录)。

---

## 问题链 6:事实去重、置信度阈值与容量控制

**Q6.1(基础)** LLM 每轮都可能抽出一堆 fact,你们凭什么决定哪些能落库?

**参考回答**:两道闸:置信度阈值和内容去重。`_apply_updates` 里只有 `confidence >= config.fact_confidence_threshold`(默认 **0.7**,可配 0.0-1.0)的 fact 才会进入后续流程;prompt 侧也引导模型打分——明确陈述 0.9-1.0、强烈暗示 0.7-0.8、推测模式 0.5-0.6,0.7 阈值正好把"推测"档挡在门外,见 [updater.py:648-649](../backend/packages/harness/deerflow/agents/memory/updater.py#L648-L649)、[memory_config.py:49-54](../backend/packages/harness/deerflow/config/memory_config.py#L49-L54) 和 [prompt.py:84-87](../backend/packages/harness/deerflow/agents/memory/prompt.py#L84-L87)。落库时还会盖上 `createdAt` 时间戳,由 `utc_now_iso_z()` 统一输出带 `Z` 后缀的 ISO-8601 UTC 时间,保证存储格式标准一致,见 [storage.py:19-21](../backend/packages/harness/deerflow/agents/memory/storage.py#L19-L21)。

**链路解析**:
```
LLM newFacts (每条带 content/category/confidence)
   │
   ▼ confidence >= 0.7 ? ──否──► 丢弃
   │ 是
   ▼ content.strip().casefold() 生成 fact_key
   │ key 已在 existing_fact_keys? ──是──► 跳过(精确去重)
   │ 否
   ▼ 生成 fact_{uuid4().hex[:8]}, source=thread_id, 落库
   │
   ▼ facts 总数 > max_facts(100)? ──是──► 按 confidence 降序截断
```

**Q6.2(深挖)** 去重是怎么做的?语义级去重(embedding 相似度)还是字符串匹配?为什么?

**参考回答**:是**精确字符串去重**,不是语义去重。`_fact_content_key` 对 content 做 `strip().casefold()`,和现有全部 fact 的 key 集合比对,重复就跳过;每插入一条新 fact 也即时把 key 加进集合,防止同一批 newFacts 内部自相重复,见 [updater.py:371-377](../backend/packages/harness/deerflow/agents/memory/updater.py#L371-L377) 和 [updater.py:645-673](../backend/packages/harness/deerflow/agents/memory/updater.py#L645-L673)。不做 embedding 相似度的原因:记忆更新本身是防抖后的低频后台任务,引入向量模型/索引成本高且带来"相似度阈值误杀"的新问题;而真正矛盾的旧 fact 是靠 prompt 里的 `factsToRemove` 机制由 LLM 显式删除的——"去重防冗余、删除解矛盾"两个职责是分开的。新 fact 的 id 是 `fact_{uuid4().hex[:8]}`,`source` 记来源 thread_id 便于溯源,见 [updater.py:658-665](../backend/packages/harness/deerflow/agents/memory/updater.py#L658-L665)。

**Q6.3(边界/异常)** fact 数量超过 `max_facts` 时怎么淘汰?这样淘汰有什么问题?

**参考回答**:超限(默认 **100** 条,可配 10-500)时按 confidence 降序排序、只保留前 100 条,见 [updater.py:675-682](../backend/packages/harness/deerflow/agents/memory/updater.py#L675-L682) 和 [memory_config.py:43-48](../backend/packages/harness/deerflow/config/memory_config.py#L43-L48)。已知短板是"时间盲点":一条 confidence 0.95 但已过时的事实会永远挤掉 0.75 的新事实——系统把"纠正过时事实"的职责完全交给了 LLM 的 `factsToRemove` 输出和 correction 类 fact(correction 要求 confidence >= 0.95,见 [prompt.py:126-127](../backend/packages/harness/deerflow/agents/memory/prompt.py#L126-L127)),淘汰策略本身不看 `createdAt`。面试时可以主动指出:如果要改进,自然的方向是淘汰时引入 recency 衰减或对 correction 类 fact 加权。

---

## 问题链 7:持久化与注入 —— 写出安全与读入预算

> 本链覆盖 memory.json 的两端:写入侧的原子性/缓存一致性,读出侧的注入预算与 token 计数。

**Q7.1(基础)** `save()` 写 `memory.json` 的过程是"先写临时文件再 rename",这套 atomic write 具体怎么做的?防的是什么?读路径的缓存又怎么保证一致?

**参考回答**:`FileMemoryStorage.save()` 先在目标文件同目录下生成 `{uuid}.tmp` 临时文件,把完整 JSON 写进去并 close,然后 `temp_path.replace(file_path)` 原子替换——同目录保证在同一文件系统上,replace 在 POSIX/Windows 上都是原子语义,见 [storage.py:165-189](../backend/packages/harness/deerflow/agents/memory/storage.py#L165-L189)。防的是**写一半进程崩溃/掉电留下截断的 JSON**:读者要么看到旧文件、要么看到新文件,永远看不到半个文件。save 前先 `{**memory_data, "lastUpdated": ...}` 浅拷贝,避免副作用污染调用方 dict 和缓存引用,见 [storage.py:167-170](../backend/packages/harness/deerflow/agents/memory/storage.py#L167-L170)。读路径 `load()` 用 **mtime 校验**缓存(key 是 `(user_id, agent_name)`),mtime 变了就重读文件,所以跨进程写也能被发现,见 [storage.py:119-143](../backend/packages/harness/deerflow/agents/memory/storage.py#L119-L143);但多进程同时写仍是"最后一个 replace 赢",这是文件存储的固有限制,所以 `get_memory_storage()` 支持配置 `storage_class` 换成数据库实现,加载失败校验子类后 fallback 回 `FileMemoryStorage`,见 [storage.py:196-231](../backend/packages/harness/deerflow/agents/memory/storage.py#L196-L231)。

**链路解析**:
```
save(memory_data)
   │
   ▼ {**memory_data, "lastUpdated": now}   # 浅拷贝, 不污染调用方
   ▼ open({uuid}.tmp, "w") → json.dump → close
   ▼ temp_path.replace(memory.json)        # 原子替换, 读者无中间态
   ▼ stat mtime → 更新 _memory_cache[(user_id, agent_name)]
   │
   ├─ 任何 OSError → logger.error, return False (不抛, best-effort)
   └─ import_memory_data 等强一致入口检查 False → raise OSError
```

**Q7.2(深挖)** `_finalize_update` 里为什么要 `copy.deepcopy(current_memory)` 再 `_apply_updates`?直接在缓存对象上改不行吗?

**参考回答**:不行,因为 `load()` 返回的是**缓存里的同一个对象引用**。如果直接在它上面做 in-place 的删除/追加,之后 `save()` 一旦失败(return False),缓存里的对象已经被改了一半——内存状态和磁盘状态不一致,下次 load(mtime 没变,命中缓存)拿到的就是这份"脏"数据,见 [updater.py:451-465](../backend/packages/harness/deerflow/agents/memory/updater.py#L451-L465) 的注释。deepcopy 之后所有 mutation 都发生在副本上,save 成功才用副本内容刷新缓存,save 失败缓存原对象毫发无损。反例分析:删掉这个 deepcopy,遇到一次磁盘满/权限错误,就会出现"API 读到的 fact 列表和磁盘 JSON 不一样"的灵异 bug,而且进程不重启不会自愈。失败语义上 `save()` 返回 False 而不抛异常,只有 `import_memory_data`/`clear_memory_data` 这类用户显式发起的写才转成 `OSError` 抛出,见 [updater.py:75-86](../backend/packages/harness/deerflow/agents/memory/updater.py#L75-L86)。

**Q7.3(深挖)** 记忆注入 system prompt 有 2000 token 的预算,这个预算具体是怎么执行的?超了怎么办?

**参考回答**:`format_memory_for_injection` 按三段拼装:User Context、History 全量放入(它们自身有 prompt 侧的长度约束),Facts 按 confidence 降序逐条**试装**——先算已有 sections 的 base token 数,再逐条增量计算每行 fact 的 token,`running_tokens + line_tokens <= max_tokens` 才收入,第一条放不下的 fact 直接 break;最后兜底再整体 count 一次,仍超则按 char/token 比例截断到 95% 并加 `"\n..."`,见 [prompt.py:319-439](../backend/packages/harness/deerflow/agents/memory/prompt.py#L319-L439) 中排序与增量预算的 [prompt.py:377-419](../backend/packages/harness/deerflow/agents/memory/prompt.py#L377-L419)、截断的 [prompt.py:431-437](../backend/packages/harness/deerflow/agents/memory/prompt.py#L431-L437)。`max_injection_tokens` 默认 **2000**(可配 100-8000),见 [memory_config.py:59-64](../backend/packages/harness/deerflow/config/memory_config.py#L59-L64)。correction 类 fact 带 sourceError 时渲染成 `- [correction | 0.95] 正确做法 (avoid: 错误做法)`,让 agent 同时看到"该怎么做"和"别怎么做",见 [prompt.py:405-409](../backend/packages/harness/deerflow/agents/memory/prompt.py#L405-L409)。增量式 token 记账(而不是每加一条就全文重算)是个有意的性能优化:fact 最多 100 条,全量重算就是 O(n^2) 次 encode。

**链路解析**:
```
memory.json → get_memory_data(user_id=...)
   ▼
format_memory_for_injection(max_tokens=2000)
   ├─ User Context  (work/personal/topOfMind)
   ├─ History       (recent/earlier/background)
   └─ Facts: 按 confidence 降序
        for fact in ranked:
            line_tokens = _count_tokens("\n" + line)
            running + line <= 2000 ? 装入 : break
   ▼
整体再 count 一次 → 超预算则按 95% 字符比例截断
   ▼
<memory>...</memory> 注入 system prompt (lead_agent/prompt.py:589-601)
```

**Q7.4(边界/异常)** token 计数用 tiktoken,但 tiktoken 首次加载要联网下载 BPE 数据。在断网/GFW 环境会发生什么?注入侧整个挂了怎么办?

**参考回答**:首次 `tiktoken.get_encoding("cl100k_base")` 可能阻塞数十分钟等 OS TCP 超时(issue #3402/#3429)。代码做了三层防御:(1) 进程启动时 gateway 用 `asyncio.to_thread(warm_tiktoken_cache)` 预热,把阻塞挪出请求路径;(2) 加载失败缓存 `(None, monotonic_timestamp)`,**600 秒**内(`_TIKTOKEN_RETRY_COOLDOWN_S`)后续调用直接走字符估算,不再重试阻塞;600 秒后允许自愈重试;加载中还放了 `_TIKTOKEN_ENCODING_LOADING` 哨兵防止并发重复触发下载,见 [prompt.py:184-240](../backend/packages/harness/deerflow/agents/memory/prompt.py#L184-L240);(3) 配置项 `token_counting: "char"` 可彻底跳过 tiktoken,见 [memory_config.py:65-75](../backend/packages/harness/deerflow/config/memory_config.py#L65-L75)。字符估算还是 CJK 感知的:非 CJK 按 4 字符/token、CJK(中日韩)按 2 字符/token,修正了朴素 `len//4` 对中文记忆的严重低估,见 [prompt.py:243-260](../backend/packages/harness/deerflow/agents/memory/prompt.py#L243-L260)。注入路径整体 fail-open:lead agent 组 prompt 时整个 memory 块包在 try/except 里,异常返回空字符串;文件损坏时 `_load_memory_from_file` 返回 `create_empty_memory()` 空骨架,见 [prompt.py(lead_agent):574-604](../backend/packages/harness/deerflow/agents/lead_agent/prompt.py#L574-L604) 和 [storage.py:104-117](../backend/packages/harness/deerflow/agents/memory/storage.py#L104-L117)。

---

## 问题链 8:per-user 隔离、上传事件清洗与迁移

**Q8.1(基础)** 多用户部署下,用户 A 的记忆会写到用户 B 的文件里吗?路径隔离怎么做的?

**参考回答**:不会。文件路径按 `{base_dir}/users/{user_id}/memory.json`(per-user)或 `{base_dir}/users/{user_id}/agents/{name}/memory.json`(per-user-per-agent)解析,`user_id` 经过 `_validate_user_id` 清洗,agent_name 经过 `AGENT_NAME_PATTERN` 正则校验防路径穿越,见 [paths.py:210-224](../backend/packages/harness/deerflow/config/paths.py#L210-L224) 和 [storage.py:73-102](../backend/packages/harness/deerflow/agents/memory/storage.py#L73-L102)。缓存 key 同样是 `(user_id, agent_name)` 二元组,不同用户的缓存互不污染,见 [storage.py:119-121](../backend/packages/harness/deerflow/agents/memory/storage.py#L119-L121)。一个刻意的例外:`storage_path` 配成**绝对路径**时所有用户共享同一个文件——配置注释里明说这会"opt out of per-user isolation",见 [memory_config.py:15-28](../backend/packages/harness/deerflow/config/memory_config.py#L15-L28) 和路径解析分支 [storage.py:90-93](../backend/packages/harness/deerflow/agents/memory/storage.py#L90-L93)。

**链路解析**:
```
get_memory_file_path(agent_name, user_id)
   │
   ├─ user_id + agent_name → users/{uid}/agents/{name}/memory.json
   ├─ user_id only         → users/{uid}/memory.json        (默认)
   │     └─ storage_path 为绝对路径? → 直接用(共享, 放弃隔离)
   ├─ agent_name only(legacy) → agents/{name}/memory.json
   └─ 皆无(legacy)            → {base_dir}/memory.json
   ▼
agent_name 过 AGENT_NAME_PATTERN;user_id 过 _validate_user_id → 防 ../ 穿越
```

**Q8.2(深挖)** 用户上传文件这件事,为什么系统三处地方都在防它被写进长期记忆?

**参考回答**:因为上传文件是 session 级的——`/mnt/user-data/uploads/` 下的文件下个会话就不存在了,记进长期记忆会让 agent 未来去找不存在的文件。三道防线:(1) 输入侧,`filter_messages_for_memory` 把 human 消息里的 `<uploaded_files>...</uploaded_files>` 块剥掉,剥空后整条消息连同紧随的 AI 回复一起跳过(`skip_next_ai`),见 [message_processing.py:56-85](../backend/packages/harness/deerflow/agents/memory/message_processing.py#L56-L85),`format_conversation_for_update` 再剥一次并把超长消息截断到 **1000** 字符,见 [prompt.py:468-478](../backend/packages/harness/deerflow/agents/memory/prompt.py#L468-L478);(2) prompt 侧,MEMORY_UPDATE_PROMPT 显式禁止记录上传事件,见 [prompt.py:134-136](../backend/packages/harness/deerflow/agents/memory/prompt.py#L134-L136);(3) 输出侧,`_strip_upload_mentions_from_memory` 用 `_UPLOAD_SENTENCE_RE` 从 summary 和 fact 里清洗"uploaded a file"这类句子,正则刻意收窄以免误伤"User works with CSV files"这种合法事实,见 [updater.py:334-368](../backend/packages/harness/deerflow/agents/memory/updater.py#L334-L368)。典型的 defense-in-depth:任何一道失守都还有下一道。

**Q8.3(边界/异常)** 老版本是全局单文件 `memory.json`,升级到 per-user 布局后老数据怎么办?

**参考回答**:提供一次性迁移脚本 `migrate_user_isolation.py`,幂等、支持 `--dry-run`。其中 `migrate_memory` 把 `{base_dir}/memory.json` 移动到 `users/{user_id}/memory.json`(默认由 `--user-id default` 认领);若目标已存在,不覆盖,而是把 legacy 文件改名成 `memory.legacy.json` 留人工处理,见 [migrate_user_isolation.py:133-162](../backend/scripts/migrate_user_isolation.py#L133-L162)。thread 目录的归属则从 sqlite 的 `threads_meta` 表查 `thread_id → user_id` 映射(查不到归 "default"),目标已存在的冲突目录挪到 `migration-conflicts/` 而不是删除,见 [migrate_user_isolation.py:40-69](../backend/scripts/migrate_user_isolation.py#L40-L69) 和 [migrate_user_isolation.py:165-185](../backend/scripts/migrate_user_isolation.py#L165-L185)。整个迁移策略是"宁可留冲突让人看,绝不静默覆盖用户数据";运行结束还会显式 warning 列出被归到 default 的无主 thread 和 agent,提示运维手工搬移,见 [migrate_user_isolation.py:227-238](../backend/scripts/migrate_user_isolation.py#L227-L238)。

---

## 面试官最爱追问的 3 个点

1. **"30 秒后 timer 线程里 user_id 是哪来的?"** —— 这是全系统最容易踩的坑。应答策略:一句话点破"ContextVar 不跨裸线程传播,所以入队时在请求线程物化进 `ConversationContext`",并主动补充 summarization hook 路径用 `runtime.context["user_id"]` 三级兜底,展示你读过 [queue.py:66-69](../backend/packages/harness/deerflow/agents/memory/queue.py#L66-L69) 的注释而不仅是猜。能顺带点出"不这样做就是多用户串号+隐私泄露"的反例,直接拉满。

2. **"LLM 输出的脏 JSON/脏数据你们兜底了吗?"** —— 应答策略:按防线顺序背:raw_decode 逐 `{` 扫描提取 → 逐字段归一化(bool/NaN/字符串 confidence 处理)→ "factsToRemove + 畸形 newFacts" fail-closed 整体作废 → 置信度阈值 0.7 + casefold 精确去重 + max_facts=100 截断。强调"删除+新增必须同时有效,否则宁可不更新"这条核心不变量。

3. **"写文件崩溃了会不会留下半个 JSON?"** —— 应答策略:讲 temp+rename 原子写(同目录 uuid tmp → replace),再补 deepcopy 防缓存污染的次级细节,最后承认文件存储的多进程"最后写入赢"局限并指出 `storage_class` 可插拔是留给数据库实现的扩展口。主动暴露局限比假装完美更加分。

三个点的共同主线:这套系统的每一处"麻烦设计"(物化 user_id、fail-closed、temp+rename)都是在为"异步 + LLM 不可靠 + 多用户"三个现实约束付保费——回答时点出这条主线,就能把散点答案串成系统设计观。
