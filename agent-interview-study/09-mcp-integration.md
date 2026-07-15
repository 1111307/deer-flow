# 第九部分：MCP 协议集成（Model Context Protocol）

第六部分讲过 sandbox 里的虚拟路径体系：agent 看到的是 `/mnt/user-data/...`，真实文件在 host 上按 `thread_id`（现在还要加上 `user_id`）隔离存放，两者之间靠 `LocalSandboxProvider` 的 `path_mappings` 做转换。MCP 这块要解决同一个问题，但起因完全不同——stdio 类型的 MCP server 是一个独立子进程，根本不在 sandbox 抽象之内，它自己把文件写到自己的 cwd 里，DeerFlow 必须在 MCP 这一层重新、手工地把"host 路径"翻译成"虚拟路径"。这一部分要看的六个文件里，`client.py`/`cache.py`/`oauth.py` 是相对简单的配置和缓存层，但 `session_pool.py` 和 `tools.py` 藏着这个仓库里数一数二精巧的并发控制代码——而且这两个文件里的复杂度，无论是 `learn.md` 现有的第 10 节，还是 `backend/CLAUDE.md` 的 "MCP System" 一节，都基本没有体现。

## 1. 六个文件的分工

```
mcp/
├── client.py        配置 → langchain-mcp-adapters 参数的翻译层
├── cache.py          懒加载 + mtime 缓存失效 + 同步/异步双路径入口
├── oauth.py          HTTP/SSE 的 OAuth token 生命周期管理
├── session_pool.py   stdio 持久会话池（anyio 同任务栈约束的解决方案）
├── tools.py           主装配：MultiServerMCPClient 封装 + 虚拟路径翻译 + 同步包装
└── __init__.py        只对外重新导出 6 个符号
```

先看 [`__init__.py`](../backend/packages/harness/deerflow/mcp/__init__.py)：

```python
from .cache import (
    get_cached_mcp_tools,
    initialize_mcp_tools,
    reset_mcp_tools_cache,
)
from .client import build_server_params, build_servers_config
from .tools import get_mcp_tools

__all__ = [
    "build_server_params",
    "build_servers_config",
    "get_mcp_tools",
    "initialize_mcp_tools",
    "get_cached_mcp_tools",
    "reset_mcp_tools_cache",
]
```

注意它**没有**重新导出 `session_pool.py` 或 `oauth.py` 里的任何东西——`MCPSessionPool`、`OAuthTokenManager`、`get_session_pool` 全部是模块内部实现细节，外部只应该通过 `get_mcp_tools()` 拿到已经包装好的工具列表。这是一个信号：这两个文件的复杂度是**故意**被封装起来、不暴露给调用方的，但也正因为不暴露，文档也就顺理成章地把它们漏掉了。

## 2. client.py：配置到 transport 参数的翻译层

