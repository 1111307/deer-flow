# MCP 协议集成 —— 面试问题链

> 本文档按"大厂面试追问链"组织:每条问题链从基础问题开始,层层追问到实现细节、设计权衡、异常边界。配套深读笔记:[09-mcp-integration.md](09-mcp-integration.md)(深读笔记讲"怎么实现",本文档讲"怎么被问、怎么答")。
> 代码引用格式:[文件名.py:起始行-结束行](../backend/仓库相对路径#L起始行-L结束行)。只引用你实际读过的行,禁止编造行号。

## 全局链路总览

```
extensions_config.json (Gateway API 可改, 独立进程写盘)
   │ ExtensionsConfig.from_file()          # 每次重新读盘, 不走进程内缓存
   ▼
build_servers_config()  →  build_server_params()   # 逐 server 翻译 stdio/sse/http
   ▼
get_initial_oauth_headers()  # 先取一次 token, 注进 sse/http 的 headers(建连/发现用)
build_oauth_tool_interceptor()  # 再包一层拦截器(每次工具调用时续期)
   ▼
MultiServerMCPClient(servers_config, tool_interceptors, tool_name_prefix=True)
   │ await client.get_tools()              # 临时 session 做工具发现
   ▼
按 transport 分流:
   ├─ stdio → _make_session_pool_tool()    # 包持久会话(cwd/TMPDIR 钉住 + 路径翻译)
   └─ sse/http → 原样返回                  # 不池化(#3203 anyio TaskGroup 跨任务清理崩溃)
   ▼
make_sync_tool_wrapper()  # 给只有 coroutine 的工具补 func, 支持同步流式调用
   ▼
cache.py: _mcp_tools_cache + _config_mtime   # mtime 失效 + 懒加载
   ▼
运行时每次工具调用(stdio 池化工具):
   thread_id = _extract_thread_id(runtime)        # runtime.context → config → langgraph
   user_id   = resolve_runtime_user_id(runtime)   # context["user_id"] → ContextVar → 兜底
   scope_key = f"{user_id}:{thread_id}"
   session = pool.get_session(server, scope_key, connection)  # LRU + in-flight 去重 + owner-task
   拦截器链(reversed 包装)→ session.call_tool(original_name, args, meta={headers})
   结果:快照 diff + 正则 → /mnt/user-data/... 虚拟路径(树外/远程原样保留)
```

## 问题链 1:MultiServerMCPClient 配置翻译与装配

**Q1.1(基础)** 你们的 MCP 接入是基于 langchain-mcp-adapters 的,用户配置到 `MultiServerMCPClient` 之间做了一层翻译。这个翻译具体怎么实现的?

**参考回答**:入口是 `build_servers_config()`,它先调 `get_enabled_mcp_servers()` 过滤掉 disabled 的 server,然后逐个调 `build_server_params()` 做翻译,产出 `dict[str, dict]` 直接喂给 `MultiServerMCPClient`,见 [client.py:45-68](../backend/packages/harness/deerflow/mcp/client.py#L45-L68)。翻译逻辑按 transport 分三支,transport 取 `config.type or "stdio"` 即缺省按 stdio 处理:stdio 必须有 `command`(缺了直接 `ValueError`),带上 `args` 和可选 `env`;sse/http 必须有 `url`,带上可选 `headers`;其他 transport 类型直接抛 unsupported,见 [client.py:21-42](../backend/packages/harness/deerflow/mcp/client.py#L21-L42)。关键设计是**单点失败不拖垮整体**:`build_servers_config` 里每个 server 的翻译包在 try/except 里,配置坏的 server 只记 error 日志,其余 server 照常装配,见 [client.py:61-66](../backend/packages/harness/deerflow/mcp/client.py#L61-L66)。

**链路解析**:
```
extensions_config.json
   │ get_enabled_mcp_servers()      # 过滤 enabled=false
   ▼
for name, cfg in servers:
   build_server_params(name, cfg)
      ├─ stdio → {transport, command, args, env?}     # 无 command → ValueError
      ├─ sse/http → {transport, url, headers?}        # 无 url → ValueError
      └─ 其他 → ValueError(unsupported transport)
   ▼  try/except 单 server 隔离, 坏配置只 log, 不中断循环
servers_config: dict[str, dict]  →  MultiServerMCPClient
   ▼
client.get_tools()  →  tool 名带前缀 "server_tool"(tool_name_prefix=True)
```

**Q1.2(深挖)** `get_mcp_tools()` 里为什么不直接用 `get_extensions_config()` 的进程内缓存,而要 `ExtensionsConfig.from_file()` 重新读盘?

**参考回答**:代码里有显式 NOTE 注释:Gateway API 改配置跑在**独立进程**里,LangGraph 运行时的进程内配置缓存永远看不到那次写入,所以 `get_mcp_tools()` 每次用 `ExtensionsConfig.from_file()` 从磁盘读最新配置,见 [tools.py:558-562](../backend/packages/harness/deerflow/mcp/tools.py#L558-L562)。这是"配置写方与消费方不同进程"下的必然选择,而且它跟缓存层是分工关系:mtime 缓存失效(见问题链 2)负责发现"配置变了、需要重建工具列表",`from_file()` 负责在重建时读到"变成什么样"。如果两处都用进程内缓存,Gateway 上改 MCP 配置就永远不会生效,只能重启进程。

**Q1.3(边界/异常)** 如果 langchain-mcp-adapters 没装,或者初始化整体抛异常(网络断、server 起不来),系统会怎么样?

**参考回答**:两条路径都是**优雅降级为空工具列表**,MCP 是可拔插能力。文件顶部 `from langchain_mcp_adapters.client import MultiServerMCPClient` 的 import 失败时记 warning(提示 pip install)并返回 `[]`,见 [tools.py:552-556](../backend/packages/harness/deerflow/mcp/tools.py#L552-L556);`get_mcp_tools()` 主体包在大的 try/except 里,任何异常(连接失败、握手错误、协议错误)记 error 后同样返回 `[]`,见 [tools.py:651-653](../backend/packages/harness/deerflow/mcp/tools.py#L651-L653)。此外没有启用任何 server 时也直接返回 `[]`,见 [tools.py:565-567](../backend/packages/harness/deerflow/mcp/tools.py#L565-L567)。也就是说 MCP 全线挂掉时 agent 只是少了一批工具,主对话流程不受影响。

**Q1.4(反例)** 不这样设计会怎样?——为什么不让一个坏 server 直接让 `get_mcp_tools()` 整体失败,快速暴露问题?

**参考回答**:如果 fail-fast,运营在 Gateway 上配错一个 server(比如 stdio 漏写 `command`、url 打不通),所有 MCP 工具全灭;而 `get_mcp_tools()` 在 agent 装配路径上,处理不好会连带整个 agent 不可用——爆炸半径从"一个 server"扩大成"整个系统"。现在的双层隔离把半径压到最小:单 server 翻译 try/except(坏的那一个不进 `servers_config`)+ 整体兜底返回 `[]`(全坏时退化为无 MCP),其余工具和主流程不受影响,见 [client.py:61-66](../backend/packages/harness/deerflow/mcp/client.py#L61-L66) 和 [tools.py:651-653](../backend/packages/harness/deerflow/mcp/tools.py#L651-L653)。fail-fast 适合启动期校验必填依赖,不适合这种"用户可动态增删"的扩展点。

## 问题链 2:mtime 缓存失效与懒加载

**Q2.1(基础)** MCP 工具加载很重(要起子进程/建连接做发现),你们做了缓存。但 Gateway 改了配置后缓存怎么失效?

**参考回答**:用的是**配置文件 mtime 失效**。初始化成功时记录当时的 `_config_mtime`,见 [cache.py:64-77](../backend/packages/harness/deerflow/mcp/cache.py#L64-L77);每次 `get_cached_mcp_tools()` 先调 `_is_cache_stale()`,经 `_get_config_mtime()` 重新 resolve 配置路径并 `os.path.getmtime` 拿当前磁盘 mtime,与记录值比较,`current_mtime > _config_mtime` 就判定 stale,先 `reset_mcp_tools_cache()` 再走重新初始化,见 [cache.py:17-28](../backend/packages/harness/deerflow/mcp/cache.py#L17-L28)、[cache.py:31-53](../backend/packages/harness/deerflow/mcp/cache.py#L31-L53) 和 [cache.py:97-101](../backend/packages/harness/deerflow/mcp/cache.py#L97-L101)。失效的保守原则是"拿不到 mtime 就当作没过期":`_config_mtime` 或 `current_mtime` 任一为 None 都返回 False,宁可多用一会旧缓存也不误清,见 [cache.py:44-46](../backend/packages/harness/deerflow/mcp/cache.py#L44-L46)。注意 reset 是同步函数、可能在 running loop 所在线程被调,所以它走 `close_all_sync()` 且对当前 loop 的会话**只 signal 不等待**(同步等就是自死锁:loop 要在同步调用返回后才能跑 teardown),teardown 由 owner task 异步完成,旧会话关闭与新池子单例(`reset_session_pool()`)互不干扰,见 [cache.py:144-166](../backend/packages/harness/deerflow/mcp/cache.py#L144-L166),关闭的完整分支见问题链 5 的 Q5.4。

**链路解析**:
```
get_cached_mcp_tools()
   │
   ▼ _is_cache_stale()?        # 磁盘 mtime > 缓存时记录的 mtime ?
   ├─ 是 → reset_mcp_tools_cache()
   │        ├─ 清 _mcp_tools_cache / _cache_initialized / _config_mtime
   │        ├─ session_pool.close_all_sync()   # 关掉旧持久会话(当前 loop 只 signal)
   │        └─ reset_session_pool()            # 换池子单例
   ▼
_cache_initialized?
   ├─ 否 → 懒加载初始化(三种事件循环姿态, 见 Q2.2)
   ▼
return _mcp_tools_cache or []
```

**Q2.2(深挖)** 懒加载 `get_cached_mcp_tools()` 是同步函数,但初始化是 async 的。同步函数里跑协程,你们处理了几种事件循环姿态?

**参考回答**:三种,见 [cache.py:102-127](../backend/packages/harness/deerflow/mcp/cache.py#L102-L127)。第一种:当前线程**已有 running loop**(比如 LangGraph Studio 内),不能 `run_until_complete`,于是开 `concurrent.futures.ThreadPoolExecutor`,在新线程里 `asyncio.run(initialize_mcp_tools())` 并 `future.result()` 同步等结果,见 [cache.py:107-114](../backend/packages/harness/deerflow/mcp/cache.py#L107-L114)。第二种:能拿到 loop 但**没在跑**,直接 `loop.run_until_complete(...)`,见 [cache.py:115-117](../backend/packages/harness/deerflow/mcp/cache.py#L115-L117)。第三种:连 loop 都没有(`asyncio.get_event_loop()` 抛 `RuntimeError`),直接 `asyncio.run(...)` 新建,见 [cache.py:118-124](../backend/packages/harness/deerflow/mcp/cache.py#L118-L124)。所有异常兜底记日志返回 `[]`,见 [cache.py:125-127](../backend/packages/harness/deerflow/mcp/cache.py#L125-L127)。并发初始化由模块级 `asyncio.Lock` 串行化,`initialize_mcp_tools()` 进锁后先双检 `_cache_initialized`,见 [cache.py:66-69](../backend/packages/harness/deerflow/mcp/cache.py#L66-L69)。

**链路解析**:
```
get_cached_mcp_tools() (同步上下文)
   ▼ asyncio.get_event_loop()
   ├─ loop.is_running() == True        # 如 LangGraph Studio
   │     → ThreadPoolExecutor 新线程里 asyncio.run(initialize_mcp_tools())
   │     → future.result() 同步等
   ├─ loop 存在但 idle
   │     → loop.run_until_complete(initialize_mcp_tools())
   └─ RuntimeError(无 loop)
         → asyncio.run(initialize_mcp_tools())
   ▼ 异常 → logger.exception + return []
```

## 问题链 3:OAuth token 管理——双检锁与主动续期

**Q3.1(基础)** HTTP/SSE 的 MCP server 需要 OAuth,token 会过期。你们的 token 管理怎么做的?支持哪些 grant type?

**参考回答**:核心是 `OAuthTokenManager`,按 server 维度缓存 `_OAuthToken(access_token, token_type, expires_at)`,每个 server 配一把专属 `asyncio.Lock`,见 [oauth.py:16-31](../backend/packages/harness/deerflow/mcp/oauth.py#L16-L31)。取 header 走**双检锁(double-checked locking)**:锁外先看缓存 token 未临期就直接返回;否则进该 server 的锁,锁内再检查一次(等待期间可能已被别的协程刷新),仍未命中才 `_fetch_token()` 并回填缓存,见 [oauth.py:47-65](../backend/packages/harness/deerflow/mcp/oauth.py#L47-L65)。grant type 支持两种:`client_credentials`(必须 client_id + client_secret,缺了 ValueError)和 `refresh_token`(必须 refresh_token,client 凭证可选),其他 grant_type 直接抛 unsupported,见 [oauth.py:85-99](../backend/packages/harness/deerflow/mcp/oauth.py#L85-L99)。请求体还支持 `scope`、`audience`、`extra_token_params` 透传,token 响应的字段名(`token_field`/`token_type_field`/`expires_in_field`)全部可配,适配非标准 IdP,见 [oauth.py:75-83](../backend/packages/harness/deerflow/mcp/oauth.py#L75-L83)。

**链路解析**:
```
get_authorization_header(server)
   │
   ▼ token = _tokens.get(server)              # 第一检(无锁快路径)
   ├─ 命中且未临期 → return "Bearer xxx"
   ▼
async with _locks[server]:                    # per-server 锁, server 间互不阻塞
   ▼ 再检一次(第二检, 等待期间可能已被刷新)
   ├─ 命中 → return
   ▼
_fetch_token(oauth): POST token_url (httpx, timeout=15.0s)
   │ grant_type=client_credentials → client_id+client_secret
   │ grant_type=refresh_token      → refresh_token(+可选 client 凭证)
   │ expires_in 缺省 3600s, 非法值也兜底 3600; max(expires_in, 1) 防 0/负
   ▼
_tokens[server] = fresh → return
```

**Q3.2(深挖)** "临期"是怎么判定的?为什么要提前刷新而不是过期了再刷?

**参考回答**:判定在 `_is_expiring()`:`token.expires_at <= now + timedelta(seconds=max(refresh_skew_seconds, 0))`,见 [oauth.py:67-70](../backend/packages/harness/deerflow/mcp/oauth.py#L67-L70)。`refresh_skew_seconds` 默认 **60 秒**(`McpOAuthConfig` 的字段默认,用 `max(..., 0)` 夹住负数配置)。提前 60 秒刷新是为了消掉时钟偏移和请求在途时间:如果等到真正过期那一刻才刷,一次正好跨过过期点的长请求就会带着失效 token 发出被 server 401;skew 把"过期"这个点事件变成一个 60 秒窗口,进窗口就主动换新。`_fetch_token` 里 `expires_in` 解析失败兜底 **3600 秒**,且 `max(expires_in, 1)` 防止 IdP 返回 0/负数导致 expires_at 落在过去,httpx 超时 **15.0 秒**,见 [oauth.py:101-118](../backend/packages/harness/deerflow/mcp/oauth.py#L101-L118)。token 端点故障时 `raise_for_status()` 直接上抛,缓存里没有旧 token 可回退则本次调用失败,但锁已释放,下次调用重新尝试,见 [oauth.py:101-104](../backend/packages/harness/deerflow/mcp/oauth.py#L101-L104)。

**Q3.3(深挖)** OAuth 用在两个地方:建连时的初始 header 和每次工具调用。为什么要注两次?只用一处不够吗?

**参考回答**:不够,两个时机的失效风险不同。建连/工具发现阶段还没有工具调用上下文,所以 `get_initial_oauth_headers()` 先逐 server 取一次 token,只对 `transport in ("sse", "http")` 的 server 写入 `headers`(stdio 的连接配置不会被塞 Authorization),保证 `MultiServerMCPClient.get_tools()` 的握手和发现请求能过认证,见 [tools.py:573-581](../backend/packages/harness/deerflow/mcp/tools.py#L573-L581) 和 [oauth.py:140-150](../backend/packages/harness/deerflow/mcp/oauth.py#L140-L150)。但初始 header 是**静态快照**,长会话中 token 必然过期,所以每次工具调用还要过 `oauth_interceptor`:它调 `get_authorization_header()` 走双检锁拿**当前有效**的 token,通过 `request.override(headers=...)` 覆盖后放行,没拿到 header 时原样透传,见 [oauth.py:128-137](../backend/packages/harness/deerflow/mcp/oauth.py#L128-L137)。一句话:前者管"连得上",后者管"长跑一直有效"。

**链路解析**:
```
装配期(静态):                      运行期(动态, 每次调用):
get_initial_oauth_headers()        oauth_interceptor(request, handler)
   │ per-server 取 token              │ get_authorization_header(request.server_name)
   ▼ 仅 sse/http 写入                 ▼ 双检锁拿当前有效 token
servers_config[s]["headers"]       request.override(headers={Authorization})
   │                                  ▼
   ▼                                handler(request) → base_handler → session.call_tool
get_tools() 握手/发现过认证          (token 过期时自动续期, 不靠装配期快照)
```

## 问题链 4:session pool 的 owner-task 模式与 anyio 同任务约束

**Q4.1(基础)** 你们给 MCP 会话做了个池子。直接每次调用新建 session 有什么问题?为什么池子本身还被 anyio 逼出了一个特殊设计?

**参考回答**:每次新建 session 的问题是有状态 server(比如 Playwright)的浏览器状态——打开的页面、填过的表单——在调用间全丢,模块 docstring 开篇就点了这个,见 [session_pool.py:1-10](../backend/packages/harness/deerflow/mcp/session_pool.py#L1-L10)。但池化遇到一个硬约束:`ClientSession` 底层是 anyio task group,anyio 强制 **cancel scope 必须由进入它的同一个 task 退出**,跨 task 退出直接抛 `RuntimeError: Attempted to exit cancel scope in a different task than it was entered in`(GitHub issue #3379),见 [session_pool.py:12-25](../backend/packages/harness/deerflow/mcp/session_pool.py#L12-L25)。解法是 **owner-task 模式**:每个会话由一个专属 `_run_session` task 拥有整个生命周期——它 `__aenter__`、initialize、通过 future 把活 session 发布给调用方、然后阻塞等 close 事件、最后自己跑 `__aexit__`;所有关闭路径只**发信号**,从不亲自关,保证 enter/exit 永远在同一 task,见 [session_pool.py:27-31](../backend/packages/harness/deerflow/mcp/session_pool.py#L27-L31)。

**链路解析**:
```
调用方 task                       owner task(_run_session)
   │ create_task(...) ───────────► │ cm = create_session(connection)
   │                               │ session = await cm.__aenter__()
   │                               │ await session.initialize()
   │ ◄──── ready.set_result ────── │
   │ 用 session.call_tool()        │ await close_evt.wait()     ← 阻塞整个生命期
   │ ...(任意多个调用方共享)       │
   │ (关闭时) close_evt.set() ───► │ finally: await cm.__aexit__(...)  ← 同一 task 退出
```

**Q4.2(深挖)** `_run_session` 内部,`__aenter__` 失败和 `initialize()` 失败,处理路径有什么区别?为什么这么分?

**参考回答**:两段是严格分开的,分界点是"cancel scope 进没进去",见 [session_pool.py:99-124](../backend/packages/harness/deerflow/mcp/session_pool.py#L99-L124)。`__aenter__` 抛异常意味着**还没进入 cancel scope**,没有什么可退出的,直接把异常经 `ready.set_exception(e)` 上报给等待方然后 return,绝不碰 `__aexit__`,见 [session_pool.py:101-106](../backend/packages/harness/deerflow/mcp/session_pool.py#L101-L106)。而 `initialize()` 抛异常时 scope 已经进入了,必须走 `finally: await cm.__aexit__(None, None, None)` 在本 task 内退出,否则 stdio 子进程会泄漏,见 [session_pool.py:108-124](../backend/packages/harness/deerflow/mcp/session_pool.py#L108-L124)。另外 `__aexit__` 自身的异常只记 warning 不再上抛,防止清理异常掩盖原始异常,见 [session_pool.py:121-124](../backend/packages/harness/deerflow/mcp/session_pool.py#L121-L124)。这个"enter 失败不 exit、enter 成功必 exit"的对称性是资源管理的基本功。

**Q4.3(反例)** 不这样设计会怎样?——如果不用 owner task,就让"最后一个用完的调用方"或"关闭方"直接 `await cm.__aexit__()`,会发生什么?

**参考回答**:必崩。sync-tool 路径(`make_sync_tool_wrapper`)每次调用都开新的 `asyncio.run` 事件循环,会话在回答第 N 次调用时 enter,却在回答第 N+1 次调用时从另一个 task、甚至另一个 loop 的 task exit——正是 anyio 明令禁止的跨 task cancel-scope 退出,直接 `RuntimeError`(#3379),见 [session_pool.py:22-25](../backend/packages/harness/deerflow/mcp/session_pool.py#L22-L25)。就算绕过 anyio 不崩,"谁来关"本身也是分布式难题:多调用方共享一个 session,引用计数或 GC 触发关闭都可能在错误的 task 上下文里执行。owner-task 模式把会话生命周期收敛到一个专属 task 内,用一个常驻 task 的成本换来 enter/exit 同 task 的确定性。

**Q4.4(深挖)** 等 `ready` future 时为什么套 `asyncio.shield`?调用方自己在这期间被 cancel 了会怎样?

**参考回答**:`session = await asyncio.shield(ready)`,见 [session_pool.py:219](../backend/packages/harness/deerflow/mcp/session_pool.py#L219)。shield 保证**调用方被取消不会连带取消 ready future**——否则 owner task 之后 `ready.set_result()` 会打在已取消的 future 上抛 `InvalidStateError`,会话也丢了归属。异常分支精确区分两种 case:case 1 是 **owner 自己失败**(已通过 `ready.set_exception` 上报),此时 owner 正在自己的 finally 里跑 `__aexit__`,**绝不能 cancel 它**——那会打断清理,只等它 unwinding 完;case 2 是**调用方自己被 cancel**,此时因 shield 保护 `ready` 仍 pending、owner 还活着,要 `close_evt.set()` + `task.cancel()` 让 owner 在自己 task 里退 scope,然后等它结束,见 [session_pool.py:220-246](../backend/packages/harness/deerflow/mcp/session_pool.py#L220-L246)。判别式是 `ready.done() and not ready.cancelled() and ready.exception() is not None`,两种 case 都保证不泄漏 session 和 owner task。

**链路解析**:
```
await asyncio.shield(ready) 抛异常
   ▼
owner_already_failed = ready.done() and not ready.cancelled() and ready.exception() is not None
   ├─ True (owner 失败, 正在 finally 里 __aexit__)
   │     → 不 cancel! 只 await shield(task) 等它 unwinding 完
   └─ False (调用方自己被 cancel, owner 还活着)
         → close_evt.set() + task.cancel()   # owner 在自己 task 里退 scope
         → await shield(task) 等结束
   ▼
锁内清理 _inflight[key](同一性校验后 pop) → raise
```

## 问题链 5:并发去重、LRU 与四级关闭

**Q5.1(基础)** 多个协程同时第一次请求同一个 `(server, scope)` 的会话,会不会建起重复会话?

**参考回答**:不会,有 **in-flight 去重**。`get_session()` 的 Phase 1 在 `threading.Lock` 内无 await 地做三选一判定:已注册且归属当前 loop → `move_to_end`(LRU touch)后直接返回;有同 loop 的 in-flight 记录 → 拿它的 `ready` future 去 join;都没有 → 自己成为 creator,创建 future/task 并**在任何 await 之前**把 in-flight 记录发布进 `_inflight`,让并发调用 join 自己而不是另起会话,见 [session_pool.py:146-189](../backend/packages/harness/deerflow/mcp/session_pool.py#L146-L189)。join 方走 `await asyncio.shield(join)` 共享同一个创建结果,见 [session_pool.py:212-213](../backend/packages/harness/deerflow/mcp/session_pool.py#L212-L213)。锁选 `threading.Lock` 而非 `asyncio.Lock` 的原因注释也写明了:threading.Lock 不绑定任何 event loop,sync/worker 线程路径(如 `close_all_sync`)也能安全拿同一把锁,见 [session_pool.py:76-78](../backend/packages/harness/deerflow/mcp/session_pool.py#L76-L78)。

**链路解析**:
```
get_session(server, scope)                (Phase 1: 持 threading.Lock, 无 await)
   │
   ├─ _entries 命中且 loop 是当前 → move_to_end, return session   [快路径]
   ├─ _entries 命中但 loop 是别的/已关 → evict, 准备重建
   ├─ _inflight 命中且 loop 是当前 → join = inflight.ready
   │     → Phase 2b: await asyncio.shield(join)                   [去重共享]
   └─ 否则成为 creator:
         ready=create_future(); task=create_task(_run_session)
         _inflight[key] = (loop, ready, task, close_evt)   # 先发布再 await, 防 race
   ▼
LRU: while len(_entries) >= MAX_SESSIONS(256): 弹出最老 → evicted
   ▼
Phase 2: 锁外关闭 evicted(同 loop await / 异 loop signal)
Phase 3: await shield(ready)  →  Phase 4: 提升 _inflight → _entries(still_ours 校验)
```

**Q5.2(深挖)** 池子容量上限是多少?LRU 淘汰时旧会话怎么关,属于别的 event loop 的会话又怎么关?

**参考回答**:上限 `MAX_SESSIONS = 256`,见 [session_pool.py:50](../backend/packages/harness/deerflow/mcp/session_pool.py#L50)。淘汰分两段:Phase 1 锁内只做**记录摘除**——从 OrderedDict 头部 pop 最老 entry 进 evicted 列表,不在锁内做任何 IO;真正的关闭放到锁外 Phase 2,见 [session_pool.py:191-208](../backend/packages/harness/deerflow/mcp/session_pool.py#L191-L208)。关闭按 owner 所在 loop 分路:同 loop → `await self._shutdown(...)` 确定性等 teardown 完成;异 loop → `_signal_close` 用 `loop.call_soon_threadsafe(close_evt.set)` 把 set 调度到 owner loop 上执行——因为 `asyncio.Event.set` 本身不是线程安全的,不能跨线程直接调,见 [session_pool.py:269-282](../backend/packages/harness/deerflow/mcp/session_pool.py#L269-L282);loop 已关闭说明 owner task 已随 loop 死亡,直接跳过。`get_session` 里遇到"旧会话属于另一个/已关闭 loop"也是先 evict 再为当前 loop 重建,注释明说了这个意图,见 [session_pool.py:134-136](../backend/packages/harness/deerflow/mcp/session_pool.py#L134-L136) 和 [session_pool.py:166-168](../backend/packages/harness/deerflow/mcp/session_pool.py#L166-L168)。

**Q5.3(深挖)** 关闭语义有几个层级?为什么 in-flight 的创建要 `cancel=True` 而正常会话不用?

**参考回答**:四个层级:`close_scope(scope_key)` 按 scope(通常 thread_id)关、`close_server(server_name)` 按 server 关、`close_all()` async 全关、`close_all_sync()` sync 全关,见 [session_pool.py:342-431](../backend/packages/harness/deerflow/mcp/session_pool.py#L342-L431)。`cancel=True` 只用于 in-flight 创建:owner 可能正阻塞在 `initialize()` 里——它在等初始化返回,**没有在等 close_evt**,set 事件唤不醒它,所以必须 `task.cancel()` 注入 `CancelledError`;关键点在于 cancel 只是中断等待点,owner 的 `finally` 仍会在**它自己的 task** 里跑 `__aexit__`,不违反 anyio 同任务约束,见 [session_pool.py:284-303](../backend/packages/harness/deerflow/mcp/session_pool.py#L284-L303)。已初始化的会话阻塞在 `close_evt.wait()` 上,set 事件即可温和唤醒走正常关闭,不需要 cancel。

**链路解析**:
```
close_scope / close_server / close_all / close_all_sync
   ▼ 锁内摘除记录(不在锁内关闭)
entries(已初始化)      inflight(创建中)
   │ close_evt.set()       │ close_evt.set() + task.cancel()
   │ (owner 阻塞在          │ (owner 可能阻塞在 initialize(),
   │  close_evt.wait(),     │  事件唤不醒, 必须 cancel 注入
   │  温和唤醒即可)          │  CancelledError; finally 仍同 task 退 scope)
   ▼                       ▼
owner task 的 finally: await cm.__aexit__(None, None, None)
```

**Q5.4(边界/异常)** `close_all_sync()` 要处理"owner loop 就是本线程/在别的线程跑/闲置"三种情况,分别怎么做?同步等待的超时是多少?

**参考回答**:三分支,见 [session_pool.py:407-431](../backend/packages/harness/deerflow/mcp/session_pool.py#L407-L431)。owner 是**当前 running loop**:同步等它会自死锁(同 Q2.3),只 `close_evt.set()`(in-flight 再补 `task.cancel()`),teardown 等控制权还给 loop 后异步完成——docstring 明确要求调用方之后保持 loop 运行,否则 `__aexit__` 可能不执行,要确定性关闭就该 `await close_all()`,见 [session_pool.py:378-396](../backend/packages/harness/deerflow/mcp/session_pool.py#L378-L396)。owner 在**别的线程 running**:`asyncio.run_coroutine_threadsafe(self._shutdown(...), loop)` 再 `future.result(timeout=SESSION_CLOSE_TIMEOUT)` 同步等,超时上限 **5.0 秒**,见 [session_pool.py:51](../backend/packages/harness/deerflow/mcp/session_pool.py#L51) 和 [session_pool.py:424-427](../backend/packages/harness/deerflow/mcp/session_pool.py#L424-L427)。owner loop **闲置**(存在但没在跑):`loop.run_until_complete(...)` 直接驱动它跑完 teardown,见 [session_pool.py:428-429](../backend/packages/harness/deerflow/mcp/session_pool.py#L428-L429)。还有一个防御分支:owner loop 既非当前也非 running(理论不该发生),降级为 best-effort 信号并 warning 可能泄漏,见 [session_pool.py:324-340](../backend/packages/harness/deerflow/mcp/session_pool.py#L324-L340)。

**链路解析**:
```
close_all_sync()  (可在任意线程, 可无 loop)
   ▼ per owner:
   ├─ loop.is_closed()          → skip(owner 已死)
   ├─ loop is current_running   → close_evt.set()(+cancel)   # 只 signal, 防自死锁
   ├─ loop.is_running()(异线程) → run_coroutine_threadsafe → result(timeout=5.0s)
   └─ loop 闲置                  → loop.run_until_complete(_shutdown(...))
   ▼ 任何异常 → logger.debug 继续下一个, 不中断整体
```

## 问题链 6:stdio 池化(#3203)与 cwd/TMPDIR 钉住

**Q6.1(基础)** 池化只池了 stdio,HTTP/SSE 的工具原样返回。为什么?都池了不是更统一吗?

**参考回答**:因为 HTTP/SSE transport 内部也跑在 anyio TaskGroup 上,池化后跨 async task 清理会抛 `RuntimeError`,代码注释明确指向 issue #3203,见 [tools.py:623-626](../backend/packages/harness/deerflow/mcp/tools.py#L623-L626)。分流逻辑是靠 `tool_name_prefix=True` 加的前缀反查 server——遍历 `servers_config` 找 `tool.name.startswith(f"{name}_")`,只有命中的 server transport 是 stdio 才走 `_make_session_pool_tool()` 包装,其余(包括反查不到 server 的)原样追加,见 [tools.py:628-642](../backend/packages/harness/deerflow/mcp/tools.py#L628-L642)。这是"正确性优先于统一性"的取舍:stdio 才是有状态大头(Playwright 这类本地子进程),池化收益最大;HTTP/SSE 的有状态性可以靠服务端自身机制保持,不值得为统一抽象冒崩溃风险。最后所有只有 coroutine 没有 func 的工具统一补 `make_sync_tool_wrapper`,支持同步流式调用方,见 [tools.py:644-647](../backend/packages/harness/deerflow/mcp/tools.py#L644-L647)。

**链路解析**:
```
client.get_tools()  (tool_name_prefix=True, 名字形如 "server_tool")
   │
   ▼ for tool in tools:
   反查 server: tool.name.startswith(f"{name}_")
   ▼
transport == "stdio"?
   ├─ 是 → _make_session_pool_tool()     # 池化 + cwd/TMPDIR 钉住 + 路径翻译
   └─ 否(sse/http 或未匹配) → 原样追加   # #3203: 池化会 RuntimeError
   ▼
for tool: 若无 func 且有 coroutine → tool.func = make_sync_tool_wrapper(coroutine, name)
```

**Q6.2(深挖)** stdio 子进程的 cwd 和临时目录被"钉"进了线程的 user-data 挂载树,具体怎么做的?不钉会怎样?

**参考回答**:每次调用前先做工作区准备(经 `asyncio.to_thread` 避免阻塞 loop):`ensure_thread_dirs` 建目录、在工作目录下建 `.mcp/tmp`(`_MCP_TMP_SUBDIR`)并 `chmod 0o700` 收紧权限、做调用前文件快照,见 [tools.py:153-170](../backend/packages/harness/deerflow/mcp/tools.py#L153-L170)。然后两处钉住:`session_connection["cwd"]` 默认设为该线程的 sandbox 工作目录(运维显式配了 `cwd` 则尊重配置),子进程解析相对路径输出都会落进挂载树;`env` 里用 `setdefault` 钉 `TMPDIR`/`TMP`/`TEMP` 三个变量到 `.mcp/tmp`——`setdefault` 保证不覆盖运维手工配的值,见 [tools.py:460-478](../backend/packages/harness/deerflow/mcp/tools.py#L460-L478)。不钉的后果:Node 的 `os.tmpdir()`、Python 的 `tempfile.gettempdir()`、各种 CLI 会把产物写到宿主机系统临时目录,sandbox/artifact API 只认 `/mnt/user-data` 挂载树,够不到那个路径——文件"生成了但读不出来"。设计动机就写在 `_MCP_TMP_SUBDIR` 常量注释里,见 [tools.py:28-33](../backend/packages/harness/deerflow/mcp/tools.py#L28-L33)。边界上,SSE/HTTP 会话**不走**这段准备:`is_stdio` 判定后对它们跳过全部文件系统工作(没有本地 cwd 可钉,避免无谓的递归遍历),见 [tools.py:447-455](../backend/packages/harness/deerflow/mcp/tools.py#L447-L455);且所有重 IO(准备、调用后快照 diff、结果转换)都 `asyncio.to_thread` 卸下 loop,结果无文本内容时还直接跳过第二次递归 diff,见 [tools.py:514-529](../backend/packages/harness/deerflow/mcp/tools.py#L514-L529)。

**链路解析**:
```
call_with_persistent_session (stdio 分支)
   ▼ asyncio.to_thread(_prepare_stdio_workspace)
   ├─ ensure_thread_dirs(thread_id, user_id)
   ├─ tmp_dir = work_dir/.mcp/tmp; mkdir -p; chmod 0o700
   └─ before_files = 递归快照 {path: (mtime_ns, size)}
   ▼
session_connection["cwd"] = configured_cwd or str(work_dir)     # 钉 cwd
session_env.setdefault("TMPDIR"/"TMP"/"TEMP", str(tmp_dir))     # 钉 temp, 不覆盖运维配置
   ▼
pool.get_session(server, scope_key, session_connection)
```

## 问题链 7:虚拟路径两层翻译

**Q7.1(基础)** Playwright 截图存到了宿主路径,但前端/sandbox API 只认 `/mnt/user-data/...` 虚拟路径。这个翻译怎么做的?

**参考回答**:分两层。第一层是**结构化翻译**:`ResourceLink` block 的 uri 走 `_local_uri_to_virtual_path()`——先 `_local_path_from_uri` 解析(file:// 解 quote、裸路径直收、http/https/data 等远程 scheme 返回 None 不动),`resolve()` 后校验是真实存在的文件,再 `relative_to(user_data_root)` 确认在本线程挂载树内,最后拼 `VIRTUAL_PATH_PREFIX` 前缀,见 [tools.py:50-74](../backend/packages/harness/deerflow/mcp/tools.py#L50-L74) 和 [tools.py:77-124](../backend/packages/harness/deerflow/mcp/tools.py#L77-L124)。注意这是**纯映射不拷贝**:因为 cwd/TMPDIR 已钉住(问题链 6),文件本来就在树里,缺的只是虚拟前缀,docstring 明说"no copy, no trusted-root list, no exposure of files outside the thread's own tree",见 [tools.py:84-96](../backend/packages/harness/deerflow/mcp/tools.py#L84-L96)。第二层是**自由文本翻译**:有的 server 只在文本里说 "saved as temp/page.png",用 `_LOCAL_PATH_IN_TEXT_RE` 正则扫出疑似 token,逐个喂给同一个翻译函数,翻译不了的**原样保留**——过度匹配无害是刻意设计,见 [tools.py:35-45](../backend/packages/harness/deerflow/mcp/tools.py#L35-L45) 和 [tools.py:240-288](../backend/packages/harness/deerflow/mcp/tools.py#L240-L288)。路径结尾的句子标点(如 `saved as temp/a.png.`)先 `rstrip(_TEXT_PATH_TRAILING_CHARS)` 剥掉、翻译后再拼回去,见 [tools.py:263-277](../backend/packages/harness/deerflow/mcp/tools.py#L263-L277)。

**链路解析**:
```
CallToolResult
   ├─ ResourceLink.uri ─────────────► _local_uri_to_virtual_path()
   ├─ TextContent.text ──► _rewrite_local_paths_in_text()
   │        │  _LOCAL_PATH_IN_TEXT_RE 扫 token(绝对/相对/file://)
   │        │  剥尾标点(.,;:...)→ 翻译(同段文本内 token 级缓存)→ 还原尾标点
   │        ▼
   │   _rewrite_unique_bare_filenames()   # 裸文件名 ↔ 快照 diff 关联, 唯一才改写
   ▼
/mnt/user-data/<...>   (树外/不存在/远程 → 原样保留, 不动)
   ▼
content_and_artifact: text/image/file block; isError → ToolException;
structuredContent → {"structured_content": ...} artifact(见 tools.py:401-409)
```

**Q7.2(深挖)** "Saved as page-2026.yml" 这种裸文件名连路径都不是,你们怎么处理?会不会误改写?

**参考回答**:靠**快照 diff 关联 + 唯一性约束**,见 `_rewrite_unique_bare_filenames()` [tools.py:193-237](../backend/packages/harness/deerflow/mcp/tools.py#L193-L237)。调用前后各做一次工作区快照,签名是 `(st_mtime_ns, st_size)` 二元组,diff 出"本次调用新建或修改"的文件,按 basename 分桶,见 [tools.py:127-150](../backend/packages/harness/deerflow/mcp/tools.py#L127-L150)。只有某 basename **恰好唯一映射**到一个虚拟路径(`len(set(paths)) == 1`)时才改写;歧义(同名多文件)或无候选都直接放弃并记 debug 日志,见 [tools.py:208-226](../backend/packages/harness/deerflow/mcp/tools.py#L208-L226)。改写用正则 `(?<![\w./-])name(?!(?:[\w/-]|\.[\w]))`——前后向断言保证不嵌在更长路径/单词里,句末句号允许、`.bak` 这类扩展不允许;且按名字长度降序替换,防短名先吃掉长名的子串,见 [tools.py:228-237](../backend/packages/harness/deerflow/mcp/tools.py#L228-L237)。

**链路解析**:
```
before 快照 ──┐
              ├─ diff → changed_files(本次调用新建/修改)
after 快照 ──┘        签名: (st_mtime_ns, st_size)
   ▼ 按 basename 分桶: candidates[name] = [virtual_path, ...]
unique = 恰好唯一映射的 name
   ├─ 歧义/无候选 → 放弃改写, debug 日志
   ▼
按 len(name) 降序, 正则 (?<![\w./-])name(?!(?:[\w/-]|\.[\w])) 替换
   (不嵌长路径/单词; 句末"."允许, ".bak"不允许)
```

**Q7.3(边界/异常)** 翻译后的结果怎么交给 LangChain?拦截器短路返回的 ToolMessage、MCP 的 error 结果和 structuredContent 怎么处理?

**参考回答**:转换出口是 `_convert_call_tool_result`,返回 `(content, artifact)` 的 `content_and_artifact` 格式,见 [tools.py:309-409](../backend/packages/harness/deerflow/mcp/tools.py#L309-L409)。两个直通短路:结果已是 `ToolMessage`(拦截器直接短路返回的)或 LangGraph `Command`,原样返回不动,见 [tools.py:336-348](../backend/packages/harness/deerflow/mcp/tools.py#L336-L348)。内容块逐类型转换:TextContent → text block(过 `_resolve_text` 做文本路径翻译)、ImageContent → base64 image block、ResourceLink → 按 mime 分 image/file block(url 过 `_resolve_link_url`)、EmbeddedResource 再分 Text/Blob 两路,见 [tools.py:370-399](../backend/packages/harness/deerflow/mcp/tools.py#L370-L399)。`call_tool_result.isError` 为真时,把所有 text block 拼起来抛 `ToolException`,让上层工具错误处理接管,见 [tools.py:401-403](../backend/packages/harness/deerflow/mcp/tools.py#L401-L403);`structuredContent` 非空时包成 `{"structured_content": ...}` 作为 artifact 返回,见 [tools.py:405-409](../backend/packages/harness/deerflow/mcp/tools.py#L405-L409)。

## 问题链 8:拦截器链与 user_id:thread_id 隔离

**Q8.1(基础)** 池化路径上,配置的工具拦截器(OAuth、自定义)是怎么挂进去的?为什么包装时要剥掉工具名的 server 前缀?

**参考回答**:在 `_make_session_pool_tool()` 里手工组洋葱链。最内层 `base_handler` 真正把请求发到池化 session:对 stdio,拦截器注入的 headers 通过 MCP call 的 `meta={"headers": ...}` 透传(非 Mapping 类型的 headers 记 warning 丢弃),见 [tools.py:484-493](../backend/packages/harness/deerflow/mcp/tools.py#L484-L493)。然后 `for interceptor in reversed(tool_interceptors)` 逐层外包,最后构造 `MCPToolCallRequest(name=original_name, args=arguments, server_name=..., runtime=...)` 从链头进入,见 [tools.py:495-510](../backend/packages/harness/deerflow/mcp/tools.py#L495-L510)。`reversed` 保证**列表第一个拦截器最先看到请求**。剥前缀是因为 `tool_name_prefix=True` 把工具名改成了 `server_tool` 形式防跨 server 撞名,但 MCP server 自己的 `call_tool` 只认原始名,所以包装时把 `f"{server_name}_"` 前缀剥掉恢复 `original_name`,见 [tools.py:428-432](../backend/packages/harness/deerflow/mcp/tools.py#L428-L432) 和 [tools.py:504-512](../backend/packages/harness/deerflow/mcp/tools.py#L504-L512)。自定义拦截器从 `extensions_config.json` 的 `mcpInterceptors` 列表经 `resolve_variable` 反射加载 builder 再调用得到;单个加载失败、返回非 callable 都只 warning 跳过,不会拖垮 OAuth 拦截器或整体装配,见 [tools.py:588-610](../backend/packages/harness/deerflow/mcp/tools.py#L588-L610)。没配任何拦截器时直接 `session.call_tool(original_name, arguments)` 裸调,不组链,见 [tools.py:511-512](../backend/packages/harness/deerflow/mcp/tools.py#L511-L512)。

**链路解析**:
```
tool_interceptors = [oauth, custom1, custom2]   # 配置顺序
   ▼ reversed 包装(闭包默认值防御)
wrapped(custom2(wrapped(custom1(wrapped(oauth(base_handler))))))
   ▼ 调用时
oauth → custom1 → custom2 → base_handler
                              │ headers → meta={"headers": ...}(stdio 透传)
                              ▼
                     session.call_tool(original_name, args, **call_kwargs)
   ▲ 响应沿链原路返回
(无拦截器 → 直接 session.call_tool(original_name, arguments))
```

**Q8.2(深挖)** 闭包循环里 `async def wrapped(req, _i=interceptor, _h=outer)` 这两个默认参数是干什么的?不写会怎样?

**参考回答**:这是 Python 经典**循环闭包陷阱**的防御。`wrapped` 函数体引用循环变量 `interceptor` 和 `outer`,如果直接引用,闭包捕获的是**变量本身**而不是那一轮的值——Python 闭包是 late binding,循环结束后所有 `wrapped` 看到的都是最后一轮的 `interceptor` 和最终的 `handler`,链会退化成"最后一个拦截器被调 N 次、前面的全被跳过",而且 `_h` 全部指向同一个 handler 还会把链压扁。用默认参数 `_i=interceptor, _h=outer` 在 def 语句执行时就把当前值快照为函数默认值,每层闭包各存各的,见 [tools.py:496-502](../backend/packages/harness/deerflow/mcp/tools.py#L496-L502)。这类 bug 的阴险之处在于:只配一个拦截器时永远正确,单测很容易漏,线上配第二个拦截器才爆——面试里主动讲出这个点是加分项。

**Q8.3(深挖)** 池化会话的隔离粒度是什么?为什么 scope_key 要拼成 `user_id:thread_id`,只用 thread_id 不行吗?

**参考回答**:不行,代码注释给了明确的串扰场景:文件系统隔离本来就是 per-(user_id, thread_id) 的,如果 scope_key 只用 thread_id,两个不同用户撞了同一个 thread_id(比如都没传、都兜底成 "default")就会**共享同一个有状态 MCP 会话**——A 的浏览器页面、登录态被 B 看到,这是安全事故,见 [tools.py:442-445](../backend/packages/harness/deerflow/mcp/tools.py#L442-L445)。两个分量的来源也都有讲究:`user_id` 来自 `resolve_runtime_user_id(runtime)`,优先取 `runtime.context["user_id"]`(网关鉴权后注入、最能穿越任务边界的通道),再退 ContextVar,最后兜底默认用户;`thread_id` 来自 `_extract_thread_id`,依次查 `runtime.context["thread_id"]`、`runtime.config["configurable"]["thread_id"]`、LangGraph 的 `get_config()`,三层都拿不到兜底 `"default"`,见 [tools.py:291-306](../backend/packages/harness/deerflow/mcp/tools.py#L291-L306) 和 [tools.py:440-445](../backend/packages/harness/deerflow/mcp/tools.py#L440-L445)。隔离粒度与会话池的 key、文件系统的挂载树三者对齐,这是整个 MCP 安全模型的地基。

**链路解析**:
```
runtime ─► _extract_thread_id:  context["thread_id"] → config.configurable → get_config() → "default"
runtime ─► resolve_runtime_user_id: context["user_id"] → ContextVar → DEFAULT_USER_ID
   ▼
scope_key = f"{user_id}:{thread_id}"
   ▼
pool key = (server_name, scope_key)     # 与文件系统隔离粒度 (user_id, thread_id) 对齐
   ▼
同 user+thread 共享有状态会话; 跨 user 即使 thread_id 相撞也各自独立
```

## 面试官最爱追问的 3 个点

1. **"anyio cancel scope 同任务约束到底怎么回事?你们的 owner-task 模式所有关闭路径怎么保证不违反它?"**——应答策略:先一句话讲清约束(anyio 强制 cancel scope 由进入它的同一 task 退出,跨 task 抛 RuntimeError,issue #3379),再讲 owner-task 三段式(`__aenter__` → 发布 ready → 等 close_evt → finally 里自己 `__aexit__`),强调"所有关闭方只 set 事件或 cancel,从不亲自 exit",并能区分 `__aenter__` 失败(无 scope 可退,直接 set_exception 走人)与 `initialize()` 失败(必须 finally 退 scope)两条路径,引用 [session_pool.py:84-124](../backend/packages/harness/deerflow/mcp/session_pool.py#L84-L124);顺带说出 HTTP/SSE 不池化是同一个约束的另一种规避(issue #3203)。

2. **"并发首次创建/LRU 淘汰/close_all 三路并发时,怎么保证不重复建、不泄漏、不复活、不死锁?"**——应答策略:按 `get_session` 的 Phase 1-4 背:锁内原子三选一(返回/join/成为 creator)、in-flight 记录"先发布再 await"防 race、`asyncio.shield` 把调用方取消与 owner 解耦、Phase 4 用 `still_ours` 判定防止"池子中途被 close_all 清空后又把新建会话复活进 `_entries`"(此时自己 signal + await owner 退 scope,再抛 CancelledError),引用 [session_pool.py:248-261](../backend/packages/harness/deerflow/mcp/session_pool.py#L248-L261);再补三个数字:LRU 上限 **256**、异 loop 同步关闭超时 **5.0 秒**、in-flight 必须 cancel 因为 `close_evt` 唤不醒 `initialize()`。

3. **"mtime 失效 + 懒加载这套缓存,什么情况下会判错或初始化崩掉?为什么 reset 里不能等当前 loop 关闭完?"**——应答策略:主动说保守策略(任一 mtime 拿不到就当没过期,宁用旧缓存不误清)、`from_file()` 解决跨进程配置可见性(Gateway 独立进程写盘)、懒加载的三种 loop 姿态(running loop 走 ThreadPoolExecutor 新线程跑 `asyncio.run`、idle loop `run_until_complete`、无 loop `asyncio.run`)、以及 reset 对当前 running loop 只 signal 不等待的自死锁规避——"loop 只有在同步调用返回后才能跑 teardown",引用 [cache.py:102-127](../backend/packages/harness/deerflow/mcp/cache.py#L102-L127) 和 [cache.py:144-161](../backend/packages/harness/deerflow/mcp/cache.py#L144-L161)。
