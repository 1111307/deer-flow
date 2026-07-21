# Agent 主循环与状态管理 —— 面试问题链

> 本文档按"大厂面试追问链"组织:每条问题链从基础问题开始,层层追问到实现细节、设计权衡、异常边界。配套深读笔记:[01-state-and-middleware.md](01-state-and-middleware.md)(深读笔记讲"怎么实现",本文档讲"怎么被问、怎么答")。
> 代码引用格式:[文件名.py:起始行-结束行](../backend/仓库相对路径#L起始行-L结束行)。只引用实际读过的行。

## 问题链 1:为什么用 LangGraph 图模型而不是手写 while 循环

**Q1.1(基础)** 你们的 Agent 主循环是怎么实现的?是自己写 `while True: call_llm -> call_tool` 吗?

**参考回答**:不是手写循环,而是基于 LangGraph 的 `create_agent` 构建一张状态图,模型节点和工具节点之间由图引擎驱动,中断/恢复由 checkpointer 负责。工厂入口是 `make_lead_agent(config)`,它把模型、工具、middleware 链、system prompt 和 `ThreadState` 组装进 `create_agent` 返回一张编译好的图 [agent.py:520-540](../backend/packages/harness/deerflow/agents/lead_agent/agent.py#L520-L540)。

**链路解析**:
```
langgraph.json
   │ 注册
   ▼
make_lead_agent(config)  ──► _make_lead_agent(config, app_config)
   │                            │
   │                            ├─ create_chat_model(attach_tracing=False)
   │                            ├─ get_available_tools() → filter → assemble_deferred_tools
   │                            ├─ build_middlewares(...)   ← 19 个组件
   │                            ├─ apply_prompt_template(...) ← system prompt
   │                            └─ state_schema=ThreadState
   ▼
create_agent(...) → CompiledStateGraph → 由 LangGraph 驱动 model↔tools 循环
```

**Q1.2(深挖)** 那手写循环到底差在哪?LangGraph 给了你什么手写拿不到的东西?

**参考回答**:三样东西:第一,**可中断的人机协同**——`ClarificationMiddleware` 拦截 `ask_clarification` 后返回 `Command(goto=END)` 中断执行,等用户回答再从 checkpoint 恢复,手写循环要自己实现持久化和断点恢复 [clarification_middleware.py:25-36](../backend/packages/harness/deerflow/agents/middlewares/clarification_middleware.py#L25-L36);第二,**声明式状态合并**——多个节点并行写同一个 state key 时由 reducer 定义合并语义,不用自己加锁;第三,**middleware 横切机制**——`before_model`/`after_model`/`wrap_tool_call` 钩子可以在不改主循环的情况下插入 19 个横切组件。反例:如果手写循环,`DanglingToolCallMiddleware` 这种"用户打断后补齐缺失 ToolMessage"的逻辑就得塞进主循环的异常分支里,主循环会迅速变成一锅粥。

**Q1.3(边界/异常)** 图模型有什么坑?比如 tracing 回调挂错地方会发生什么?

**参考回答**:最大的坑是 tracing 回调的挂载位置。`agent.py` 开头的 docstring 把这条写成 INVARIANT:所有图内 `create_chat_model` 必须传 `attach_tracing=False`,回调只挂在**图调用根**的 `config["callbacks"]` 上 [agent.py:1-19](../backend/packages/harness/deerflow/agents/lead_agent/agent.py#L1-L19)。如果忘了这个 flag,会产生重复 span(图根一个、模型一个),并且 Langfuse handler 的 `propagate_attributes` 路径不会触发,`session_id`/`user_id` 永远到不了 trace。目前有四处以身作则:bootstrap agent、默认 agent、summarization middleware、`TitleMiddleware` 的异步路径 [agent.py:16-18](../backend/packages/harness/deerflow/agents/lead_agent/agent.py#L16-L18)。

---

## 问题链 2:ThreadState 与自定义 reducer 语义

**Q2.1(基础)** 你们的 Agent 状态长什么样?消息之外的字段怎么设计?

**参考回答**:`ThreadState` 继承 LangChain 的 `AgentState`,扩展了 8 个字段:`sandbox`、`thread_data`、`title`、`artifacts`、`todos`、`uploaded_files`、`viewed_images`、`promoted` [thread_state.py:111-119](../backend/packages/harness/deerflow/agents/thread_state.py#L111-L119)。其中 5 个字段带自定义 reducer,用 `Annotated[T, reducer]` 声明合并语义。

**链路解析**:
```
tool / middleware 节点返回 partial state
        │  (可能多个节点同 step 写同一 key)
        ▼
LangGraph 按 key 查 reducer:
  sandbox        → merge_sandbox        (fail-closed)
  artifacts      → merge_artifacts      (保序去重)
  viewed_images  → merge_viewed_images  ({} = 清空)
  todos          → merge_todos          (last-non-None wins)
  promoted       → merge_promoted       (按 catalog_hash 作用域)
        ▼
合并后的 ThreadState 进入下一节点
```

**Q2.2(深挖)** `merge_sandbox` 为什么冲突时直接 raise 而不是选一个?这不是把一次普通写入变成整个 run 崩溃吗?

**参考回答**:这是刻意的 **fail-closed** 设计。多个 sandbox 工具在同一个图步骤里懒初始化时,会通过 `Command(update=...)` 写同一个 `sandbox_id`,这是幂等写,reducer 放行;但如果同一 thread 出现**两个不同的** sandbox id,说明生命周期/隔离出了 bug(比如串了别的线程的沙箱),此时静默选一个会把文件写进错误的隔离域——这是安全事故而不是可用性问题。所以实现里 `existing_id == new_id` 返回 existing,否则 `raise ValueError` [thread_state.py:21-39](../backend/packages/harness/deerflow/agents/thread_state.py#L21-L39)。反例:若改成"后写覆盖",一次串线就会把用户 A 的文件落到用户 B 的目录,且无任何报错信号。

**Q2.3(深挖)** `merge_viewed_images` 里"空 dict 表示清空"这个特殊语义不别扭吗?为什么不单独开个 action?

**参考回答**:这是用 reducer 语义表达"读完后消费掉"的技巧:reducer 收到 `new == {}` 时返回 `{}` 清空全部已看图片 [thread_state.py:55-69](../backend/packages/harness/deerflow/agents/thread_state.py#L55-L69)。`ViewImageMiddleware` 把 base64 注入模型上下文后写回空 dict,防止图片在 state 里无限累积撑爆 checkpoint。对比 `merge_todos`:它的 `new is None` 表示"节点没碰 todos,保留原值",而空 list 是"显式清空" [thread_state.py:72-82](../backend/packages/harness/deerflow/agents/thread_state.py#L72-L82)——两个 reducer 对"空"的解释正好相反,因为图片是**累积型**状态(默认要 merge),todos 是**快照型**状态(默认要覆盖)。

**Q2.4(边界/异常)** `merge_promoted` 为什么要带 `catalog_hash`?如果工具目录漂移了会怎样?

**参考回答**:`promoted` 记录"本线程已通过 tool_search 提升为可见的 deferred MCP 工具名"。reducer 比较新旧值的 `catalog_hash`:hash 变了就整体替换并丢弃旧名字,防止"持久化下来的裸工具名在目录漂移后指向另一个工具";hash 相同才做并集去重且保序 [thread_state.py:90-108](../backend/packages/harness/deerflow/agents/thread_state.py#L90-L108)。反例:如果只做 names 并集而不校验 hash,运维改了 MCP server 后,旧 checkpoint 里提升过的 `some_tool` 可能静默绑定到新 server 的同名但语义不同的工具上——这是供应链级别的隐患。

---

## 问题链 3:middleware 链 19 个组件的装配顺序

**Q3.1(基础)** 你们的 middleware 链有多少个组件?顺序是随便排的吗?

**参考回答**:全量开启时 19 个,顺序是严格设计的,代码里有一段 10 行的注释块把关键约束写死在 `build_middlewares` 上方 [agent.py:260-269](../backend/packages/harness/deerflow/agents/lead_agent/agent.py#L260-L269)。链分两段装配:前 9 个共享运行时组件由 `build_lead_runtime_middlewares` 产出(以 `ToolOutputBudgetMiddleware` 开头、`ToolErrorHandlingMiddleware` 收尾)[tool_error_handling_middleware.py:129-187](../backend/packages/harness/deerflow/agents/middlewares/tool_error_handling_middleware.py#L129-L187),其余 10 个在 `build_middlewares` 里按条件逐个 append [agent.py:299-377](../backend/packages/harness/deerflow/agents/lead_agent/agent.py#L299-L377)。

**链路解析**:
```
build_lead_runtime_middlewares (共享段, 9 个)
  0 ToolOutputBudget → 1 ThreadData → 2 Uploads(insert) → 3 Sandbox
  → 4 DanglingToolCall → 5 LLMErrorHandling → 6 Guardrail* → 7 SandboxAudit
  → 8 ToolErrorHandling
build_middlewares (lead 专属段, 10 个)
  9 DynamicContext → 10 SkillActivation → 11 Summarization* → 12 Todo*
  → 13 TokenUsage* → 14 Title → 15 Memory → 16 ViewImage*
  → 17 DeferredToolFilter* → 18 SubagentLimit* → Loop* → custom*
  → SafetyFinishReason* → 19 Clarification  (永远最后)
(* = 按配置/运行时参数条件挂载)
```

**Q3.2(深挖)** 挑三个关键约束讲讲:为什么 ThreadData 必须在 Sandbox 前面?为什么 ToolErrorHandling 必须在 Clarification 前面?

**参考回答**:第一,`ThreadDataMiddleware` 先于 `SandboxMiddleware`,因为 sandbox 获取需要 `thread_id` 来建 per-thread 目录;`UploadsMiddleware` 插在两者之间(index 2),它也要读 `thread_id` [tool_error_handling_middleware.py:142-151](../backend/packages/harness/deerflow/agents/middlewares/tool_error_handling_middleware.py#L142-L151)。第二,`ToolErrorHandlingMiddleware` 用 `wrap_tool_call` 把工具异常转成 error ToolMessage 让 run 继续,它必须包住所有可能抛异常的工具调用;如果 Clarification 在它前面拦截,`ask_clarification` 的 `Command(goto=END)` 控制流就会被 error handler 误吞——所以实现里特意对 `GraphBubbleUp` 异常直接 re-raise 放行 [tool_error_handling_middleware.py:102-110](../backend/packages/harness/deerflow/agents/middlewares/tool_error_handling_middleware.py#L102-L110)。第三,Summarization 要尽量靠前,先压缩上下文再让后面的 middleware 处理,能省 token。

**Q3.3(深挖)** `SafetyFinishReasonMiddleware` 注册在列表尾部,注释却说它"最先"看到模型输出,这不矛盾吗?

**参考回答**:不矛盾,这正是 LangChain middleware 的 **after_model 反向 dispatch** 语义:factory 把 `after_model` 边按列表逆序接线,**最后注册的最先观察到模型输出**。Safety 注册在 LoopDetection 之后,于是它先拿到原始 AIMessage——如果 provider 以 `content_filter`/`refusal`/`SAFETY` 之类的原因截断响应却还带着半截 `tool_calls`,Safety 先剥掉这些 tool_calls,清理过的消息再流过 Loop/Subagent 的计数逻辑,不会误触发告警 [safety_finish_reason_middleware.py:21-33](../backend/packages/harness/deerflow/agents/middlewares/safety_finish_reason_middleware.py#L21-L33)。选 `after_model` 而不是 `wrap_model_call`,是因为安全截断是**正常返回**而非异常。

**Q3.4(边界/异常)** 为什么 `ClarificationMiddleware` 必须是最后一个?如果有人通过 `custom_middlewares` 把它挤走了会怎样?

**参考回答**:因为它要在"所有其他 after_model/wrap 逻辑都放行之后"才拦截 `ask_clarification` 并中断到 END;不在最后,后面的 middleware 可能在澄清问题发出后还继续改写消息。`build_middlewares` 里 custom middlewares 被插在 Clarification 之前,Safety 也在它之前 append,最后一行无条件 `middlewares.append(ClarificationMiddleware())` [agent.py:362-377](../backend/packages/harness/deerflow/agents/lead_agent/agent.py#L362-L377)。SDK 侧的 `create_deerflow_agent` 还有一道保险:用 `@Next`/`@Prev` 插入外部 middleware 后,会显式检查 Clarification 的索引,不在尾部就 pop 出来重新 append 到末尾 [factory.py:292-296](../backend/packages/harness/deerflow/agents/factory.py#L292-L296)。

---

## 问题链 4:system prompt 组装与注入点

**Q4.1(基础)** system prompt 是怎么拼出来的?memory、skills、日期这些动态内容放哪?

**参考回答**:模板是 `SYSTEM_PROMPT_TEMPLATE`,由 `apply_prompt_template` 一次性 format,注入 soul、self_update、skills、deferred tools、subagent、ACP/mounts 等段落 [prompt.py:757-813](../backend/packages/harness/deerflow/agents/lead_agent/prompt.py#L757-L813)。关键设计:**memory 和当前日期不在 system prompt 里**,而是由 `DynamicContextMiddleware` 每轮以 `<system-reminder>` 形式注入第一条 HumanMessage,让 system prompt 完全静态以吃满 prefix cache [agent.py:302-306](../backend/packages/harness/deerflow/agents/lead_agent/agent.py#L302-L306)。

**链路解析**:
```
apply_prompt_template(...)
  │ agent_name / SOUL.md ──► get_agent_soul()           [静态]
  │ skills 元数据 ─────────► get_skills_prompt_section() [lru_cache]
  │ deferred MCP 工具 ─────► get_deferred_tools_prompt_section()
  │ subagent 编排规则 ─────► _build_subagent_section(n)
  │ ACP / custom mounts ───► _build_acp_section()
  ▼
完全静态的 system prompt ──► prefix-cache 友好
每轮动态部分(日期/memory)──► DynamicContextMiddleware → 注入 HumanMessage
```

**Q4.2(深挖)** skills 列表注入 prompt 时做了哪些性能优化?缓存失效怎么处理并发?

**参考回答**:两层缓存。一是 `get_cached_enabled_skills`:请求路径绝不阻塞磁盘 I/O,cache miss 时后台线程刷新、本次先返回空列表,下一次调用就能读到热数据 [prompt.py:115-128](../backend/packages/harness/deerflow/agents/lead_agent/prompt.py#L115-L128);warm-up 等待超时是 **5.0 秒** [prompt.py:20](../backend/packages/harness/deerflow/agents/lead_agent/prompt.py#L20)。二是渲染结果用 `@lru_cache(maxsize=32)`,key 是 skill 签名(名称/描述/类别/容器路径四元组)+ 可用集合 + 容器路径 [prompt.py:607-614](../backend/packages/harness/deerflow/agents/lead_agent/prompt.py#L607-L614)。失效时用 version 计数器解决并发:worker 加载期间若版本号变了,说明有更新的失效事件,丢弃旧结果循环重载,保证缓存收敛到最新版本 [prompt.py:41-63](../backend/packages/harness/deerflow/agents/lead_agent/prompt.py#L41-L63)。

**Q4.3(边界/异常)** memory 注入失败会怎样?会影响主对话吗?

**参考回答**:不会。`_get_memory_context` 整个包在 try/except 里,任何异常(存储坏了、tiktoken 下载失败等)都 `logger.exception` 后返回空字符串,prompt 照常组装 [prompt.py:563-604](../backend/packages/harness/deerflow/agents/lead_agent/prompt.py#L563-L604)。这是"增强型上下文可以降级、主链路不能断"的典型取舍;对比之下 `_load_enabled_skills_for_tool_policy` 加载失败是**直接 raise** 的 [agent.py:388-395](../backend/packages/harness/deerflow/agents/lead_agent/agent.py#L388-L395)——因为 skills 白名单决定工具过滤,属于安全边界,不能降级。

---

## 问题链 5:runtime configurable 与模型解析

**Q5.1(基础)** 运行时参数(`thinking_enabled`、`is_plan_mode`、`subagent_enabled`)从哪进来?默认值是什么?

**参考回答**:从 `RunnableConfig` 进,`_get_runtime_config` 把 legacy 的 `configurable` 和 LangGraph 的 `context` 两个字典合并,context 覆盖 configurable [agent.py:55-61](../backend/packages/harness/deerflow/agents/lead_agent/agent.py#L55-L61)。默认值:`thinking_enabled=True`、`reasoning_effort=None`、`is_plan_mode=False`、`subagent_enabled=False`、`max_concurrent_subagents=3`、`is_bootstrap=False` [agent.py:418-424](../backend/packages/harness/deerflow/agents/lead_agent/agent.py#L418-L424)。

**链路解析**:
```
请求 config
  ├─ configurable: {model_name, is_plan_mode, subagent_enabled, ...}
  └─ context:      {...}   (LangGraph runtime context, 优先级更高)
        ▼ _get_runtime_config 合并
cfg ──► 模型解析: requested → agent_config.model → 全局默认
   ──► thinking 降级: supports_thinking=False 时强制关 + warning
   ──► middleware 条件挂载: plan_mode→Todo, subagent→SubagentLimit
   ──► prompt 参数: subagent_enabled / max_concurrent_subagents
```

**Q5.2(深挖)** 用户请求了一个配置里不存在的模型名,会发生什么?直接报错吗?

**参考回答**:不报错,降级。`_resolve_model_name` 先取 `app_config.models[0].name` 作为默认模型(一个模型都没配才 raise);请求的模型名在配置里查得到就用,查不到且不等于默认名时打 warning 并回落到默认模型 [agent.py:64-76](../backend/packages/harness/deerflow/agents/lead_agent/agent.py#L64-L76)。模型解析优先级是"请求 > custom agent 自带 model > 全局默认" [agent.py:430-433](../backend/packages/harness/deerflow/agents/lead_agent/agent.py#L430-L433)。

**Q5.3(边界/异常)** 模型不支持 thinking 但用户开了 `thinking_enabled`,你们静默降级还是报错?为什么?

**参考回答**:静默降级 + warning 日志:`thinking_enabled and not model_config.supports_thinking` 时置 False 并记录"fallback to non-thinking mode" [agent.py:439-441](../backend/packages/harness/deerflow/agents/lead_agent/agent.py#L439-L441)。选降级不选报错,因为 thinking 是体验增强而非正确性前提,用户换了一个不支持 thinking 的模型不该让整个对话 400;反例是模型完全解析不出来(`model_config is None`)时直接 raise——那是"没有任何大脑可用",必须 fail-fast [agent.py:437-438](../backend/packages/harness/deerflow/agents/lead_agent/agent.py#L437-L438)。这些解析结果还会写进 `config["metadata"]` 供 LangSmith trace 打标 [agent.py:455-469](../backend/packages/harness/deerflow/agents/lead_agent/agent.py#L455-L469)。

---

## 问题链 6:SDK 工厂与声明式 feature 装配

**Q6.1(基础)** 除了 `make_lead_agent`,你们还有别的 agent 创建入口吗?区别是什么?

**参考回答**:有,`create_deerflow_agent` 是 SDK 级入口:纯参数、不读 YAML、不依赖全局单例,介于裸 `create_agent` 和配置驱动的 `make_lead_agent` 之间 [factory.py:61-147](../backend/packages/harness/deerflow/agents/factory.py#L61-L147)。它提供两种互斥的 middleware 供给方式:`middleware=` 全量接管(传了就不能再给 features/extra_middleware,否则 ValueError),或 `features=RuntimeFeatures(...)` 声明式自动装配 [factory.py:110-117](../backend/packages/harness/deerflow/agents/factory.py#L110-L117)。

**链路解析**:
```
create_deerflow_agent(model, tools, features=..., extra_middleware=...)
   │ 互斥校验: middleware ⊕ (features|extra_middleware)
   ▼
_assemble_from_features(feat, plan_mode, extra_middleware)
   │ 内置链固定顺序 append (14 个槽位, 0-13)
   │ feature 值三态: False=跳过 / True=内置默认 / 实例=自定义替换
   ▼
_insert_extra(chain, extras)   ← @Next/@Prev 锚点插入
   │ 无锚点 → 插到 Clarification 之前
   │ 有锚点 → 多轮迭代解析(支持外部互锚)
   ▼
不变量修复: Clarification 不在尾部则 pop 重 append
```

**Q6.2(深挖)** `RuntimeFeatures` 里 `summarization` 和 `guardrail` 为什么不能像别的 feature 一样传 `True`?

**参考回答**:因为这两个没有内置默认实现——summarization 必须拿到一个 model 参数,guardrail 需要 provider 实例,factory 是纯参数的、不该偷偷去读全局配置造一个出来,所以传 `True` 会直接 raise ValueError,只能传 `False` 或一个现成的 `AgentMiddleware` 实例 [factory.py:209-223](../backend/packages/harness/deerflow/agents/factory.py#L209-L223)。类型签名上也用了 `Literal[False] | AgentMiddleware` 把这件事静态钉死 [features.py:29-33](../backend/packages/harness/deerflow/agents/features.py#L29-L33)。

**Q6.3(边界/异常)** `@Next`/`@Prev` 锚点插入有哪些失败模式?循环锚定怎么办?

**参考回答**:`_insert_extra` 有四类显式报错:同一 middleware 同时挂 @Next 和 @Prev;两个 extras 锚同一个锚点(同向或反向都算冲突);锚点在链上找不到(多轮迭代后仍无法解析);以及锚点间循环依赖——检测方法是迭代一轮后 remaining 没减少,取"剩余锚点类型 ∩ 剩余 middleware 类型"的交集,非空即循环 [factory.py:306-379](../backend/packages/harness/deerflow/agents/factory.py#L306-L379)。迭代轮数上限是 `len(pending) + 1`,防止死循环 [factory.py:353-355](../backend/packages/harness/deerflow/agents/factory.py#L353-L355)。

---

## 问题链 7:subagent 并发限制与 prompt/系统双重约束

**Q7.1(基础)** `max_concurrent_subagents=3` 这个限制是怎么落地的?只靠 prompt 告诉模型吗?

**参考回答**:双保险。Prompt 侧:`_build_subagent_section(n)` 把上限 n 写进 `<subagent_system>` 段,反复强调"HARD CONCURRENCY LIMIT"、教模型先 COUNT 再分批 [prompt.py:236-261](../backend/packages/harness/deerflow/agents/lead_agent/prompt.py#L236-L261)。系统侧:`SubagentLimitMiddleware` 在 `after_model` 截断超额的 `task` 调用,仅当 `subagent_enabled` 时挂载,默认值 **3** [agent.py:351-355](../backend/packages/harness/deerflow/agents/lead_agent/agent.py#L351-L355)。

**链路解析**:
```
模型输出 AIMessage (含 5 个 task 调用, 上限 3)
        ▼ after_model (反向 dispatch)
SafetyFinishReason ──► LoopDetection ──► SubagentLimitMiddleware
                                              │ 截断第 4、5 个 task 调用
                                              ▼
                                       只放行前 3 个 → ToolNode
```

**Q7.2(深挖)** 为什么 prompt 里要告诉模型"超额调用会被静默丢弃"?不怕模型因此行为怪异吗?

**参考回答**:恰恰相反,必须明说。中间件截断是**静默**的,如果模型不知道,它会以为 5 个子任务都在跑,下一轮干等不存在的结果;prompt 里写明"Any excess calls are silently discarded by the system — you will lose that work",并给出多批次编排范式(每轮 ≤n,等结果再发下一批,最后统一 SYNTHESIZE)[prompt.py:246-256](../backend/packages/harness/deerflow/agents/lead_agent/prompt.py#L246-L256)。这是"系统行为必须在 prompt 里可观测"的原则:任何对模型输出的隐形改写,都要在 prompt 里找到对应说明。

**Q7.3(边界/异常)** bootstrap 模式和普通模式在工具/prompt 上有什么区别?为什么要单独搞一个 bootstrap agent?

**参考回答**:bootstrap 用于"创建 custom agent"的初始化流程,刻意收窄:skills 只暴露 `{"bootstrap"}` 一个 [agent.py:52](../backend/packages/harness/deerflow/agents/lead_agent/agent.py#L52),工具额外绑定 `setup_agent` 而不是 `update_agent`,且不加载 agent config(`agent_config=None`)[agent.py:486-511](../backend/packages/harness/deerflow/agents/lead_agent/agent.py#L486-L511)。理由是确定性:在 custom agent 自己的 config 还不存在时,创造它的流程不能依赖那份不存在的配置,否则就是鸡生蛋问题。普通 custom agent 则绑定 `update_agent` 让它自改 SOUL.md/config [agent.py:513-515](../backend/packages/harness/deerflow/agents/lead_agent/agent.py#L513-L515)。

---

## 面试官最爱追问的 3 个点

1. **after_model 反向 dispatch 的顺序推理**:"Safety 注册在最后为什么最先执行?"——应答策略:先讲 LangChain 按列表逆序接 after_model 边的事实,再用 Safety→Loop 的协作举例(先剥 tool_calls 再做循环计数),最后补一句 Clarification 必须最后是因为它走 wrap_tool_call + `Command(goto=END)` 中断,不参与 after_model 逆序问题但语义上必须兜底。

2. **reducer 的 fail-closed 取舍**:"merge_sandbox 直接 raise,不怕把整个 run 打挂?"——应答策略:承认会挂,但强调"同 thread 出现两个 sandbox id"本身就是隔离 bug,静默合并会把数据写错隔离域;崩溃是显性的、可告警的,串数据是隐性的、不可挽回的。顺便对比 merge_artifacts 的宽容(保序去重)说明"每个 reducer 的严格程度跟着安全等级走"。

3. **静态 prompt 与 prefix cache**:"日期和 memory 为什么不放 system prompt?"——应答策略:讲清注入点转移(DynamicContextMiddleware 以 `<system-reminder>` 注入首条 HumanMessage),量化收益是 system prompt 跨用户/跨会话完全一致、prefix cache 命中率最大化;再补一个配套证据:skills prompt 段用 `lru_cache(maxsize=32)` 缓存渲染结果,enabled-skills 加载用后台线程 + 5 秒 warm-up 超时,请求路径零磁盘 I/O。