[`build_server_params`](../backend/packages/harness/deerflow/mcp/client.py#L11-L42) 做的事情很直接：把 DeerFlow 自己的 `McpServerConfig` 翻译成 `langchain-mcp-adapters` 期望的字典格式，按 transport 类型校验必填字段：

```python
transport_type = config.type or "stdio"
params: dict[str, Any] = {"transport": transport_type}

if transport_type == "stdio":
    if not config.command:
        raise ValueError(f"MCP server '{server_name}' with stdio transport requires 'command' field")
    params["command"] = config.command
    params["args"] = config.args
    if config.env:
        params["env"] = config.env
elif transport_type in ("sse", "http"):
    if not config.url:
        raise ValueError(f"MCP server '{server_name}' with {transport_type} transport requires 'url' field")
    params["url"] = config.url
    if config.headers:
        params["headers"] = config.headers
else:
    raise ValueError(f"MCP server '{server_name}' has unsupported transport type: {transport_type}")
```

stdio 要 `command`，sse/http 要 `url`——缺了就直接 `raise ValueError`，不是静默跳过。真正"容错"的地方在上一层 [`build_servers_config`](../backend/packages/harness/deerflow/mcp/client.py#L45-L68)：它遍历所有启用的 server，每个都单独包一层 `try/except`，一个 server 配置写错了只会 `logger.error` 然后跳过，不会拖累其他 server 的加载。这是个很常见但值得说的模式：**校验要严格（该 raise 就 raise），但校验的调用点要宽容（一个失败不能拖垮全部）**——严格和宽容不矛盾，看你把 try/except 放在哪一层。

## 3. cache.py：懒加载、mtime 失效、三分支同步入口

[`get_cached_mcp_tools()`](../backend/packages/harness/deerflow/mcp/cache.py#L82-L129) 是 MCP 工具真正的入口——`tools/tools.py` 里的 `get_available_tools()` 调的就是它，而不是直接调 `get_mcp_tools()`。它要解决两个问题：

**懒加载**：第一次调用时才真正去连接 MCP server 拉取工具 schema，而不是应用启动时全量拉取。

**过期检测**：[`_is_cache_stale()`](../backend/packages/harness/deerflow/mcp/cache.py#L31-L53) 比较 `extensions_config.json` 的 mtime 和上次缓存时记录的 mtime，变了就认为缓存过期。里面有一处防御性判断值得注意：

```python
# If we couldn't get mtime before or now, assume not stale
if _config_mtime is None or current_mtime is None:
    return False
```

如果拿不到 mtime（文件被删了、或者上次压根没拿到），**默认判定为"不过期"**而不是"过期"——这是保守的一侧：宁可用旧缓存，也不要在文件系统抖动的时候反复触发重新初始化。

真正有意思的是 [`get_cached_mcp_tools()`](../backend/packages/harness/deerflow/mcp/cache.py#L102-L127) 里那段三分支调度逻辑，因为它要同时兼容"在 FastAPI 里跑（有运行中的事件循环）"和"在 LangGraph Studio / 同步脚本里跑（没有）"两种调用环境：

```python
try:
    loop = asyncio.get_event_loop()
    if loop.is_running():
        # 已经有循环在跑（比如 LangGraph Studio），
        # 只能另开一个线程去跑一个全新的循环
        import concurrent.futures
        with concurrent.futures.ThreadPoolExecutor() as executor:
            future = executor.submit(asyncio.run, initialize_mcp_tools())
            future.result()
    else:
        loop.run_until_complete(initialize_mcp_tools())
except RuntimeError:
    # 压根没有事件循环，asyncio.get_event_loop() 在非主线程里会抛这个
    asyncio.run(initialize_mcp_tools())
```

这是一个同步函数，却要去驱动一个 `async def initialize_mcp_tools()`。三条路径分别应对：循环存在且正在跑（不能直接 `run_until_complete`，会跟当前循环打架，必须换个线程另起一个循环）、循环存在但没在跑（可以直接借用）、压根没有循环（`asyncio.run` 新建一个）。**这个"探测当前有没有正在运行的事件循环，有就换线程"的模式，在这个模块里还会重复出现两次**（第 12 节展开讲）。

最后，[`reset_mcp_tools_cache()`](../backend/packages/harness/deerflow/mcp/cache.py#L132-L166) 不只是清空缓存字典，还耦合了会话池的清理：

```python
try:
    from deerflow.mcp.session_pool import get_session_pool
    get_session_pool().close_all_sync()
except Exception:
    logger.debug("Could not close MCP session pool on cache reset", exc_info=True)

from deerflow.mcp.session_pool import reset_session_pool
reset_session_pool()
```

为什么缓存重置要牵连会话池？因为如果 `extensions_config.json` 改了（比如某个 stdio server 的 `command` 变了），旧的持久 session 引用的是**旧的连接配置**，留着就是脏数据，必须连同工具缓存一起作废，下次 `get_mcp_tools()` 才会用新配置重新建连接、重新建 session。这一处耦合，也是本模块唯一一个"缓存层主动触碰会话池"的地方——理解这行代码需要先理解第 5-7 节的 `session_pool.py`，所以先往下看。

## 4. oauth.py：双重检查锁 + 主动续期

[`OAuthTokenManager.get_authorization_header`](../backend/packages/harness/deerflow/mcp/oauth.py#L47-L65) 是标准的双重检查锁（double-checked locking）：

```python
token = self._tokens.get(server_name)
if token and not self._is_expiring(token, oauth):
    return f"{token.token_type} {token.access_token}"          # 锁外第一次检查

lock = self._locks[server_name]
async with lock:
    token = self._tokens.get(server_name)
    if token and not self._is_expiring(token, oauth):
        return f"{token.token_type} {token.access_token}"      # 锁内第二次检查

    fresh = await self._fetch_token(oauth)                      # 真正发起刷新
    self._tokens[server_name] = fresh
    return f"{fresh.token_type} {fresh.access_token}"
```

锁外先查一次是为了让"token 还新鲜"这个最常见的路径完全不用等锁；锁内再查一次，是因为在你等锁的那段时间里，可能已经有另一个并发调用刷新过了——不重复检查的话，两个几乎同时到达的请求会都以为自己需要刷新，发出两次没必要的 token 请求。每个 server 一把独立的 `asyncio.Lock`（[`__init__`](../backend/packages/harness/deerflow/mcp/oauth.py#L28-L31)），server A 刷新不会挡住 server B。

[`_is_expiring`](../backend/packages/harness/deerflow/mcp/oauth.py#L67-L70) 判断"是否需要刷新"用的不是字面过期时间，而是提前一个 `refresh_skew_seconds`（默认 60 秒）：

```python
return token.expires_at <= now + timedelta(seconds=max(oauth.refresh_skew_seconds, 0))
```

这是主动续期而不是被动等过期——避免"token 恰好在请求发出的路上过期"这种边界竞态。[`_fetch_token`](../backend/packages/harness/deerflow/mcp/oauth.py#L72-L119) 处理两种 grant type（`client_credentials`/`refresh_token`），[`build_oauth_tool_interceptor`](../backend/packages/harness/deerflow/mcp/oauth.py#L122-L137) 把这套 token 管理包装成一个**责任链拦截器**：

```python
async def oauth_interceptor(request: Any, handler: Any) -> Any:
    header = await token_manager.get_authorization_header(request.server_name)
    if not header:
        return await handler(request)
    updated_headers = dict(request.headers or {})
    updated_headers["Authorization"] = header
    return await handler(request.override(headers=updated_headers))
```

`(request, handler) -> Any` 这个签名——拿到请求，可选地改写它，再调用 `handler` 把它传给链条里的下一环——和 DeerFlow 别处的 `AgentMiddleware` 责任链是同一套设计语言。第 11 节会看到 `tools.py` 里怎么用同样的接口手工拼出一条拦截器链。

## 5. session_pool.py（上）：为什么需要一个"owner task"

这是整个 MCP 模块里最值得讲的一段。模块开头的 docstring 直接点出了问题：

> MCP 的 `ClientSession` 底层建立在 anyio 的 task group 上，而 anyio 强制要求：一个 cancel scope 必须由**进入它的那个 task** 亲自退出。如果换一个 task 去调用 `cm.__aexit__`，会抛出：
> `RuntimeError: Attempted to exit cancel scope in a different task than it was entered in`（[anyio issue #3379](../backend/packages/harness/deerflow/mcp/session_pool.py#L1-L32)）

问题出在哪？[`tools/sync.py`](../backend/packages/harness/deerflow/tools/sync.py) 里的同步工具包装器每次调用都跑一个全新的 `asyncio.run()`——也就是全新的 event loop、全新的 task。如果 session 是在第一次调用时 `__aenter__` 进入的，第二次调用又想复用这同一个 session、并在某个时刻 `__aexit__` 退出它，那这个退出动作就发生在**另一个** `asyncio.run()` 开的另一个 task 里——直接触发上面那条 `RuntimeError`。

解决方案是"owner task"模式（[`_run_session`](../backend/packages/harness/deerflow/mcp/session_pool.py#L84-L124)）：session 的整个生命周期——进入、初始化、使用、退出——全部固定在**同一个专属 task** 里跑，其它任何想关闭这个 session 的代码，都只能**发信号**，不能自己动手 `__aexit__`：

```python
async def _run_session(self, connection, ready, close_evt) -> None:
    from langchain_mcp_adapters.sessions import create_session
    cm = create_session(connection)
    try:
        session = await cm.__aenter__()
    except BaseException as e:
        if not ready.done():
            ready.set_exception(e)
        return   # 从未进入 cancel scope，没什么需要退出的

    try:
        await session.initialize()
        if not ready.done():
            ready.set_result(session)     # 把活着的 session 交给等待者
        await close_evt.wait()            # 挂起，直到被要求关闭
    except BaseException as e:
        if not ready.done():
            ready.set_exception(e)
    finally:
        try:
            await cm.__aexit__(None, None, None)   # 永远由这个 task 自己退出
        except Exception:
            logger.warning("Error closing MCP session", exc_info=True)
```

`ready`（一个 `asyncio.Future`）是这个 task 和外部世界之间唯一的数据出口；`close_evt`（一个 `asyncio.Event`）是外部世界关闭它的唯一入口。无论 `initialize()` 失败、被取消，还是正常收到关闭信号，`finally` 块都保证 `__aexit__` 被这个 task 自己调用——这就是为什么整个类要设计成"每个 session 配一个专属的常驻 task"，而不是让调用方直接持有裸的 `ClientSession` 对象。

## 6. session_pool.py（中）：get_session 的四阶段状态机

[`get_session`](../backend/packages/harness/deerflow/mcp/session_pool.py#L126-L263) 要同时应对三件事：复用已有 session、避免并发创建重复 session、LRU 容量淘汰。它把整个决策拆成四个阶段。

**阶段一：锁内原子决策，零 await**（[L160-195](../backend/packages/harness/deerflow/mcp/session_pool.py#L160-L195)）。用的是 `threading.Lock` 而不是 `asyncio.Lock`——因为这把锁既要在异步路径里用，也要在没有运行中事件循环的同步路径（`close_all_sync`）里用，`asyncio.Lock` 做不到这点。锁内只做字典读写，不做任何 `await`，所以持锁时间极短：

```python
with self._lock:
    if key in self._entries:
        session, loop, ent_task, ent_close = self._entries[key]
        if loop is current_loop and not loop.is_closed():
            self._entries.move_to_end(key)      # LRU：命中即刷新到队尾
            return session
        # 属于别的/已关闭的事件循环——淘汰
        self._entries.pop(key)
        evicted.append((loop, ent_task, ent_close, False))

    inflight = self._inflight.get(key)
    if inflight is not None and inflight[0] is current_loop and not inflight[0].is_closed():
        join = inflight[1]                       # 已有人在建，等他的结果
    else:
        if inflight is not None:
            self._inflight.pop(key)              # 别的循环留下的残留创建，淘汰
            evicted.append((inflight[0], inflight[2], inflight[3], True))
        ready = current_loop.create_future()
        close_evt = asyncio.Event()
        task = current_loop.create_task(self._run_session(connection, ready, close_evt))
        self._inflight[key] = (current_loop, ready, task, close_evt)

    while len(self._entries) >= self.MAX_SESSIONS:   # LRU 容量淘汰
        oldest_key, (_, loop, ent_task, ent_close) = next(iter(self._entries.items()))
        self._entries.pop(oldest_key)
        evicted.append((loop, ent_task, ent_close, False))
```

三种结果二选一——`return session`（复用）、`join = ...`（等别人）、成为创建者——外加顺手做 LRU 淘汰。`OrderedDict.move_to_end` 是标准的 LRU 实现手法：命中就挪到队尾，淘汰永远从队头拿。

**阶段二**（[L197-208](../backend/packages/harness/deerflow/mcp/session_pool.py#L197-L208)）在锁外处理阶段一收集到的 `evicted` 列表：跟当前 task 同一个循环的，直接 `await self._shutdown(...)` 等它关完；不同循环但是残留的 in-flight 创建（可能卡在 `initialize()` 里，`close_evt` 唤不醒它），必须 `cancel=True` 地路由到它自己的循环去关；普通的、别的循环上的已注册 session，只是**发个信号就不等了**（`_signal_close`）——不阻塞自己去等一个跟当前请求无关的旧 session 收尾。

**阶段三**（[L215-246](../backend/packages/harness/deerflow/mcp/session_pool.py#L215-L246)）等自己创建的 owner task 把 session 准备好：

```python
try:
    session = await asyncio.shield(ready)
except BaseException:
    owner_already_failed = ready.done() and not ready.cancelled() and ready.exception() is not None
    if not owner_already_failed:
        close_evt.set()
        task.cancel()
    await asyncio.shield(task)
    ...
    raise
```

用 `asyncio.shield` 包一层，是因为**这次 `get_session` 调用本身被取消**，不应该连带取消共享的 owner task（万一还有别的并发调用者在等同一个 `ready`）。`except` 分支要分辨两种情况：owner 自己失败了（此时它已经在自己的 `finally` 里跑 `__aexit__`，绝不能再 cancel 它，只能等它收尾）；还是这次调用自己被取消了（此时 `ready` 还挂着、owner 还活着，需要主动 `close_evt.set()` + `task.cancel()` 去唤醒它退出）。

**阶段四**（[L248-263](../backend/packages/harness/deerflow/mcp/session_pool.py#L248-L263)）把 in-flight 记录"晋升"为正式注册的 entry——但要再检查一次自己的 in-flight 记录是否还是"活的"：

```python
with self._lock:
    still_ours = self._inflight.get(key) == (current_loop, ready, task, close_evt)
    if still_ours:
        self._inflight.pop(key)
        self._entries[key] = (session, current_loop, task, close_evt)
if not still_ours:
    await self._shutdown(close_evt, task)
    raise asyncio.CancelledError("MCP session pool was closed while the session was being created")
```

为什么还要再检查？因为在"等 `ready`"这段时间里（`initialize()` 可能要走网络/子进程握手，不是瞬间完成），可能有另一个 `close_scope`/`close_all` 已经把这条 in-flight 记录摘掉了。如果这时候不检查、直接把 session 塞进 `_entries`，就是把一个"本该已经被关闭"的 session 复活。检查失败就自己把刚创建好的 session 关掉，然后以 `CancelledError` 的形式让调用方知道：你等的这个东西，其实已经被别人判了死刑。

## 7. session_pool.py（下）：四级关闭与"别在自己的循环里等自己"

关闭粒度由粗到细分四层：[`close_scope`](../backend/packages/harness/deerflow/mcp/session_pool.py#L342-L352)（按 `scope_key`，即按线程关）、[`close_server`](../backend/packages/harness/deerflow/mcp/session_pool.py#L354-L364)（按 server 名关）、[`close_all`](../backend/packages/harness/deerflow/mcp/session_pool.py#L366-L376)（异步、关全部）、[`close_all_sync`](../backend/packages/harness/deerflow/mcp/session_pool.py#L378-L431)（同步、关全部，供 `cache.py` 在没有事件循环保证的地方调用）。

最后这个同步版本是全文件里逻辑最微妙的地方，因为它要在**不知道自己被从哪个线程调用**的前提下，正确关闭**分布在不同事件循环上**的 session：

```python
for loop, task, close_evt, cancel in owners:
    if loop.is_closed():
        continue
    if loop is current_running_loop:
        # 我们正在这个循环自己的线程里执行——同步等它跑完等于自己等自己，
        # 死锁到 timeout。只发信号，让它在这次同步调用返回、
        # 循环重新拿回控制权之后自己去跑收尾。
        close_evt.set()
        if cancel:
            task.cancel()
    elif loop.is_running():
        # 目标循环在别的线程上正在跑，可以安全地跨线程调度过去、同步等结果
        future = asyncio.run_coroutine_threadsafe(self._shutdown(close_evt, task, cancel), loop)
        future.result(timeout=self.SESSION_CLOSE_TIMEOUT)
    else:
        # 目标循环闲置着（没在跑，但也没关闭）——可以直接借来跑
        loop.run_until_complete(self._shutdown(close_evt, task, cancel))
```

三分支对应三种"目标循环处于什么状态"：**是我自己正在跑的那个循环**（同步等待=自己等自己=死锁，只能发信号不等）、**是别的线程正在跑的循环**（`run_coroutine_threadsafe` 跨线程调度，可以带 `timeout` 同步等结果）、**闲置但没关闭的循环**（`run_until_complete` 直接借用）。这跟第 3 节 `cache.py` 的三分支表面相似，但判断依据完全不同——`cache.py` 问的是"当前有没有循环在跑"，这里问的是"**每一个 session 自己的 owner 循环**跟当前调用者的循环是什么关系"，是逐条 entry 判断，不是一次性判断。

第一分支"只发信号不等"隐含一个使用约束，代码注释里也明确写了：**调用方必须保证那个循环之后还会继续跑**，否则 owner task 永远没有机会执行 `close_evt.wait()` 之后的收尾代码，`__aexit__` 就无法运行——如果需要确定性的关闭且当前正处于那个循环内部，应该用 `await close_all()` 而不是 `close_all_sync()`。这是"同步接口套在异步生命周期上"典型会踩的坑：同步函数没有办法阻塞等待"当前正在执行自己的这段代码"的循环，因为循环此刻正忙着执行你，不可能同时去执行别的任务。

## 8. tools.py（上）：get_mcp_tools 主装配

[`get_mcp_tools()`](../backend/packages/harness/deerflow/mcp/tools.py#L541-L653) 是把前面所有零件拧在一起的地方，主线：

1. `ExtensionsConfig.from_file()`——注意不是缓存版 `get_extensions_config()`，是每次都从磁盘重新读，为的是让 Gateway API 保存的配置改动立刻生效（跟 `tools/tools.py:114-117` 那条注释是同一个理由）。
2. `get_initial_oauth_headers()`——只给 sse/http server 的 `headers` 注入初始 `Authorization`（stdio 没有 HTTP headers 这个概念）,这是给"工具发现/建立初始连接"这一步用的静态 header。
3. 组装 `tool_interceptors` 列表：OAuth 拦截器（如果有 OAuth server）在前，再加上从 `extensions_config.json` 的 `mcpInterceptors` 字段动态加载的自定义拦截器（通过 `resolve_variable` 反射加载）。
4. `MultiServerMCPClient(servers_config, tool_interceptors=tool_interceptors, tool_name_prefix=True)`——`tool_name_prefix=True` 让每个工具名自动加上 `{server_name}_` 前缀,避免不同 server 出现同名工具时互相冲突。
5. `client.get_tools()`——这一步只是**发现**工具 schema,内部会临时开连接握手,跟第 5-7 节讲的持久化会话池是两件事。
6. 按 transport 类型分流包装（[L623-642](../backend/packages/harness/deerflow/mcp/tools.py#L623-L642)）：

```python
# 只池化 stdio session。HTTP/SSE transport 内部同样用 anyio TaskGroup,
# 从不同的 async task 去关闭会触发 cleanup 时的 RuntimeError（见 #3203）。
for tool in tools:
    tool_server = None
    for name in servers_config:
        if tool.name.startswith(f"{name}_"):
            tool_server = name
            break
    if tool_server is not None:
        transport = servers_config[tool_server].get("transport", "stdio")
        if transport == "stdio":
            wrapped_tools.append(_make_session_pool_tool(tool, tool_server, servers_config[tool_server], tool_interceptors))
        else:
            wrapped_tools.append(tool)   # HTTP/SSE 不包装,原样返回
    else:
        wrapped_tools.append(tool)
```

这里有个容易忽略但很重要的点：**HTTP/SSE 工具不进会话池，不是因为它们不需要状态复用，而是因为同一个 anyio 同任务栈约束在 HTTP/SSE 传输上也存在**（[issue #3203](../backend/packages/harness/deerflow/mcp/tools.py#L624-L626)）——按第 5 节的逻辑，理论上也可以给它们做一套 owner-task 池化。但 stdio 会话值得池化的理由（本地子进程重启开销大、Playwright 之类要保留浏览器状态）在 HTTP/SSE 上基本不成立：远程服务器本来就是无状态或自己管状态,少了池化最多是"每次调用重新握手"的开销,而不是"丢失有状态的浏览器/进程"。**同一个底层技术约束（anyio 同任务栈退出），两种 transport 给出了不同的最优解**——stdio 花力气解决它换来状态保留的好处;HTTP/SSE 判断这个好处不值得换,干脆不做,直接受益于"不做也没什么损失"。这是一个很好的"权衡而非教条"的例子。

## 9. tools.py（中）：stdio 子进程的工作区钉住

stdio server 是本地子进程，写文件、读临时目录都是相对于自己的 cwd/`TMPDIR` 进行的。[`_prepare_stdio_workspace`](../backend/packages/harness/deerflow/mcp/tools.py#L153-L170) 把这些同步文件系统操作打包成一个函数，好让调用方用 `asyncio.to_thread` 整体丢到线程池，不卡事件循环：

```python
def _prepare_stdio_workspace(paths, *, thread_id, user_id):
    paths.ensure_thread_dirs(thread_id, user_id=user_id)
    source_base_dir = paths.sandbox_work_dir(thread_id, user_id=user_id)
    tmp_dir = source_base_dir / _MCP_TMP_SUBDIR   # ".mcp/tmp"
    tmp_dir.mkdir(parents=True, exist_ok=True)
    tmp_dir.chmod(0o700)
    before_files = _snapshot_workspace_files(source_base_dir)
    return source_base_dir, tmp_dir, before_files
```

`ensure_thread_dirs` 建的 workspace/uploads/outputs 目录权限是 `0o777`（第六部分讲过，因为可能被 Docker sandbox 用不同 UID 挂载读写），但这里 `.mcp/tmp` 只给 `0o700`——原因是这个临时目录只被本地直接拉起的 MCP 子进程使用，这个子进程和 DeerFlow 后端进程本身是同一个 UID，不存在跨容器 UID 不一致的问题，收紧权限不会破坏可写性，反而减少了同一台机器上其它进程/用户读到临时文件的面。

在调用点（[`_make_session_pool_tool`](../backend/packages/harness/deerflow/mcp/tools.py#L455-L478)），cwd 和三个临时目录环境变量的钉法是这样的：

```python
configured_cwd = session_connection.get("cwd", str(source_base_dir))
session_connection["cwd"] = str(configured_cwd)
...
session_env = dict(session_connection.get("env") or {})
session_env.setdefault("TMPDIR", str(tmp_dir))
session_env.setdefault("TMP", str(tmp_dir))
session_env.setdefault("TEMP", str(tmp_dir))
session_connection["env"] = session_env
```

`cwd` 用 `.get(key, default)`——如果 operator 在配置里已经写了 `cwd`，这里读到的就是配置值，原样写回；只有配置没写时才落到 `source_base_dir`。三个临时目录变量用 `setdefault`——同理，operator 显式配置的环境变量永远优先。这跟第 4 节 OAuth 的"锁外先查一次"是同一种编程习惯的不同体现：**永远给"外部已经决定好的值"让路，只在没人决定的时候才用默认值补位**。钉住 cwd/TMPDIR 的目的很直接：让 Node 的 `os.tmpdir()`、Python 的 `tempfile`、各种 CLI 默认写文件的位置都落在挂载的 user-data 树里，而不是一个 sandbox/artifact API 完全找不到的 host 临时路径。

## 10. tools.py（下）：虚拟路径翻译的两层匹配

stdio 子进程的 cwd/temp 已经钉在 user-data 树里了，所以它产出的文件天然就在一个"可服务"的位置——唯一缺的是虚拟路径前缀。核心翻译函数是 [`_local_uri_to_virtual_path`](../backend/packages/harness/deerflow/mcp/tools.py#L77-L124)：

```python
def _local_uri_to_virtual_path(uri, *, thread_id, user_id, source_base_dir=None):
    src = _local_path_from_uri(uri, base_dir=source_base_dir)
    if src is None:
        return None
    real = src.resolve()
    if not real.is_file():
        return None
    user_data_root = get_paths().sandbox_user_data_dir(thread_id, user_id=user_id).resolve()
    try:
        relative = real.relative_to(user_data_root)
    except ValueError:
        # 文件在这个线程的 user-data 挂载树之外——没法表达成虚拟路径，原样保留
        return None
    return f"{VIRTUAL_PATH_PREFIX}/{relative.as_posix()}"
```

`Path.relative_to()` 在这里既是路径计算,也是安全校验:如果 `real` 不在 `user_data_root` 之内,它会抛 `ValueError`,被 `except` 捕获后返回 `None`(原样保留、不翻译)。这是"translate,don't copy"的安全边界——**只有确认这个文件确实躺在当前线程自己的 user-data 树里,才会给它生成一个虚拟路径**;不会把任何"文件确实存在但在别的地方"的路径错误地包装成一个看起来合法的虚拟路径。

但 MCP server 上报文件的方式不总是结构化的。结构化的 `ResourceLink` 直接给出 URI，一次 `_local_uri_to_virtual_path` 调用就能解决。麻烦的是像 Playwright 的 `browser_take_screenshot` 那样，只在自由文本里说"存到了 `temp/page.yml`"——这种情况分两层处理：

**第一层**，正则扫描（[`_LOCAL_PATH_IN_TEXT_RE`](../backend/packages/harness/deerflow/mcp/tools.py#L42)）找出文本里所有"长得像路径"的 token（绝对路径、`file://` URI、带斜杠的相对路径），每个候选都单独喂给 `_local_uri_to_virtual_path` 校验：

```python
_LOCAL_PATH_IN_TEXT_RE = re.compile(r"(?:file://)?/[^\s'\"<>|*?]+|(?:\.{0,2}/|[\w.-]+/)[^\s'\"<>|*?]+")
```

匹配不到真实文件的候选，直接原样保留——正则宽松地"过度匹配"是安全的，因为后面的校验会挡掉假阳性。

**第二层**专门补第一层的盲区：像 `page-2026.yml` 这种**不带任何斜杠**的裸文件名，正则天然匹配不到（正则的两个分支都要求至少有一个 `/`）。[`_rewrite_unique_bare_filenames`](../backend/packages/harness/deerflow/mcp/tools.py#L193-L237) 换一种思路——不靠文本模式，靠**这次调用前后 workspace 文件快照的 diff**：

```python
candidates: dict[str, list[str]] = {}
for path in changed_files:                      # 本次调用期间新建/改动过的文件
    virtual_path = _local_uri_to_virtual_path(str(path), thread_id=thread_id, user_id=user_id, ...)
    if virtual_path is None:
        continue
    candidates.setdefault(path.name, []).append(virtual_path)

unique = {name: paths[0] for name, paths in candidates.items() if len(set(paths)) == 1}
```

只有当某个文件名在"这次调用改动过的文件"里**恰好对应唯一一条虚拟路径**时才重写；如果同名文件有歧义（`len(set(paths)) != 1`），宁可什么都不做，也不猜。这个"前后快照 diff"能力靠 [`_snapshot_workspace_files`/`_changed_workspace_files`](../backend/packages/harness/deerflow/mcp/tools.py#L127-L150) 提供，只在结果确实含有文本内容时才触发（[`_result_has_text_content`](../backend/packages/harness/deerflow/mcp/tools.py#L173-L190)），避免每次工具调用都白白多走一次全量目录遍历。

两层加起来的设计哲学是一致的：**宁可漏翻译，不能错翻译**。过度匹配没关系，因为后面总有一步"确认文件真实存在于自己的树里"做最终把关；唯一不能接受的是把一个不确定的引用强行映射到一个可能是错的虚拟路径上。

## 11. 把一切粘起来：_convert_call_tool_result 与 _make_session_pool_tool

[`_convert_call_tool_result`](../backend/packages/harness/deerflow/mcp/tools.py#L309-L409) 是 MCP SDK 的 `CallToolResult`（`TextContent`/`ImageContent`/`ResourceLink`/`EmbeddedResource`）到 LangChain `content_and_artifact` 格式的手写转换。docstring 里说得很直接：这**没有**复用 `langchain_mcp_adapters.tools._convert_call_tool_result`——一个下划线开头的私有符号——因为 DeerFlow 需要在转换过程中间插入自己的 `_resolve_text`/`_resolve_link_url` 路径翻译钩子，而私有符号没法从外部注入钩子，只能自己整个重写一遍。

[`_make_session_pool_tool`](../backend/packages/harness/deerflow/mcp/tools.py#L412-L538) 是最终把会话池、拦截器、路径翻译全部串起来的地方。安全上最值得注意的一行：

```python
scope_key = f"{user_id}:{thread_id}"
```

注释写得很明确：文件系统隔离是按 `(user_id, thread_id)` 二元组做的,如果 session 只按 `thread_id` 隔离,**两个不同用户如果凑巧撞出同一个 thread_id,就会共享同一个有状态的 MCP session**——这是一个跨用户数据泄露的风险点,而不只是"命名冲突"那么简单。这跟第 8 部分讲的记忆系统按 `user_id` 分桶存储是同一层安全考量的不同体现。

拦截器链的构建（[L495-503](../backend/packages/harness/deerflow/mcp/tools.py#L495-L503)）：

```python
handler = base_handler
for interceptor in reversed(tool_interceptors):
    outer = handler
    async def wrapped(req: Any, _i: Any = interceptor, _h: Any = outer) -> Any:
        return await _i(req, _h)
    handler = wrapped
```

`reversed(tool_interceptors)` 是关键——如果拦截器列表是 `[A, B]`（A 先配置），倒序遍历先包 B、再包 A，最终 `handler = wrapped_A(wrapped_B(base_handler))`，调用时 A 最先看到请求，符合"先配置的先执行"的直觉。这里还有一处值得单独指出的 Python 细节：`wrapped` 用 `_i: Any = interceptor, _h: Any = outer` 把 `interceptor`/`outer` 通过**默认参数**捕进闭包，而不是直接在函数体里引用循环变量 `interceptor`/`outer`。如果去掉这两个默认参数、直接写 `return await interceptor(req, outer)`，所有循环迭代产生的 `wrapped` 闭包会共享同一个自由变量单元格——等真正调用任何一个 `wrapped` 的时候，循环早就跑完了，`interceptor`/`outer` 已经是循环结束后的**最后一次**取值，链条里除了最后一环全部失效。这是 Python 里"循环体内定义闭包"的经典陷阱，用默认参数在定义时就把当前值拷贝一份是标准解法。

另外一个不显眼但值得点出的设计：`tool_interceptors` 在 `get_mcp_tools()` 里被**传了两次**——一次是构造 `MultiServerMCPClient(..., tool_interceptors=tool_interceptors)` 时,一次是传给 `_make_session_pool_tool(tool, tool_server, servers_config[tool_server], tool_interceptors)`。这不是重复代码:stdio 工具被 `_make_session_pool_tool` 整个替换了调用路径(用池化 session 手动 `call_tool`),库自带的拦截器应用逻辑被绕过了,所以必须在包装器内部重新手工搭一条一模一样的链;而 HTTP/SSE 工具没被替换,走的是库自己内部应用拦截器的路径,只需要在构造 `MultiServerMCPClient` 时传一次就够。**同一份拦截器配置,喂给两条不同的调用路径,因为其中一条路径的默认拦截器应用逻辑被手动接管了。**

## 12. 三处"检测运行中事件循环"的模式——为什么不抽成一个公共函数

这个模块里，"判断当前是否有事件循环在跑，有就换个线程处理"这个动作独立出现了三次：

| 位置 | 触发条件 | 做法 |
|---|---|---|
| [`cache.py:get_cached_mcp_tools`](../backend/packages/harness/deerflow/mcp/cache.py#L104-L127) | 循环存在且在跑 | 开线程池 `executor.submit(asyncio.run, ...)`；循环存在不在跑就 `run_until_complete`；压根没循环就 `asyncio.run` |
| [`session_pool.py:close_all_sync`](../backend/packages/harness/deerflow/mcp/session_pool.py#L411-L429) | 是不是**当前**这个正在跑的循环 | 是自己就只发信号不等；是别的线程的就 `run_coroutine_threadsafe(...).result(timeout=...)`；闲置的就 `run_until_complete` |
| [`tools/sync.py:make_sync_tool_wrapper`](../backend/packages/harness/deerflow/tools/sync.py#L64-L78) | 循环存在且在跑 | `contextvars.copy_context()` + 共享线程池执行 `asyncio.run`；否则直接 `asyncio.run` |

三处解决的是同一个约束——**一个线程只能有一个正在跑的事件循环，你不能在一个已经在跑的循环里同步阻塞等待另一个协程**——但没有抽成一个共享函数，原因是它们各自"跑完之后要不要等结果""跑完之后要不要清理状态"的语义不一样：`cache.py` 需要拿到初始化结果（工具列表），必须等；`close_all_sync` 分三种情况有的等有的不等，取决于会不会自死锁；`sync.py` 除了 offload 执行，还要用 `contextvars.copy_context()` 保证被调用的协程能看到发起调用时的 context（这是第 8 部分提到的 ContextVar 跨线程传播问题在这里的第三次出现，解法比记忆系统"入队时提取成普通值"更直接——直接把整个 context 拷贝过去）。

这是一个很实在的架构判断：**同一个约束不代表要用同一个抽象**。硬把这三处收敛成一个共享 helper，为了照顾"要不要等结果""要不要拷贝 context"这些细微差异，那个共享函数的参数列表会迅速膨胀成一堆布尔开关，可读性反而更差。分开写三次，每次都是这个具体调用点真正需要的最小逻辑，反而更清楚。

## 13. extensions_config.py：配置面的两个细节

[`McpServerConfig._accept_transport_alias`](../backend/packages/harness/deerflow/config/extensions_config.py#L50-L66) 是个 `model_validator(mode="before")`：

```python
@model_validator(mode="before")
@classmethod
def _accept_transport_alias(cls, data: Any) -> Any:
    if isinstance(data, dict):
        transport = data.get("transport")
        if transport and not data.get("type"):
            data = {**data, "type": transport}
    return data
```

MCP 官方配置规范用 `transport` 字段表示传输方式，DeerFlow 内部历史上一直用自己的 `type` 字段。这个 validator 在 pydantic 校验**之前**跑，把 `transport` 悄悄映射成 `type`（`type` 已存在时优先用 `type`），让两种写法都能被接受——不用强迫所有已有配置迁移到新字段名，也能兼容照抄官方文档写 `transport` 的新用户。

[`resolve_env_variables`](../backend/packages/harness/deerflow/config/extensions_config.py#L169-L201) 递归地把配置里所有 `"$VAR"` 形式的字符串值换成 `os.getenv("VAR")` 的结果：

```python
if isinstance(config, str):
    if not config.startswith("$"):
        return config
    env_value = os.getenv(config[1:])
    if env_value is None:
        return ""     # 而不是原样返回 "$VAR"
    return env_value
```

关键在于环境变量不存在时返回空字符串，而不是把 `"$VAR"` 这个字面占位符原样传下去——如果放任 `"$VAR"` 流到下游（比如塞进某个 MCP server 的 `headers`），日志或错误信息里可能会把这串看起来像密钥占位符的文本原样打印出来，看起来像是泄露了一个值、实际上只是一个没解析成功的占位符，容易误导排查方向；返回空字符串至少保证下游拿到的是一个"确定为空"的值。

## 14. 文档纠偏：两份文档都漏掉了整整两个文件的复杂度

`learn.md` 现有的第 10 节只列出三个"核心文件"：

> **文件**：`mcp/client.py`、`mcp/cache.py`、`mcp/oauth.py`

`session_pool.py`（456 行）和 `tools.py`（654 行）——两个文件加起来比前三个的总和还长好几倍——完全没有出现在文件列表里，对应的机制（anyio 同任务栈约束、owner-task 模式、LRU 淘汰、in-flight 创建去重、四级关闭生命周期、两层虚拟路径翻译）自然也就一句没提。

`backend/CLAUDE.md` 的 "MCP System" 一节比 `learn.md` 好一些——它提到了 "Persistent stdio sessions are scoped by `user_id:thread_id`"、提到了 "MCP-returned local file references are not copied... translated deterministically to `/mnt/user-data/...`"，说明这两个结论性事实是写文档的人确实读过、确实知道的。但连接这两个结论性事实和实际代码之间的，恰恰是这个模块里工程含量最高的部分——**为什么需要 owner task、并发创建怎么去重、关闭时怎么避免自死锁**——在 CLAUDE.md 里同样一个字没有。

这跟第 7、8 部分遇到的情况不太一样：第 7 部分是文档里的中间件装配顺序跟代码实际不符（具体的事实错误），第 8 部分是"top 15 facts"这个说法本身就是编的（虚构的具体数字）。这里没有任何一句话是错的——`learn.md`/`CLAUDE.md` 写的每一条都能在代码里找到对应——只是**深度上系统性地浅**：两份文档共同证明了一件事，即"MCP 集成"这个标题下面，实际藏着这个仓库里数一数二复杂的并发控制代码，而这份复杂度目前完全只存在于源码本身，没有被沉淀成任何一句文档。这也是这次学习本身最大的收获之一：**判断一份文档是否可信，不能只看它写的话对不对，还要看它有没有把真正难的那部分写出来。**

## 15. 小结：一次 MCP 工具调用的全链路

```
应用启动 / 首次调用
  └─ get_cached_mcp_tools()                              [cache.py]
       ├─ 缓存新鲜？──是──▶ 直接返回 _mcp_tools_cache
       └─ 否 → 三分支同步调度 → initialize_mcp_tools()
                                    └─ get_mcp_tools()      [tools.py]
                                         ├─ ExtensionsConfig.from_file()（每次读最新）
                                         ├─ build_servers_config()          [client.py]
                                         ├─ get_initial_oauth_headers()     [oauth.py]
                                         ├─ 组装 tool_interceptors（OAuth + 自定义）
                                         ├─ MultiServerMCPClient(...).get_tools()
                                         └─ 按 transport 分流：
                                              ├─ stdio → _make_session_pool_tool(...)
                                              └─ sse/http → 原样返回（不池化，#3203）
                                         └─ make_sync_tool_wrapper 打同步补丁 [tools/sync.py]

一次工具调用（stdio 场景）
  call_with_persistent_session(runtime, **args)
       ├─ _extract_thread_id / resolve_runtime_user_id
       ├─ scope_key = f"{user_id}:{thread_id}"            ← 跨用户隔离
       ├─ _prepare_stdio_workspace（cwd/TMPDIR 钉住 + 前置快照）
       ├─ pool.get_session(server, scope_key, connection)  [session_pool.py]
       │     ├─ 命中已有 entry（同循环）→ 直接复用
       │     ├─ 命中 in-flight 创建（同循环）→ asyncio.shield(join)
       │     └─ 都没有 → _run_session 起一个 owner task，同任务栈进/出
       ├─ 拦截器链（reversed 构建）→ session.call_tool(...)
       ├─ 有文本内容？→ 计算 changed_files（后置快照 diff）
       └─ _convert_call_tool_result（ResourceLink 精确翻译 + 自由文本两层匹配）
                                                             [虚拟路径 → 第 6 部分 sandbox]

应用关闭 / 配置变更
  reset_mcp_tools_cache() → get_session_pool().close_all_sync()
       ├─ 是当前正在跑的循环 → 只发信号，不等（避免自死锁）
       ├─ 是别的线程在跑的循环 → run_coroutine_threadsafe(...).result(timeout=5s)
       └─ 闲置的循环 → run_until_complete(...)
```

从"agent 看到一个叫 `playwright_browser_take_screenshot` 的工具"到"这个工具背后是一个专属 task 常驻管理的子进程 session、返回的截图路径被安全翻译成 `/mnt/user-data/outputs/...`"，中间这条链路涉及的并发控制、安全边界检查和配置兼容处理，比大多数人对"接入一个 MCP server"这件事的直觉复杂得多。这也是这个模块最适合拿来讲的地方：MCP 协议本身解决的是"标准化"问题（不同工具提供方只要实现协议就能被复用），但**把一个标准协议安全、高效地接入一个多用户、多线程、同步/异步混合调用的生产系统**，才是这几个文件真正在做的工程工作。
