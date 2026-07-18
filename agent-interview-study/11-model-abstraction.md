# 第十一部分：模型抽象层 —— 一个工厂 + 一堆"打补丁"的 provider

第十部分末尾说,Skills 的内核是"如何安全执行不受信任的内容"。模型抽象层要解决的是另一类脏活:**每一家 LLM 厂商的 API 都有自己的怪癖,但 agent 主逻辑不应该知道这些怪癖存在**。DeerFlow 支持 OpenAI、Anthropic、DeepSeek、MiniMax、Qwen(vLLM)、MindIE、Codex、Stepfun、MiMo 等一堆模型,它们的"思考模式"配置方式各不相同、`reasoning` 字段的存放位置各不相同、多轮对话要不要回传思考内容各不相同。这一部分要看的是:**这些差异是怎么被一个统一的 `create_chat_model()` 入口和一层"打补丁的 provider"吸收掉的**,让上层永远只调 `model.ainvoke(messages)`。

`learn.md` 第 12 节只列了 `factory.py` 和 `vllm_provider.py` 两个文件,而 `models/` 目录有 13 个文件——8 个是各家厂商的 provider 补丁。这次要讲的重点恰恰是那 8 个补丁背后的**共同套路**,以及 `factory.py` 那 200 行里藏着的、文档一句没提的一堆"参数归一化"逻辑。

## 1. 对外只有一个符号

[`models/__init__.py`](../backend/packages/harness/deerflow/models/__init__.py) 只导出一个东西:

```python
from .factory import create_chat_model
__all__ = ["create_chat_model"]
```

`VllmChatModel`、`ClaudeChatModel`、`PatchedChatMiniMax` 这些具体 provider 类**全部不导出**——它们只出现在 `config.yaml` 的 `use:` 字段里,通过反射被加载,业务代码永远不 import 它们。这跟第九部分 MCP 的 `__init__.py` 只导出 `get_mcp_tools`、把 `session_pool` 藏起来是同一个信号:**上层只应该看到"工厂",不应该看到"工厂里生产的具体型号"。** 上层拿到的永远是一个 `BaseChatModel` 抽象类型,不关心背后是哪家。

## 2. 反射实例化:配置驱动,零 if-else

