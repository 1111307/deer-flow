# DeerFlow 源码学习指南 —— Agent 岗位面试准备

> 本文档基于 `bytedance/deer-flow` 2.0 后端代码整理，目标是把项目里体现的工程设计，转化成可以在 agent 岗位面试中讲清楚的知识点。每个模块包含：**核心文件**、**设计原理**、**关键代码**、**面试问答**。

> **逐行精读版**：本文件是速查/总览性质的学习指南。更细致的"逐行代码精读"笔记（打开真实源码、一行一行讲为什么这么写、换一种写法会在哪里炸）单独存放在同目录下，按模块编号：
> - [01-state-and-middleware.md](01-state-and-middleware.md) —— Agent 状态管理（`ThreadState` / reducer）+ Middleware 责任链组装
> - [02-loop-detection-and-safety.md](02-loop-detection-and-safety.md) —— 循环检测（LoopDetectionMiddleware）+ 安全兜底（SafetyFinishReasonMiddleware）
> - [03-dangling-tool-call.md](03-dangling-tool-call.md) —— Dangling Tool Call 消息补偿（DanglingToolCallMiddleware）
> - [04-deferred-tool-binding.md](04-deferred-tool-binding.md) —— 延迟工具绑定（tool_search + DeferredToolFilterMiddleware）
> - [05-subagent-delegation.md](05-subagent-delegation.md) —— Sub-Agent 多智能体委派（task_tool + SubagentExecutor + registry）
> - [06-sandbox-execution.md](06-sandbox-execution.md) —— Sandbox 执行环境与安全（抽象接口/Provider 模式 + 虚拟路径三层防御 + LocalSandboxProvider LRU 缓存 + AioSandboxProvider Docker 生命周期编排 + host bash 安全开关）
> - [07-context-engineering.md](07-context-engineering.md) —— Context Engineering：摘要与预算控制（DeerFlowSummarizationMiddleware skill救援/TAG_NOSTREAM + ToolOutputBudgetMiddleware 外部化双路径/便宜预检查 + memory_flush_hook 长期记忆联动 + 中间件装配位置文档落差）
> - [08-long-term-memory.md](08-long-term-memory.md) —— 长期记忆系统（MemoryUpdateQueue防抖/OR合并 + ContextVar跨线程陷阱 + MemoryUpdater同步/异步分发规避#2615 + fail-closed部分更新防御 + FileMemoryStorage原子写 + format_memory_for_injection真实截断逻辑纠正"top 15 facts"文档误传）
> - [09-mcp-integration.md](09-mcp-integration.md) —— MCP 协议集成（client.py配置翻译层 + cache.py懒加载/mtime失效/三分支同步入口 + oauth.py双检锁主动续期 + session_pool.py anyio同任务栈约束/owner-task模式/四阶段get_session/LRU淘汰/四级关闭自死锁规避#3379 + tools.py stdio会话池化选择#3203/cwd·TMPDIR钉住/虚拟路径两层匹配/拦截器链闭包陷阱/user_id:thread_id跨用户隔离 + 三处"检测运行循环"模式为何不抽公共函数 + 文档系统性漏掉两个文件的复杂度）
> - [10-skills-system.md](10-skills-system.md) —— Skills 系统（types public/custom二分 + parser报错行号对齐/allowed-tools三态 + 渐进式加载system prompt只放目录@lru_cache + slash.py保留命令排除 + SkillActivationMiddleware幂等/软链逃逸防御/XML转义/审计哈希 + security_scanner LLM分类+fail-closed兜底 + tool_policy三态白名单合并 + installer zip炸弹/路径穿越/软链/二次校验 + permissions跨UID抹写位 + storage模板方法把校验固化在基类 + skill_manage_tool自演化闸门 + 文档漏掉一半安全边界代码）
> - [11-model-abstraction.md](11-model-abstraction.md) —— 模型抽象层（__init__只导出create_chat_model + resolve_class反射实例化/能力声明+extra透传 + factory把thinking_enabled一个bool翻译成关思考四分支方言 + 三个LangChain默认值补丁stream_usage/chunk_timeout/deep_merge + provider补丁统一套路重写_get_request_payload/_create_chat_result保留reasoning + assistant_payload_replay共享匹配骨架+字段回调/vLLM未收敛重复 + MiniMax删user name一致性坑2013 + claude_provider OAuth Bearer/prompt caching 4断点/思考预算/重试 + credential_loader多来源+过期检查 + attach_tracing避免双重span + 文档漏掉8个provider补丁的统一模式）
> - [12-reliability-practices.md](12-reliability-practices.md) —— 工程可靠性实践（约定即测试的统一思路 + Blockbuster运行时钩子scanned_modules限定业务代码避免假阳性/hookwrapper包setup+call+teardown/回归锚点锁offload/test_gate_smoke元测试守卫者也要被守卫 + test_harness_boundary AST扫import把分层规则变CI gate + reload_boundary单一事实来源注册表+双向漂移检测锁注册表⇄schema一致 + 共同原则把"人易违反/成本高/当下不报错"的约定转成"违反即失败"的确定性检查）
> - [13-sandbox-docker.md](13-sandbox-docker.md) —— 沙箱 Docker 深入（三层抽象Sandbox/Provider/Backend职责分离 + LocalContainerBackend docker run每个坑的处理端口异步重试/容器名冲突adopt/Windows --mount/DooD 0.0.0.0绑定/凭证日志脱敏/macOS Apple Container + AioSandbox HTTP客户端单session锁/ErrorObservation恢复/Fern SDK close链 + AioSandboxProvider warm pool释放≠销毁/replicas软上限只限闲置不限active/idle checker + 跨进程确定性ID+文件锁防容器名冲突 + 孤儿容器启动reconcile兜底进程崩溃泄漏 + 生命周期澄清"一个thread一个容器复用"而非"每次对话新建" + warm pool vs线程池区别 + docker-compose部署拓扑DooD opt-in/单worker约束）
> - [14-sandbox-tools-and-local.md](14-sandbox-tools-and-local.md) —— 沙箱工具与 LocalSandbox（PathMapping路径映射正向/反向解析/防逃逸 + _agent_written_paths只替换agent写的文件 + tools.py 7个工具完整流程验证路径→替换路径→执行→反向替换→截断 + write_file 80KB限制防LLM流式超时 + LocalSandboxProvider LRU缓存vs AioSandboxProvider warm pool轻量vs重量 + SandboxMiddleware lazy init+wrap_tool_call把sandbox_id写进LangGraph state + search.py忽略规则set+正则性能优化 + file_operation_lock WeakValueDictionary防内存泄漏）

