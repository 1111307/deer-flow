# 模块一:Agent 状态管理 + Middleware 责任链(逐行精读)

> 教学形式:逐行代码精读。不是转述文档,是直接打开源码逐行解释"这行在干什么、为什么这么写、换一种写法会在哪里炸"。

---

## 第一部分:Agent 状态管理 —— [thread_state.py](../backend/packages/harness/deerflow/agents/thread_state.py)

### 1.1 为什么需要这个文件?

LangGraph 的 Agent 循环本质上是一个状态机:每一步(模型调用、工具调用)都会产出一个"状态更新"(`dict`),LangGraph 把这个更新合并进全局 `State`。默认合并策略是**覆盖**(后写的值覆盖旧值),但很多字段这样合并是错的——比如 `messages` 需要追加而不是覆盖,`artifacts` 需要去重合并。`thread_state.py` 就是在定义:"DeerFlow 这个 Agent 的状态里有哪些字段,每个字段该怎么合并"。

### 1.2 逐行看

```python
from typing import Annotated, NotRequired, TypedDict
from langchain.agents import AgentState
```
[thread_state.py:1-3](../backend/packages/harness/deerflow/agents/thread_state.py#L1-L3)

- `TypedDict`:声明一个"长得像 dict,但有类型检查"的结构,运行时它就是普通 dict,类型只在 IDE/mypy 层面起作用。
- `NotRequired[X]`:标记这个 key 可以不存在于 dict 里(区别于 `X | None`——`| None` 是"key 存在但值可能是 None",`NotRequired` 是"key 可能压根不存在")。
- `Annotated[T, reducer_fn]`:这是全篇的核心机制。LangGraph 在看到 State 的某个字段被标注为 `Annotated[T, fn]` 时,会用 `fn(old_value, new_value)` 来合并,而不是直接覆盖。
- `AgentState`:LangChain 提供的基础状态类,已经内置了 `messages: Annotated[list[AnyMessage], add_messages]` 这类字段(注意:`messages` 的默认 reducer 是"追加",不是覆盖——这是 Agent 能保留对话历史的根本原因)。`ThreadState` 继承它,相当于"在标准对话状态之上,叠加 DeerFlow 特有的字段"。

```python
class SandboxState(TypedDict):
    sandbox_id: NotRequired[str | None]
```
[thread_state.py:6-7](../backend/packages/harness/deerflow/agents/thread_state.py#L6-L7)

一个只有一个字段的小结构,记录当前线程绑定的沙箱 ID。为什么不直接用 `str | None` 而要包一层 `TypedDict`?因为下面要给它写一个**结构化的 reducer**(`merge_sandbox`),reducer 需要能区分"没有更新"(`None`)和"更新为某个结构体"这两种情况,包一层更清晰。

```python
def merge_sandbox(existing: SandboxState | None, new: SandboxState | None) -> SandboxState | None:
    if new is None:
        return existing
    if existing is None:
        return new
    existing_id = existing.get("sandbox_id")
    new_id = new.get("sandbox_id")
    if existing_id == new_id:
        return existing
    raise ValueError(f"Conflicting sandbox state updates: {existing_id!r} != {new_id!r}")
```
[thread_state.py:21-39](../backend/packages/harness/deerflow/agents/thread_state.py#L21-L39)

这是全文件里最值得细品的一个 reducer,体现了一种设计哲学:**"幂等写入允许,冲突写入报错",而不是"谁写得晚谁生效"**。

- 为什么会有这个需求?注释里写了:多个沙箱相关工具可能在**同一个 graph step**里并发初始化,各自都通过 `Command(update=...)` 往 state 写 `sandbox_id`。LangGraph 会把同一 step 内所有并发的更新收集起来一起调用 reducer 合并。
- 如果两次写入的 `sandbox_id` 相同(典型场景:两个工具都懒加载同一个沙箱,拿到同一个 ID),那就是无害的重复写,直接放行,返回 `existing` 即可。
- 如果两次写入的 `sandbox_id` **不同**,说明出现了"同一个线程被分配了两个不同的沙箱"这种隔离性 bug——这时候绝对不能"选一个生效",因为无论选哪个都会导致后续工具调用操作错误的沙箱(比如文件写到了 A 沙箱,但 Agent 以为是 B 沙箱)。所以选择 `raise ValueError` **快速失败**而不是静默吞掉。

*追问自己:如果把这里改成"直接用 new 覆盖 existing"会怎样?* —— 会把一个原本能在开发阶段被发现的隔离性 bug,变成一个生产环境里"文件读写到了错误沙箱"的诡异线上问题,而且没有任何报错线索。这就是"fail closed"设计原则的具体体现。

```python
SandboxStateField = Annotated[NotRequired[SandboxState | None], merge_sandbox]
```
[thread_state.py:42](../backend/packages/harness/deerflow/agents/thread_state.py#L42)

把类型和 reducer 打包成一个别名,方便在 `ThreadState` 里直接用一个名字引用,而不用每次都写一遍完整的 `Annotated[...]`。

```python
def merge_artifacts(existing: list[str] | None, new: list[str] | None) -> list[str]:
    if existing is None:
        return new or []
    if new is None:
        return existing
    return list(dict.fromkeys(existing + new))
```
[thread_state.py:45-52](../backend/packages/harness/deerflow/agents/thread_state.py#L45-L52)

`artifacts` 是 Agent 生成的产出文件列表(比如 `present_files` 工具暴露出来的文件)。这里用 `dict.fromkeys(...)` 做去重——这是个常见 Python trick:字典的 key 是唯一的,`dict.fromkeys(iterable)` 会保留**首次出现的顺序**同时去重,比 `set()` 更好,因为 `set` 会打乱顺序。

*为什么需要去重?* 想象 Agent 在多轮对话里重复调用同一个工具生成了同一个文件路径,如果不去重,前端展示的产出列表里会有重复项。

```python
def merge_viewed_images(existing, new) -> dict[str, ViewedImageData]:
    if existing is None:
        return new or {}
    if new is None:
        return existing
    if len(new) == 0:
        return {}
    return {**existing, **new}
```
[thread_state.py:55-69](../backend/packages/harness/deerflow/agents/thread_state.py#L55-L69)

这里有一个容易被忽略的**"空字典是有意义的信号"**的设计:一般人写代码会觉得"空更新等于没更新",但这里显式区分了 `new is None`(没碰这个字段,保留原值)和 `new == {}`(显式地清空)。

*这是为什么?* 图像被 Agent"看过"一次后,`ViewImageMiddleware` 不希望这些 base64 数据在后续每一轮里持续占用 token 预算和 payload 体积,于是设计成:某个 middleware 处理完当前这批图片后,主动写入 `{}` 来清空。如果没有这个特殊 case,middleware 永远没法把这个字段"清空"——因为按照默认语义,任何"不想更新"就该传 `None`,但传 `None` 又拿不到"清空"的能力,所以专门开了个后门:传空字典 = 清空指令。这是一个从需求(控制 token 成本)反推出数据结构设计的典型例子,面试时如果被问"如何设计一个可清空又可合并的 state 字段",这就是标准答案模板。

```python
def merge_todos(existing: list | None, new: list | None) -> list | None:
    if new is None:
        return existing
    return new
```
[thread_state.py:72-82](../backend/packages/harness/deerflow/agents/thread_state.py#L72-L82)

这个最简单:"新值非 None 就整体替换,None 就保留原值"。为什么 todos 不需要合并去重,而是整体替换?因为 `write_todos` 工具每次调用都会传入**完整的**任务列表(不是增量),模型自己负责维护整个列表的状态(参考 [agent.py:158-192](../backend/packages/harness/deerflow/agents/lead_agent/agent.py#L158-L192) 里 `write_todos` 工具的 prompt,要求模型"实时更新整个列表")。所以这里不需要像 `artifacts` 那样做增量合并。

```python
class PromotedTools(TypedDict):
    catalog_hash: str
    names: list[str]

def merge_promoted(existing, new):
    if not new:
        return existing
    if existing is None or existing.get("catalog_hash") != new["catalog_hash"]:
        return {"catalog_hash": new["catalog_hash"], "names": list(dict.fromkeys(new["names"]))}
    return {"catalog_hash": existing["catalog_hash"], "names": list(dict.fromkeys(existing["names"] + new["names"]))}
```
[thread_state.py:85-108](../backend/packages/harness/deerflow/agents/thread_state.py#L85-L108)

这是为"延迟工具绑定"(tool_search / MCP 大工具目录场景)服务的字段,记录哪些原本被隐藏的工具已经被"提升"(promoted)为模型可见。关键逻辑是 `catalog_hash` 校验:

- 工具目录(所有可用工具的集合)可能会变化(比如运维改了 MCP server 配置)。`catalog_hash` 就是当前工具目录的一个指纹。
- 如果存量的 `catalog_hash` 和新的对不上,说明"目录已经变了",那么旧的 `names` 列表可能已经过时——比如旧目录里 `names` 记录的是"第 5 个工具已提升",但新目录里第 5 个位置换成了完全不同的工具,继续沿用旧的提升记录就会误将新工具直接暴露给模型,绕过了本该走的"先搜索再提升"的安全检查。所以这里选择**整体丢弃重建**,而不是合并。
- 如果 hash 一致,说明目录没变,才安全地把两次的 `names` 合并去重。

这是这个文件里第二个体现"fail closed / 宁可保守也不要悄悄出错"哲学的 reducer,和 `merge_sandbox` 遥相呼应——面试时如果被问"这个项目里状态设计有什么通用原则",可以直接举这两个例子。

```python
class ThreadState(AgentState):
    sandbox: SandboxStateField
    thread_data: NotRequired[ThreadDataState | None]
    title: NotRequired[str | None]
    artifacts: Annotated[list[str], merge_artifacts]
    todos: Annotated[list | None, merge_todos]
    uploaded_files: NotRequired[list[dict] | None]
    viewed_images: Annotated[dict[str, ViewedImageData], merge_viewed_images]
    promoted: Annotated[PromotedTools | None, merge_promoted]
```
[thread_state.py:111-119](../backend/packages/harness/deerflow/agents/thread_state.py#L111-L119)

最后汇总成一个类。注意这里**没有自定义 reducer 的字段**(`thread_data`、`title`、`uploaded_files`)用的是纯 `NotRequired[...]`,意味着它们走 LangGraph 默认合并策略——直接覆盖。这些字段的语义天然就是"覆盖式"的(比如 `title` 每次生成新标题就该整体替换旧标题),所以不需要额外 reducer。

**小结一句话**:这个文件教会你的核心能力是——**看到一个 LangGraph state 字段,先问自己"并发写入这个字段会不会有多个 step 同时发生?合并语义应该是追加、去重、替换还是要检测冲突?"**,而不是无脑用默认覆盖。

---

## 第二部分:Middleware 责任链

### 2.1 责任链的载体:`AgentMiddleware` 是什么

先看一个具体例子建立直觉,而不是死记硬背 19 个类名。[thread_data_middleware.py](../backend/packages/harness/deerflow/agents/middlewares/thread_data_middleware.py) 是最简单的一个:

```python
class ThreadDataMiddlewareState(AgentState):
    thread_data: NotRequired[ThreadDataState | None]

class ThreadDataMiddleware(AgentMiddleware[ThreadDataMiddlewareState]):
    state_schema = ThreadDataMiddlewareState
```
[thread_data_middleware.py:18-37](../backend/packages/harness/deerflow/agents/middlewares/thread_data_middleware.py#L18-L37)

注意这里又声明了一个**局部的** `ThreadDataMiddlewareState`,只包含这个 middleware 关心的那一个字段 `thread_data`,并且注释写明"Compatible with the `ThreadState` schema"。

*为什么不直接用第一部分的 `ThreadState`?* —— 关键的架构原则:**每个 middleware 只声明自己需要读写的那一小片 state,而不是依赖完整的 `ThreadState`**。好处是这个 middleware 可以被复用在别的 Agent(比如 subagent,它们的 state schema 可能不完全等于 `ThreadState`),只要字段名和类型兼容就行,不需要引入对 `ThreadState` 这个具体类的硬依赖。这是关注点分离(SoC)在 LangGraph middleware 体系里的具体落地方式。

```python
@override
def before_agent(self, state: ThreadDataMiddlewareState, runtime: Runtime) -> dict | None:
    context = runtime.context or {}
    thread_id = context.get("thread_id")
    if thread_id is None:
        config = get_config()
        thread_id = config.get("configurable", {}).get("thread_id")
    if thread_id is None:
        raise ValueError("Thread ID is required in runtime context or config.configurable")
    ...
```
[thread_data_middleware.py:81-91](../backend/packages/harness/deerflow/agents/middlewares/thread_data_middleware.py#L81-L91)

这里揭示了 `AgentMiddleware` 的第一个 hook 点:`before_agent(state, runtime) -> dict | None`——**在整个 Agent 循环开始前跑一次**(不是每次模型调用前,是每个 graph run 开始时跑一次)。返回的 `dict` 会被当作一次 state 更新,交给对应字段的 reducer 合并进全局 state。

```python
messages = list(state.get("messages", []))
last_message = messages[-1] if messages else None
if last_message and isinstance(last_message, HumanMessage):
    messages[-1] = HumanMessage(
        content=last_message.content, id=last_message.id,
        name=last_message.name or "user-input",
        additional_kwargs={**last_message.additional_kwargs, "run_id": ..., "timestamp": ...},
    )
return {"thread_data": {**paths}, "messages": messages}
```
[thread_data_middleware.py:102-118](../backend/packages/harness/deerflow/agents/middlewares/thread_data_middleware.py#L102-L118)

这里顺手做了一件事:给最新的用户消息打上 `run_id` 和 `timestamp` 元数据(用于审计/追踪),用**替换整个消息对象**的方式(因为 `HumanMessage` 是不可变风格的对象,要改 `additional_kwargs` 就得重新构造一个)。返回的 `messages` 会走 `AgentState` 默认的 `add_messages` reducer——但这里传的是"替换最后一条"而不是"追加新的一条",这依赖 `add_messages` reducer 对相同 `id` 的消息做的是**替换**而不是重复追加(这是 LangGraph `add_messages` reducer 的既有行为:按 message id 去重合并)。

*划重点(常见面试坑)*:如果你不知道 `add_messages` 是按 `id` 去重的,你会以为这里"追加了一条新消息",从而误判消息历史多了一条——这正是我在教你读这段代码时希望你注意到的隐藏契约。

### 2.2 责任链是怎么"拼"出来的 —— 两段真实拼接代码

这是这次专门去读的重点:责任链**不是**写在一个文件里的一份静态列表,而是**分两段拼接**——第一段是"lead agent 和 subagent 共享的基础中间件"(在 [tool_error_handling_middleware.py](../backend/packages/harness/deerflow/agents/middlewares/tool_error_handling_middleware.py)),第二段是"lead agent 独有的中间件"(在 [lead_agent/agent.py](../backend/packages/harness/deerflow/agents/lead_agent/agent.py))。

#### 第一段:`_build_runtime_middlewares`

```python
def _build_runtime_middlewares(*, app_config, include_uploads, include_dangling_tool_call_patch, lazy_init=True):
    middlewares: list[AgentMiddleware] = [
        ToolOutputBudgetMiddleware.from_app_config(app_config),
        ThreadDataMiddleware(lazy_init=lazy_init),
        SandboxMiddleware(lazy_init=lazy_init),
    ]
    if include_uploads:
        middlewares.insert(2, UploadsMiddleware())
    if include_dangling_tool_call_patch:
        middlewares.append(DanglingToolCallMiddleware())
    middlewares.append(LLMErrorHandlingMiddleware(app_config=app_config))
    ...
```
[tool_error_handling_middleware.py:129-158](../backend/packages/harness/deerflow/agents/middlewares/tool_error_handling_middleware.py#L129-L158)

先看这几行的顺序设计:

- `ThreadDataMiddleware` 在 index 0 之后(实际是 index 1,列表初始化时排第 2 位)——它必须在 `SandboxMiddleware` **之前**,因为 `SandboxMiddleware` 需要用到 `thread_id` 对应的目录信息才能挂载沙箱。
- `UploadsMiddleware` 用 `middlewares.insert(2, ...)` **插入到 index 2**,也就是排在 `ThreadDataMiddleware`(index 1)之后、`SandboxMiddleware`(原 index 2,插入后变成 index 3)之前。为什么?因为 `UploadsMiddleware` 要往用户上传目录里查文件,而这个目录路径是 `ThreadDataMiddleware` 算出来的,所以必须排在它后面;同时它跟 `SandboxMiddleware` 谁先谁后其实没有强依赖,这里选择插在中间纯粹是逻辑分组(先把"线程相关的数据准备类"中间件放一起)。

*追问自己:如果这里用 `middlewares.append(UploadsMiddleware())` 而不是 `insert(2, ...)` 会怎样?* —— `UploadsMiddleware` 就会排到 `SandboxMiddleware`、`DanglingToolCallMiddleware`、`LLMErrorHandlingMiddleware` 之后。这些中间件之间大概率不冲突,所以功能上未必会立刻炸,但破坏了"线程数据准备类中间件放在最前面统一收口"的分组意图,后续如果新增一个依赖 `uploaded_files` 字段的中间件并且排在了 `SandboxMiddleware` 之前,就会读到还没准备好的空值——这就是为什么顺序类 bug 往往是"过一阵子才炸"的隐藏地雷。

- `DanglingToolCallMiddleware` 排在这批"资源准备"中间件之后。它的职责是修补消息历史里缺失的 `ToolMessage`,必须发生在**模型看到消息历史之前**,但同时不依赖 sandbox/thread_data,所以放在这个位置只是"越早处理越好,避免带着脏消息历史进入后续处理"。
- `LLMErrorHandlingMiddleware` 追加在最后,顾名思义是兜底处理模型调用失败,放在这批之后合理,因为它包裹的是"模型调用"这个动作本身,而不是"调用前的准备工作"。

```python
    guardrails_config = app_config.guardrails
    if guardrails_config.enabled and guardrails_config.provider:
        ...
        provider_cls = resolve_variable(guardrails_config.provider.use)
        provider = provider_cls(**provider_kwargs)
        middlewares.append(GuardrailMiddleware(provider, fail_closed=guardrails_config.fail_closed, passport=guardrails_config.passport))

    middlewares.append(SandboxAuditMiddleware())
    middlewares.append(ToolErrorHandlingMiddleware())
    return middlewares
```
[tool_error_handling_middleware.py:160-187](../backend/packages/harness/deerflow/agents/middlewares/tool_error_handling_middleware.py#L160-L187)

- `GuardrailMiddleware` 是可选的(取决于配置),用 `resolve_variable`(反射机制)动态实例化一个 provider 类。这里体现了"策略可插拔"的设计——provider 可以是内置的 `AllowlistProvider`,也可以是第三方 OAP 策略提供商,通过配置的类路径字符串动态加载,新增一种 provider 完全不需要改这个工厂函数。
- **顺序关键点**:`GuardrailMiddleware`(工具调用前授权检查) → `SandboxAuditMiddleware`(审计) → `ToolErrorHandlingMiddleware`(把工具异常转成 ToolMessage) —— 这个顺序为什么重要?因为这三个都是围绕"工具调用"这个动作的 `wrap_tool_call` 钩子。Guardrail 必须在实际执行工具**之前**拦截(比如检测到危险命令直接拒绝并返回错误 ToolMessage,不让工具真的跑起来),如果把 `ToolErrorHandlingMiddleware` 放在 Guardrail 前面,理论上不会有直接冲突(它们包裹的是同一次调用),但语义上 Guardrail 的"拒绝"和 ToolErrorHandling 的"捕获异常"是两种不同性质的失败(前者是策略拒绝,后者是运行时异常),让 Guardrail 排在前面能保证"策略检查"发生在"真正执行工具"之前,而不是执行完了才想起来查权限。

*这是本节的第一个"如果调换顺序会怎样"的具体案例,记住这个思考模式,后面看那 19 项顺序注释时就不需要死记,而是能自己推导。*

#### 第二段:两个复用入口

```python
def build_lead_runtime_middlewares(*, app_config, lazy_init=True):
    return _build_runtime_middlewares(app_config=app_config, include_uploads=True, include_dangling_tool_call_patch=True, lazy_init=lazy_init)

def build_subagent_runtime_middlewares(*, app_config=None, model_name=None, lazy_init=True, deferred_setup=None):
    ...
    middlewares = _build_runtime_middlewares(app_config=app_config, include_uploads=False, include_dangling_tool_call_patch=True, lazy_init=lazy_init)
    ...
```
[tool_error_handling_middleware.py:190-249](../backend/packages/harness/deerflow/agents/middlewares/tool_error_handling_middleware.py#L190-L249)

这里能看出为什么要把公共逻辑抽成 `_build_runtime_middlewares` 私有函数,再用两个公开函数包一层:lead agent 需要 `include_uploads=True`(用户直接和 lead agent 对话,才需要处理文件上传),而 subagent 是被 `task` 工具派发出去执行子任务的,它没有独立的"用户上传"概念(`include_uploads=False`),但同样需要沙箱、线程数据、dangling tool call 修复这些基础设施。这是**用参数化组合避免复制粘贴**的教科书写法——如果你在面试里被问"如何设计 lead agent 和 subagent 共享中间件又允许差异化",这段代码就是标准答案。

### 2.3 lead agent 独有部分:[agent.py::build_middlewares](../backend/packages/harness/deerflow/agents/lead_agent/agent.py#L270-L377)

```python
def build_middlewares(config, model_name, agent_name=None, custom_middlewares=None, *, available_skills=None, app_config=None, deferred_setup=None):
    resolved_app_config = app_config or get_app_config()
    middlewares = build_lead_runtime_middlewares(app_config=resolved_app_config, lazy_init=True)
```
[agent.py:270-300](../backend/packages/harness/deerflow/agents/lead_agent/agent.py#L270-L300)

第一行调用就是刚读的第一段——**先把共享基础中间件拿到手**,后面在这个 `middlewares` list 上继续 `append`。这就是"共享中间件在前,lead-only 中间件在后"这句话的真实代码来源。

```python
    from deerflow.agents.middlewares.dynamic_context_middleware import DynamicContextMiddleware
    middlewares.append(DynamicContextMiddleware(agent_name=agent_name, app_config=resolved_app_config))

    from deerflow.agents.middlewares.skill_activation_middleware import SkillActivationMiddleware
    middlewares.append(SkillActivationMiddleware(available_skills=available_skills, app_config=resolved_app_config))
```
[agent.py:304-313](../backend/packages/harness/deerflow/agents/lead_agent/agent.py#L304-L313)

注意这两处是**函数内部的 lazy import**,不是文件顶部的 import。为什么?文件顶部已经导入了一大堆 middleware 类,但这两个没有放在顶部——这通常是为了**打破循环导入**,或者单纯是"减少模块加载时的导入面"。这是一个值得在面试时提及的 Python 工程细节:看到函数体内的 import,第一反应应该是"这里大概率在规避循环依赖",而不是代码风格随意。

```python
    summarization_middleware = _create_summarization_middleware(app_config=resolved_app_config)
    if summarization_middleware is not None:
        middlewares.append(summarization_middleware)

    cfg = _get_runtime_config(config)
    is_plan_mode = cfg.get("is_plan_mode", False)
    todo_list_middleware = _create_todo_list_middleware(is_plan_mode)
    if todo_list_middleware is not None:
        middlewares.append(todo_list_middleware)
```
[agent.py:315-325](../backend/packages/harness/deerflow/agents/lead_agent/agent.py#L315-L325)

这里看到责任链组装的通用模式:**"工厂函数返回 None 就跳过 append"**,用来实现"可选中间件"。`_create_summarization_middleware` 内部会检查 `config.summarization.enabled`,不开启就返回 `None`。这是一种很干净的"条件性责任链装配"写法——不需要在主函数里写一堆 `if config.xxx.enabled: middlewares.append(...)` 的重复模式,而是把"要不要启用"的判断下沉到各自的工厂函数里,主函数只管"有就加"。

```python
    model_config = resolved_app_config.get_model_config(model_name) if model_name else None
    if model_config is not None and model_config.supports_vision:
        middlewares.append(ViewImageMiddleware())
```
[agent.py:339-341](../backend/packages/harness/deerflow/agents/lead_agent/agent.py#L339-L341)

`ViewImageMiddleware` 只有在**当前模型支持视觉**时才加入责任链。这是"能力探测驱动的条件装配"——不是配置开关,而是运行时查模型元数据(`supports_vision`)。如果一个不支持视觉的模型被硬塞了这个中间件,后果是:中间件试图往 messages 里注入 base64 图片,模型 API 大概率直接因为 payload 格式不被支持而报错,或者更隐蔽地,模型收到了图片但完全无法"看懂"只会浪费 token——这就是为什么要在装配阶段就做能力门控,而不是让运行时才发现不匹配。

```python
    if deferred_setup is not None and deferred_setup.deferred_names:
        from deerflow.agents.middlewares.deferred_tool_filter_middleware import DeferredToolFilterMiddleware
        middlewares.append(DeferredToolFilterMiddleware(deferred_setup.deferred_names, deferred_setup.catalog_hash))

    subagent_enabled = cfg.get("subagent_enabled", False)
    if subagent_enabled:
        max_concurrent_subagents = cfg.get("max_concurrent_subagents", 3)
        middlewares.append(SubagentLimitMiddleware(max_concurrent=max_concurrent_subagents))

    loop_detection_config = resolved_app_config.loop_detection
    if loop_detection_config.enabled:
        middlewares.append(LoopDetectionMiddleware.from_config(loop_detection_config))

    if custom_middlewares:
        middlewares.extend(custom_middlewares)
```
[agent.py:343-364](../backend/packages/harness/deerflow/agents/lead_agent/agent.py#L343-L364)

这里出现了本节第二个关键的"顺序为什么重要"的例子,紧接着代码给出了权威解释:

```python
    # SafetyFinishReasonMiddleware — suppress tool execution when the provider
    # safety-terminated the response. Registered after custom middlewares so
    # that LangChain's reverse-order after_model dispatch runs Safety first;
    # cleared tool_calls then flow through Loop/Subagent accounting without
    # firing extra alarms.
    safety_config = resolved_app_config.safety_finish_reason
    if safety_config.enabled:
        middlewares.append(SafetyFinishReasonMiddleware.from_config(safety_config))

    middlewares.append(ClarificationMiddleware())
    return middlewares
```
[agent.py:366-377](../backend/packages/harness/deerflow/agents/lead_agent/agent.py#L366-L377)

这条注释信息量很大,拆开讲:

1. **`after_model` 钩子是反序执行的**——这是 LangChain middleware 体系里一个容易踩坑的隐藏规则:`before_model`/`wrap_model_call` 是按 middleware 列表的**正序**执行(先加的先跑,像洋葱从外往里剥),但 `after_model` 是**反序**执行(后加的先跑,相当于洋葱从里往外剥回去)。这符合"栈"的直觉——你最后进入的处理层,应该最先在"返回路上"处理完。
2. 正因为反序,`SafetyFinishReasonMiddleware` 被 `append` 在 `LoopDetectionMiddleware`/`SubagentLimitMiddleware` **之后**,反而意味着它的 `after_model` 会**先于**它们执行。
3. 为什么要"Safety 先跑"?因为 Safety 中间件的职责是:当供应商因为安全策略强制截断了响应(`finish_reason=content_filter` 之类),这时候返回的 `tool_calls` 可能是不完整/被截断的,必须先把这些"有问题的 tool_calls"清空掉。如果换成 `LoopDetectionMiddleware` 或 `SubagentLimitMiddleware` 先跑,它们会看到这些不完整的 tool_calls 并尝试去做循环检测计数、并发限制裁剪等处理,对着一个本就不该存在的 tool_call 做业务逻辑判断,可能触发不必要的"循环告警"或裁剪逻辑,造成误报。
4. `ClarificationMiddleware` 必须真正物理排最后(`append` 在所有其它中间件之后),原因是它的职责是**拦截 `ask_clarification` 工具调用并用 `Command(goto=END)` 直接结束整个 graph**——它是一个"熔断点"。如果它不是最后一个,而是排在比如 `LoopDetectionMiddleware` 前面,由于反序执行,`ClarificationMiddleware` 的 `after_model` 会先跑,一旦触发 `Command(goto=END)` 提前结束,后面本该跑的 `LoopDetectionMiddleware.after_model` 就永远不会执行到,导致循环检测的计数状态出现缺口。

*留一个思考题,不直接给答案,自己在心里过一遍*:如果把 `SubagentLimitMiddleware` 和 `LoopDetectionMiddleware` 的 `append` 顺序对调(先加 Loop 再加 Subagent),会不会破坏什么?提示:想想反序执行规则,再想想这两个中间件各自在 `after_model` 里做什么事情(一个是裁剪超额的并行 task 调用,一个是检测重复调用模式)——它们之间有没有类似 Safety 那样的强依赖关系,还是只是弱耦合?

---

## 本模块小结

| 概念 | 关键文件 | 一句话记忆点 |
|---|---|---|
| 自定义 reducer | `thread_state.py` | 并发写入同一字段时,追加/去重/替换/冲突报错,四选一想清楚 |
| middleware 局部 state | `thread_data_middleware.py` | 每个 middleware 只声明自己需要的那片 state,不绑死具体 Agent |
| 责任链两段拼接 | `tool_error_handling_middleware.py` + `lead_agent/agent.py` | 共享基础设施抽成私有函数,lead/subagent 用参数差异化复用 |
| `before_model`/`wrap_model_call` 正序,`after_model` 反序 | `lead_agent/agent.py::build_middlewares` 尾部注释 | 决定了"谁该在 append 顺序里靠后但逻辑上先跑" |
