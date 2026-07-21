# 工具体系与延迟工具绑定 —— 面试问题链

> 本文档按"大厂面试追问链"组织:每条问题链从基础问题开始,层层追问到实现细节、设计权衡、异常边界。配套深读笔记:[04-deferred-tool-binding.md](04-deferred-tool-binding.md)(深读笔记讲"怎么实现",本文档讲"怎么被问、怎么答")。
> 代码引用格式:[文件名.py:起始行-结束行](../backend/仓库相对路径#L起始行-L结束行)。只引用你实际读过的行,禁止编造行号。
> 覆盖范围:`tools/` 目录全量(tools.py / types.py / sync.py / mcp_metadata.py / builtins/tool_search.py / builtins/task_tool.py)、`DeferredToolFilterMiddleware`、`tool_search` 配置,以及 lead / embedded client / subagent 三处调用方。

## 问题链 1:get_available_tools 的多来源聚合

**Q1.1(基础)** 说说 DeerFlow 里一个 agent 的工具列表是怎么来的?`get_available_tools` 从哪几个来源聚合?

**参考回答**:`get_available_tools` 聚合四个来源:(1) config.yaml 里声明的 tools,按 `groups` 过滤后用 `use` 字段反射加载;(2) 内置工具 `BUILTIN_TOOLS`(present_file、ask_clarification)和条件注入的 skill_manage / view_image;(3) MCP 缓存工具 `get_cached_mcp_tools()`;(4) ACP agent 工具。此外 `subagent_enabled=True` 时追加 `SUBAGENT_TOOLS`(task 工具)。见 [tools.py:15-23](../backend/packages/harness/deerflow/tools/tools.py#L15-L23)、[tools.py:44-66](../backend/packages/harness/deerflow/tools/tools.py#L44-L66)、[tools.py:99-100](../backend/packages/harness/deerflow/tools/tools.py#L99-L100)。装配结束时会打一行汇总日志,把 config 加载、built-in、MCP、ACP 四类工具的数量分别列出,方便定位"工具没进来"类问题,见 [tools.py:159](../backend/packages/harness/deerflow/tools/tools.py#L159)。

**链路解析**:

```
        ┌──────────── config.yaml: tools: [{name, group, use, ...extra}] ────────────┐
        │ ToolConfig: name/group/use 三个必填字段,extra="allow" 放行厂商自定义键     │
        ▼                                                                            │
 groups 过滤(tool.group in groups)                                                   │
        ▼                                                                            │
 is_host_bash_allowed? ──否──► 剔除 bash 组/_is_host_bash_tool 命中的条目            │
        ▼                                                                            │
 resolve_variable(cfg.use, BaseTool) 反射加载 ─► name 不一致打 warning ──────────────┤
        ▼                                                                            ▼
 BUILTIN_TOOLS + skill_manage(skill_evolution.enabled) + view_image(supports_vision) ─┐
 SUBAGENT_TOOLS(subagent_enabled=True 才追加 task)                                    ─┤
 MCP: ExtensionsConfig.from_file()(每次读盘,跨进程新鲜)
      ─► get_cached_mcp_tools() ─► tag_mcp_tool 打标("deerflow_mcp"=True)           ─┼─► 合并
 ACP: acp_agents 配置非空 ─► build_invoke_acp_agent_tool                              ─┘
        ▼
 按 t.name 去重,遍历顺序即优先级:
   config 加载 > built-in > MCP > ACP(重复名 warning 并跳过后者)
        ▼
   unique_tools ─► 返回给 agent 构建处
        │ 汇总日志:Total tools loaded / built-in / MCP / ACP 数量分列
        ▼
 下游两道加工:
   ① filter_tools_by_skill_allowed_tools(skill allowed-tools 策略)
   ② assemble_deferred_tools(MCP 工具转 deferred,见问题链 3)
```

**Q1.2(深挖)** config 里的 `use: "deerflow.sandbox.tools:bash_tool"` 这个字符串具体怎么变成一个 BaseTool 实例的?如果 config 里的 name 和工具对象自己的 `.name` 不一致会怎样?

**参考回答**:`resolve_variable(cfg.use, BaseTool)` 按 `module:variable` 路径反射加载并做 `isinstance` 校验,见 [tools.py:73](../backend/packages/harness/deerflow/tools/tools.py#L73)。name 不一致时不会报错,而是打 warning 并以工具对象自己的 `.name` 为准做绑定——这是 issue #1803 的教训:LLM 收到的 schema 名和运行时路由名不一致会产生 "not a valid tool" 错误,见 [tools.py:79-86](../backend/packages/harness/deerflow/tools/tools.py#L79-L86)。回答时要点出"以工具自身 `.name` 为准"这个取舍:配置里的 name 只是提示,运行时路由和 LLM schema 都跟着对象走,保证两端一致。随后每个加载出的工具都会过 `_ensure_sync_invocable_tool`,async-only 的工具在这里被补上 sync 入口,见 [tools.py:88](../backend/packages/harness/deerflow/tools/tools.py#L88)。

**Q1.3(边界/异常)** 如果两个来源出现同名工具怎么办?MCP 模块没装或者加载抛异常呢?

**参考回答**:最后统一按 `t.name` 去重,优先级是 config 加载 > built-in > MCP > ACP,重复名字打 warning 并跳过后者,见 [tools.py:164-176](../backend/packages/harness/deerflow/tools/tools.py#L164-L176)。MCP 部分分两级兜底:`ImportError` 提示装 `langchain-mcp-adapters`,其他异常只记 error 日志继续跑,`mcp_tools` 退化为空列表,见 [tools.py:137-140](../backend/packages/harness/deerflow/tools/tools.py#L137-L140)。**不这样设计会怎样**:如果 MCP 加载失败直接抛出去,一个配错的第三方 MCP server 就会让整个 agent 启动失败;现在的设计是"降级到无 MCP 工具",可用性优先,同时 error 日志保留了排查线索。去重之所以让 config 加载的工具赢,是因为它是用户显式声明的意图,优先级理应高于框架自带和动态发现的来源。

**Q1.4(深挖)** 代码注释说 MCP 配置特意用 `ExtensionsConfig.from_file()` 从磁盘读,而不用内存里的 `config.extensions`,为什么?

**参考回答**:因为 Gateway API 跑在独立进程里改 MCP 配置,内存里的 AppConfig 快照感知不到;每次装配工具时从磁盘重读,能让另一个进程刚写入的 MCP server 增删立即生效,见 [tools.py:113-126](../backend/packages/harness/deerflow/tools/tools.py#L113-L126)。这是一个典型的多进程配置一致性取舍:牺牲一次文件读的开销,换"跨进程配置变更免重启生效"。工具对象本身仍走 `get_cached_mcp_tools()` 缓存,磁盘读只决定"有哪些 server 启用",两件事的刷新粒度是分开的——server 列表每次新鲜,工具对象复用启动期初始化的缓存。如果 `from_file()` 读不到任何启用的 MCP server,就直接跳过缓存加载,`mcp_tools` 保持空列表继续走,见 [tools.py:125-126](../backend/packages/harness/deerflow/tools/tools.py#L125-L126)。

## 问题链 2:安全门控与条件注入

**Q2.1(基础)** host bash 这种危险工具,默认是怎么挡住的?

**参考回答**:`get_available_tools` 在 group 过滤之后、反射加载之前,先调 `is_host_bash_allowed(config)`;不允许时把 `group == "bash"` 或 `use == "deerflow.sandbox.tools:bash_tool"` 的条目从 tool_configs 里剔除,见 [tools.py:69-71](../backend/packages/harness/deerflow/tools/tools.py#L69-L71) 和判定函数 [tools.py:26-34](../backend/packages/harness/deerflow/tools/tools.py#L26-L34)。注意它是在"配置层"过滤,连反射加载都不会发生——危险工具的类对象根本不会进入进程的工具列表,下游所有以工具列表为输入的策略(policy filter、deferred catalog)自然都看不到它。判定用双条件"或":即使有人不遵守 group 命名约定、直接按 use 路径声明 bash 工具,也逃不过这个过滤。

**Q2.2(深挖)** `view_image_tool` 和 `task` 工具不是默认就有的,注入条件分别是什么?

**参考回答**:`view_image_tool` 看模型能力——`model_name` 缺省取 `config.models[0].name`,对应 model_config 的 `supports_vision=True` 才追加,见 [tools.py:104-111](../backend/packages/harness/deerflow/tools/tools.py#L104-L111)。`task` 工具看运行时参数 `subagent_enabled`,只有为 True 才 extend `SUBAGENT_TOOLS`,见 [tools.py:99-101](../backend/packages/harness/deerflow/tools/tools.py#L99-L101)。两个门控一个是"静态能力"(模型配置),一个是"运行时开关"(每次请求的 config),作用域不同,面试时别混为一谈。并发上限 `max_concurrent_subagents` 默认 **3**,但那是 SubagentLimitMiddleware 的职责,不在工具装配这一层。二者的共同点是"默认不加、条件满足才追加",保持默认工具面最小。

**Q2.3(边界/异常)** subagent 自己也会调 `get_available_tools`,怎么防止 subagent 再套 subagent?父 agent 的工具组限制能传下去吗?

**参考回答**:`task_tool` 内部调 `get_available_tools` 时硬编码 `subagent_enabled=False`,递归嵌套在工具层面被掐断;同时把父 agent 的 `metadata["tool_groups"]` 作为 `groups` 透传,subagent 继承同样的工具组限制,见 [task_tool.py:284-303](../backend/packages/harness/deerflow/tools/builtins/task_tool.py#L284-L303)。另外 `bash` 型 subagent 还有二道校验:`subagent_type == "bash"` 且 host bash 不允许时直接返回错误文案,见 [task_tool.py:239-242](../backend/packages/harness/deerflow/tools/builtins/task_tool.py#L239-L242)。也就是说"递归嵌套"和"危险 subagent 类型"是分别在工具装配层和 task 执行层两道闸拦的。父 agent 的 skill 白名单也会和子 agent 配置求交集(`_merge_skill_allowlists`),subagent 的能力只会收敛不会放大,见 [task_tool.py:176-184](../backend/packages/harness/deerflow/tools/builtins/task_tool.py#L176-L184)。

**链路解析**:

```
lead agent (config.metadata: tool_groups=["web"], subagent_enabled=True)
   │ metadata 由 agent 构建处写入,task_tool 从 runtime.config 读取
   │ 模型调用 task(description, prompt, subagent_type="general-purpose")
   ▼
task_tool:
   ├─ get_subagent_config(subagent_type) ──未知类型──► "Error: Unknown subagent type ..."
   ├─ subagent_type=="bash" 且 !is_host_bash_allowed ─► "Error: LOCAL_BASH_SUBAGENT_DISABLED"
   ├─ metadata["available_skills"] ─► _merge_skill_allowlists(parent, child) 求交集
   └─ metadata["tool_groups"] ─► 作为 groups 透传(继承父工具组)
   ▼
get_available_tools(groups=parent_tool_groups, subagent_enabled=False)
   │ subagent_enabled=False ──► 工具列表不含 task ──► 递归嵌套被工具层掐断
   │ groups 继承 ──► subagent 与 lead 看到同一工具组
   ▼
SubagentExecutor(tools) ─► execute_async(prompt, task_id=tool_call_id)
   │ 后台线程跑子 graph;主协程每 5s 轮询一次
   │ max_poll_count = (config.timeout_seconds + 60) // 5
   │ 状态机:RUNNING ─► COMPLETED / FAILED / CANCELLED / TIMED_OUT
   │ 兜底:poll_count > max_poll_count ─► request_cancel_background_task
   │   + _schedule_deferred_subagent_cleanup(协作式取消,防后台任务泄漏)
   ▼
get_background_task_result ─► 终态 ─► cleanup_background_task + 写回 ToolMessage
   │ 全程通过 stream writer 发 SSE: task_started / task_running / task_completed...
```

## 问题链 3:延迟绑定(tool_search)的动机与整体链路

**Q3.1(基础)** 为什么要搞 tool_search 这套延迟绑定?直接把 MCP 工具 schema 全绑给模型不行吗?

**参考回答**:动机是省 context:MCP 工具的完整 JSON schema 很占 token,几十个 MCP 工具全量 bind 会显著挤占上下文窗口并抬高每轮调用成本。开启后(`ToolSearchConfig.enabled`,默认 `False`,见 [tool_search_config.py:14-17](../backend/packages/harness/deerflow/config/tool_search_config.py#L14-L17))MCP 工具只把**名字**列进 system prompt 的 `<available-deferred-tools>` 段,schema 不进 bind_tools,模型需要时用 `tool_search` 按需拉取,见 [tool_search.py:10-15](../backend/packages/harness/deerflow/tools/builtins/tool_search.py#L10-L15) 和 [tool_search.py:207-221](../backend/packages/harness/deerflow/tools/builtins/tool_search.py#L207-L221)。本质是把"工具发现"从编译期(bind 时全量)挪到运行期(按 query 检索),用一轮 tool_search 调用换常驻 context 的缩减;且单次拉取有 **MAX_RESULTS = 5** 的上限,见 [tool_search.py:35](../backend/packages/harness/deerflow/tools/builtins/tool_search.py#L35),防止一次 search 把省下的 context 又还回去。开关本身也极简——`ToolSearchConfig` 只有一个 `enabled` 字段,AppConfig 加载时写入模块级单例供全局读取,见 [tool_search_config.py:20-35](../backend/packages/harness/deerflow/config/tool_search_config.py#L20-L35)。

**Q3.2(深挖)** 那从 agent 构建到模型真正调起一个 deferred 工具,完整链路走一遍?

**参考回答**:五步:(1) `assemble_deferred_tools` 在 policy 过滤后的工具列表上挑出带 `deerflow_mcp` 标记的工具建 `DeferredToolCatalog`;(2) 把 `tool_search` 工具 append 进最终工具列表;(3) prompt 里渲染 `<available-deferred-tools>` 名单;(4) `DeferredToolFilterMiddleware` 在每次 model call 前把未 promote 的 schema 从 `request.tools` 里过滤掉;(5) 模型调 `tool_search`,它通过 `Command(update={"promoted": ...})` 把命中的名字写进 graph state,下一轮 middleware 放行。见 [tool_search.py:184-201](../backend/packages/harness/deerflow/tools/builtins/tool_search.py#L184-L201)、[deferred_tool_filter_middleware.py:51-60](../backend/packages/harness/deerflow/agents/middlewares/deferred_tool_filter_middleware.py#L51-L60)、[tool_search.py:153-158](../backend/packages/harness/deerflow/tools/builtins/tool_search.py#L153-L158)。三件套(tool_search_tool / deferred_names / catalog_hash)作为 `DeferredToolSetup` 整体流转,调用方只需判断 `tool_search_tool` 是否为 None 就能分支,见 [tool_search.py:109-127](../backend/packages/harness/deerflow/tools/builtins/tool_search.py#L109-L127)。

**链路解析**:

```
build 期(每个 agent 构建一次):
  raw_tools ─► filter_tools_by_skill_allowed_tools ─► filtered_tools
       │(顺序是硬要求:catalog 绝不能包含 policy 禁止的工具)
       ▼
  assemble_deferred_tools(filtered_tools, enabled=app_config.tool_search.enabled)
       ├─ enabled=False ───────► DeferredToolSetup(None, ∅, None),全部照常 bind
       ├─ 无 MCP 工具存活 ─────► 同上(空 setup,原因不同,见 DeferredToolSetup docstring)
       └─ 有 MCP 工具 ─► DeferredToolCatalog(tuple(deferred))   # frozen dataclass
                           ├─ names ─► get_deferred_tools_prompt_section ─► system prompt
                           ├─ hash = sha256(canonical JSON)[:16] ─► middleware 构造参数
                           └─ tool_search 工具(闭包持有 catalog) append 进 final_tools
run 期(每轮模型调用):
  wrap_model_call: hidden = deferred_names - promoted(state)
       └► request.override(tools=active) ─► LLM 只见 active + 已 promote 的 schema
  模型调 tool_search("select:X") ─► Command 写 state["promoted"](reducer 合并)
       ▼ 下一轮
  X ∈ promoted ─► X 的 schema 进入 bind ─► 模型调 X ─► wrap_tool_call 放行 ─► ToolNode 执行
  异常兜底: 模型 hallucinate 直接调未 promote 的工具名
       └► wrap_tool_call 拦截 ─► error ToolMessage,引导先 tool_search(见问题链 5)
```

**Q3.3(深挖)** 文档和代码里反复强调 catalog 必须在 tool-policy filtering **之后**构建,为什么这个顺序是硬要求?

**参考回答**:因为 catalog 是模型发现工具的唯一窗口。`build_deferred_tool_setup` 的 docstring 明确写 "Must be called after skill/agent tool-policy filtering so the catalog never exposes a tool the current agent is not allowed to use",见 [tool_search.py:163-172](../backend/packages/harness/deerflow/tools/builtins/tool_search.py#L163-L172)。**不这样设计会怎样**:如果先建 catalog 再过滤,被 skill 的 `allowed-tools` 或 agent 的 `disallowed_tools` 禁掉的 MCP 工具仍然能被 `tool_search` 搜到完整 schema,而 schema 一旦进入模型上下文,模型就可能直接构造调用——过滤中间件只挡"未 promote"的调用,而 tool_search 本身就完成了 promote,等于绕过策略越权。这是 fail-closed 设计的第一道防线。prompt 里的 `<available-deferred-tools>` 名单也是从同一个过滤后的 deferred_names 渲染的,发现面与可调用面严格一致,见 [tool_search.py:207-221](../backend/packages/harness/deerflow/tools/builtins/tool_search.py#L207-L221)。

## 问题链 4:promotion 状态——为什么是 hash-scoped 的 graph state

**Q4.1(基础)** 工具被 promote 之后,这个"已 promote"状态存在哪?为什么不放 ContextVar 或者 middleware 实例属性里?

**参考回答**:存在 graph state 的 `state["promoted"]` 字段,类型是 `PromotedTools(catalog_hash, names)`,挂在 `ThreadState` 上并配了 `merge_promoted` reducer,见 [thread_state.py:85-119](../backend/packages/harness/deerflow/agents/thread_state.py#L85-L119)。middleware 的 docstring 明确说 deferred 集合和 catalog hash 是**构造时注入**,不用 ContextVar,见 [deferred_tool_filter_middleware.py:9-12](../backend/packages/harness/deerflow/agents/middlewares/deferred_tool_filter_middleware.py#L9-L12)。原因:ContextVar 在多 agent 并发/嵌套(lead + subagent 共跑一个 event loop)下会串扰,而 graph state 天然 per-thread、随 checkpoint 持久化、可跨轮恢复;实例属性则无法跨进程恢复,且多 thread 共享同一 agent 实例时会互相污染。构造注入还带来可测试性:middleware 行为完全由构造参数和输入 state 决定,没有隐藏全局状态。`PromotedTools` 本身是个只有两个字段的 TypedDict,见 [thread_state.py:85-87](../backend/packages/harness/deerflow/agents/thread_state.py#L85-L87)。

**Q4.2(深挖)** catalog_hash 具体怎么算的?为什么 promotion 要带 hash,直接存一版名字不行吗?

**参考回答**:hash 把目录里所有工具按名字排序后,取 `{name, openai_function_schema}` 的 canonical JSON,做 sha256 再截前 **16** 个 hex 字符,见 [tool_search.py:66-70](../backend/packages/harness/deerflow/tools/builtins/tool_search.py#L66-L70)。带 hash 是为了防"陈旧 promotion":`_promoted` 只有 `state["promoted"]["catalog_hash"]` 等于当前构造注入的 hash 才采纳,否则返回空集,见 [deferred_tool_filter_middleware.py:42-46](../backend/packages/harness/deerflow/agents/middlewares/deferred_tool_filter_middleware.py#L42-L46)。**不这样设计会怎样**:thread 被 checkpoint 持久化后,如果 MCP server 升级导致同名工具 schema 变了(参数漂移)或工具改名,旧 state 里 bare name 的 promotion 依然有效,模型会拿着过期 schema 的认知去调一个已经变了的工具——hash 不一致时整版 promotion 作废,强制模型重新 tool_search,见 [thread_state.py:90-108](../backend/packages/harness/deerflow/agents/thread_state.py#L90-L108)。实现上 hash 是 `cached_property`,catalog 是 frozen dataclass、不可变,一次构建只算一次且结果稳定,见 [tool_search.py:56-70](../backend/packages/harness/deerflow/tools/builtins/tool_search.py#L56-L70)。

**Q4.3(边界/异常)** `merge_promoted` 这个 reducer 的三种分支分别是什么语义?同一轮多次 tool_search 会互相覆盖吗?

**参考回答**:三种分支:(1) `new` 为空 → 保留 existing(本轮没碰 promotion);(2) hash 变了 → 整体替换,丢掉旧名字;(3) hash 相同 → names 取并集去重保序,见 [thread_state.py:98-108](../backend/packages/harness/deerflow/agents/thread_state.py#L98-L108)。所以同一线程内多次 tool_search 是**累积**的,不会互相覆盖;只有目录漂移才整体重置。注意去重用 `dict.fromkeys`,保序的同时幂等,reducer 重复应用不会膨胀——这对 LangGraph 可能多次重放 state 更新的场景很重要。如果两次 search 之间 MCP 目录变了(比如 Gateway 进程刚改了 server 列表并重建缓存),第二次 search 写入的新 hash 会让旧 names 整体失效——这正是分支②在实际运行中的触发场景。

**链路解析**:

```
tool_search 执行完毕
   │ Command(update={"promoted": {"catalog_hash": h_new, "names": N_new}, "messages": [...]})
   ▼
LangGraph 按 Annotated reducer 合并 state["promoted"]:
   (reducer 在每次节点返回 update 时触发,不只 tool_search 一种来源)
   ┌───────────────────────────────────────────────────────────────────┐
   │ merge_promoted(existing, new)                                     │
   │  ① new 为 None/空 ────────────────► 返回 existing(本轮未触碰)    │
   │  ② existing=None 或 hash 不同 ────► 整体替换为 new(丢弃 stale)   │
   │  ③ hash 相同 ─────────────────────► names = existing ∪ new        │
   │                                     (dict.fromkeys 去重保序)      │
   └───────────────────────────────────────────────────────────────────┘
   ▼
下一次 wrap_model_call / wrap_tool_call:
   middleware._promoted(state):
     state["promoted"]["catalog_hash"] == self._catalog_hash(构造注入)?
       ├─ 是 ─► set(names) 参与"放行集合"计算
       └─ 否 ─► 返回 ∅(checkpoint 里的旧 promotion 整版作废)
   hidden = frozenset(deferred_names) - promoted ─► 决定过滤与拦截

checkpoint 恢复场景(thread 重开 / 进程重启后):
   state["promoted"] 从持久化恢复 ─► 与当前构造注入的 catalog_hash 比对
     ├─ 一致 ────► promotion 继续有效,模型无需重新 search
     └─ 不一致 ──► 整版作废(MCP 目录漂移),模型须重新 tool_search
        ("hash-scoped per-thread"的全部含义:状态归线程,有效性归目录)
```

## 问题链 5:DeferredToolFilterMiddleware 的两道闸

**Q5.1(基础)** 这个 middleware 在 agent 执行的哪两个点上拦截?分别干什么?

**参考回答**:两个点:`wrap_model_call` 在模型绑定前把仍处于 deferred 状态的工具 schema 从 `request.tools` 里滤掉,让 LLM 根本看不见;`wrap_tool_call` 在工具执行前拦截对未 promote 工具的调用,直接返回 error ToolMessage 而不进 handler,见 [deferred_tool_filter_middleware.py:76-93](../backend/packages/harness/deerflow/agents/middlewares/deferred_tool_filter_middleware.py#L76-L93)。同步和 async(`awrap_*`)各有一份,逻辑一致,见 [deferred_tool_filter_middleware.py:95-112](../backend/packages/harness/deerflow/agents/middlewares/deferred_tool_filter_middleware.py#L95-L112)。两道闸的分工是"可见性"与"可执行性"分离:第一道让模型看不见,第二道兜底"万一看见了(或 hallucinate 了)也调不动"。值得强调 ToolNode 仍持有全量工具——过滤的只是绑给模型的 schema 视图,执行路由不受影响,见 [deferred_tool_filter_middleware.py:30-35](../backend/packages/harness/deerflow/agents/middlewares/deferred_tool_filter_middleware.py#L30-L35)。

**Q5.2(深挖)** 过滤工具列表时为什么用 `request.override(tools=active)` 生成新 request,而不是原地改 `request.tools`?deferred_names 为什么在构造时注入而不是每次现算?

**参考回答**:`ModelRequest` 在 middleware 链里是共享对象,原地改会污染下游其他 middleware 看到的视图;`override` 返回拷贝,只影响本次 handler 调用,见 [deferred_tool_filter_middleware.py:51-60](../backend/packages/harness/deerflow/agents/middlewares/deferred_tool_filter_middleware.py#L51-L60)。构造注入是因为 deferred 集合来自 agent build 时 policy 过滤后的快照,跑期现算既没有输入来源也会破坏"build 时确定、run 时只读"的不变量;`ToolNode` 仍持有全量工具做执行路由,middleware 只控制"模型可见性",见 [deferred_tool_filter_middleware.py:30-40](../backend/packages/harness/deerflow/agents/middlewares/deferred_tool_filter_middleware.py#L30-L40)。还有一个性能点:`_deferred` 为空时两个 wrap 都直接短路返回,见 [deferred_tool_filter_middleware.py:52-53](../backend/packages/harness/deerflow/agents/middlewares/deferred_tool_filter_middleware.py#L52-L53),未开启 tool_search 的部署零开销。每次实际过滤掉几个 schema 也会记 debug 日志,见 [deferred_tool_filter_middleware.py:58-59](../backend/packages/harness/deerflow/agents/middlewares/deferred_tool_filter_middleware.py#L58-L59)。

**Q5.3(边界/异常)** 如果模型 hallucinate,直接调了一个它在 schema 里从没见过的 deferred 工具名,会发生什么?

**参考回答**:`_blocked_tool_message` 命中 hidden 集合,返回一条 `status="error"` 的 ToolMessage,内容明确提示 "Call tool_search first to expose and promote this tool's schema, then retry",并把 `tool_call_id` 填上(缺省用 `missing_tool_call_id`)保证消息协议完整,见 [deferred_tool_filter_middleware.py:62-74](../backend/packages/harness/deerflow/agents/middlewares/deferred_tool_filter_middleware.py#L62-L74)。这样不会执行工具、不会让 graph 崩,还给模型一条可自我纠正的错误反馈——下一轮它大概率会去调 tool_search 再重试,正好走完设计好的 promote 流程。注意 `_blocked_tool_message` 查的是 `_hidden(state)` 而不是整个 `_deferred` 集合——已 promote 的工具不会被误拦,见 [deferred_tool_filter_middleware.py:66](../backend/packages/harness/deerflow/agents/middlewares/deferred_tool_filter_middleware.py#L66)。

**链路解析**:

```
构造期(agent build 时一次):
  DeferredToolFilterMiddleware(deferred_names: frozenset, catalog_hash)
   │ 无 ContextVar;deferred 集合与 hash 来自 build 时 policy 过滤后的快照
   ▼
┌────────────────────── agent loop 每一轮 ──────────────────────┐
│ 闸 1: wrap_model_call(模型调用前 —— schema 可见性)            │
│   request.tools(ToolNode 持有全量,含 deferred)               │
│      │ _hidden(state) = deferred_names - promoted(state)      │
│      ▼                                                        │
│   active = [t for t in tools if t.name ∉ hidden]              │
│      │ request.override(tools=active)(拷贝,不污染共享对象)  │
│      ▼                                                        │
│   handler(override 后的 request) ─► LLM 只 bind active schema │
│                                                               │
│ 闸 2: wrap_tool_call(工具执行前 —— 调用合法性)                │
│   模型发起 tool_call(name=X)                                  │
│      │ X ∈ _hidden(state)?                                    │
│      ├─ 否 ─► handler(request) ─► 正常执行                    │
│      └─ 是 ─► ToolMessage(status="error",                     │
│           content="Error: Tool 'X' is deferred and has not    │
│           been promoted yet. Call tool_search first ...")     │
│           ─► 不执行,回到消息流,模型下一轮自我纠正            │
│                                                               │
│ 短路: _deferred 为空 ─► 两道闸都原样透传,零开销              │
│ 日志: 实际过滤时记 debug(过滤掉的 schema 数量);拦截不另记   │
│       日志 —— error 已通过 ToolMessage 回给模型,无需重复告警 │
└───────────────────────────────────────────────────────────────┘
```

## 问题链 6:tool_search 工具本身

**Q6.1(基础)** `tool_search` 支持哪几种 query 形式?一次最多返回几个?

**参考回答**:三种形式,写在工具 docstring 里:`select:Read,Edit` 按名字精确取;`keyword` 普通关键词(regex 匹配 name+description,name 命中权重 **2**、description 命中权重 **1**);`+slack send` 要求名字必须含某个 token 再按剩余词打分。每次最多返回 **MAX_RESULTS = 5** 个,见 [tool_search.py:35](../backend/packages/harness/deerflow/tools/builtins/tool_search.py#L35)、[tool_search.py:72-98](../backend/packages/harness/deerflow/tools/builtins/tool_search.py#L72-98)、[tool_search.py:142-146](../backend/packages/harness/deerflow/tools/builtins/tool_search.py#L142-L146)。`select:` 按逗号切分名字集合再精确匹配,见 [tool_search.py:77-79](../backend/packages/harness/deerflow/tools/builtins/tool_search.py#L77-L79);`+` 形式的必填 token 只在 name 里匹配,见 [tool_search.py:81-89](../backend/packages/harness/deerflow/tools/builtins/tool_search.py#L81-L89);裸 `+` 后面没有 token 时直接返回空,注释写明 "nothing to require",见 [tool_search.py:83-84](../backend/packages/harness/deerflow/tools/builtins/tool_search.py#L83-L84)。

**Q6.2(深挖)** query 是模型生成的,如果模型给了一个非法 regex(比如括号不配对)会怎样?

**参考回答**:`_compile_catalog_regex` 先 `re.compile(..., re.IGNORECASE)`,捕获 `re.error` 后降级为 `re.escape(pattern)` 的字面子串匹配,绝不向模型抛异常,见 [tool_search.py:38-47](../backend/packages/harness/deerflow/tools/builtins/tool_search.py#L38-L47)。**不这样设计会怎样**:模型一次手滑的非法 pattern 就会让工具调用以异常收场,轻则浪费一轮交互,重则在没有 tool-error 兜底的路径上打断整个 agent loop;降级为字面匹配是"对 LLM 输入永远防御性处理"的典型做法,同样的防御还有空 query 直接返回空列表,见 [tool_search.py:73-75](../backend/packages/harness/deerflow/tools/builtins/tool_search.py#L73-L75)。大小写不敏感(`re.IGNORECASE`)也是刻意的:模型对工具名大小写的记忆并不可靠。

**Q6.3(深挖)** tool_search 的返回值类型是 `Command` 而不是字符串,为什么?它往 state 里写了什么?

**参考回答**:因为它要同时干两件事:给模型返回 ToolMessage(命中工具的完整 OpenAI function schema JSON),以及把 promote 名单写进 graph state——这两件事通过 `Command(update={"promoted": {...}, "messages": [ToolMessage(...)]})` 一次完成,见 [tool_search.py:147-158](../backend/packages/harness/deerflow/tools/builtins/tool_search.py#L147-L158)。没命中时写空 names 列表和 "No tools found matching" 文案,promotion 不产生实际效果(merge 后 names 为空集)。`catalog_hash` 在闭包构建时捕获,保证写进 state 的 hash 和 middleware 构造注入的 hash 是同一个值,见 [tool_search.py:130-131](../backend/packages/harness/deerflow/tools/builtins/tool_search.py#L130-L131)。schema 序列化走 `convert_to_openai_function` 再 `json.dumps(indent=2)`,和 bind 路径用的是同一份 schema 表示,模型看到的格式一致,见 [tool_search.py:151](../backend/packages/harness/deerflow/tools/builtins/tool_search.py#L151)。

**链路解析**:

```
query(模型生成,不可信输入)
   │
   ├─ strip 后为空 ─────────────────────────► return []
   ├─ "select:A,B" ─► 按逗号切分名字 ──────► wanted ∩ catalog.names,[:5]
   │   (精确定向拉取:模型已从名单知道名字时的最短路径)
   ├─ "+token rest" ─► name 必含 token ─────► 再按 rest 的 findall 次数排序,[:5]
   └─ 其他 ─► _compile_catalog_regex(query)
                │ re.error? ─► re.escape(query) 降级为字面匹配(绝不抛异常)
                ▼
             regex.search(f"{name} {description}")
                │ name 命中权重 2,description 命中权重 1,降序,[:MAX_RESULTS=5]
                │ (scored.sort 按权重排序;同分保持 catalog 内的原始遍历序)
   ▼
matched?
   ├─ 空 ───► Command{promoted.names=[], ToolMessage("No tools found matching: ...")}
   └─ 非空 ─► Command{
                "promoted": {"catalog_hash": h, "names": [t.name, ...]},
                "messages": [ToolMessage(json.dumps(
                    [convert_to_openai_function(t) for t in matched], indent=2))]}
                ─► schema 进消息流(模型当下可读)
                ─► names 进 state(下一轮 wrap_model_call 放行 bind)
   Command 是 langgraph 标准的"工具 → state"通道:
   一次工具调用同时产出 ToolMessage 和 state 更新,无需额外节点
```

## 问题链 7:subagent 里的一致性与 MCP metadata 透传

**Q7.1(基础)** 系统怎么知道一个工具是 MCP 来源的?这个标记写在哪、谁读?

**参考回答**:靠 `metadata["deerflow_mcp"] = True` 标记。`MCP_TOOL_METADATA_KEY` 常量和 `tag_mcp_tool` / `is_mcp_tool` 一对函数定义在 `mcp_metadata.py` 这个刻意设计的 leaf module(只依赖 BaseTool,避免 import cycle);写在 `tools.py` 加载 MCP 缓存工具处,读在 `tool_search.py` 的 deferred 装配处,见 [mcp_metadata.py:18-29](../backend/packages/harness/deerflow/tools/mcp_metadata.py#L18-L29)、[tools.py:135-136](../backend/packages/harness/deerflow/tools/tools.py#L135-L136)、[tool_search.py:176](../backend/packages/harness/deerflow/tools/builtins/tool_search.py#L176)。把 magic string 收在一个 leaf module 里,写入方和读取方都 import 公开谓词,避免了跨模块私有常量和循环依赖。`tag_mcp_tool` 是原地 mutate 并返回自身,方便链式使用,见 [mcp_metadata.py:21-24](../backend/packages/harness/deerflow/tools/mcp_metadata.py#L21-L24)。

**Q7.2(深挖)** subagent 跑在独立的 graph 里,deferred 语义怎么保证和 lead agent 一致?

**参考回答**:三条路径共享同一套装配函数,保证一致:(1) `SubagentExecutor._build_initial_state` 在 skill policy 过滤之后调同一个 `assemble_deferred_tools`;(2) 用同一个 `get_deferred_tools_prompt_section` 把 `<available-deferred-tools>` 拼进 subagent 的 SystemMessage;(3) `build_subagent_runtime_middlewares` 在 `deferred_setup.deferred_names` 非空时挂同一个 `DeferredToolFilterMiddleware`,见 [executor.py:441-468](../backend/packages/harness/deerflow/subagents/executor.py#L441-L468)、[tool_error_handling_middleware.py:229-237](../backend/packages/harness/deerflow/agents/middlewares/tool_error_handling_middleware.py#L229-L237)。lead 侧的挂法完全相同,见 [agent.py:343-349](../backend/packages/harness/deerflow/agents/lead_agent/agent.py#L343-L349);embedded client 也走同一个 `assemble_deferred_tools`,见 [client.py:242](../backend/packages/harness/deerflow/client.py#L242)。注意:subagent 有自己独立的 graph state,所以它的 promotion 是 per-subagent-run 的,不会泄漏回 lead 的 state——隔离性和一致性同时成立。subagent 的 prompt 段是把 system_prompt、skill 消息、deferred 段拼成**一条** SystemMessage——有些 LLM API 拒绝多条 system 消息,见 [executor.py:456-472](../backend/packages/harness/deerflow/subagents/executor.py#L456-L472)。

**链路解析**:

```
MCP 标记的写入与读取(leaf module 承载 magic string):
  tools.py 加载缓存 MCP 工具 ─► tag_mcp_tool(t): t.metadata["deerflow_mcp"] = True
                                        │ mcp_metadata.py(只依赖 BaseTool,无循环依赖)
                                        ▼
  tool_search.py / executor.py ─► is_mcp_tool(t) ─► 决定是否进 deferred catalog

三条 agent 构建路径共享同一装配(fail-closed 只实现一次):
  lead(agent.py):        get_available_tools ─► skill 过滤 ─► assemble_deferred_tools
  embedded(client.py):   tools ─► assemble_deferred_tools ─► build_middlewares(deferred_setup)
  subagent(executor.py): _apply_skill_allowed_tools ─► assemble_deferred_tools
        ├─ final_tools(含 tool_search)──► create_agent(tools=...)
        ├─ deferred_names ─► get_deferred_tools_prompt_section ─► SystemMessage
        └─ deferred_setup ─► build_subagent_runtime_middlewares
                              ─► DeferredToolFilterMiddleware(names, hash)
  一致性: 同一 catalog 构造函数、同一 prompt 渲染函数、同一 middleware 类
  隔离性: subagent 独立 graph state,promotion 不回传 lead
  例外: subagent 的 tool_search 不受 config.tools/disallowed_tools 名单约束——
        catalog 已从过滤后的列表构建,不可能搜出被禁工具(executor.py 注释明示)
```

**Q7.3(边界/异常)** 什么叫 fail-closed?tool_search 开着、MCP 工具也活过了 policy 过滤,但 deferred set 没建出来,会发生什么?

**参考回答**:`assemble_deferred_tools` 直接 `raise RuntimeError`,拒绝继续构建 agent——宁可启动失败也不把 MCP 全量 schema 静默绑给模型,见 [tool_search.py:196-197](../backend/packages/harness/deerflow/tools/builtins/tool_search.py#L196-L197)。`DeferredToolSetup` 还有一个显式不变量:`tool_search_tool is None` ⟺ `deferred_names` 为空 ⟺ `catalog_hash is None`,三个字段必须同生同灭,见 [tool_search.py:121-123](../backend/packages/harness/deerflow/tools/builtins/tool_search.py#L121-L123)。**不这样设计会怎样**:如果此处静默放行,操作者以为开了延迟绑定、实际上模型拿到了全量 schema——配置语义和实际行为背离,context 成本和安全面都在无人察觉的情况下劣化。这个 raise 之所以放在共享的 `assemble_deferred_tools` 里,就是让 lead / client / subagent 三条路径不用各自重复这份检查。

## 问题链 8:同步/异步桥接(sync wrapper)

**Q8.1(基础)** `make_sync_tool_wrapper` 是解决什么问题的?线程池参数是多少?

**参考回答**:解决"只有 coroutine 的 async 工具被同步 agent 调用方使用"的问题:`_ensure_sync_invocable_tool` 检测工具没有 `func` 但有 `coroutine` 时,给它套上 sync wrapper,见 [tools.py:37-41](../backend/packages/harness/deerflow/tools/tools.py#L37-L41)。wrapper 跑在共享线程池 `_SYNC_TOOL_EXECUTOR` 上,**max_workers=10**,`thread_name_prefix="tool-sync"`,进程退出时 `atexit` 触发 `shutdown(wait=False)`,见 [sync.py:17-19](../backend/packages/harness/deerflow/tools/sync.py#L17-L19)。`wait=False` 表示退出阶段不等池内任务跑完,避免 atexit 被挂住。这个 wrapper 在装配末尾去重前也会统一再过一遍,保证所有来源的工具都可被同步调用,见 [tools.py:164](../backend/packages/harness/deerflow/tools/tools.py#L164)。

**Q8.2(深挖)** 如果调用时已经有一个 running event loop(比如 async 的 LangGraph runtime 里走了同步工具路径),直接 `asyncio.run` 会炸,这里怎么处理的?

**参考回答**:`run_coroutine` 先 `asyncio.get_running_loop()` 探测:有 running loop 时,`contextvars.copy_context()` 复制当前上下文,提交到线程池里用 `asyncio.run(coro(...))` 起一个新 loop 跑,主线程 `future.result()` 阻塞等结果;没有 running loop 就直接 `asyncio.run`,见 [sync.py:64-78](../backend/packages/harness/deerflow/tools/sync.py#L64-L78)。复制 context 是为了让 ContextVar(如 tracing、user context)在子线程的新 loop 里仍然可见;异常不吞,记 error 日志后原样抛出,见 [sync.py:76-78](../backend/packages/harness/deerflow/tools/sync.py#L76-L78)。`future.result()` 的阻塞只发生在同步调用方线程里,新 loop 跑在池线程中,不会干扰原来的 event loop。

**Q8.3(深挖)** 有些工具(如 `invoke_acp_agent`)需要 LangChain 注入 `RunnableConfig`,sync wrapper 怎么把 config 传进 coroutine 的?

**参考回答**:`_get_runnable_config_param` 用 `get_type_hints` 扫描 coroutine 签名,找出类型标注为 `RunnableConfig` 的形参名(支持 `functools.partial` 解包);找到就生成带 `config: RunnableConfig = None` 形参的 wrapper,让 LangChain 能把 config 注入进来,再转发到 coroutine 真正的那个参数名上,见 [sync.py:22-35](../backend/packages/harness/deerflow/tools/sync.py#L22-L35)、[sync.py:80-92](../backend/packages/harness/deerflow/tools/sync.py#L80-L92)。docstring 也坦白了局限:不合成动态签名,如果未来某工具既有用户侧 `config` 参数又有另名的 RunnableConfig 参数会撞名,需要先改名或扩展这个 helper,见 [sync.py:55-60](../backend/packages/harness/deerflow/tools/sync.py#L55-L60)。`functools.partial` 解包那一步保证了被 partial 包装的工具也能正确识别 config 参数,见 [sync.py:24-25](../backend/packages/harness/deerflow/tools/sync.py#L24-L25)。

**链路解析**:

```
入口: _ensure_sync_invocable_tool(tool)
   │ 条件: tool.func is None 且 tool.coroutine 非空 ─► 才套 wrapper
   ▼
sync 调用方: tool.func(*args, **kwargs)(make_sync_tool_wrapper 的产物)
   │
   ▼ run_coroutine
   asyncio.get_running_loop()
   ├─ RuntimeError(无 loop)──► asyncio.run(coro(...)) 直接新建 loop 跑
   └─ 已有 running loop ───────────────────────────────────────────┐
        │ contextvars.copy_context()(保留 tracing/user 等 ContextVar)│
        ▼                                                            │
        _SYNC_TOOL_EXECUTOR(max_workers=10, prefix="tool-sync")       │
        └ submit(context.run, lambda: asyncio.run(coro(...))) ────────┘
        │ future.result() 阻塞等待 ─► 返回结果;异常记 error 后透传
   config 敏感工具分支(如 invoke_acp_agent):
   _get_runnable_config_param(coro): get_type_hints 找 RunnableConfig 形参名
     ├─ 找到 ─► wrapper(*args, config: RunnableConfig=None, **kwargs)
     │          └ kwargs[检测到的参数名] = config ─► 转发进 coroutine
     └─ 未找到 ─► 朴素 wrapper,参数原样透传
   已知局限: 用户侧参数若也叫 config 且 RunnableConfig 参数另名,会撞名
            (docstring 明示,需先改名或扩展 helper)
   生命周期: atexit ─► _SYNC_TOOL_EXECUTOR.shutdown(wait=False)
            (进程退出不等池内任务跑完,避免 atexit 阶段挂死)
   设计要点: 探测 loop ─► 复制 context ─► 池线程新 loop ─► 阻塞等结果
```

## 面试官最爱追问的 3 个点

1. **"promotion 为什么用 catalog_hash 而不用 bare names / ContextVar?"** —— 应答策略:一句话锁定两个正交理由:graph state 解决"per-thread 持久化 + 多 agent 并发不串扰",hash scope 解决"checkpoint 恢复后目录漂移导致 stale promotion 放行已改名/变 schema 的工具";引用 [deferred_tool_filter_middleware.py:42-46](../backend/packages/harness/deerflow/agents/middlewares/deferred_tool_filter_middleware.py#L42-L46) 和 [thread_state.py:90-108](../backend/packages/harness/deerflow/agents/thread_state.py#L90-L108)。如果对方追问"hash 撞了怎么办",可以答:hash 输入是名字+完整 schema 的 canonical JSON,16  hex 截断在单 agent 目录规模下碰撞概率可忽略,且撞了的最坏结果只是沿用旧 promotion,语义仍自洽。
2. **"catalog 为什么必须在 policy filtering 之后构建?顺序反了会怎样?"** —— 应答策略:先讲越权链路(被禁工具仍能被 tool_search 搜出 schema → promote → 可调用),再补 fail-closed 的 RuntimeError 兜底,引用 [tool_search.py:163-172](../backend/packages/harness/deerflow/tools/builtins/tool_search.py#L163-L172) 和 [tool_search.py:196-197](../backend/packages/harness/deerflow/tools/builtins/tool_search.py#L196-L197)。强调这个检查只做一次、放在共享装配函数里,三条 agent 构建路径免费获得同一保证。若被追问"为什么不在 middleware 里查 policy",可答:middleware 只持有 build 时的快照,policy 的输入(skills、agent config)在 run 期不可得,防线只能前置到 catalog 构建点。
3. **"模型直接调了没 promote 的工具会发生什么?会不会崩 graph?"** —— 应答策略:强调两道闸的分工——wrap_model_call 让模型"看不见",wrap_tool_call 兜底"看见了也调不动",返回带明确纠错指引的 error ToolMessage 而非异常;引用 [deferred_tool_filter_middleware.py:62-74](../backend/packages/harness/deerflow/agents/middlewares/deferred_tool_filter_middleware.py#L62-L74)。可以补一句:错误文案本身就是给模型的 prompt 工程,引导它下一步去调 tool_search,把异常路径转化为设计好的 promote 流程。若继续追问"为什么不用 exception",可答:agent loop 里异常会中断 run 或触发上层重试,而 error ToolMessage 让纠错留在模型循环内,成本最低。