---

## 面试问题链（QA 题库，372 问）

> 按"大厂面试追问链"组织：每条问题链从基础问题开始，层层追问到实现细节、设计权衡、异常边界，附 ASCII 链路图和精确行号引用。深读笔记（01-14）讲"怎么实现"，问题链讲"怎么被问、怎么答"。全部 15 篇均由 agent 通读对应模块**全部源码**后撰写，行号引用经第二轮校验 agent 抽查核对。
> - [qa-01-agent-loop-and-state.md](qa-01-agent-loop-and-state.md) —— Agent 主循环与状态管理（7 链 22 问：LangGraph 图模型选型/5 个自定义 reducer/19 组件 middleware 链装配顺序与 after_model 反向 dispatch/静态 prompt+prefix cache/runtime configurable/tracing 回调挂 graph 根 INVARIANT）
> - [qa-02-safety-middlewares.md](qa-02-safety-middlewares.md) —— 安全中间件（7 链 25 问：循环检测 hash/frequency 两层+优雅降级/safety finish reason 截断 tool_calls/dangling 补偿/LLM 错误分类重试 vs 工具错误转 ToolMessage/ClarificationMiddleware Command(goto=END) 殿后）
> - [qa-03-tools-deferred-binding.md](qa-03-tools-deferred-binding.md) —— 工具体系与延迟绑定（8 链 25 问：get_available_tools 四来源/tool_search 省 context/hash-scoped per-thread promotion+merge_promoted/两道闸/fail-closed 三路径一致/sync wrapper 线程池）
> - [qa-04-subagents.md](qa-04-subagents.md) —— Sub-Agent 委派（8 链 25 问：task_tool 后台执行+轮询/持久隔离 event loop 线程模型/Limit 中间件截断强制 MAX=3/status_contract/token 按位置合并/checkpointer=False/1800s+372 轮询+max_turns=150 三层治理/disallowed_tools 防递归）
> - [qa-05-sandbox.md](qa-05-sandbox.md) —— Sandbox 执行环境与安全（8 链 25 问：三层抽象/PathMapping 正反向解析/工具安全流水线/warm pool/确定性 ID+文件锁/孤儿 reconcile/docker run 细节/replicas=3 软上限/DooD/默认禁 host bash）
> - [qa-06-context-engineering.md](qa-06-context-engineering.md) —— Context Engineering（8 链 25 问：summarization 三维触发/TAG_NOSTREAM/skill 救援三层预算/tool output 外部化 12000 字符/tiktoken+CJK 双轨计数/memory_flush_hook 联动）
> - [qa-07-memory.md](qa-07-memory.md) —— 长期记忆（8 链 25 问：防抖队列 30s+user_id 捕获防 ContextVar 陷阱/LLM 事实抽取去重/原子写+fail-closed/注入预算 2000 tokens+0.7 置信度/per-user 隔离迁移）
> - [qa-08-mcp.md](qa-08-mcp.md) —— MCP 协议集成（8 链 25 问：配置翻译/mtime 缓存失效/OAuth 双检锁续期/session pool anyio 同任务约束+owner-task+四级关闭/stdio 池化#3203/虚拟路径两层翻译/拦截器链闭包陷阱）
> - [qa-09-skills.md](qa-09-skills.md) —— Skills 系统（8 链 25 问：渐进式加载+@lru_cache/slash 激活与保留命令/Activation 幂等+软链逃逸防御/security scanner fail-closed/tool_policy 三态/installer zip 炸弹 512MB/storage 模板方法/自演化闸门）
> - [qa-10-models.md](qa-10-models.md) —— 模型抽象层（7 链 25 问：create_chat_model 反射+extra 透传/关思考四分支方言/三个 LangChain 默认值补丁/provider 补丁统一模式保留 reasoning/vLLM 重复代码/Claude OAuth+prompt caching 4 断点/MiniMax error 2013/attach_tracing 防双 span）
> - [qa-11-gateway-runtime.md](qa-11-gateway-runtime.md) —— Gateway 与运行时🆕（8 链 25 问：HTTP→RunManager→LangGraph→SSE 全链路/on_disconnect 断连语义/cancel 内存态 vs 水合 409/multitask_strategy interrupt/rollback/wait 为何不裸 await#3265/auth+CSRF+internal 三层/GATEWAY_WORKERS=1 原因）
> - [qa-12-persistence-migrations.md](qa-12-persistence-migrations.md) —— 持久化与 Schema 迁移🆕（8 链 25 问：混合 bootstrap 三分支/legacy 先 create_all 再 stamp 否则后加基线表永不建/_BASELINE_TABLE_NAMES 守卫/pg_advisory_lock vs SQLite busy_timeout/checkpointer 表过滤/safe_add_column 幂等）
> - [qa-13-channels.md](qa-13-channels.md) —— IM Channels 多平台接入🆕（8 链 25 问：MessageBus pub/sub/channel:chat→thread 映射/Feishu 卡片原地 patch vs Telegram editMessageText 节流/DingTalk AI Card/connect code 600s TTL/单 owner 转移 partial unique index/owner-scoped 文件存储）
> - [qa-14-reliability.md](qa-14-reliability.md) —— 工程可靠性实践（7 链 25 问：Blockbuster scanned_modules 防假阳性/hookwrapper 包 setup+call+teardown/回归锚点/test_gate_smoke 元测试/AST 边界测试/reload_boundary 双向漂移检测）
> - [qa-15-observability-guardrails-misc.md](qa-15-observability-guardrails-misc.md) —— 可观测/护栏/上传/社区工具/反射/配置🆕（8 链 25 问：Langfuse/LangSmith 双挂载+graph 根回调/trace metadata 映射/guardrails 三 provider deny 转 ToolMessage/uploads 转换流水线/community tools 统一模式/resolve_class 风险/config 热重载边界）

