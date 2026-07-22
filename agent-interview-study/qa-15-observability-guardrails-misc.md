# 可观测 / 护栏 / 上传 / 社区工具 / 反射 / 配置 —— 面试问题链

> 本文档按"大厂面试追问链"组织:每条问题链从基础问题开始,层层追问到实现细节、设计权衡、异常边界。配套深读笔记:无(本模块尚无深读笔记,本文档是第一份)。
> 代码引用格式:[文件名.py:起始行-结束行](../backend/仓库相对路径#L起始行-L结束行)。只引用你实际读过的行,禁止编造行号。

本模块覆盖六个"小而硬"的支撑面:tracing 双 provider 挂载与 Langfuse metadata 提升(`tracing/`)、工具调用前授权护栏(`guardrails/`)、上传目录安全写入与转换流水线(`uploads/` + `utils/file_conversion.py`)、社区搜索/抓取工具的统一模式(`community/tavily|jina_ai|firecrawl`)、类路径反射加载(`reflection/resolvers.py`)、以及配置热重载与 startup-only 边界(`config/app_config.py` + `config/reload_boundary.py`)。

---

## 问题链 1:Langfuse / LangSmith 双挂载与回调挂载位置

**Q1.1(基础)** 你们的 Agent 系统是怎么接可观测平台的?LangSmith 和 Langfuse 能同时开吗?

**参考回答**:能同时开。`build_tracing_callbacks()` 先校验再遍历 `get_enabled_tracing_providers()`,为 langsmith 构造 `LangChainTracer(project_name=...)`,为 langfuse 先初始化 `Langfuse` 客户端单例再返回 `LangfuseCallbackHandler(public_key=...)`,两者放进同一个 callbacks 列表返回([factory.py:32-54](../backend/packages/harness/deerflow/tracing/factory.py#L32-L54))。是否启用完全由环境变量决定:`LANGSMITH_TRACING`/`LANGFUSE_TRACING` 为 truthy(`{"1","true","yes","on"}`)且对应 key 齐全才算 configured([tracing_config.py:86-129](../backend/packages/harness/deerflow/config/tracing_config.py#L86-L129))。构造失败会被包成 `RuntimeError(f"Langfuse tracing initialization failed: {exc}")` 抛出,而不是静默降级([factory.py:48-52](../backend/packages/harness/deerflow/tracing/factory.py#L48-L52))。

**链路解析**:

```
env: LANGSMITH_TRACING=1 + LANGFUSE_TRACING=1
  -> get_tracing_config() (进程级单例, threading.Lock 双检)
       -> enabled_providers = ["langsmith", "langfuse"]
  -> build_tracing_callbacks()
       +-- langsmith -> LangChainTracer(project_name)
       +-- langfuse  -> Langfuse(secret/public key, host)  # 初始化 client 单例
       |             -> LangfuseCallbackHandler(public_key) # 挂到该 client
       v
  [tracer, handler] 追加到 config["callbacks"] (graph 调用根)
       -> 一次 stream() = 一条 trace,node/LLM/tool 全部嵌套其下
```

**Q1.2(深挖)** 这些回调挂在哪里?我注意到 `create_chat_model` 也有个 `attach_tracing` 参数,为什么图内的模型调用都必须传 `False`?

**参考回答**:回调必须挂在 **graph 调用根**——gateway worker 和 embedded client 都在 `stream()` 入口把 `build_tracing_callbacks()` 的结果追加进 `config["callbacks"]`([client.py:607-610](../backend/packages/harness/deerflow/client.py#L607-L610))。图内任何 `create_chat_model(...)` 必须传 `attach_tracing=False`,这是 `lead_agent/agent.py` 模块 docstring 里写死的 INVARIANT,当前有四个位置:bootstrap agent、default agent、summarization middleware、`TitleMiddleware` 异步路径([agent.py:1-19](../backend/packages/harness/deerflow/agents/lead_agent/agent.py#L1-L19))。如果忘了,`create_chat_model` 会把同一组 callbacks 直接钉到模型实例上([factory.py:198-204](../backend/packages/harness/deerflow/models/factory.py#L198-L204)),于是同一次 LLM 调用产生两条 span(一条根在 graph、一条根在 model);更隐蔽的是 Langfuse handler 的 `propagate_attributes` 路径不再触发,`session_id`/`user_id` 永远到不了 trace 上。反过来说,`attach_tracing` 默认 `True` 是给图外独立调用者(如 `MemoryUpdater`)留的,它们没有 graph 根可挂([factory.py:89-99](../backend/packages/harness/deerflow/models/factory.py#L89-L99))。还有测试把这条不变量钉死了:`test_lead_agent_model_resolution.py` 断言所有图内 `create_chat_model` 调用的 `attach_tracing` 都是 `False`,且 `build_tracing_callbacks` 的输出确实进了 `config["callbacks"]`。**不这样设计会怎样**:假设把 `attach_tracing` 默认值改成 `False`、让图外调用者显式开,那么每个新增的工具函数都要记得传参,漏一个就丢一条 trace——把默认放在"图外要挂、图内要关"这个方向,是因为图内调用点少且集中(4 处)、图外调用点多且分散,出错的期望成本更低。

**链路解析**:

```
错误挂法 (attach_tracing=True 在图内):
  graph stream (callbacks=[langfuse])  -> trace root
    └─ LLM node (model.callbacks=[langfuse]) -> 模型自己又开一条 span
         => 同一次调用 2 条 span;嵌套层里 langfuse_* metadata 被剥离
            session_id / user_id 丢失

正确挂法 (attach_tracing=False 在图内):
  graph stream (callbacks=[langfuse])  -> 唯一 trace root
    ├─ agent node
    │    └─ LLM call  (继承根 callbacks, 一条 span)
    └─ tool node     (一条 span)
         => 一条 trace 全嵌套;根 metadata 提升 session/user
```

**Q1.3(边界/异常)** 如果我只是在 config.yaml 里写了 `LANGFUSE_TRACING=true` 但忘了配 secret key,系统行为是什么?会不会跑到一半才炸?

**参考回答**:会在第一次构造回调时快速失败,不会跑到一半。`validate_enabled_tracing_providers()` 在 `build_tracing_callbacks()` 第一行就被调用([factory.py:34](../backend/packages/harness/deerflow/tracing/factory.py#L34)),`LangfuseTracingConfig.validate()` 发现 `enabled=True` 但缺 `LANGFUSE_PUBLIC_KEY`/`LANGFUSE_SECRET_KEY` 时抛 `ValueError` 并列出缺失的变量名([tracing_config.py:38-47](../backend/packages/harness/deerflow/config/tracing_config.py#L38-L47))。注意"显式开启"和"配置完整"是两套列表:`explicitly_enabled_providers` 只看开关,`enabled_providers` 要求 key 齐全([tracing_config.py:60-76](../backend/packages/harness/deerflow/config/tracing_config.py#L60-L76))——所以"没配 key 就当没开"这种静默降级被刻意排除,运维写错了必须立刻知道。另外 tracing config 是进程级缓存单例,运行中改环境变量不会生效,只能靠 `reset_tracing_config()` 清缓存(测试用)([tracing_config.py:107-162](../backend/packages/harness/deerflow/config/tracing_config.py#L107-L162))。

---

## 问题链 2:Langfuse trace metadata 提升机制

**Q2.1(基础)** Langfuse 的 Users 页面和 Sessions 分组是靠什么数据驱动的?thread_id 怎么变成 Langfuse 的 session?

**参考回答**:靠 `RunnableConfig.metadata` 里一组保留 key。Langfuse v4 的 `langchain.CallbackHandler` 只在根 run(`on_chain_start` 且 `parent_run_id=None`)时从 metadata 里提升四个 key:`langfuse_session_id`、`langfuse_user_id`、`langfuse_trace_name`、`langfuse_tags`([metadata.py:1-15](../backend/packages/harness/deerflow/tracing/metadata.py#L1-L15))。`build_langfuse_trace_metadata()` 把 LangGraph 的 `thread_id` 直接映射为 `langfuse_session_id`,`user_id` 缺省回退到 `DEFAULT_USER_ID` 让无认证模式下 Users 页面也能用,`assistant_id` 缺省为 `"lead-agent"`;`environment` 和 `model_name` 则编码成 `env:xxx`、`model:xxx` 形式的 tags([metadata.py:28-70](../backend/packages/harness/deerflow/tracing/metadata.py#L28-L70))。

**链路解析**:

```
client.stream(thread_id, user_id, model_name, environment)
  -> inject_langfuse_metadata(config, ...)
       -> build_langfuse_trace_metadata()
            langfuse_session_id = thread_id
            langfuse_user_id    = user_id or DEFAULT_USER_ID
            langfuse_trace_name = assistant_id or "lead-agent"
            langfuse_tags       = ["env:production", "model:gpt-x"]
       -> setdefault 合并进 config["metadata"]
  -> graph.astream(..., config)
       -> LangfuseCallbackHandler.on_chain_start(parent_run_id=None)
            -> 提升 4 个 key 到 root trace
               (sessionId / userId / name / tags)
```

**Q2.2(深挖)** 合并 metadata 时为什么用 `setdefault` 而不是直接赋值?这个细节有什么讲究?

**参考回答**:`inject_langfuse_metadata()` 对已有 key 用 `setdefault`,即"调用方提供的值优先"——比如前端已经在 metadata 里塞了 `langfuse_session_id`,gateway 注入的值不能覆盖它([metadata.py:102-105](../backend/packages/harness/deerflow/tracing/metadata.py#L102-L105))。如果改成直接赋值,上游(前端/ACP 客户端)显式指定的会话分组会被后端静默改写,排查"为什么两个用户的 trace 串到一个 session"会非常痛苦。这个 helper 被 gateway worker 和 embedded client 两条路径共享,目的就是"两条路径不能 drift"([metadata.py:82-90](../backend/packages/harness/deerflow/tracing/metadata.py#L82-L90);[worker.py:241-246](../backend/packages/harness/deerflow/runtime/runs/worker.py#L241-L246))。

**Q2.3(边界/异常)** 如果 Langfuse 没开,只开了 LangSmith,这套注入逻辑会不会污染 LangSmith 的 metadata?

**参考回答**:不会。`build_langfuse_trace_metadata()` 第一行就检查 `"langfuse" not in get_enabled_tracing_providers()` 并返回 `{}`([metadata.py:51-52](../backend/packages/harness/deerflow/tracing/metadata.py#L51-L52)),`inject_langfuse_metadata()` 拿到空 dict 直接 return,完全不碰 `config["metadata"]`([metadata.py:99-100](../backend/packages/harness/deerflow/tracing/metadata.py#L99-L100))。这就是注释里说的"callers can unconditionally merge"——调用方不需要自己判断 provider 开关。反例:如果让调用方各自判断,gateway 和 client 两条路径迟早有一边忘了同步,trace 属性就会出现"这个入口有 session、那个入口没有"的不一致。

---

## 问题链 3:GuardrailMiddleware —— deny 转 error ToolMessage 与 fail-closed

**Q3.1(基础)** 你们的护栏(guardrail)是在哪个环节介入的?拦截一个工具调用后系统怎么继续走?

**参考回答**:护栏以 `AgentMiddleware` 形式挂在 `wrap_tool_call`/`awrap_tool_call` 上,在每次工具执行前调用 provider 的 `evaluate()`/`aevaluate()` 拿到 allow/deny 判定([middleware.py:54-75](../backend/packages/harness/deerflow/guardrails/middleware.py#L54-L75))。deny 时不抛异常,而是构造一条 `status="error"` 的 `ToolMessage` 返回,内容包含 reason code 和 "Choose an alternative approach." 提示([middleware.py:42-52](../backend/packages/harness/deerflow/guardrails/middleware.py#L42-L52))。这样 tool_call 有对应的 ToolMessage,agent 循环继续,LLM 能看到"被拒了"并换方案。

**链路解析**:

```
LLM 发出 tool_call
  -> GuardrailMiddleware.wrap_tool_call(request, handler)
       -> _build_request(): GuardrailRequest(tool_name, args, agent_id=passport, ts)
       -> provider.evaluate(gr)
            +-- allow=True  -> handler(request) -> 正常执行工具
            +-- allow=False -> _build_denied_message()
            |                  ToolMessage(status="error",
            |                    content="Guardrail denied: ... (oap.tool_not_allowed)...")
            |                  -> 回到 agent 循环,LLM 读到拒绝原因
            +-- provider 抛异常 -> fail_closed=True: 按 deny 处理
                                 fail_closed=False: 放行 + warning 日志
```

**Q3.2(深挖)** 为什么 deny 要返回 error ToolMessage 而不是直接 raise?抛异常不是更"硬"吗?

**参考回答**:抛异常会留下一个没有对应 ToolMessage 的 dangling tool_call,直接破坏 LangGraph 的消息配对不变量,agent 循环会崩或进入修复路径;而 error ToolMessage 让拒绝成为 LLM 可读的上下文,agent 能"adapt"——换工具、降级、或向用户解释([middleware.py:21-27](../backend/packages/harness/deerflow/guardrails/middleware.py#L21-L27))。**不这样设计会怎样**:raise 等于把"策略性拒绝"和"系统性故障"混为一谈,LLM 失去自我修正机会,一次误拦就终结整个 run;而且 deny 日志里已经带了 `policy_id` 和 reason code 供审计([middleware.py:73](../backend/packages/harness/deerflow/guardrails/middleware.py#L73)),可观测性并不依赖异常。这正是 guardrail 和 circuit breaker 的本质区别:前者是业务判定,要给 agent 留活路;后者是故障隔离,才需要中断。消息文案也是设计过的:`"Choose an alternative approach."` 直接给 LLM 下指令,而不是只陈述事实([middleware.py:48](../backend/packages/harness/deerflow/guardrails/middleware.py#L48))——拒绝消息本质是一段注入给模型的 prompt,写法要按 prompt 工程的标准来。

**链路解析**:

```
方案对比:
  raise GuardrailDenied
    -> ToolNode 不产出 ToolMessage -> tool_call_id 悬空
    -> 下轮 LLM 输入校验失败 / 需 DanglingToolCallMiddleware 兜底修复
    -> 模型永远不知道"为什么被拒",无法调整策略

  return ToolMessage(status="error")
    -> tool_call_id 配对完整,graph 状态机继续
    -> 模型读到: "Guardrail denied: tool 'shell' was blocked
       (oap.tool_not_allowed). Reason: ... Choose an alternative approach."
    -> 模型换用 file_write 等被允许的工具继续任务
```

**Q3.3(边界/异常)** provider 自己抛异常怎么办?为什么要单独 catch `GraphBubbleUp` 再 re-raise?

**参考回答**:两层处理。第一层:`GraphBubbleUp` 是 LangGraph 的控制流信号(interrupt/pause/resume),必须原样 re-raise,否则护栏会把"人工审批中断"误判成 provider 故障,fail-closed 分支会把它吞成一条 deny 消息,整个 HITL 流程就断了([middleware.py:63-65](../backend/packages/harness/deerflow/guardrails/middleware.py#L63-L65))。第二层:其他 Exception 按 `fail_closed` 分流——默认 `True` 时构造 `oap.evaluator_error` 的 deny 判定(宁可错杀),`False` 时记 warning 后放行([middleware.py:66-71](../backend/packages/harness/deerflow/guardrails/middleware.py#L66-L71))。`fail_closed` 默认 `True` 是有意的安全默认:护栏存在的前提是"不可信输入宁可拦错",fail-open 只在 provider 是装饰性审计时才合理([guardrails_config.py:21-23](../backend/packages/harness/deerflow/config/guardrails_config.py#L21-L23))。异步路径 `awrap_tool_call` 逻辑完全对称([middleware.py:77-98](../backend/packages/harness/deerflow/guardrails/middleware.py#L77-L98))。

---

## 问题链 4:GuardrailProvider 插件化与内置 AllowlistProvider

**Q4.1(基础)** 护栏策略是怎么配置的?我要接一个自己的 OAP 服务或者自研鉴权,需要改框架代码吗?

**参考回答**:不用改框架。config.yaml 里 `guardrails.provider.use` 写一个类路径(如 `deerflow.guardrails.builtin:AllowlistProvider`),运行时 `resolve_variable()` 反射加载,`provider.config` 字典作为 kwargs 传给构造函数([tool_error_handling_middleware.py:160-181](../backend/packages/harness/deerflow/agents/middlewares/tool_error_handling_middleware.py#L160-L181))。provider 只需满足 `GuardrailProvider` Protocol——`@runtime_checkable` 的鸭子类型协议,有 `name`、`evaluate()`、`aevaluate()` 即可,不需要继承任何基类([provider.py:39-56](../backend/packages/harness/deerflow/guardrails/provider.py#L39-L56))。加载时还会用 `inspect.signature` 探测构造函数是否接受 `framework` 参数,接受就注入 `"deerflow"` 提示,内置 provider 不需要所以不注入([tool_error_handling_middleware.py:170-179](../backend/packages/harness/deerflow/agents/middlewares/tool_error_handling_middleware.py#L170-L179))。

**链路解析**:

```
config.yaml:
  guardrails:
    enabled: true
    fail_closed: true
    provider:
      use: "my_company.oap:OAPProvider"
      config: {endpoint: "...", timeout: 3}
        |
        v
_build_runtime_middlewares()
  -> resolve_variable("my_company.oap:OAPProvider")   # importlib + getattr
  -> provider_cls(**config)                            # 实例化
  -> middlewares.append(GuardrailMiddleware(provider, fail_closed, passport))
  -> 每次 tool_call 前 evaluate()
```

**Q4.2(深挖)** 内置的 AllowlistProvider 判定逻辑讲一下?`allowed_tools` 和 `denied_tools` 同时配了谁先谁后?

**参考回答**:allowlist 优先。`allowed_tools` 非空时,不在名单里的工具直接 deny(`oap.tool_not_allowed`);通过 allowlist 后再查 denylist,命中也 deny;都通过才 allow([builtin.py:15-20](../backend/packages/harness/deerflow/guardrails/builtin.py#L15-L20))。语义上 `allowed_tools=None` 表示"不启用白名单"(存成 `None`),而 `denied_tools=None` 归一化成空 `set`,两种"没配"的表示不同是因为白名单需要区分"未启用"和"空名单(全拒)"([builtin.py:11-13](../backend/packages/harness/deerflow/guardrails/builtin.py#L11-L13))。`aevaluate` 直接委托同步 `evaluate`,因为判定是纯内存集合查询,没有 IO([builtin.py:22-23](../backend/packages/harness/deerflow/guardrails/builtin.py#L22-L23))。

**Q4.3(边界/异常)** 决策对象里 `reasons` 是列表,但构造 denied 消息时只取 `reasons[0]`,空 reasons 会怎样?多 reason 的信息不是丢了吗?

**参考回答**:空 reasons 有兜底:code 回退 `"oap.denied"`,message 回退 `"blocked by guardrail policy"`([middleware.py:45-46](../backend/packages/harness/deerflow/guardrails/middleware.py#L45-L46))。只取 `reasons[0]` 是给 LLM 读的妥协——拒绝上下文要短,模型只需要一个可行动的理由;完整 reasons 列表仍在 `GuardrailDecision` 对象里,审计侧可以从 provider 日志拿到([provider.py:29-36](../backend/packages/harness/deerflow/guardrails/provider.py#L29-L36))。另外注意 `GuardrailRequest` 里其实预留了 `thread_id`、`is_subagent` 字段但中间件当前没填,`_build_request` 只填了 `tool_name/tool_input/agent_id/timestamp`([middleware.py:34-40](../backend/packages/harness/deerflow/guardrails/middleware.py#L34-L40))——协议先行、填充渐进,面试时主动指出这一点是加分项。

---

## 问题链 5:Uploads —— 安全写入、重名处理与转换流水线

**Q5.1(基础)** 用户往一个 thread 上传文件,完整链路是怎样的?有哪些默认限制?

**参考回答**:网关 `POST /api/threads/{thread_id}/uploads` 接收 multipart,先校验数量,再逐文件:`normalize_filename` 消毒文件名 → `claim_unique_filename` 处理同请求内重名 → `open_upload_file_no_symlink` 安全打开 → 按 8192 字节 chunk 流式写入并实时累计大小([uploads.py:254-273](../backend/app/gateway/routers/uploads.py#L254-L273);[uploads.py:166-195](../backend/app/gateway/routers/uploads.py#L166-L195))。默认限制:单次最多 10 个文件、单文件 50 MB、总计 100 MB,超限返回 413([uploads.py:36-39](../backend/app/gateway/routers/uploads.py#L36-L39))。写完后统一补 group/other 读位让 sandbox 容器内非 root 进程可读,非挂载模式还要 `sandbox.update_file` 同步进沙箱([uploads.py:322-335](../backend/app/gateway/routers/uploads.py#L322-L335))。落盘位置由 `Paths` 单例决定:host 侧是 `{base}/users/{user_id}/threads/{thread_id}/user-data/uploads/`,sandbox 内看到的是 `/mnt/user-data/uploads/`(`VIRTUAL_PATH_PREFIX`),`resolve_virtual_path` 负责把虚拟路径解回 host 路径并做段边界匹配 + traversal 检查([paths.py:11](../backend/packages/harness/deerflow/config/paths.py#L11);[paths.py:254-260](../backend/packages/harness/deerflow/config/paths.py#L254-L260);[paths.py:346-380](../backend/packages/harness/deerflow/config/paths.py#L346-L380))。

**链路解析**:

```
POST /api/threads/{tid}/uploads (multipart)
  -> 数量检查 (默认 max 10)
  -> ensure_uploads_dir(tid)   # 校验 thread_id 字符集
  -> for each file:
       normalize_filename()        # basename、拒 ".."、拒反斜杠、<=255 字节
       claim_unique_filename()     # 同请求重名 -> name_1.ext
       open_upload_file_no_symlink # O_NOFOLLOW / 双 lstat + fstat, 0o600
       流式写 (8192B/chunk, 单文件>50MB 或 总计>100MB -> 413 + 清理)
       [auto_convert_documents=true 且 .pdf/.docx/...]
           -> convert_file_to_markdown() -> 伴生 .md
  -> chmod 加组/他人读位 -> (非挂载模式) sandbox.update_file 同步
```

**Q5.2(深挖)** `open_upload_file_no_symlink` 防的是什么攻击?POSIX 和 Windows 实现为什么不一样?

**参考回答**:防的是"沙箱预置 symlink 劫持写入"。uploads 目录会挂载进本地 sandbox,sandbox 里的进程可以提前在未来上传文件名处放一个指向外部的 symlink;普通 `write_bytes` 会跟随 symlink,用 gateway 权限改写 uploads 目录外的文件([manager.py:118-128](../backend/packages/harness/deerflow/uploads/manager.py#L118-L128))。POSIX 上用 `O_NOFOLLOW` 打开,目标是 symlink 时 `open()` 直接失败(ELOOP),再用 `fstat` 确认是普通文件且 `st_nlink == 1`(独占硬链接),最后 `ftruncate(0)`([manager.py:143-168](../backend/packages/harness/deerflow/uploads/manager.py#L143-L168))。Windows 没有 `O_NOFOLLOW`,只能用"打开前 lstat + 打开后 fstat"双检查收窄 TOCTOU 窗口——docstring 坦承这只是显著加大利用难度,无法完全消除竞态([manager.py:170-209](../backend/packages/harness/deerflow/uploads/manager.py#L170-L209))。**不这样设计会怎样**:一个能在 sandbox 里执行代码的 agent(本来就是 DeerFlow 的正常用法)可以通过 `ln -s ~/.ssh/authorized_keys /mnt/user-data/uploads/evil.txt` 等待用户上传同名文件,实现容器逃逸到宿主文件写。

**链路解析**:

```
POSIX 防线 (open_upload_file_no_symlink):
  lstat(dest)          -> 已存在且非普通文件? 拒绝 (目录/symlink/设备)
  validate_path_traversal(dest, base) -> resolve 后必须仍在 base 内
  os.open(dest, O_WRONLY|O_CREAT|O_NOFOLLOW|O_NONBLOCK, 0o600)
      -> symlink 触发 ELOOP -> UnsafeUploadPathError
  fstat(fd)            -> S_ISREG 且 st_nlink == 1 (防硬链接共享)
  ftruncate(fd, 0)     -> 清空旧内容
       |
       v
  Windows 退化防线:
  打开前 lstat (检查+st_nlink>1 拒绝) -> os.open -> 打开后 fstat 复查
  => 两次检查之间仍有 TOCTOU 缝隙, docstring 明说 "cannot prevent
     an attacker who can atomically replace dest with a symlink"
```

**Q5.3(深挖)** 同一次请求传了两个同名文件会怎样?`claim_unique_filename` 为什么只管"本次请求内"的重名?

**参考回答**:重名文件会被改名为 `stem_N.suffix`,从 `_1` 开始递增直到不与 `seen` 集合冲突,返回前已自动加入 `seen`([manager.py:81-103](../backend/packages/harness/deerflow/uploads/manager.py#L81-L103))。`seen_filenames` 是请求级集合,只防止同批 multipart 里后一个文件静默截断前一个;而历史已存在的同名文件保持"覆盖"语义,注释明确说这是保留的历史行为([uploads.py:239-242](../backend/app/gateway/routers/uploads.py#L239-L242))。`normalize_filename` 先做基础消毒:取 basename、拒绝 `"."`/`".."`、拒绝反斜杠(Linux 下 `Path.name` 会把 `..\x` 当普通字符)、UTF-8 编码后超 255 字节拒绝([manager.py:53-78](../backend/packages/harness/deerflow/uploads/manager.py#L53-L78))。thread_id 也有独立白名单正则 `^[a-zA-Z0-9._-]+$`,从源头挡住路径分隔符([manager.py:26-37](../backend/packages/harness/deerflow/uploads/manager.py#L26-L37))。

**Q5.4(边界/异常)** 上传 PDF 后的 markdown 转换是怎么做的?为什么 `auto_convert_documents` 默认是关的?

**参考回答**:转换是双策略:auto 模式下先试 pymupdf4llm(标题识别更好),输出太稀疏——每页不足 50 字符,或拿不到页数时总量不足 200 字符——判定为图片型 PDF,回退 MarkItDown([file_conversion.py:40-47](../backend/packages/harness/deerflow/utils/file_conversion.py#L40-L47);[file_conversion.py:105-135](../backend/packages/harness/deerflow/utils/file_conversion.py#L105-L135))。大于 1 MB 的文件用 `asyncio.to_thread` 丢线程池,避免阻塞事件循环(修复 issue #1569)([file_conversion.py:37-40](../backend/packages/harness/deerflow/utils/file_conversion.py#L37-L40);[file_conversion.py:151-158](../backend/packages/harness/deerflow/utils/file_conversion.py#L151-L158))。`auto_convert_documents` 默认 `False` 是刻意的安全默认:MarkItDown/pymupdf 解析用户上传的任意文档属于"重解析攻击面",host 侧自动转换要运维显式 opt-in([uploads.py:198-210](../backend/app/gateway/routers/uploads.py#L198-L210))。转换失败不阻塞上传,记 error 日志返回 `None`,原文件保留([file_conversion.py:165-167](../backend/packages/harness/deerflow/utils/file_conversion.py#L165-L167));删除原文件时伴生 `.md` 会被一并清理([manager.py:279-281](../backend/packages/harness/deerflow/uploads/manager.py#L279-L281))。

**链路解析**:

```
convert_file_to_markdown(pdf):
  file_size > 1MB ? --yes--> asyncio.to_thread(_do_convert)  (不阻塞事件循环)
                   \--no---> _do_convert 直接跑
  _do_convert:
    .pdf 且 converter != "markitdown"
      -> pymupdf4llm.to_markdown()
           输出/页数 >= 50 chars/page ? -> 用 pymupdf 结果
           否则 (图片型/加密)           -> 回退 MarkItDown
    其他格式 (.docx/.pptx/.xlsx...)   -> 直接 MarkItDown
  成功 -> 写伴生 file.md;失败 -> log error, 返回 None, 上传本身不受影响
```

---

## 问题链 6:社区工具统一模式 —— tavily / jina / firecrawl

**Q6.1(基础)** 你们支持好几家搜索/抓取服务,接入层是怎么统一的?模型侧看到的是几个工具?

**参考回答**:模型侧永远只看到 `web_search` 和 `web_fetch` 两个工具,每家 provider 用自己的包实现同名工具,通过 config.yaml 的 `use` 类路径选一家装载([tavily/tools.py:17](../backend/packages/harness/deerflow/community/tavily/tools.py#L17);[jina_ai/tools.py:44](../backend/packages/harness/deerflow/community/jina_ai/tools.py#L44))。三个 provider 连 docstring 都逐字相同——包括"只抓用户给过的或搜索结果里出现过的 EXACT URL"、"不能访问登录墙内容"这些行为约束,保证换 provider 时 prompt 稳定性([tavily/tools.py:43-53](../backend/packages/harness/deerflow/community/tavily/tools.py#L43-L53);[firecrawl/tools.py:49-59](../backend/packages/harness/deerflow/community/firecrawl/tools.py#L49-L59))。配置都从 `get_app_config().get_tool_config("web_search")` 读,额外参数(api_key、max_results)走 pydantic `model_extra`([tavily/tools.py:24-27](../backend/packages/harness/deerflow/community/tavily/tools.py#L24-L27))。

**链路解析**:

```
config.yaml: tools: [{name: web_search, use: "deerflow.community.tavily.tools:web_search_tool", api_key: $TAVILY_API_KEY}]
        |
        v
resolve_variable(use, BaseTool)  # 装载其中一家的 @tool("web_search")
        |
        +-- tavily:   client.search(query, max_results=5)
        +-- firecrawl: client.search(query, limit=max_results) (try/except 包全体)
        +-- jina(fetch): JinaClient.crawl(html) -> Readability -> markdown
        v
统一输出: search -> JSON [{title,url,snippet}]  /  fetch -> "# title\n\n" + content[:4096]
```

**Q6.2(深挖)** 三家的 `web_fetch` 实现差异在哪?为什么 jina 版本要把 Readability 抽取丢进 `asyncio.to_thread`?

**参考回答**:tavily 用自家 `extract` API 直接返回 `raw_content` 并截到 4096 字符([tavily/tools.py:54-62](../backend/packages/harness/deerflow/community/tavily/tools.py#L54-L62));firecrawl 用 `scrape(formats=["markdown"])` 同样截 4096([firecrawl/tools.py:60-73](../backend/packages/harness/deerflow/community/firecrawl/tools.py#L60-L73));jina 是自己爬 HTML 后本地用 `ReadabilityExtractor` 抽正文再转 markdown,`asyncio.to_thread` 是因为 Readability 是 CPU 密集的同步解析,直接 await 会卡住事件循环([jina_ai/tools.py:64-68](../backend/packages/harness/deerflow/community/jina_ai/tools.py#L64-L68))。jina 版本还有一套配置 coercion:`timeout` 默认 10 秒、字符串 `"30"` 也能转 int,`trust_env` 默认 `True`,布尔字符串 `{"1","true","yes","on"}` 都算真——配置文件里 YAML 类型写飘了也不会崩([jina_ai/tools.py:12-41](../backend/packages/harness/deerflow/community/jina_ai/tools.py#L12-L41);[jina_ai/tools.py:55-63](../backend/packages/harness/deerflow/community/jina_ai/tools.py#L55-L63))。输出统一截 4096 字符是刻意的 token 预算控制,网页正文无上限,不截会把上下文窗口吃光。

**链路解析**:

```
三家 web_fetch 对比 (输入都是 url, 输出都 <= 4096 字符):
+-----------+---------------------------+------------------------------+
| provider  | 抓取方式                  | 正文抽取                     |
+-----------+---------------------------+------------------------------+
| tavily    | client.extract([url])     | 服务端已完成, 取 raw_content |
| firecrawl | client.scrape(markdown)   | 服务端转好 markdown          |
| jina      | JinaClient.crawl(html)    | 本地 ReadabilityExtractor    |
|           |  (timeout 默认 10s)       |  -> asyncio.to_thread 抽正文 |
+-----------+---------------------------+------------------------------+
错误处理: 全部转 "Error: ..." 字符串返回 (不 raise)
```

**Q6.3(边界/异常)** 抓取失败时这些工具是抛异常还是返回错误字符串?为什么?

**参考回答**:返回 `"Error: ..."` 字符串,不抛。tavily 检查 `failed_results` 数组返回 `f"Error: {error}"`([tavily/tools.py:56-57](../backend/packages/harness/deerflow/community/tavily/tools.py#L56-L57));firecrawl 更彻底,整个函数体包在 try/except 里,任何异常都转成 `f"Error: {str(e)}"`([firecrawl/tools.py:60-71](../backend/packages/harness/deerflow/community/firecrawl/tools.py#L60-L71));jina 的 client 出错时返回以 `"Error:"` 开头的字符串,tool 层原样透传([jina_ai/tools.py:64-66](../backend/packages/harness/deerflow/community/jina_ai/tools.py#L64-L66))。理由和 guardrail 的 deny 转 ToolMessage 一脉相承:工具失败是 LLM 应该"看到并适应"的正常业务信号(换 URL、换查询词),抛异常会把控制权从 agent 手里夺走。**不这样设计会怎样**:一个 404 就让整条 run 中断,用户看到的不是"这个链接抓不到,我换个来源"而是系统报错。

---

## 问题链 7:reflection —— resolve_variable / resolve_class

**Q7.1(基础)** 代码里到处可见 `resolve_variable`,它解决什么问题?

**参考回答**:把配置文件里的字符串类路径(如 `"langchain_openai:ChatOpenAI"`)变成运行时的 Python 对象,是 DeerFlow 所有可插拔组件的统一装载机制——模型、工具、sandbox、guardrail provider、skills storage 都走它([resolvers.py:25-70](../backend/packages/harness/deerflow/reflection/resolvers.py#L25-70))。实现就是 `rsplit(":", 1)` 拆模块路径和变量名,`import_module` + `getattr`,可选 `expected_type` 做 isinstance 校验;`resolve_class` 在此基础上追加 `issubclass` 检查,比如模型必须继承 `BaseChatModel`([resolvers.py:73-95](../backend/packages/harness/deerflow/reflection/resolvers.py#L73-95))。`GuardrailProvider` 的 docstring 也明确说 provider 就是用这个机制加载的,与 models/tools/sandbox 同一套([provider.py:41-46](../backend/packages/harness/deerflow/guardrails/provider.py#L41-L46))。

**链路解析**:

```
config.yaml: use: "langchain_openai:ChatOpenAI"
        |
        v
resolve_class(path, BaseChatModel)
  -> rsplit(":",1) -> ("langchain_openai", "ChatOpenAI")
  -> import_module("langchain_openai")
       +-- ModuleNotFoundError -> 查 MODULE_TO_PACKAGE_HINTS
       |    "langchain_openai" -> "langchain-openai"
       |    报错带 "uv add langchain-openai" 提示
  -> getattr(module, "ChatOpenAI")
       +-- AttributeError -> ImportError("does not define ...")
  -> isinstance(cls, type) 且 issubclass(cls, BaseChatModel)
  -> 返回类对象 -> factory 实例化
```

**Q7.2(深挖)** 导入失败时的报错信息有什么设计?`MODULE_TO_PACKAGE_HINTS` 是干嘛的?

**参考回答**:报错信息被刻意做成"可行动"的:`_build_missing_dependency_hint` 把模块名映射到 pip 包名(模块用下划线、包用连字符,如 `langchain_openai` → `langchain-openai`),直接告诉用户 `uv add langchain-openai` 然后重启([resolvers.py:3-22](../backend/packages/harness/deerflow/reflection/resolvers.py#L3-22))。有个精细的分支:即使 ImportError 是传递依赖触发的(比如缺 `google`),也优先用模块根的 hint,因为用户该装的还是 `langchain-google-genai` 这个集成包([resolvers.py:16-20](../backend/packages/harness/deerflow/reflection/resolvers.py#L16-L20))。只有"根模块本身缺失"才走 hint 路径,其他 ImportError 保留原始消息——区分"缺包"和"包在但导入炸"两种故障([resolvers.py:50-57](../backend/packages/harness/deerflow/reflection/resolvers.py#L50-L57))。这是典型的开发者体验工程:把 Python 生态"模块名≠包名"的坑在框架层抹平。

**Q7.3(边界/异常)** 这个机制有什么风险?配置文件里写任意类路径就能加载任意类,这不是 RCE 吗?

**参考回答**:确实是"配置即代码"的信任模型——能写 config.yaml 的人本来就等价于能执行代码,所以框架不在这一层做防护,这和插件系统(如 pytest 插件、Django settings)是同一假设。框架能做的是收窄失误面:`expected_type`/`base_class` 校验让"路径写错指向了错误类型"在装载期就炸,而不是运行到一半才出诡异行为([resolvers.py:64-68](../backend/packages/harness/deerflow/reflection/resolvers.py#L64-L68));guardrail 装配处还用 `inspect.signature` 探测构造函数签名,`framework` 参数只在被接受时才注入,避免 kwargs 不匹配([tool_error_handling_middleware.py:173-179](../backend/packages/harness/deerflow/agents/middlewares/tool_error_handling_middleware.py#L173-L179))。另一个隐性风险是 import 副作用:`import_module` 会执行模块顶层代码,配置里指向一个有重型顶层逻辑的模块,装载阶段就会付出启动耗时甚至网络调用的代价。面试时如果被追问"怎么加固",方向是:多租户 SaaS 化后配置来自用户而非运维,此时需要类路径白名单(只允许 `deerflow.` 前缀或注册表内的 provider),当前单体部署下这个威胁模型不成立。

---

## 问题链 8:配置热重载与 startup-only 边界

**Q8.1(基础)** 改 config.yaml 需要重启服务吗?`get_app_config()` 是怎么实现热重载的?

**参考回答**:大部分字段不用重启。gateway 每个请求都走 `get_app_config()`,它对比"路径 + 内容签名"(mtime + size + 全文 SHA-256,按 1 MB 块流式哈希),不一致就重新加载并缓存([app_config.py:450-482](../backend/packages/harness/deerflow/config/app_config.py#L450-L482);[app_config.py:419-434](../backend/packages/harness/deerflow/config/app_config.py#L419-L434))。用内容签名而不是只看 mtime,是为了抓住"写回但 mtime 没变"(某些编辑器/同步工具)和"mtime 变了但内容没变"两种情况;签名一致就不重载,避免每请求解析 YAML([app_config.py:471-481](../backend/packages/harness/deerflow/config/app_config.py#L471-L481))。这正是 reload_boundary 模块 docstring 引用的 issue #3144 的语义:per-run 字段下一条消息即生效([reload_boundary.py:1-9](../backend/packages/harness/deerflow/config/reload_boundary.py#L1-L9))。

**链路解析**:

```
每个请求 -> get_app_config()
  -> ContextVar 有 override? -> 直接用 (测试/租户隔离)
  -> resolve_config_path()          # 参数 > env > 项目根 > legacy
  -> stat + sha256 全文 (1MB/chunk) -> (mtime, size, digest)
  -> 与缓存签名比较
       +-- 一致 -> 返回缓存单例
       +-- 不一致 -> AppConfig.from_file() 重载
            -> _apply_singleton_configs()
                 各子系统 load_xxx_config_from_dict()
                 checkpointer 变了 -> reset_checkpointer() + reset_store()
```

**Q8.2(深挖)** 那哪些字段必须重启?框架怎么防止运维改了 `database` 然后疑惑"为什么不生效"?

**参考回答**:8 个 startup-only 字段登记在 `STARTUP_ONLY_FIELDS` 注册表里:`database`、`checkpointer`、`run_events`、`stream_bridge`、`sandbox`、`log_level`、`channels`、`channel_connections`,每条都附带"哪段代码在 startup 捕获了快照"的原因——例如 `database` 是 `init_engine_from_config()` 在启动时跑一次,SQLAlchemy 连接池不会随配置重建([reload_boundary.py:45-62](../backend/packages/harness/deerflow/config/reload_boundary.py#L45-L62))。schema 层的 `Field(description=...)` 用 `format_field_description()` 生成统一的 `"startup-only:"` 前缀,IDE hover 就能看到;而且有 drift 测试双向钉死——注册表里的字段必须带前缀、带前缀的字段必须在注册表里([reload_boundary.py:32-36](../backend/packages/harness/deerflow/config/reload_boundary.py#L32-L36);[reload_boundary.py:80-107](../backend/packages/harness/deerflow/config/reload_boundary.py#L80-L107))。**不这样设计会怎样**:边界靠口口相传,运维改完 `sandbox.use` 发现沙箱没换实现,要么怀疑人生要么重启全站——把"需要重启"从隐性知识变成机器可校验的注册表,是这个模块的核心价值。

**链路解析**:

```
字段改了 config.yaml 之后的两条命运:

  per-run 字段 (models/tools/guardrails/uploads/...)
    -> 下一请求 get_app_config() 签名不符 -> 重载 -> 立即生效

  startup-only 字段 (database/checkpointer/sandbox/...)
    -> AppConfig 对象确实重载了 (对象里是新值)
    -> 但运行时单例早已捕获旧值:
         engine / checkpointer / sandbox provider / logger level
    -> 必须重启进程才生效
    -> Field(description="startup-only: ...") + drift 测试
       双向钉住注册表,防止新增字段漏登记
```

**Q8.3(边界/异常)** 热重载在并发下安全吗?`push_current_app_config`/`pop_current_app_config` 这对 API 是干什么的?

**参考回答**:两个机制解两个问题。第一,`get_app_config` 的热重载是"读签名 → 比对 → 整体替换全局单例",替换是原子赋值,读者要么拿到旧实例要么拿到新实例,不会读到半成品;重载里 `checkpointer` 配置变化会触发 `reset_checkpointer()`/`reset_store()`,因为这两个运行时单例的后端是从 checkpointer 配置派生的([app_config.py:273-280](../backend/packages/harness/deerflow/config/app_config.py#L273-L280))。第二,push/pop 是基于 `ContextVar` 的运行时覆盖栈:push 时把当前值压栈再 set 新值,pop 时恢复栈顶,用于测试或多租户场景在单个执行上下文里临时换配置而不污染全局单例([app_config.py:532-553](../backend/packages/harness/deerflow/config/app_config.py#L532-L553))。`ContextVar` 的选择很关键——它按 async task/线程上下文隔离,天然适合"每个请求一个配置视图"。另外加载侧还有两个防御细节:全注释掉的 YAML section 会被 PyYAML 解析成 `None`,validator 统一归一成 `[]`,保证 `cp config.example.yaml config.yaml` 的首次启动不崩([app_config.py:162-175](../backend/packages/harness/deerflow/config/app_config.py#L162-L175));`config_version` 落后时会向上最多找 5 层目录定位 `config.example.yaml` 比对并提示 `make config-upgrade`([app_config.py:294-337](../backend/packages/harness/deerflow/config/app_config.py#L294-L337))。

---

## 面试官最爱追问的 3 个点

1. **"为什么 tracing 回调必须挂 graph 根?挂模型上不行吗?"** —— 应答策略:先答双重 span,再补更致命的一刀:模型变成嵌套 observation 后 Langfuse handler 的 `propagate_attributes` 不触发,`session_id`/`user_id` 被剥离,Users/Sessions 页面直接没数据;并指出仓库把这条写成 `lead_agent/agent.py` 的模块级 INVARIANT,四个调用点全部 `attach_tracing=False`([agent.py:1-19](../backend/packages/harness/deerflow/agents/lead_agent/agent.py#L1-L19))。
2. **"护栏拒绝了工具调用,agent 怎么继续?"** —— 应答策略:强调"deny 是业务信号不是系统故障",所以转成 `status="error"` 的 ToolMessage 让 LLM 读到 reason code 后自行换路;再补两个边界:provider 抛异常走 fail-closed(默认)、`GraphBubbleUp` 必须原样透传否则 HITL 中断被吞([middleware.py:63-71](../backend/packages/harness/deerflow/guardrails/middleware.py#L63-L71))。
3. **"上传功能的安全威胁模型是什么?"** —— 应答策略:别只答路径校验,要讲清楚"uploads 目录会被挂载进 sandbox,sandbox 内进程不可信"这个前提,所以文件名消毒(basename/反斜杠/255 字节)只是第一层,核心防线是 `O_NOFOLLOW` + `fstat` + `st_nlink==1` 防 symlink 劫持写,以及 Windows 上承认 TOCTOU 窗口只能收窄不能消除([manager.py:118-128](../backend/packages/harness/deerflow/uploads/manager.py#L118-L128))。
