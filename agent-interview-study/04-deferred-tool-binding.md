# Module 4：Tool / Function Calling 与延迟工具绑定

**核心文件**：
- [tools/builtins/tool_search.py](backend/packages/harness/deerflow/tools/builtins/tool_search.py)
- [agents/middlewares/deferred_tool_filter_middleware.py](backend/packages/harness/deerflow/agents/middlewares/deferred_tool_filter_middleware.py)
- [tools/mcp_metadata.py](backend/packages/harness/deerflow/tools/mcp_metadata.py)

## 一、问题定义

接入多个 MCP server 后，工具可能有几十上百个。如果全部 `bind_tools` 给模型：
1. 每个工具的完整 JSON Schema 都要塞进请求，占用大量 context
2. 候选工具太多，模型选择时的准确率会下降（"选择困难症"）

DeerFlow 的解法是**延迟工具绑定（Deferred Tool）**：大部分 MCP 工具默认不绑定 schema，只在 system prompt 里列"名字+一句话描述"；模型需要时先调用 `tool_search` 检索，命中的工具才会在**下一轮**被真正绑定。

## 二、核心数据结构：`DeferredToolCatalog`

[tool_search.py:56-70](backend/packages/harness/deerflow/tools/builtins/tool_search.py#L56-L70)：

```python
@dataclass(frozen=True)
class DeferredToolCatalog:
    tools: tuple[BaseTool, ...]

    @cached_property
    def names(self) -> frozenset[str]:
        return frozenset(t.name for t in self.tools)

    @cached_property
    def hash(self) -> str:
        canon = [{"name": t.name, "schema": convert_to_openai_function(t)} for t in sorted(self.tools, key=lambda t: t.name)]
        blob = json.dumps(canon, sort_keys=True, ensure_ascii=False, default=str)
        return hashlib.sha256(blob.encode("utf-8")).hexdigest()[:16]
```

**为什么 `hash` 是"内容哈希"而不是版本号/时间戳**：`hash` 是对**排序后的全部工具 name+schema** 做 sha256，这意味着只要工具目录的实际内容（新增、删除、改名、改参数）发生任何变化，hash 就一定变；如果内容完全没变，hash 就完全一样——它不是"这次启动的版本号"，而是"这份目录内容的指纹"。这个指纹后面要拿去给 `promoted` 状态做**版本校验**（下面会讲为什么这是安全关键点）。

**`frozen=True` 但不加 `slots=True` 的那个注释**（[L53-55](backend/packages/harness/deerflow/tools/builtins/tool_search.py#L53-L55)）：`@cached_property` 需要往实例的 `__dict__` 写缓存值，而 `frozen=True` 的 dataclass 本来会拦截 `__setattr__`——但 `cached_property` 走的是直接操作 `instance.__dict__`，绕过了 `__setattr__` 检查，所以能在"冻结"对象上正常缓存。如果手滑加上 `slots=True`，实例就没有 `__dict__` 了，`cached_property` 会在运行时直接报错——这是一个"两个特性看起来正交，实际耦合"的经典坑。

## 三、`search` 的三种查询语法

[tool_search.py:72-98](backend/packages/harness/deerflow/tools/builtins/tool_search.py#L72-L98)：

```python
def search(self, query: str) -> list[BaseTool]:
    if query.startswith("select:"):
        wanted = {n.strip() for n in query[7:].split(",")}
        return [t for t in self.tools if t.name in wanted][:MAX_RESULTS]
    if query.startswith("+"):
        required = parts[0].lower()
        candidates = [t for t in self.tools if required in t.name.lower()]
        ... # 按剩余关键词打分排序
    regex = _compile_catalog_regex(query)
    ... # 对 name+description 做正则匹配，name 命中权重更高(2) 于 description 命中(1)
```

三种语法覆盖三种使用场景：`select:Read,Edit` 是模型已经**确切知道要哪几个工具名**（比如之前 tool_search 过一次，现在换个 session）；`+slack send` 是"必须含 slack，再按 send 相关性排序"的两阶段过滤；裸查询走正则/子串兜底。**`_compile_catalog_regex` 遇到非法正则会降级为字面匹配**（[L38-47](backend/packages/harness/deerflow/tools/builtins/tool_search.py#L38-L47)）——因为查询词来自模型输出，模型完全可能生成一个不平衡括号的"伪正则"，这里选择"优雅降级"而不是让 `tool_search` 直接抛异常炸掉整个 agent 循环。

## 四、`tool_search` 工具本身——用 `Command` 同时干两件事

[tool_search.py:130-160](backend/packages/harness/deerflow/tools/builtins/tool_search.py#L130-L160)：

```python
@tool
def tool_search(query: str, tool_call_id: Annotated[str, InjectedToolCallId]) -> Command:
    matched = catalog.search(query)[:MAX_RESULTS]
    ...
    return Command(
        update={
            "promoted": {"catalog_hash": catalog_hash, "names": names},
            "messages": [ToolMessage(content=content, tool_call_id=tool_call_id, name="tool_search")],
        }
    )
```

这里复用了 Module 1 里讲过的 `merge_promoted` reducer（[thread_state.py:90-108](backend/packages/harness/deerflow/agents/thread_state.py#L90-L108)）——工具本身不直接"暴露 schema 给模型"，而是把匹配结果**写进 state**（`promoted.names` + `promoted.catalog_hash`），同时把工具的完整 schema 塞进这次 `tool_search` 调用自己的 `ToolMessage` 内容里（让模型这一轮就能看到 schema 文本）。真正"能不能在下一轮被模型调用"，则要看 `DeferredToolFilterMiddleware` 在 state 里读到的 `promoted` 是否覆盖这个工具名。

**`catalog_hash` 在这里的作用**：`tool_search` 返回的 `promoted` 只在这次调用发生时的 `catalog.hash` 下有效。如果 MCP 配置后来变了（服务器重连、工具改名），新的 catalog hash 会不同，`merge_promoted` 里"hash 变了就整体丢弃旧名单"的逻辑会让这条旧的 promotion 记录自动失效——**防止一个持久化在 state 里的裸名字，在工具目录漂移后意外指向了一个完全不同的新工具**（比如两次不同 MCP 版本里同名但参数语义完全不同的工具）。

## 五、`DeferredToolFilterMiddleware`——真正执行"隐藏"的地方

[deferred_tool_filter_middleware.py:42-60](backend/packages/harness/deerflow/agents/middlewares/deferred_tool_filter_middleware.py#L42-L60)：

```python
def _promoted(self, state) -> set[str]:
    promoted = (state or {}).get("promoted")
    if promoted and promoted.get("catalog_hash") == self._catalog_hash:
        return set(promoted.get("names") or [])
    return set()

def _hidden(self, state) -> set[str]:
    return set(self._deferred) - self._promoted(state)

def _filter_tools(self, request: ModelRequest) -> ModelRequest:
    ...
    hide = self._hidden(request.state)
    active = [t for t in request.tools if getattr(t, "name", None) not in hide]
    return request.override(tools=active)
```

`_hidden` 就是一次集合差：**全部延迟工具 − 已提升工具 = 当前该隐藏的工具**。`wrap_model_call` 在每次真正发起模型请求前用这个差集过滤 `request.tools`，模型永远看不到还没被 `tool_search` 命中过的工具 schema。

**`self._deferred`/`self._catalog_hash` 是构造时传入的（"no ContextVar"）**：类文档字符串（[L9](backend/packages/harness/deerflow/agents/middlewares/deferred_tool_filter_middleware.py#L9)）特意强调这点——延迟集合和目录哈希是**构建 agent 时**就固定死的（一次 agent 构建 = 一份工具目录 = 一个 catalog），不依赖运行时的 ContextVar 去动态取值。这样设计的好处是这个 middleware 实例本身是无状态的纯函数式过滤器，真正会变化的只有 `state["promoted"]`（每个 thread 独立），不存在多线程共享同一个 middleware 实例时的状态串号风险。

## 六、双重防御：不只隐藏 schema，还要拦截调用

[deferred_tool_filter_middleware.py:62-93](backend/packages/harness/deerflow/agents/middlewares/deferred_tool_filter_middleware.py#L62-L93)：

```python
def _blocked_tool_message(self, request: ToolCallRequest) -> ToolMessage | None:
    name = str(request.tool_call.get("name") or "")
    if not name or name not in self._hidden(request.state):
        return None
    return ToolMessage(
        content=f"Error: Tool '{name}' is deferred and has not been promoted yet. Call tool_search first...",
        tool_call_id=..., name=name, status="error",
    )

@override
def wrap_tool_call(self, request, handler):
    blocked = self._blocked_tool_message(request)
    if blocked is not None:
        return blocked
    return handler(request)
```

**为什么隐藏了 schema 还不够，还要在 `wrap_tool_call` 再拦一次**：模块文档字符串说得很直接（[L1-7](backend/packages/harness/deerflow/agents/middlewares/deferred_tool_filter_middleware.py#L1-L7)）——ToolNode（LangGraph 真正执行工具调用的节点）**始终持有全部工具**（包括还没提升的），这是为了让"已经提升过的工具"能被正常路由执行。但这意味着如果模型通过某种方式（比如复述了之前对话里看到的工具名，或者 hallucinate 出一个工具名）生成了一个指向未提升工具的 `tool_call`，光靠"没绑定 schema"是拦不住**执行**的——`wrap_tool_call` 就是这层"即使被调用了也要在真正执行前挡住"的纵深防御，返回一条明确的 error ToolMessage 告诉模型"先去 `tool_search`"。

这跟 Module 2 讲过的设计哲学一致：**"不给模型看见"和"不允许模型做"是两件独立的事，安全关键场景要两层都做**（类似 Sandbox 章节里"虚拟路径 + 强制翻译校验"的两层设计）。

## 七、装配层：fail-closed 设计

[tool_search.py:184-201](backend/packages/harness/deerflow/tools/builtins/tool_search.py#L184-L201)：

```python
def assemble_deferred_tools(filtered_tools, *, enabled):
    deferred_setup = build_deferred_tool_setup(filtered_tools, enabled=enabled)
    if enabled and not deferred_setup.deferred_names and any(is_mcp_tool(t) for t in filtered_tools):
        raise RuntimeError("tool_search enabled and MCP tools survived policy filtering, but no deferred set was recovered - refusing to bind MCP schemas (fail-closed).")
    ...
```

如果配置里开了 `tool_search.enabled`，且确实有 MCP 工具通过了策略过滤，但因为某种 bug 没能正确构建出 `deferred_names`——这里选择**直接抛异常拒绝启动**，而不是"退化成把所有 MCP 工具原样绑定"。这是典型的 fail-closed：宁可这次 agent 构建失败，也不要在"用户以为工具被延迟保护，实际上全量暴露了"这种静默安全降级状态下继续跑。

## 八、总结表

| 组件 | 角色 |
|---|---|
| `is_mcp_tool`/`tag_mcp_tool`（`mcp_metadata.py`） | 单一来源标记"这个工具是不是 MCP 来的"，避免多处判断逻辑发散 |
| `DeferredToolCatalog` | 不可变、可搜索的延迟工具目录；`hash` 是内容指纹，用于给 promotion 打版本戳 |
| `tool_search` 工具 | 模型主动检索入口；命中后通过 `Command` 同时更新 `state["promoted"]` 并把 schema 文本塞进本轮 ToolMessage |
| `merge_promoted` reducer | 按 `catalog_hash` 分区合并已提升工具名单，目录漂移时整体失效（见 Module 1） |
| `DeferredToolFilterMiddleware.wrap_model_call` | 每次请求前过滤 `request.tools`，模型看不到未提升工具的 schema |
| `DeferredToolFilterMiddleware.wrap_tool_call` | 纵深防御第二层：即使有调用漏网，执行前也会被挡住 |
| `assemble_deferred_tools` 的 fail-closed 检查 | 宁可启动失败，不要静默退化成"全量暴露" |