---

## 概念补充（外部知识体系，非源码精读）

> - [concept-ai-evaluation.md](concept-ai-evaluation.md) —— AI/Agent 评测体系（为什么 agent 评测难没有标准答案 + Langfuse 整体框架图四大块共享数据底座 + 数据模型关系图Session→Trace→Observation树/Score胶水挂任意层 + 评估闭环九步链路离线experiment→线上judge→bad case回流dataset + 链路分析一judge线上打分rubric四要素组装/三种score类型/三种挂载点 + 链路分析二experiment离线回归按item对比不只看平均分 + 链路分析三人工annotation校准judge一致性检验 + 链路分析四deer-flow接入graph根回调是评测第一公里 + 评测工具全景langfuse 31.6k/promptfoo 23.5k/deepeval 17k/ragas 15k + 面试五问三层+闭环结构）

---

## 读者背景说明

- **Go 开发者**：强烈建议先读 [go-developer-guide.md](go-developer-guide.md) —— Python asyncio 和 Go goroutine 在结构上很像但实现机制完全不同（单线程协作式调度 vs 多线程抢占式调度），这份对照文档把前 13 篇里所有"为什么 Python 要这么写"的设计选择（`asyncio.to_thread` 到处都是、两套锁并存、单 worker 部署、Blockbuster 检测）串起来了。
- **Python 开发者**：可以直接按 01-13 顺序读，遇到并发相关机制如果感觉"这不是显而易见吗"，说明你已经内化了 Python asyncio 的约束。

---

## 目录