[`create_chat_model`](../backend/packages/harness/deerflow/models/factory.py#L82-L204) 的核心只有三行:

```python
model_config = config.get_model_config(name)
model_class = resolve_class(model_config.use, BaseChatModel)   # 反射加载
model_instance = model_class(**kwargs, **model_settings_from_config)
```

`resolve_class`(第九部分和 tool 系统里都见过的 `resolve_variable` 的兄弟)按 `config.yaml` 里的 `use: "deerflow.models.claude_provider:ClaudeChatModel"` 这个字符串,import 模块、取出类、并校验它是 `BaseChatModel` 的子类。这意味着**加一个新模型厂商,不用改一行 factory 代码**——写一个新的 provider 类,在配置里指一下 `use:` 就行。

[`ModelConfig`](../backend/packages/harness/deerflow/config/model_config.py) 是配置侧的 schema。除了 `use`/`model`/`name`,关键是几个**能力声明**字段:`supports_thinking`、`supports_reasoning_effort`、`supports_vision`,加一个 `model_config = ConfigDict(extra="allow")`——`extra="allow"` 让配置可以塞任意 provider 特有的字段(比如 `base_url`、`api_key`、`temperature`),这些会原样透传给 provider 构造函数。**能力用显式字段声明,provider 私有参数走 extra 透传**,这个组合让一个 schema 能覆盖所有厂商。

`supports_vision` 就是第十部分之前提过的:它驱动 `get_available_tools` 是否挂载 `view_image_tool`、`ViewImageMiddleware` 是否注入图片。能力声明和消费它的中间件是解耦的——config 说"这个模型能看图",中间件据此决定行为。

## 3. factory 真正干的活:一个 bool 翻译成七种"关思考"的方言

文档把 factory 一句话带过("做了别名归一化"),但 factory 里真正复杂的,是把 `thinking_enabled` 这**一个布尔值**翻译成七八家厂商各不相同的开关方式。先看"开思考"([factory.py:126-137](../backend/packages/harness/deerflow/models/factory.py#L126-L137)):

```python
has_thinking_settings = (model_config.when_thinking_enabled is not None) or (model_config.thinking is not None)
effective_wte: dict = dict(model_config.when_thinking_enabled) if model_config.when_thinking_enabled else {}
if model_config.thinking is not None:
    merged_thinking = {**(effective_wte.get("thinking") or {}), **model_config.thinking}
    effective_wte = {**effective_wte, "thinking": merged_thinking}
if thinking_enabled and has_thinking_settings:
    if not model_config.supports_thinking:
        raise ValueError(f"Model {name} does not support thinking. ...")
    if effective_wte:
        model_settings_from_config.update(effective_wte)
```

这里有个向后兼容的合并:`thinking` 是 `when_thinking_enabled` 的一个"快捷字段"(shortcut),旧配置可能只写了 `thinking`,新配置写 `when_thinking_enabled`,两者都存在时要 merge。这跟第七部分的配置层适配、第九部分 MCP 的 `transport`→`type` 别名归一化是同一类工作:**接受多种历史写法,内部收敛成一种。**

真正体现"每家方言不同"的是"关思考"那段([factory.py:138-157](../backend/packages/harness/deerflow/models/factory.py#L138-L157)),连着四个 `elif` 分支:

```python
if not thinking_enabled:
    if model_config.when_thinking_disabled is not None:
        model_settings_from_config.update(model_config.when_thinking_disabled)          # ① 用户显式配置的关闭设置,最高优先
    elif has_thinking_settings and effective_wte.get("extra_body", {}).get("thinking", {}).get("type"):
        model_settings_from_config["extra_body"] = _deep_merge_dicts(...)               # ② OpenAI 兼容网关:thinking 嵌在 extra_body 里
        model_settings_from_config["reasoning_effort"] = "minimal"
    elif has_thinking_settings and (disable_chat_template_kwargs := _vllm_disable_chat_template_kwargs(...)):
        model_settings_from_config["extra_body"] = _deep_merge_dicts(...)               # ③ vLLM:靠 chat_template_kwargs 开关
    elif has_thinking_settings and effective_wte.get("thinking", {}).get("type"):
        model_settings_from_config["thinking"] = {"type": "disabled"}                   # ④ 原生 langchain_anthropic:thinking 是直接的构造参数
```

同样是"把思考关掉"这一个语义,四家的表达完全不同:OpenAI 兼容网关要往 `extra_body.thinking.type` 里塞 `disabled` 还要配 `reasoning_effort=minimal`;vLLM 要改 `chat_template_kwargs`;原生 Anthropic 直接给构造函数传 `thinking={"type":"disabled"}`。**factory 的价值就是把这四种方言藏在一个 `thinking_enabled=False` 后面**——上层只说"这次不要思考",不需要知道当前这个模型是靠哪种机制关的。分支顺序也有讲究:①用户显式的 `when_thinking_disabled` 永远第一优先,和第九、十部分反复出现的"永远给外部已决定的值让路"是同一个原则。

## 4. 三个"补丁默认值"函数:LangChain 的默认不适合生产

factory 里还有三个小函数,专门修 LangChain 自身默认值在 DeerFlow 场景下不合适的地方。它们都只对 `langchain_openai:ChatOpenAI` 这条路径生效,因为这些 kwarg 是 OpenAI 适配器特有的。

**`_enable_stream_usage_by_default`**([factory.py:34-47](../backend/packages/harness/deerflow/models/factory.py#L34-L47)):LangChain 只在"没配自定义 base_url"时才自动开 `stream_usage`。但 DeerFlow 大量用 OpenAI 兼容网关(豆包、DeepSeek 都配了 base_url),于是 token 用量统计会**静默为空**,第七部分讲的 `TokenUsageMiddleware` 就没数据可记。所以这里主动补上:配了 base_url 且没显式设过,就默认开 `stream_usage=True`。

**`_apply_stream_chunk_timeout_default`**([factory.py:61-79](../backend/packages/harness/deerflow/models/factory.py#L61-L79)):LangChain 默认 60 秒没收到 chunk 就抛 `StreamChunkTimeoutError`,但推理模型(DeepSeek-R1、GPT-5)第一个 chunk 可能要 90~150 秒才来。所以默认放宽到 240 秒。而且注意——**非 OpenAI 路径要把这个 key 直接 pop 掉**,否则透传给别家 provider 的构造函数会报 `TypeError: unexpected keyword argument`。这是"透传任意参数"这个灵活性的代价:某些参数只有特定 provider 认,喂错了会炸,所以要按 provider 精确地加/删。

**`_deep_merge_dicts`**([factory.py:13-21](../backend/packages/harness/deerflow/models/factory.py#L13-L21)):递归合并,不改原输入。因为 `extra_body` 这种嵌套结构,浅拷贝会互相覆盖掉对方的子键。

后面还有针对 Codex(把 thinking 映射成 `reasoning_effort`,并 pop 掉 Codex 端点拒绝的 `max_tokens`,[factory.py:165-179](../backend/packages/harness/deerflow/models/factory.py#L165-L179))和 MindIE(强制保守的 `max_retries`,[factory.py:183-185](../backend/packages/harness/deerflow/models/factory.py#L183-L185))的特判。这些特判为什么可以留在 factory 里、而不是塞进各自的 provider 类?因为它们是**参数层面**的调整(构造前改 kwargs),放 factory 集中管更清楚;而下面第 5-7 节讲的是**行为层面**的差异(请求/响应的序列化),那些就必须放进 provider 类。

## 5. provider 补丁的共同套路:重写 `_get_request_payload` / `_create_chat_result`

8 个 provider 文件里,除了 Claude,其余全是同一个套路:**继承 LangChain 自带的 `ChatOpenAI`/`ChatDeepSeek`,只重写两三个钩子方法**,修某一家厂商的具体行为差异。重写的钩子无非三个:

- `_get_request_payload` —— 请求发出前,改 payload(加字段、删字段、恢复字段)。
- `_create_chat_result` —— 非流式响应回来后,把 provider 特有字段提取到 LangChain 的标准结构里。
- `_convert_chunk_to_generation_chunk` —— 流式 delta 逐块处理,同上。

它们要修的,几乎都是同一个问题:**推理模型的"思考内容"(`reasoning`)不是 OpenAI 标准字段,LangChain 默认会把它丢掉**,而很多厂商在多轮对话里又要求你把上一轮的思考内容**原样回传**,否则报错。

以 [`vllm_provider.py`](../backend/packages/harness/deerflow/models/vllm_provider.py) 为例(文档唯一提到的 provider):它三个钩子全重写了。请求侧 [`_get_request_payload`](../backend/packages/harness/deerflow/models/vllm_provider.py#L168-L191) 把历史 `AIMessage` 里存的 `reasoning` 重新塞回 payload;响应侧 `_create_chat_result` 和流式侧 `_convert_chunk_to_generation_chunk` 则把 vLLM 返回的 `reasoning` 字段提取到 `additional_kwargs` 里。一进一出,就把"vLLM 有个非标准的 reasoning 字段"这件事对上层完全隐藏了。

`vllm_provider.py` 里还顺带做了第九部分见过的**别名归一化**([_normalize_vllm_chat_template_kwargs](../backend/packages/harness/deerflow/models/vllm_provider.py#L39-L62)):DeerFlow 老文档写的是 `chat_template_kwargs.thinking`,而 vLLM 0.19 的 Qwen 解析器读的是 `enable_thinking`,所以发送前把老 key 映射成新 key。

## 6. 把重复的匹配逻辑抽出来:assistant_payload_replay

"多轮对话里把上一轮的 reasoning 回传"这件事,vLLM、DeepSeek 都要做。但它们面临一个共同的难题:**payload 里的 assistant 消息,怎么和原始的 `AIMessage` 对应起来**?理想情况下两个列表一一对应,但序列化过程可能会丢弃或重排消息,数量就对不上了。

这个匹配逻辑被抽进了 [`assistant_payload_replay.py`](../backend/packages/harness/deerflow/models/assistant_payload_replay.py),而 DeepSeek 的补丁([patched_deepseek.py](../backend/packages/harness/deerflow/models/patched_deepseek.py))就薄薄一层——它只重写 `_get_request_payload`,把匹配和恢复全委托给共享函数:

```python
restore_assistant_payloads(
    payload.get("messages", []),
    original_messages,
    restore_reasoning_content,     # 只传一个"要恢复哪个字段"的回调
)
```

[`restore_assistant_payloads`](../backend/packages/harness/deerflow/models/assistant_payload_replay.py#L20-L39) 的匹配策略是分层降级的:数量相等就按位置一一对应;数量不等就退化到"内容+tool_call_id 签名匹配"([_match_ai_message](../backend/packages/harness/deerflow/models/assistant_payload_replay.py#L54-L72)),签名唯一命中才用,否则再降级到"从当前序号往后找第一个没用过的"。**匹配逻辑共享、"恢复哪个字段"用回调注入**——这跟第九部分 MCP `_convert_call_tool_result` 里用 `_resolve_text`/`_resolve_link_url` 钩子注入路径翻译是同一个设计模式:**把稳定的骨架抽成公共函数,把易变的那一小块用回调传进去。**

但注意:vLLM 没用这个共享函数,自己写了一份几乎一样的匹配逻辑([vllm_provider.py:181-189](../backend/packages/harness/deerflow/models/vllm_provider.py#L181-L189))。这是一处**没有完全收敛的重复**——可能是历史演进先后不同步。面试里如果被追问"你觉得这块代码有什么可改进的",这就是个真实的例子:DeepSeek 已经用了共享的 `restore_assistant_payloads`,vLLM 却还留着一份手写的,理应也切过去。

## 7. MiniMax 的补丁:"reasoning 保留"只是它一半的工作

[`patched_minimax.py`](../backend/packages/harness/deerflow/models/patched_minimax.py) 除了同样的 reasoning 保留,还多做两件很能说明"厂商怪癖"的事:

**`<think>` 标签剥离**([_strip_inline_think_tags](../backend/packages/harness/deerflow/models/patched_minimax.py#L52-L63)):有些 MiniMax 模型不走结构化的 `reasoning_details` 字段,而是直接把思考塞在正文里用 `<think>...</think>` 包起来。补丁用正则把它抠出来、从正文里删掉、放进 `reasoning_content`。所以它同时处理两种来源(结构化字段 + 内联标签),再 `_merge_reasoning` 合并。

**`_strip_user_message_names`**([patched_minimax.py:120-136](../backend/packages/harness/deerflow/models/patched_minimax.py#L120-L136)):这个最典型。DeerFlow 的中间件会给 user 消息打内部来源标签(`user-input`、`summary`、`loop_warning`——第七、八部分见过这些),LangChain 把这些 name 序列化进请求。但 MiniMax 要求所有 user 消息的 name 必须一致,否则直接报 `invalid params, user name must be consistent (2013)`。MiniMax 又不用这个 name,所以补丁在发送前把它全删掉。

这就是 provider 补丁存在的全部理由:**上层(中间件)有它自己的合理设计(给消息打来源标签),某一家厂商的 API 恰好和这个设计冲突,补丁负责在"上层设计"和"厂商现实"之间做一次翻译**,让两边都不用为对方妥协。上层继续打它的标签,MiniMax 继续要它的一致性,补丁在中间把标签抹掉。

## 8. claude_provider:唯一一个"重"provider,以及 OAuth 的一整套

[`claude_provider.py`](../backend/packages/harness/deerflow/models/claude_provider.py) 是唯一继承 `ChatAnthropic`(不是 ChatOpenAI)的,也是最重的一个。它做三件正交的事:

**OAuth Bearer 认证**。Claude Code 的 OAuth token(`sk-ant-oat` 前缀,靠 [`is_oauth_token`](../backend/packages/harness/deerflow/models/credential_loader.py#L29-L31) 识别)不能用标准的 `x-api-key` header,得用 `Authorization: Bearer`,还要带特定的 `anthropic-beta` header,并且注入一个 billing header 到 system prompt 第一块。[`model_post_init`](../backend/packages/harness/deerflow/models/claude_provider.py#L69-L126) 里检测到 OAuth token 就切认证模式、[`_patch_client_oauth`](../backend/packages/harness/deerflow/models/claude_provider.py#L128-L132) 把底层 SDK client 的 `api_key` 换成 `auth_token`。凭证从哪来?[`credential_loader.py`](../backend/packages/harness/deerflow/models/credential_loader.py) 有一套多来源查找(环境变量 → 文件描述符 → 显式路径 → `~/.claude/.credentials.json`),还带过期检查。

**Prompt Caching**([_apply_prompt_caching](../backend/packages/harness/deerflow/models/claude_provider.py#L192-L248))。Anthropic 的缓存断点(`cache_control`)硬上限是 4 个,所以补丁按文档顺序收集候选块(system → 最近几条消息 → 最后一个 tool 定义),**只给最后 4 个打断点**——因为越靠后的断点覆盖的前缀越长、命中率越高。这是一个很实在的成本优化细节:缓存断点是稀缺资源,要花在覆盖面最大的地方。

**自动思考预算**([_apply_thinking_budget](../backend/packages/harness/deerflow/models/claude_provider.py#L250-L261)):思考模式开启时,自动把 `budget_tokens` 设成 `max_tokens` 的 80%。

还有一个安全细节:OAuth token 最多只允许 4 个 `cache_control` 块,而且和 prompt caching 冲突,所以检测到 OAuth 时**直接把 prompt caching 关掉**([claude_provider.py:109-110](../backend/packages/harness/deerflow/models/claude_provider.py#L109-L110)),并且在真正发请求前用 [`_strip_cache_control`](../backend/packages/harness/deerflow/models/claude_provider.py#L263-L289) 把所有 `cache_control` 标记清干净。加上它自带的指数退避重试([_generate](../backend/packages/harness/deerflow/models/claude_provider.py#L296-L319) 同步、[_agenerate](../backend/packages/harness/deerflow/models/claude_provider.py#L321-L346) 异步各一份),这个 provider 几乎是一个独立的小系统。

## 9. tracing 挂载:又一处"同一次调用不能记两遍"

factory 结尾([factory.py:198-204](../backend/packages/harness/deerflow/models/factory.py#L198-L204))有个 `attach_tracing` 开关,默认 True。但 docstring 里花了一大段解释**什么时候必须传 False**:如果调用方(比如 `make_lead_agent`、in-graph 的 `TitleMiddleware`)已经在 graph 调用根部挂了 tracing,这里就绝不能再在 model 层挂一遍——否则同一次 LLM 调用会产生两条 span(一条挂在 graph 根、一条挂在 model),而且 model 变成嵌套观测后,`langfuse_*` 的 session_id/user_id 元数据会被剥掉传不上去。只有**脱离 graph 单独调模型的场景**(`MemoryUpdater` 这种)才保持默认 True。

这和第九部分 MCP、第八部分记忆里反复出现的"同一件事只能做一次"是同一类考量,只是这次是"同一次调用的 trace 只能记在一个层级"。它也印证了第八部分讲过的 tracing 必须挂在 graph 根的原因(Langfuse 只在 `parent_run_id=None` 时才把元数据提升到根 trace)。

## 10. 文档纠偏:2 个文件讲不清"吸收厂商差异"这件事

`learn.md` 第 12 节列的两个文件(`factory.py`/`vllm_provider.py`)覆盖的是"反射实例化 + 一个 provider 示例"。但这个模块真正的工作量在**用一层 provider 补丁吸收 8 家厂商的 API 差异**,而承载它的 `claude_provider.py`(OAuth + prompt caching + 重试)、`credential_loader.py`(多来源凭证 + 过期检查)、`patched_minimax.py`/`patched_deepseek.py`/`patched_mimo.py`/`patched_stepfun.py`/`patched_openai.py`(各家 reasoning/参数怪癖)、`assistant_payload_replay.py`(共享的消息匹配骨架)一个都没列。

CLAUDE.md 的 "Model Factory" / "vLLM Provider" 两节比 learn.md 细,提到了反射实例化、`when_thinking_enabled` 归一化、`supports_vision`、vLLM 保留 reasoning。但它同样没有触及:factory 里"关思考"的四分支方言、三个 LangChain 默认值补丁、Claude 的 OAuth/缓存断点预算、MiniMax 的 user name 一致性坑、以及"provider 补丁"这个贯穿 8 个文件的统一模式本身。

规律和第七~十部分完全一致:**文档写的结论都对,但"为什么需要一层补丁""每家厂商到底怪在哪""factory 那个 bool 背后有多少方言"这些工程密度最高、最能讲出深度的部分,恰恰被略过了。** 模型抽象层尤其如此——它表面是"配置驱动 + 反射",内核是"如何让一个统一接口吸收掉现实世界里参差不齐的厂商 API",后者才是这十几个文件真正在做的事。

## 11. 小结:一次 create_chat_model 调用的全链路

```
create_chat_model(name, thinking_enabled)                         [factory.py]
  ├─ config.get_model_config(name)                                 读配置
  ├─ resolve_class(model_config.use, BaseChatModel)                反射加载 provider 类
  ├─ model_config.model_dump(exclude={能力字段})                   抽出要透传的 provider 参数
  ├─ 思考模式翻译:一个 thinking_enabled → 多家方言
  │    ├─ 开:合并 when_thinking_enabled + thinking 快捷字段
  │    └─ 关:四分支(用户显式 / extra_body / vLLM kwargs / 原生 anthropic)
  ├─ LangChain 默认值补丁(仅 OpenAI 兼容路径):
  │    ├─ _enable_stream_usage_by_default   (否则 token 统计为空)
  │    ├─ _apply_stream_chunk_timeout_default (60s→240s,非 OpenAI 路径删掉此 key)
  │    └─ Codex / MindIE 特判(reasoning_effort 映射 / max_retries)
  ├─ model_class(**kwargs, **settings)                             实例化
  └─ attach_tracing ? 挂 tracing callback : 跳过(避免双重 span)   [呼应第八部分]

provider 补丁(继承 LangChain 基类,只重写钩子)
  _get_request_payload         请求发出前:加/删/恢复字段
       ├─ vLLM / DeepSeek:回传上一轮 reasoning(多轮思考连续性)
       │     └─ restore_assistant_payloads(共享匹配骨架 + 字段回调)  [assistant_payload_replay.py]
       ├─ MiniMax:开 reasoning_split + 删 user name(一致性坑 2013)
       └─ Claude:注入 OAuth billing header + prompt caching + thinking budget
  _create_chat_result          非流式响应:提取 provider 特有字段 → additional_kwargs
  _convert_chunk_to_generation_chunk  流式 delta:同上,逐块处理

凭证(Claude/Codex OAuth)                                          [credential_loader.py]
  多来源查找:env → 文件描述符 → 显式路径 → ~/.claude/.credentials.json
  + 过期检查(sk-ant-oat 前缀识别 OAuth token)
```

从"agent 调 `model.ainvoke(messages)`"到"底层可能是 Anthropic OAuth Bearer + prompt caching、也可能是 vLLM 的 Qwen 带 reasoning 回传、还可能是 MiniMax 要抹掉 user name",中间这层抽象的价值,是让上层**永远不需要知道当前是哪家、哪家有什么坑**。反射实例化解决的是"加新厂商不用改主逻辑",而占了大半代码量的 provider 补丁解决的是一个更琐碎但更真实的问题:**现实世界的 LLM API 没有一个真正统一的标准,总得有一层代码把这些参差不齐抹平**。这也是这个模块最适合在面试里讲的地方——它不是"调个 SDK"那么简单,而是一个"用统一接口 + 可插拔补丁吸收厂商碎片化"的完整方案。