1. [项目总览](#0-项目总览)
2. [Agent 状态管理与 Reducer 设计](#1-agent-状态管理与-reducer-设计)
3. [Middleware 责任链](#2-middleware-责任链)
4. [循环检测与安全兜底](#3-循环检测与安全兜底)
5. [Dangling Tool Call 消息补偿](#4-dangling-tool-call-消息补偿)
6. [Tool / Function Calling 与延迟工具绑定](#5-tool--function-calling-与延迟工具绑定)
7. [Sub-Agent 多智能体委派](#6-sub-agent-多智能体委派)
8. [Sandbox 执行环境与安全](#7-sandbox-执行环境与安全)
9. [Context Engineering：摘要与预算控制](#8-context-engineering摘要与预算控制)
10. [长期记忆系统](#9-长期记忆系统)
11. [MCP 协议集成](#10-mcp-协议集成)
12. [Skills 系统](#11-skills-系统)
13. [模型抽象层](#12-模型抽象层)
14. [工程可靠性实践](#13-工程可靠性实践)
15. [高频面试问题速查表](#14-高频面试问题速查表)
16. [建议学习路径](#15-建议学习路径)

---

## 0. 项目总览

DeerFlow 是基于 **LangGraph + LangChain** 构建的 "super agent harness"：一个主 agent（lead agent）通过一条 **19 级 middleware 责任链** 组装能力，配合 **sandbox 执行环境**、**sub-agent 委派**、**skills/MCP 扩展**、**长期记忆**，处理从几分钟到几小时的复杂任务。

```
Nginx(2026) → Gateway(8001, FastAPI + 内嵌 LangGraph runtime) → Lead Agent (LangGraph graph)
                                                                    │
                                                    ┌───────────────┼───────────────┐
                                                Middleware Chain   Tools         Sub-Agents
                                              (19 个横切关注点)   (sandbox/MCP/   (task tool →
                                                                  builtin/skill)   executor)
```

代码分两层，边界由测试强制约束：
- **harness**（`backend/packages/harness/deerflow/`，import 前缀 `deerflow.*`）：可发布的 agent 框架
- **app**（`backend/app/`，import 前缀 `app.*`）：FastAPI Gateway + IM 渠道，只能单向依赖 harness

```python
# tests/test_harness_boundary.py 的核心断言（简化）
assert not any(imports_from("app") for module in walk("packages/harness/deerflow"))
```

**面试问答**
- Q: 为什么要把 harness 和 app 拆开，而不是一个包全放在一起？
  A: harness 是要独立发布复用的框架层（不依赖具体业务），app 是绑定 FastAPI/IM 渠道的应用层。单向依赖 + CI 测试锁定边界，防止"框架层偷偷耦合业务细节"这种架构腐化，在团队协作中比 code review 更可靠。

---

## 1. Agent 状态管理与 Reducer 设计

**文件**：[thread_state.py](backend/packages/harness/deerflow/agents/thread_state.py)

LangGraph 的状态更新默认是"覆盖"或用 `add_messages` 之类内置 reducer 合并。DeerFlow 针对不同字段写了**语义不同的自定义 reducer**：

```python
def merge_sandbox(existing, new):
    """sandbox_id 只能幂等写入；出现两个不同 id 说明生命周期出 bug，直接 fail-closed 抛异常"""
    if new is None: return existing
    if existing is None: return new
    if existing["sandbox_id"] == new["sandbox_id"]: return existing
    raise ValueError(f"Conflicting sandbox state updates: ...")

def merge_artifacts(existing, new):
    """产物列表：合并 + 去重（保序）"""
    return list(dict.fromkeys((existing or []) + (new or [])))

def merge_viewed_images(existing, new):
    """已读图片字典：正常合并；但传入空字典 {} 是"清空"信号，不是"没有更新" —— 需要区分 None 和 {}"""
    if new is None: return existing
    if len(new) == 0: return {}          # 显式清空
    return {**existing, **new}

def merge_promoted(existing, new):
    """deferred tool 的"已提升"工具名单：按 catalog_hash 分区，
    hash 变了就整体替换（防止 catalog 漂移后旧的裸名字指向了不同的工具）"""
```

**设计要点**：每个 reducer 的语义都对应一个真实的并发/一致性问题：
- 多个 sandbox 工具在同一 graph step 里可能都尝试懒初始化并写 `sandbox_id` → 需要幂等合并而不是覆盖
- `artifacts` 是"追加"语义，但同一文件可能被多次生成 → 需要去重
- `viewed_images` 需要区分"这轮没更新"（`None`）和"显式清空"（`{}`），如果用同一个值表示会有歧义
- `promoted`（deferred tool 提升记录）如果 catalog 变了还继续复用旧名单，可能让一个新工具意外获得旧工具的权限

**面试问答**
- Q: LangGraph 默认的 state 更新方式是什么，什么时候必须自定义 reducer？
  A: 默认是最后写入覆盖（除非用 `Annotated[T, reducer]`）。当同一个 key 在同一 step 可能被多个节点/中间件并发写入，且"覆盖"或"简单合并"不能正确表达业务语义时（如去重、幂等校验、区分"未更新"与"清空"），就必须自定义 reducer。
- Q: 如何设计一个"字段是列表，需要合并去重"的 reducer？边界情况有哪些？
  A: 用 `dict.fromkeys` 保序去重；边界情况：`existing`/`new` 为 `None`（区分未初始化 vs 空更新）。

---

## 2. Middleware 责任链

**文件**：[middlewares/](backend/packages/harness/deerflow/agents/middlewares/) 目录，装配逻辑在 `tool_error_handling_middleware.py::build_lead_runtime_middlewares` 和 `lead_agent/agent.py::build_middlewares`

这是本项目**最具面试价值**的设计：一个 agent loop 里叠加了 19 个横切关注点，靠严格的**append 顺序**保证正确性（顺序错了会有真实 bug，不是风格问题）：

| # | Middleware | 解决的问题 |
|---|---|---|
| 1 | ThreadDataMiddleware | 按 `user_id/thread_id` 建隔离目录 |
| 2 | UploadsMiddleware | 把新上传文件注入对话 |
| 3 | SandboxMiddleware | 获取 sandbox，写 `sandbox_id` 到 state |
| 4 | **DanglingToolCallMiddleware** | 补全用户中断导致的缺失 ToolMessage（见 §4） |
| 5 | LLMErrorHandlingMiddleware | 把 provider 报错归一化成可恢复的错误 |
| 6 | GuardrailMiddleware | 工具调用前鉴权（pluggable provider） |
| 7 | SandboxAuditMiddleware | 沙箱操作审计日志 |
| 8 | ToolErrorHandlingMiddleware | 工具异常 → error ToolMessage，run 不中断 |
| 9 | SkillActivationMiddleware | `/skill-name` 语法激活对应技能 |
| 10 | SummarizationMiddleware | 接近 token 上限时压缩上下文 |
| 11 | TodoListMiddleware | Plan mode 的任务跟踪 |
| 12 | TokenUsageMiddleware | Token 用量统计 |
| 13 | TitleMiddleware | 首轮对话后自动生成标题 |
| 14 | MemoryMiddleware | 把对话丢进异步记忆更新队列 |
| 15 | ViewImageMiddleware | 视觉模型注入 base64 图片 |
| 16 | DeferredToolFilterMiddleware | 隐藏未提升的 MCP 工具 schema（见 §5） |
| 17 | SubagentLimitMiddleware | 截断超额的并发子任务调用 |
| 18 | LoopDetectionMiddleware | 检测重复调用并强制停止（见 §3） |
| 19 | **ClarificationMiddleware** | 拦截澄清请求并中断 —— **必须最后** |

**为什么顺序不能乱（举两个真实约束）**：
1. Guardrail（#6）必须在真正执行工具（更靠后的 middleware 只是错误处理，实际执行在 LangGraph 的 tools node）**之前**拦截，否则鉴权就没意义。
2. Clarification（#19）必须最后：它拦截 `ask_clarification` 工具调用并用 `Command(goto=END)` 中断整个 graph，如果放前面，后面的 middleware（比如 memory/title）就不会正确执行到。

**面试问答**
- Q: 你怎么理解 middleware 模式在 agent 系统里的作用？和传统 Web 框架的 middleware 有什么共性/差异？
  A: 共性是责任链模式，把横切关注点从核心业务逻辑剥离；差异在于 agent middleware 除了 `before/after` 请求，还要 hook 进模型调用（`wrap_model_call`）本身，因为很多问题（如工具调用配对、循环检测）只能在"即将发给 LLM 的消息列表"这个层面修正，而不是在 HTTP 请求层面。
- Q: 如果让你新增一个"敏感词过滤"能力，你会作为独立 middleware还是塞进已有的哪个里？插入顺序放在哪？
  A: 独立 middleware，放在 Guardrail 之后、工具执行之前（如果要过滤工具参数）或者放在 wrap_model_call 里过滤要发给用户的最终文本（如果是过滤输出）。核心是先确定它要拦截的是"入口"还是"出口"再决定插入点。

---

## 3. 循环检测与安全兜底

**文件**：[loop_detection_middleware.py](backend/packages/harness/deerflow/agents/middlewares/loop_detection_middleware.py)

Agent 系统最容易被问到的"P0 safety"问题：**LLM 陷入重复调用同一工具的死循环，直到把 `recursion_limit` 跑爆**。这里做了**两层检测**：

**Layer 1：哈希去重检测**（完全相同的调用集合）
```python
def _hash_tool_calls(tool_calls):
    # 对每个 tool call 提取"稳定 key"（不是全部参数——例如 read_file
    # 按 200 行分桶，避免 start_line 差 1 就被判定为"不同调用"）
    # 排序后 md5 → 12 位哈希，保证跨顺序也能命中同一组合
    ...
```
- 出现 ≥3 次同一哈希 → 注入一次性警告（"你在重复自己，尽快收尾"）
- 出现 ≥5 次 → **强制停止**：清空该 AIMessage 的 `tool_calls`，逼模型只能输出文本

**Layer 2：按工具类型的频率检测**（参数不同但同类工具被疯狂调用，比如轮询读 40 个不同文件）
```python
freq[name] += 1
if tc_count >= eff_hard:   # 默认 50 次
    return "强制停止"
if tc_count >= eff_warn:   # 默认 30 次
    return "警告"
```

**最精妙的工程细节——警告消息该插在哪里**：
- 不能在 `after_model` 直接插入一条消息，因为这时 AIMessage 已经带了 `tool_calls`，但对应的 ToolMessage 还没产生。OpenAI/Moonshot 的校验器要求 "assistant tool_calls 后面必须紧跟着 tool messages"，中间插东西会被 400 拒绝；Anthropic 则不允许在流中间插 SystemMessage。
- 解决方案：**把警告"排队"，延迟到下一次 `wrap_model_call` 时，作为 HumanMessage 追加到整个消息列表最后**（这时所有 ToolMessage 已经在列表里了，配对不受影响）。

```python
def _apply(self, state, runtime):
    warning, hard_stop = self._track_and_check(state, runtime)
    if hard_stop:
        # 直接原地清空 tool_calls，因为清空后不再需要配对的 ToolMessage
        return {"messages": [stripped_msg]}
    if warning:
        self._queue_pending_warning(runtime, warning)   # 排队，不是马上插入
        return None

def wrap_model_call(self, request, handler):
    # 在真正发起下一次模型请求前，把排队的警告追加到消息末尾
    return handler(self._augment_request(request))
```

**面试问答**
- Q: Agent 死循环怎么检测？只按"完全相同的调用"判断够不够？
  A: 不够。还要按"工具类型频率"兜底，因为很多死循环是参数在变但本质在瞎试（比如轮询读文件、反复搜索）。两层检测配合警告 → 硬停止的梯度处理，比直接杀掉 run 更优雅（能拿到部分结果）。
- Q: 给 LLM 对话流插入一条系统级警告消息，你会注意什么？
  A: 消息协议的强约束——工具调用和工具结果消息必须紧邻配对，不同 provider（OpenAI/Anthropic/Moonshot）对"能不能在中间插消息"的容忍度不同。安全的做法是把新消息放在整个消息列表末尾，绝不能插进已有的 tool_calls/tool_result 对之间。

---

## 4. Dangling Tool Call 消息补偿

**文件**：[dangling_tool_call_middleware.py](backend/packages/harness/deerflow/agents/middlewares/dangling_tool_call_middleware.py)

**场景**：用户在 agent 调用工具的过程中中断了请求（或请求被取消），导致消息历史里出现"AIMessage 带 tool_calls，但没有对应 ToolMessage"的悬空调用。下次请求这段历史会被所有主流 provider 拒绝（400）。

**解法**：`wrap_model_call` 时扫描消息历史，为每个悬空的 `tool_call_id` 插入一条合成的 error ToolMessage：

```python
def _build_patched_messages(self, messages):
    # 1. 建立 tool_call_id -> 已有 ToolMessage 的映射
    # 2. 遍历消息，每遇到一条 AIMessage 的 tool_calls，
    #    紧跟着补上：已有的 ToolMessage（如果有）或合成的错误 ToolMessage（如果没有）
    # 3. 还处理 invalid_tool_calls（provider 序列化失败的畸形调用，比如超大 write_file 内容
    #    里带了没转义的引号——这类"看起来发出去了但没执行"的调用也要补偿）
```

还处理了一个隐藏坑：`write_file` 因为携带大段 Markdown 导致 JSON 解析失败时，不能把原始报错内容（可能几十 KB）原样回填进合成消息——那样等于把坏数据又喂回给模型，所以做了 500 字符的截断防御。

**面试问答**
- Q: 用户中断请求后，为什么下一轮对话会报错？怎么修？
  A: 因为 LLM 的消息协议要求 tool_calls 和 tool 执行结果严格配对，中断导致"发了调用但没结果"。修法是在发起下一次模型请求前，扫描历史插入占位的 error ToolMessage，让协议形式合法，同时让模型知道"这次调用被打断了，没有拿到结果"。
- Q: 合成的错误消息内容要注意什么？
  A: 不能把原始的、可能巨大或畸形的错误 payload 原样回填（会污染上下文甚至再次触发同样的解析错误），要做长度截断和结构化提示，告诉模型"换一种方式重试"而不是"重复同样的失败调用"。

---

## 5. Tool / Function Calling 与延迟工具绑定

**文件**：[tools/tools.py](backend/packages/harness/deerflow/tools/tools.py)、[tools/builtins/tool_search.py](backend/packages/harness/deerflow/tools/builtins/tool_search.py)、[deferred_tool_filter_middleware.py](backend/packages/harness/deerflow/agents/middlewares/deferred_tool_filter_middleware.py)

**问题**：当接入多个 MCP server 后，工具数量可能几十上百个，全部塞进 system prompt / bound tools 会：
1. 占用大量 context
2. 让模型在选择工具时准确率下降

**解法——"延迟工具绑定"（Deferred Tool）**：
1. 启动时把大部分 MCP 工具标记为 `deferred`，**不**绑定到模型的 tool schema 上，只在 `<available-deferred-tools>` 里列出"名字 + 一句话描述"
2. 模型需要某个工具时先调用 `tool_search`（本身是一个 bound tool），传入查询词
3. 命中后，把该工具的完整 schema"提升"（promote）到 `ThreadState.promoted`（按 `catalog_hash` 分区，见 §1 的 `merge_promoted`）
4. 下一轮 `DeferredToolFilterMiddleware` 检查 `promoted`，把已提升的工具正式绑定给模型
5. Sub-agent 也走一样的机制，且每次 task run 都是全新 `ThreadState`，提升状态互不干扰

**面试问答**
- Q: 工具太多导致模型选错/context 爆炸怎么办？
  A: 核心思路是"按需绑定"——先不把 schema 都给模型看，只给目录（名字+摘要），模型需要时通过一次"检索"调用换取具体 schema 的绑定权限，下一轮才真正可调用。本质是把"工具选择"从"一次性静态列表"变成"动态检索 + 提升"的两阶段过程。
- Q: 这套机制怎么保证"过期目录"不会导致提升了错误的工具？
  A: 用 `catalog_hash` 给提升记录打版本戳，工具目录变了（增删/改名），hash 就变了，旧的 promoted 名单整体失效，不会出现"名字复用指向了不同工具"的安全问题。

---

## 6. Sub-Agent 多智能体委派

**文件**：[subagents/executor.py](backend/packages/harness/deerflow/subagents/executor.py)、[subagents/registry.py](backend/packages/harness/deerflow/subagents/registry.py)

**架构**：
```
lead agent 调用 task(description, prompt, subagent_type)
        ↓
SubagentExecutor：双线程池设计
  _scheduler_pool（3 workers）— 负责调度/排队
  _execution_pool（3 workers）— 负责真正跑子 agent 的 graph
        ↓
后台线程执行，主线程每 5s 轮询 + 发送 SSE 事件
task_started → task_running → task_completed/task_failed/task_timed_out
        ↓
结果结构化返回给 lead agent（不是原始 transcript）
```

**关键设计点**：
- **上下文隔离**：每个子 agent 有独立的 `ThreadState`，看不到主 agent 或其他子 agent 的上下文——只聚焦分配到的任务
- **并发限流**：`MAX_CONCURRENT_SUBAGENTS = 3`，由 `SubagentLimitMiddleware` 在 `after_model` 截断超额的 `task` 工具调用（而不是让请求失败）
- **一次性执行、不可恢复**：子 agent 的 graph 编译时 `checkpointer=False`，因为子任务是一次性的，永远不会被 resume，不需要背 checkpoint 的开销
- **超时/轮次上限**：默认超时 30 分钟，`general-purpose` 内置子 agent `max_turns=150`（专门为深度研究类子任务调大，避免过早触发 `GraphRecursionError`）

**面试问答**
- Q: 主 agent 怎么把一个复杂任务拆给多个子 agent 并行执行，又不互相干扰？
  A: 每个子 agent 独立 state + 独立 graph 实例，通过 `task` 工具传入的 description/prompt 是唯一输入，执行在独立线程池里，通过轮询 + 事件回调的方式异步汇报进度，最终把结构化结果（不是完整对话历史）返回给主 agent，主 agent 再决定如何整合。
- Q: 并发子任务怎么防止资源被打爆？
  A: 双重限流——线程池大小本身是硬上限，同时 middleware 层面在模型输出阶段就截断超额的调度请求，避免"模型一次性发起 20 个子任务"这种滥用场景。
- Q: 为什么子 agent 不需要 checkpointer？
  A: Checkpointer 是为了支持"中断后恢复"，但子任务的生命周期是"发起 → 跑完 → 返回结果"，从设计上就不支持恢复，去掉能省掉每步的持久化开销。

---

## 7. Sandbox 执行环境与安全

**文件**：[sandbox/sandbox.py](backend/packages/harness/deerflow/sandbox/sandbox.py)（抽象接口）、[sandbox/sandbox_provider.py](backend/packages/harness/deerflow/sandbox/sandbox_provider.py)、[sandbox/local/local_sandbox_provider.py](backend/packages/harness/deerflow/sandbox/local/local_sandbox_provider.py)、[sandbox/security.py](backend/packages/harness/deerflow/sandbox/security.py)

**Provider 模式**：统一接口 `execute_command / read_file / write_file / list_dir`，屏蔽三种实现：
- `LocalSandboxProvider`：直接在宿主机执行，per-thread LRU 缓存（默认 256 条）
- `AioSandboxProvider`：Docker 隔离，带健康检查（"检测失败"视为 unknown 而非 dead，避免误杀正常容器）
- Docker + K8s（通过 provisioner）

**虚拟路径系统**（面试常问的"agent 怎么读写文件又不越权"）：
- Agent 眼里只有 `/mnt/user-data/{workspace,uploads,outputs}` 和 `/mnt/skills`
- 物理路径是 `backend/.deer-flow/users/{user_id}/threads/{thread_id}/user-data/...`
- `LocalSandboxProvider` 在 `acquire()` 时按线程构建 `PathMapping`；`tools.py` 里的 `replace_virtual_path()` 做二次防御（既做路径翻译也做校验）
- Docker 模式下则是把宿主机目录 volume mount 到容器里的相同虚拟路径——**两种实现方式不同，但 agent 侧的接口完全一致**

**面试问答**
- Q: 让 LLM 有能力执行 shell 命令/读写文件，你怎么设计防止它跑出预期范围（路径穿越、越权访问其他用户数据）？
  A: 核心是"虚拟路径 + 强制翻译层"：agent 永远只能看到一套固定的虚拟路径（不知道物理路径长什么样），真正读写前统一做路径翻译+校验（拒绝 `..` 穿越、拒绝解析到隔离目录之外）。再加一层是 per-thread/per-user 的物理目录天然隔离，即使翻译逻辑有 bug，最坏情况也只能碰到同一用户自己的数据。
- Q: Local 和 Docker 两种 sandbox 实现，怎么让上层 agent 代码不用关心区别？
  A: 抽象出统一 `Sandbox` 接口 + `SandboxProvider` 的 acquire/get/release 生命周期，两种实现各自处理"怎么把虚拟路径映射到物理位置"，对 agent/tools 层完全透明。这是经典的策略模式（Strategy Pattern）。

---

## 8. Context Engineering：摘要与预算控制

**文件**：[middlewares/summarization_middleware.py](backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py)、[middlewares/tool_output_budget_middleware.py](backend/packages/harness/deerflow/agents/middlewares/tool_output_budget_middleware.py)、[memory/summarization_hook.py](backend/packages/harness/deerflow/agents/memory/summarization_hook.py)

长任务的核心矛盾：**上下文窗口有限，但任务链条可能很长**。DeerFlow 用了两层手段：

1. **对话级摘要**（`SummarizationMiddleware`）：按 token 数/消息数/占最大输入的比例触发，保留最近若干消息原文，把更早的历史压缩成摘要。
2. **单次工具输出预算**（`ToolOutputBudgetMiddleware`）：防止一次 `bash`/`read_file` 直接把几十 KB 输出灌进上下文——即使总对话没到摘要阈值，单个工具输出本身就要有硬上限。

**面试问答**
- Q: 长任务（几十轮工具调用）怎么防止上下文溢出？
  A: 分两层控制：入口层面限制单次工具输出体量（防止一次调用就把预算打爆），对话层面到达阈值后做摘要压缩（保留最近消息 + 摘要旧消息），本质是"流量控制 + 定期压缩"的组合，类似 TCP 滑动窗口 + 日志 compaction 的思路。
- Q: 摘要压缩会不会丢失关键信息导致后续决策出错？怎么权衡？
  A: 通常保留策略是"最近 N 轮原文 + 更早历史摘要"，摘要 prompt 会显式要求保留关键决策、产物路径、未完成的 todo，而不是无差别精简。这是一个准确性 vs 上下文成本的权衡，没有免费的解法，只能靠摘要 prompt 质量和保留窗口大小去调。

---

## 9. 长期记忆系统

**文件**：[memory/queue.py](backend/packages/harness/deerflow/agents/memory/queue.py)、[memory/updater.py](backend/packages/harness/deerflow/agents/memory/updater.py)、[memory/storage.py](backend/packages/harness/deerflow/agents/memory/storage.py)

**完整流程**：
```
MemoryMiddleware（每轮对话结束）
   → 捕获 user_id（此时用 get_effective_user_id()，因为 ContextVar 不会跨线程传播）
   → queue.add(thread_id, messages, user_id, ...)
       → 按 (thread_id, user_id, agent_name) 做 debounce key
       → 30s 内同一 key 的多次更新会合并（取最新 messages，但 correction/reinforcement 信号是"或"逻辑合并，不会被覆盖丢失）
       → threading.Timer 定时器到点触发 _process_queue()
           → 后台线程调 LLM 抽取事实/更新上下文摘要
           → 原子写入（temp file + rename），跳过重复事实（去重）
   → 下次对话开始时，注入 top 15 facts + 上下文摘要到 system prompt 的 <memory> 标签
```

**两个容易被问到的工程细节**：
1. **为什么要在 enqueue 时就捕获 `user_id`，而不是在处理时读**？
   因为处理是 `threading.Timer` 触发的独立线程，Python 的 `ContextVar`（`get_effective_user_id()` 底层依赖的机制）**不会跨越普通线程边界传播**，必须在还在正确上下文的时候把值提前拿出来存进 `ConversationContext`。
2. **debounce 合并时如何避免"关键信号被静默覆盖"**？
   `correction_detected` / `reinforcement_detected` 用 `or` 语义合并（见 `_enqueue_locked`），即使后一次入队没有检测到修正信号，只要窗口内**曾经**出现过修正信号就不会丢。

**面试问答**
- Q: 为什么记忆更新要做防抖异步处理，而不是每轮同步更新？
  A: 同步更新意味着每轮对话都要多跑一次 LLM 调用去抽取事实，直接拖慢响应延迟；而记忆信息本身不需要实时生效（下一轮才用得上）。防抖 + 批处理既省 LLM 调用次数（同一 thread 短时间内的多轮对话合并成一次抽取），又不阻塞主响应路径。
- Q: 多线程环境下，怎么把"当前请求的用户身份"正确传递到异步处理逻辑里？
  A: `ContextVar`/线程局部变量只在原始调用栈里有效，一旦跨越到 `threading.Timer`、线程池等新起的线程，就必须在入队时把需要的值显式提取出来，随数据一起传递，不能依赖上下文变量"自动跟随"。这是一个通用的并发陷阱，不只是记忆系统会遇到。

---

## 10. MCP 协议集成

**文件**：[mcp/client.py](backend/packages/harness/deerflow/mcp/client.py)、[mcp/cache.py](backend/packages/harness/deerflow/mcp/cache.py)、[mcp/oauth.py](backend/packages/harness/deerflow/mcp/oauth.py)

- 用 `langchain-mcp-adapters` 的 `MultiServerMCPClient` 统一管理多个 MCP server
- **懒加载**：工具 schema 首次使用才拉取（`get_cached_mcp_tools()`），不是启动时全量拉取
- **缓存失效**：通过配置文件 `mtime` 检测变化，避免每次请求都重新握手
- **三种 transport**：stdio（子进程）、SSE、HTTP；HTTP/SSE 支持 OAuth（`client_credentials`/`refresh_token` 自动刷新）
- stdio 场景下，为持久化会话按 `user_id:thread_id` 隔离，并固定子进程的 `cwd`/临时目录到线程 workspace，避免不同线程的临时文件互相污染

**面试问答**
- Q: MCP 相比自己定义一套 tool schema 协议，解决了什么问题？
  A: 标准化——不同工具提供方（不管是本地脚本、远程 HTTP 服务还是别的 agent 框架）只要实现 MCP 协议，就能被任意兼容客户端复用，避免每接入一个新能力都要写一套 adapter。
- Q: 大量 MCP server 接入后，怎么控制启动开销和运行时开销？
  A: 懒加载 + mtime 缓存失效——不用的工具不初始化，配置没变就不用重新拉取 schema，这是"按需 + 缓存"的标准套路。

---

## 11. Skills 系统

**文件**：[skills/parser.py](backend/packages/harness/deerflow/skills/parser.py)、[skills/slash.py](backend/packages/harness/deerflow/skills/slash.py)、[skills/security_scanner.py](backend/packages/harness/deerflow/skills/security_scanner.py)

- Skill = 目录 + `SKILL.md`（YAML frontmatter：name/description/license/allowed-tools）
- **渐进式加载**：默认只有"名字 + 描述"出现在 system prompt，真正的操作说明只有被激活（`/skill-name task` 或模型判断需要）时才注入当前这一轮的上下文
- 第三方 `.skill` 包安装时过 `security_scanner.py` 做安全扫描（防止恶意 skill 里塞恶意脚本/prompt injection）

**面试问答**
- Q: 怎么设计一个"能力可插拔又不占满 context"的系统？
  A: 分层——目录层（一句话描述，常驻 system prompt）+ 详情层（完整操作指南，按需注入）。这跟 §5 的延迟工具绑定是同一个思路的两种实现：都是把"要不要看到完整定义"从"启动时决定"变成"运行时按需决定"。
- Q: 允许用户安装第三方 skill 包，最大的安全风险是什么，怎么防？
  A: 恶意内容通过 skill 描述/操作指南对模型做 prompt injection，或者附带的脚本在 sandbox 里搞破坏。防护手段是安装时静态扫描 + sandbox 本身的路径/权限隔离作为纵深防御的第二层。

---

## 12. 模型抽象层

**文件**：[models/factory.py](backend/packages/harness/deerflow/models/factory.py)、[models/vllm_provider.py](backend/packages/harness/deerflow/models/vllm_provider.py)

- `create_chat_model(name, thinking_enabled)` 通过反射（`resolve_variable`）动态加载配置里 `use:` 指定的模型类，业务代码不需要为每个 provider 写 if-else
- `thinking_enabled` 统一了不同厂商的"思考模式"配置差异：有的是标准 `thinking` 字段，有的（如 vLLM 部署的 Qwen）要通过 `extra_body.chat_template_kwargs.enable_thinking` 开启；工厂层做了别名归一化，向后兼容旧配置
- `supports_vision` 作为能力声明，驱动 `ViewImageMiddleware` 是否注入图片

**面试问答**
- Q: 多模型/多厂商兼容层要怎么设计才不会让业务代码到处写 if-else？
  A: 用配置驱动 + 反射实例化——每个模型在配置里声明 `use: <module.path>:<ClassName>` 和能力标记（`supports_thinking`/`supports_vision`），工厂函数统一负责实例化和参数归一化，业务代码只依赖统一的能力接口，不关心具体是哪个 provider。

---

## 13. 工程可靠性实践

- **异步阻塞检测**（`tests/blocking_io/`）：用运行时 Blockbuster 包一层严格上下文，任何同步阻塞调用如果在 `app.*`/`deerflow.*` 业务代码里、且执行栈跑在 asyncio 事件循环上，直接抛 `BlockingError` 让测试失败。这是异步系统最常踩的坑（"看起来是 async def，但内部调了阻塞 IO"），用**运行时检测**比 code review 靠谱。
- **架构边界测试**（`tests/test_harness_boundary.py`）：静态检查 harness 不能 import app，把"分层规则"变成可执行的回归测试而不是文档里的一句话。
- **配置热重载边界**（`reload_boundary.py::STARTUP_ONLY_FIELDS`）：显式区分"运行时改配置立即生效的字段"和"必须重启才生效的字段"（如数据库连接、sandbox 类型），并用测试锁定这份清单，防止新加字段时忘记声明清楚。

**面试问答**
- Q: 异步系统里最容易踩的坑是什么，你怎么系统化地发现它，而不是靠线上事故？
  A: 最常见的坑是"async 函数内部调用了同步阻塞 IO"（文件读写、requests 库、time.sleep），表面上代码能跑，但会阻塞整个事件循环拖慢所有并发请求。系统化发现的手段是运行时钩子检测（比如 Blockbuster）包一层严格上下文，在测试执行路径覆盖到的代码上，一旦真的发生阻塞调用就立刻失败，把"隐性性能问题"变成"显性的测试失败"。
- Q: 怎么保证一个多层架构的依赖方向不会随着团队迭代慢慢腐化？
  A: 光靠文档和 review 约束不可靠，要写一个静态检查（扫 import 语句）作为 CI gate，任何违反依赖方向的 PR 直接挂红，把架构规则变成可执行的、不会被遗忘的东西。

---

## 14. 高频面试问题速查表

| 类别 | 问题 | 对应模块 |
|---|---|---|
| 架构 | Agent 系统怎么用责任链模式管理横切关注点？顺序为什么重要？ | §2 |
| 架构 | 状态在并发写入下怎么合并才安全？ | §1 |
| 安全 | Agent 陷入死循环怎么检测和兜底？ | §3 |
| 安全 | 给 LLM 执行 shell/文件能力，怎么防止越权？ | §7 |
| 安全 | 第三方扩展（skill/MCP）怎么防恶意内容？ | §11 |
| 工程 | LLM 消息协议（tool_calls 配对）有哪些隐藏约束？怎么应对异常场景？ | §3、§4 |
| Context | 工具/技能太多怎么不占满 context？ | §5、§11 |
| Context | 长任务怎么防止上下文溢出？ | §8 |
| 记忆 | 短期记忆和长期记忆怎么分层？怎么异步化又不丢数据？ | §9 |
| 多智能体 | 主 agent 怎么委派子任务、隔离上下文、限流？ | §6 |
| 工程质量 | 怎么用测试而不是 review 强制架构/性能约束？ | §13 |

---

## 15. 建议学习路径

1. **先啃 §1-§4**（state/middleware/循环检测/悬空调用）——这是本项目工程含金量最高、最容易被追问细节的部分，建议直接打开对应源码文件跟着读一遍。
2. **再看 §5-§7**（工具绑定/子智能体/sandbox）——回答"怎么设计一个能安全执行任意任务的 agent"这类开放题的素材库。
3. **§8-§9**（context engineering/记忆）——几乎是"agent 岗"面试必考方向，建议能自己画出记忆更新的完整时序图。
4. **§10-§12** 按目标岗位 JD 侧重选择性深入（偏 infra/生态集成岗多看 MCP/模型抽象层）。
5. 面试前建议自己口述一遍："如果让你从零设计一个类似的 agent harness，你会怎么拆模块" —— 这份文档里的每个模块基本对应了一个可以单独展开讲 5 分钟的子系统。
