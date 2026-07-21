# Context Engineering(摘要与预算)—— 面试问题链

> 本文档按"大厂面试追问链"组织:每条问题链从基础问题开始,层层追问到实现细节、设计权衡、异常边界。配套深读笔记:[07-context-engineering.md](07-context-engineering.md)(深读笔记讲"怎么实现",本文档讲"怎么被问、怎么答")。
> 代码引用格式:[文件名.py:起始行-结束行](../backend/仓库相对路径#L起始行-L结束行)。只引用你实际读过的行,禁止编造行号。

本模块覆盖四个 middleware(摘要、工具输出预算、动态上下文注入、token 用量)和三个配置类(`SummarizationConfig`、`ToolOutputConfig`、`TokenUsageConfig`),主线问题是:**怎么在长对话、大工具输出、外部依赖不稳定三重压力下,既不爆 context,又不丢信息,还不阻塞请求**。8 条问题链按"触发 → 替换 → 流式隔离 → 救援 → 联动 → 外部化 → 计数 → 记忆"的递进顺序组织,建议按序准备。

## 问题链 1:Summarization 的触发条件与保留策略

**Q1.1(基础)** DeerFlow 的对话摘要什么时候触发?触发后保留多少历史?

**参考回答**:触发和保留都由 `SummarizationConfig` 的 `trigger` / `keep` 两个参数控制。`trigger` 支持三种维度 —— `messages`(消息条数)、`tokens`(token 总数)、`fraction`(占模型最大输入 token 的比例,如 0.8 表示 80%),且可以配成列表,任一条件满足即触发;`keep` 默认保留最近 20 条消息 [summarization_config.py:32-45](../backend/packages/harness/deerflow/config/summarization_config.py#L32-L45)。运行时 middleware 在每个模型调用前的 `before_model` / `abefore_model` 钩子里先 `_ensure_message_ids` 补齐消息 ID,再用 `token_counter` 统计全量消息的总 token,交给父类 `_should_summarize` 判断,不达标直接返回 `None` 放行 [summarization_middleware.py:195-201](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L195-L201)。工厂函数 `_create_summarization_middleware` 把 pydantic 的 `ContextSize` 模型转成 LangChain 父类期望的 `(type, value)` 元组再注入 [agent.py:87-96](../backend/packages/harness/deerflow/agents/lead_agent/agent.py#L87-L96)。注意配置里 `enabled` 默认是 `False`,摘要能力默认关闭,要显式打开 [summarization_config.py:24-27](../backend/packages/harness/deerflow/config/summarization_config.py#L24-L27)。

答题时可以把三个触发维度各自的使用场景讲清楚,会显得真的用过:

- `messages`:与模型无关的硬兜底,防止"轮数太多"撑爆请求体或拖慢 TTFT;
- `tokens`:绝对预算,适合 context 窗口固定的自托管模型;
- `fraction`:相对预算,自动适配不同模型的窗口,0.8 是留出输出余量的经验值。

**链路解析**:

```
config.yaml ──> SummarizationConfig (trigger / keep: messages 20 / trim 4000)
      │  ContextSize.to_tuple() → ("tokens", 4000) 等
      ▼
_create_summarization_middleware() ──> DeerFlowSummarizationMiddleware
      ▼  每次模型调用前
before_model → _ensure_message_ids → token_counter(messages)
      ▼
_should_summarize? ──否──> return None(原样放行)
      │是
      ▼
_determine_cutoff_index → partition → 摘要 → 重写历史
```

**Q1.2(深挖)** `trigger` 传一个列表是什么语义?为什么 `keep` 默认 20 条消息而不是按 token?

**参考回答**:列表语义是"OR"——配置描述里明确写了 "When any threshold is met, summarization runs" [summarization_config.py:32-38](../backend/packages/harness/deerflow/config/summarization_config.py#L32-L38),这样可以同时兜底两种爆 context 的路径:"消息很多但每条很短"用 `messages: 50` 兜,"消息不多但单条巨长"(比如塞了几个大文件)用 `tokens` 或 `fraction` 兜。`keep` 默认 `messages: 20` 是因为消息条数是跨模型稳定的单位:不同模型的 max input tokens 差异很大,按 fraction 保留在小窗口模型上可能只剩两三轮对话,按固定条数能保证模型至少看到最近约 10 轮完整交互,行为可预期、可测试。另外触发判定每轮模型调用前都全量跑一次 `token_counter`,这是有意的取舍:阈值之下的历史规模有限,一次编码是毫秒级,真正贵的 LLM 摘要只在触发后发生一次;做增量计数反而会引入"历史指纹缓存 + 失效判断"的复杂度,收益不明显。还有个工程细节:`trigger` 缺省是 `None`,此时完全交给 LangChain 父类的默认策略,DeerFlow 自己不发明第二套默认值 [summarization_config.py:32-38](../backend/packages/harness/deerflow/config/summarization_config.py#L32-L38)、[agent.py:87-93](../backend/packages/harness/deerflow/agents/lead_agent/agent.py#L87-L93)。

一句话总结这条链的回答主线:触发是多维度 OR 的"任一过载即压缩",保留是面向交互轮次的固定窗口,两者通过父类策略参数化——DeerFlow 层只做配置翻译和守卫,不重写策略本身。这也是整个 middleware 的设计基调:扩展点全部挂在 LangChain 父类的钩子上,避免和上游策略演进冲突。

**Q1.3(边界/异常)** 触发判定通过了,但 `_determine_cutoff_index` 算出的 cutoff 是 0 或负数,会发生什么?摘要输入本身太长又怎么办?

**参考回答**:cutoff ≤ 0 时直接放弃本轮摘要、返回 `None`,历史原样送进模型——这是防御性分支:当 keep 策略已经覆盖现有历史时,强行摘要只会丢信息 [summarization_middleware.py:203-205](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L203-L205)。摘要输入端有第二道保险:`trim_tokens_to_summarize` 默认 4000 token,送入摘要模型的消息先被 `_trim_messages_for_summary` 裁剪 [summarization_config.py:46-49](../backend/packages/harness/deerflow/config/summarization_config.py#L46-L49);若裁剪后一条不剩,`_build_summary_prompt` 返回 `None`,摘要退化为固定文案 "Previous conversation was too long to summarize." [summarization_middleware.py:179-187](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L179-L187)。摘要 LLM 调用本身抛异常也不会炸主流程,而是把 `"Error generating summary: ..."` 当成摘要文本继续 [summarization_middleware.py:160-161](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L160-L161)。

把降级链记成四级,面试时按顺序背:

1. cutoff ≤ 0 → 本轮不摘要(历史原样);
2. trim 后为空 → 固定文案占位;
3. 摘要 LLM 异常 → 错误字符串当摘要;
4. 空输入 → "No previous conversation history." [summarization_middleware.py:149-150](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L149-L150)。

每一级都保证"用户请求不失败",区别只是摘要质量从高到低。

**反例分析**:如果不做"摘要失败降级为字符串"而是让异常上抛,一次摘要模型的限流/超时就会让整个用户请求失败——而保留旧历史继续跑(哪怕 context 快满)几乎是严格更优的选择。这就是 `_summarize_with` 用 try/except 把摘要错误吞成文本的原因:压缩是优化手段,不是正确性依赖。

## 问题链 2:摘要后的消息替换机制

**Q2.1(基础)** 摘要生成之后,state 里的消息列表是怎么被替换的?

**参考回答**:返回一个 LangGraph 状态更新:先放一条 `RemoveMessage(id=REMOVE_ALL_MESSAGES)` 清空全部历史,然后按顺序写入"摘要消息 + 保留消息",由 add_messages reducer 重建消息列表 [summarization_middleware.py:213-219](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L213-L219)。摘要消息被构造成一条 `HumanMessage`,内容以 "Here is a summary of the conversation to date:" 开头,并且带 `name="summary"` 标记 [summarization_middleware.py:247-252](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L247-L252)。同步和异步两条路径(`_maybe_summarize` / `_amaybe_summarize`)的替换结构完全一致,只是摘要调用分别走 `invoke` / `ainvoke` [summarization_middleware.py:221-245](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L221-L245)。

值得强调的执行顺序(摘要是"先救后压"):

1. `_partition_with_skill_rescue` —— 先切分并救回 skill;
2. `_preserve_dynamic_context_reminders` —— 再救回 reminder;
3. `_fire_hooks` —— 通知记忆系统(final 待摘要集合);
4. `_create_summary` —— 最后才调 LLM。

hooks 拿到的是救援完成后的最终待摘要集合,所以 flush 进长期记忆的语料和被 LLM 压缩的语料严格一致 [summarization_middleware.py:207-211](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L207-L211)。

**链路解析**:

```
state.messages (N 条, 超阈值)
   │
   ├─ cutoff ─→ messages_to_summarize ─→ LLM 摘要 ─→ HumanMessage(name="summary")
   │                                                       │
   └─────────→ preserved_messages ◄────────────────────────┘
   ▼
[RemoveMessage(REMOVE_ALL), summary_msg, *preserved]  → add_messages 重建历史
```

**Q2.2(深挖)** 为什么摘要要伪装成 `HumanMessage` 而不是 `SystemMessage`?`name="summary"` 这个标记有什么用?

**参考回答**:两个原因。第一,兼容性:很多模型(尤其非 OpenAI 系)对 system 消息的数量和位置有限制,放在 human 角色里最稳,LangChain 父类就是这么设计的,DeerFlow 只覆写 `_build_new_messages` 加了 name 标记。第二,`name="summary"` 是一个跨 middleware 契约:前端据此不渲染这条消息——注释明说 "ignored to display in the frontend, but still can be used as context for the model" [summarization_middleware.py:249-251](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L249-L251);DynamicContextMiddleware 也用它排除注入目标——`_is_user_injection_target` 把 `name == "summary"` 的消息踢出注入候选 [dynamic_context_middleware.py:83-85](../backend/packages/harness/deerflow/agents/middlewares/dynamic_context_middleware.py#L83-L85),防止日期/记忆 reminder 被注到摘要消息前面。摘要 prompt 侧也有讲究:消息先经 `get_buffer_string` 格式化为纯文本再套模板,而不是 `str()` 消息对象——后者会把 `additional_kwargs` / `response_metadata` 序列化进去,白白抬高 token 量并干扰摘要模型 [summarization_middleware.py:184-187](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L184-L187)。

可以把这个标记的消费方列全,说明它是事实上的协议:

- 前端渲染层:跳过 `name == "summary"` 的消息;
- DynamicContextMiddleware:`_is_user_injection_target` 排除它 [dynamic_context_middleware.py:83-85](../backend/packages/harness/deerflow/agents/middlewares/dynamic_context_middleware.py#L83-L85);
- 模型本身:仍然把它当上下文读——隐藏的只是 UI,不是语义。

**Q2.3(边界/异常)** 摘要会不会把 AIMessage 的 tool_calls 和对应 ToolMessage 拆散,造成 dangling tool call?

**参考回答**:不会。父类的 `_partition_messages` 保证 cutoff 不落在 AI tool_call 与其 ToolMessage 之间;DeerFlow 的 skill 救援在拆分 AI 消息时也维持配对不变式:被救援的 tool_calls 克隆成一条只含这些调用、`content=""` 的新 AIMessage,其余 tool_calls 留在另一条克隆里进摘要,对应 ToolMessage 按索引各自跟随 [summarization_middleware.py:300-318](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L300-L318)。也就是说一条混合调用的 AIMessage(比如同时调了 read_file 和 bash)可能被拆成两条,但每条新 AIMessage 的 tool_calls 都能在历史上找到配对 ToolMessage。skill bundle 的扫描也是按"AI 消息之后连续 ToolMessage 区间"成块推进的 [summarization_middleware.py:353-363](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L353-L363),天然不会跨块切割。拆分时还有一个细节:如果被救走 tool_calls 之后原消息"既没有剩余 tool_call 也没有 content",整条消息就不再进 remaining——避免往摘要输入里塞一条空 AI 消息 [summarization_middleware.py:306-310](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L306-L310)。万一历史里真出现了 dangling(比如旧的 checkpoint 数据),DeerFlow 另有独立的 `dangling_tool_call_middleware` 兜底修复,但那是摘要之外的安全网,不能替代 partition 自身的不变式。面试官若继续追问"为什么用 `RemoveMessage(REMOVE_ALL)` 全量重写而不是逐条删除":因为重写后的历史是 "summary + preserved" 的全新排列(preserved 里还插着被救回的 skill bundle 和 reminder,顺序与原历史不同),逐条 Remove 要精确枚举每条待删消息的 ID,既啰嗦又易漏;`REMOVE_ALL_MESSAGES` 是原子语义,一条指令清空再由 add_messages 按序重建,配合 `_ensure_message_ids` 补齐的 ID,整个替换是可重放、可 checkpoint 的确定操作 [summarization_middleware.py:213-219](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L213-L219)。

回答这类"配对完整性"问题时的通用套路:先讲不变式(tool_calls 与 ToolMessage 一一对应、顺序连续),再讲每个变换步骤分别如何维持不变式(partition 不跨块、救援按 id 集合拆分、克隆保留元数据),最后讲兜底(独立 middleware 修复历史遗留)。三层都答到,基本就封顶了。

## 问题链 3:TAG_NOSTREAM 与"幻影消息"问题

**Q3.1(基础)** 摘要也是一次 LLM 调用,它的 token 流会不会被推送给前端?

**参考回答**:不处理就会。摘要 LLM 调用发生在 LangGraph middleware 钩子内部,默认会被 messages-tuple stream callback 捕获,作为一条普通 AI 消息流式广播给前端——用户就看到一条莫名其妙的"幻影消息"。解法是在 `__init__` 里构造一个带 `TAG_NOSTREAM` 标签的模型副本 `self._summary_model`,流式处理器见到这个 tag 就跳过 [summarization_middleware.py:120-131](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L120-L131)。摘要调用统一走这个副本的 `invoke` / `ainvoke`,并带 `metadata={"lc_source": "summarization"}` 便于可观测归因 [summarization_middleware.py:154-158](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L154-L158)。

这里有个反直觉点值得主动讲:副本只换了"流式可见性",没有换模型本身——摘要用的还是工厂选好的那个 chat model(可配 `model_name`,缺省用轻量默认模型 [summarization_config.py:28-31](../backend/packages/harness/deerflow/config/summarization_config.py#L28-L31)),且创建时显式 `thinking_enabled=False`、`attach_tracing=False`:摘要不需要推理开销,tracing 回调由图级 config 统一携带,重复绑定会产生重复 span 并破坏 session/user 传播 [agent.py:98-110](../backend/packages/harness/deerflow/agents/lead_agent/agent.py#L98-L110)。

**链路解析**:

```
agent 主模型 (无 TAG_NOSTREAM) ──stream──> SSE ──> 前端渲染
_summary_model (TAG_NOSTREAM) ──stream──> stream handler 见 tag → 丢弃
                                          invoke 完整返回值 → 写成 summary HumanMessage
```

**Q3.2(深挖)** 为什么不直接给 `self.model` 加 tag,或者摘要时临时换模型?

**参考回答**:三个递进的细节。第一,父类的 `profile` / `_get_ls_params` 检查依赖原始模型,`self.model` 必须保持无 tag [summarization_middleware.py:124-125](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L124-L125)。第二,middleware 实例被缓存并跨并发 run 复用:如果在 `_create_summary` 里临时把 `self.model` 换成 tagged 副本,`await` 挂起期间另一个协程会拿到被换过的模型,`RunnableBinding` 就泄漏到别的请求里——所以必须在 `__init__` 一次性建好副本 [summarization_middleware.py:141-148](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L141-L148)。第三,合并 tag 不能覆盖已有 tag:工厂创建模型时已绑了 `"middleware:summarize"` 用于 RunJournal 归因 [agent.py:99-110](../backend/packages/harness/deerflow/agents/lead_agent/agent.py#L99-L110),而 `RunnableBinding.with_config` 对 tags 是浅合并、会整体覆盖,所以代码先读出 `existing_tags` 再追加,且 `TAG_NOSTREAM` 已存在时不重复添加(幂等)[summarization_middleware.py:129-131](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L129-L131)。

这类"实例级临时可变状态"的坑在 middleware 体系里是通病了,因为 LangChain middleware 的生命周期是"应用级单例"而不是"请求级"。凡是要随调用变化的配置,都应该像这里一样:要么预建成不可变成员(`_summary_model`),要么通过 `invoke(config=...)` 的调用级参数传入(比如 `metadata={"lc_source": "summarization"}` 就是调用级传入,不污染共享模型 [summarization_middleware.py:155-158](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L155-L158))。

**反例分析**:如果图省事直接 `self.model = self.model.with_config(tags=[TAG_NOSTREAM])`,会同时踩两个坑:已有 `"middleware:summarize"` tag 被覆盖导致 RunJournal 把摘要调用归错组件;父类的模型 introspection 拿到 tagged 副本,`ls_params` 行为改变。如果临时 swap 再换回来,并发下另一个请求的流式输出会被静默吞掉——这类 bug 极难复现,所以代码用"不可变的预建副本"从结构上消除它。反过来,如果什么 tag 都不加,摘要模型的每一个 token 都会以 AI 消息块的形式流到前端,用户看到助手"自言自语"一大段总结——这正是 TAG_NOSTREAM 要消灭的幻影消息。

## 问题链 4:Skill 救援(skill rescue)

**Q4.1(基础)** 摘要把历史压掉之后,agent 刚加载的 skill 文件内容也没了,DeerFlow 怎么解决?

**参考回答**:在父类 partition 之后加一步"skill 救援":在待摘要消息里找出"读取 skill 文件"的 AIMessage + ToolMessage 组合(抽象为 `_SkillBundle`,记录 AI 索引、ToolMessage 索引、tool_call_id 集合、token 数和去重 key [summarization_middleware.py:88-96](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L88-L96)),把最近加载的几个 bundle 从待摘要集合搬回保留集合,放在 preserved 之前 [summarization_middleware.py:272-318](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L272-L318)。判定一次 tool_call 是不是 skill 读取看两点:工具名必须在 `{"read_file", "read", "view", "cat"}` 里,且 `path`/`file_path`/`filepath` 参数必须落在 skills 容器根(默认 `/mnt/skills`)之下——判断时先 `rstrip("/")` 再比较等值或前缀,防止 `/mnt/skills2` 这种兄弟目录误判 [summarization_middleware.py:114-115](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L114-L115)、[summarization_middleware.py:410-419](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L410-L419)。

几个容易漏的判定细节:

- 路径参数名兼容三种写法(`path` / `file_path` / `filepath`),取第一个非空字符串 [summarization_middleware.py:66-75](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L66-L75);
- `args` 不是 dict 直接放弃判定,防止畸形 tool_call 崩溃 [summarization_middleware.py:68-70](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L68-L70);
- skills 根路径来自 app 配置 `skills.container_path`,缺省 `/mnt/skills`,和工具名列表一样可在 config 里覆盖 [agent.py:132-141](../backend/packages/harness/deerflow/agents/lead_agent/agent.py#L132-L141)、[summarization_config.py:69-72](../backend/packages/harness/deerflow/config/summarization_config.py#L69-L72)。

**链路解析**:

```
_partition_messages(cutoff)
   │ to_summarize / preserved
   ▼
_find_skill_bundles(to_summarize, "/mnt/skills")
   │  [AI(read_file /mnt/skills/x/SKILL.md) + ToolMessage(内容)]
   ▼
_select_bundles_to_rescue   ← 三个预算闸门(见 Q4.2)
   ▼
拆分 AI 消息: 被救 tool_calls → 新 AI 克隆(content="")
             其余 tool_calls → 留在原 AI 克隆进摘要
   ▼
remaining(去摘要) , rescued + preserved(留下)
```

**Q4.2(深挖)** 救援不是无上限的吧?具体有哪几个预算?为什么从最新往旧选?

**参考回答**:三个预算闸门,全在 `_select_bundles_to_rescue` 里:(1)数量上限 `preserve_recent_skill_count` 默认 5 个 bundle;(2)单 skill 上限 `preserve_recent_skill_tokens_per_skill` 默认 5000 token,单个超过就不救;(3)总预算 `preserve_recent_skill_tokens` 默认 25000 token,累计超出就跳过一个继续看更旧的 [summarization_middleware.py:107-119](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L107-L119)、[summarization_middleware.py:392-405](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L392-L405)。遍历用 `reversed(bundles)` 即最新优先——刚加载的 skill 大概率是 agent 当前任务正在用的,最旧的可以靠重新 `read_file` 补回。还有去重:同一组 skill 路径(排序后用 `|` 拼成 `skill_key`)只救一次,避免 agent 反复读同一文件时多份拷贝吃预算 [summarization_middleware.py:375](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L375)、[summarization_middleware.py:395-405](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L395-L405)。选完后 `selected.reverse()` 恢复时间序,保证拼回历史时顺序不乱 [summarization_middleware.py:407-408](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L407-L408)。

注意预算检查用的是 `continue` 而不是 `break`:某个 bundle 太贵就跳过它、继续看更旧更小的,而不是直接停——这样"1 个 8000 token 的大 skill + 3 个 2000 token 的小 skill"的场景下,三个小 skill 依然能被救回 [summarization_middleware.py:397-400](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L397-L400)。只有数量上限用的是 `break`,因为数量是按最新优先消耗的,再往后翻也没有意义 [summarization_middleware.py:393-394](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L393-L394)。

**Q4.3(边界/异常)** 救援逻辑本身出 bug 抛异常怎么办?怎么彻底关掉它?

**参考回答**:`_find_skill_bundles` 整段包在 try/except 里,任何异常都 `logger.exception` 后回退到父类默认 partition——摘要照常进行,只是 skill 不被特殊保留 [summarization_middleware.py:283-287](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L283-L287)。关闭方式是把 `preserve_recent_skill_count` 或 `preserve_recent_skill_tokens` 配成 0,`_partition_with_skill_rescue` 开头直接短路 [summarization_middleware.py:280-281](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L280-L281)。三个阈值在 `__init__` 里都过了 `max(0, ...)`,负数配置被钳到 0 而不是抛错 [summarization_middleware.py:117-119](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L117-L119)。

bundle 扫描的循环结构也有边界意识:遇到非 AI 消息或无 tool_calls 的 AI 消息就 `i += 1` 单步推进;遇到带 skill 调用的 AI 消息则一口气跳过其后整段连续 ToolMessage(`i = j`),保证同一块不被重复扫描 [summarization_middleware.py:327-378](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L327-L378)。扫描是 O(n) 单趟,没有嵌套循环,长历史上成本可控。

降级路径再补一刀:即使 `_find_skill_bundles` 成功但一个 bundle 都没找到、或 `_select_bundles_to_rescue` 选出来是空列表,都直接返回父类的原始 partition,不做任何克隆——也就是说没有 skill 的历史走救援路径是零成本的 [summarization_middleware.py:289-294](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L289-L294)。这与整个中间件"失败即回退父类行为"的设计基调一致:所有 DeerFlow 定制逻辑都是可选增强,不是正确性依赖。

**反例分析**:如果没有 skill 救援,长任务 agent 每到摘要点就"失忆"自己刚加载的 skill 指令,下一步行为漂移,甚至重新 read_file 把同一份内容再拉一遍——费 token 且行为不稳定。反过来,若救援不设 25000 token 总预算,一个异常大的 skill 目录会让"被保留"的消息本身撑爆 context,摘要等于白做。预算的存在是为了让救援收益与摘要的压缩目标共存,而不是互相抵消。

## 问题链 5:摘要与 DynamicContext reminder 的相互作用

**Q5.1(基础)** DynamicContextMiddleware 注入的 `<system-reminder>` 消息,摘要时会被压进摘要文本吗?

**参考回答**:不会。`_preserve_dynamic_context_reminders` 在 partition 之后、调摘要 LLM 之前,把所有 reminder(靠 `additional_kwargs["dynamic_context_reminder"]` 标记识别,而非内容匹配 [dynamic_context_middleware.py:64-66](../backend/packages/harness/deerflow/agents/middlewares/dynamic_context_middleware.py#L64-L66))从待摘要列表挪到保留列表最前面 [summarization_middleware.py:254-270](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L254-L270)。用标记而不是内容子串匹配是有意的:用户消息里自己写了 `<system-reminder>` 字样也不会被误判为注入消息 [dynamic_context_middleware.py:69-80](../backend/packages/harness/deerflow/agents/middlewares/dynamic_context_middleware.py#L69-L80)。

reminder 消息本身的长相也值得记住:内容是 `<system-reminder>` 包裹的可选 `<memory>` 段 + `<current_date>2026-05-08, Friday</current_date>` 段;`additional_kwargs` 里同时带 `hide_from_ui: True`(前端不渲染)和 `dynamic_context_reminder: True`(机器可识别)两个标记 [dynamic_context_middleware.py:14-27](../backend/packages/harness/deerflow/agents/middlewares/dynamic_context_middleware.py#L14-L27)、[dynamic_context_middleware.py:149-154](../backend/packages/harness/deerflow/agents/middlewares/dynamic_context_middleware.py#L149-L154)。"给人看的"和"给程序判的"分离,是这个设计最干净的地方。

**链路解析**:

```
partition 后:
  to_summarize = [reminder?, ..., msg_k]
        │
        ▼  _preserve_dynamic_context_reminders
  to_summarize' = [非 reminder 消息]
  preserved'    = [reminder] + preserved      ← reminder 永远留在上下文最前
        │
        ▼
  _create_summary(to_summarize')   ← 摘要文本不含 reminder
```

**Q5.2(深挖)** 为什么 reminder 不能被摘要掉?注释说的 "inject the reminder in the wrong place" 具体是什么 bug?

**参考回答**:DynamicContextMiddleware 的首轮注入条件是"当前历史扫不到任何已注入日期"(`_last_injected_date` 返回 None),注入点是第一条满足 `_is_user_injection_target` 的 HumanMessage [dynamic_context_middleware.py:177-190](../backend/packages/harness/deerflow/agents/middlewares/dynamic_context_middleware.py#L177-L190)。如果 reminder 被摘要吞掉,下一轮扫描就以为"这是第一轮",于是把一条新的完整 reminder 插到摘要消息之后或当前 turn 之前——日期/记忆被重复注入且位置错乱,这正是注释描述的 bug [summarization_middleware.py:259-264](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L259-L264)。保留 reminder 还顺带保住 prefix cache:reminder 是冻结快照、内容不再变化,后续每轮都能命中缓存前缀——这是 DynamicContextMiddleware 把 system prompt 保持全静态、把动态量塞进首条消息的整体设计 [dynamic_context_middleware.py:88-104](../backend/packages/harness/deerflow/agents/middlewares/dynamic_context_middleware.py#L88-L104)。

这个设计背后是三个 middleware 的一组合约,面试时最好主动串起来:

- DynamicContext 负责"注入并打标记";
- Summarization 负责"识别标记并豁免";
- 摘要消息用 `name="summary"` 反向标记自己,防止被 DynamicContext 误认为首轮用户消息 [dynamic_context_middleware.py:83-85](../backend/packages/harness/deerflow/agents/middlewares/dynamic_context_middleware.py#L83-L85)。

任何一环破坏合约(比如自己手写一条 `name="summary"` 的用户消息),都会让另外两环的判定失真——这是隐式契约的代价。

**Q5.3(边界/异常)** 跨午夜会发生什么?这些注入消息的消息 ID 会不会撞车?

**参考回答**:跨午夜时,`_last_injected_date` 扫到的历史日期与 `datetime.now()` 不等,middleware 给当前最后一条用户消息前插一条轻量 date-update reminder 并持久化;之后同一天的轮次看到新日期就不再重复注入 [dynamic_context_middleware.py:192-203](../backend/packages/harness/deerflow/agents/middlewares/dynamic_context_middleware.py#L192-L203)。date-update reminder 同样带 `dynamic_context_reminder` 标记,所以摘要时同样被救回——不会因为一次摘要让系统忘记"今天已经是新的一天"。ID 设计上用"ID 交换"技巧:reminder 消息复用原消息 ID(让 add_messages 原地替换、保持位置),用户消息用 `{id}__user` 派生 ID 紧随其后;原消息没 ID 时生成 UUID,避免 `None__user` 这种歧义 ID;reminder 还带 `hide_from_ui` 让前端不渲染 [dynamic_context_middleware.py:138-161](../backend/packages/harness/deerflow/agents/middlewares/dynamic_context_middleware.py#L138-L161)。

把"持久化"这点说透能加分:date-update 不是每轮临时拼在 prompt 里的,而是作为真实消息写回 state——所以历史里同时存在"首日完整 reminder"和"跨午夜的 date-update reminder"两条,`_last_injected_date` 倒序扫描总是取到最新的那条,判定逻辑天然单调 [dynamic_context_middleware.py:69-80](../backend/packages/harness/deerflow/agents/middlewares/dynamic_context_middleware.py#L69-L80)。如果改成临时注入,checkpoint 恢复后日期状态就会丢失,跨午夜的判定会重注入。

## 问题链 6:Tool output budget 外部化

**Q6.1(基础)** 一个工具返回了 10 万字符的输出,DeerFlow 会怎么处理?

**参考回答**:由 `ToolOutputBudgetMiddleware` 在 `wrap_tool_call` 里拦截,阈值来自 `ToolOutputConfig`:超过 `externalize_min_chars`(默认 12000 字符)就尝试"外部化"——完整内容写磁盘,上下文里只留 head(默认 2000 字符)+ 文件引用 + tail(默认 1000 字符)的预览;磁盘不可用时降级为 head+tail 截断,总长不超过 `fallback_max_chars`(默认 30000 字符,head 8000 / tail 3000)[tool_output_config.py:21-50](../backend/packages/harness/deerflow/config/tool_output_config.py#L21-L50)。工具返回的不只是 ToolMessage,还可能是 LangGraph `Command`,所以 `_patch_result` 分两种类型处理:Command 会遍历其 `update["messages"]` 里的 ToolMessage 逐个 patch,有改动才用 `dataclasses.replace` 生成新 Command,其他 update 字段原样保留 [tool_output_budget_middleware.py:496-528](../backend/packages/harness/deerflow/agents/middlewares/tool_output_budget_middleware.py#L496-L528)。整体开关 `enabled` 默认 True,关掉后所有钩子直通 [tool_output_budget_middleware.py:587-589](../backend/packages/harness/deerflow/agents/middlewares/tool_output_budget_middleware.py#L587-L589)。

预览的构造也有细节:head/tail 都会经 `_snap_to_line_boundary` 对齐到最近的换行符,尽量不切在半行中间;引用块里写明总字符数、估算 token 数和省略字符数,模型能据此判断要不要读全文 [tool_output_budget_middleware.py:74-87](../backend/packages/harness/deerflow/agents/middlewares/tool_output_budget_middleware.py#L74-L87)、[tool_output_budget_middleware.py:206-231](../backend/packages/harness/deerflow/agents/middlewares/tool_output_budget_middleware.py#L206-L231)。磁盘文件名是 `{工具名净化}-{uuid前12位}.{扩展名}`,bash/web_fetch 映射成 `.log`,其余 `.txt`,host 与 sandbox 两条路径共用同一个命名函数,保证行为一致 [tool_output_budget_middleware.py:94-117](../backend/packages/harness/deerflow/agents/middlewares/tool_output_budget_middleware.py#L94-L117)。

**链路解析**:

```
ToolMessage / Command(update.messages)
   │ _needs_budget 快速预检(per-tool 阈值取 min)
   ▼
len > 12000? ──否──> 原样放行
   │是
   ▼ 外部化(三选一)
 sandbox+mounts → 写 host outputs_path ──┐
 sandbox 无mount → sandbox.write_file ───┼─→ 预览: head 2000 + [ref 路径] + tail 1000
 无 sandbox     → 写 host outputs_path ──┘
   │ 全部失败
   ▼ fallback
 head 8000 + [omitted 标记] + tail 3000  (≤ 30000 chars)
```

**Q6.2(深挖)** 写到哪里?为什么有 host 和 sandbox 两条写入路径?虚拟路径是什么?

**参考回答**:`_VIRTUAL_OUTPUTS_BASE = "/mnt/user-data/outputs"` 是模型视角的统一虚拟路径,实际存储子目录是 `.tool-results` [tool_output_budget_middleware.py:39](../backend/packages/harness/deerflow/agents/middlewares/tool_output_budget_middleware.py#L39)、[tool_output_config.py:51-54](../backend/packages/harness/deerflow/config/tool_output_config.py#L51-L54)。三条路径:(1)host-mounted sandbox——host 的 outputs 目录 bind-mount 进沙箱同一路径,直接写 host 即可,省一次沙箱往返;(2)远程 AIO sandbox 不用 thread-data mount——host 写的文件沙箱里不可见(issue #3416),必须走 `sandbox.write_file` 写进沙箱文件系统,写前显式 `mkdir -p`(AIO 的 write_file 不自动建父目录),写完用 `test -s` 验证落盘,验证失败就拒绝把不可读路径交给模型、降级 inline 截断 [tool_output_budget_middleware.py:348-374](../backend/packages/harness/deerflow/agents/middlewares/tool_output_budget_middleware.py#L348-L374)、[tool_output_budget_middleware.py:173-190](../backend/packages/harness/deerflow/agents/middlewares/tool_output_budget_middleware.py#L173-L190);(3)无 sandbox 的工具(web_search、MCP 等)直接写 host。`_resolve_sandbox` 特意不调用 `provider.acquire`,只查内存注册表——acquire 可能触发阻塞远程 I/O,而这个解析在每次工具调用都跑 [tool_output_budget_middleware.py:298-322](../backend/packages/harness/deerflow/agents/middlewares/tool_output_budget_middleware.py#L298-L322)。预览文本里还嵌了 `~{total // 4} tokens` 的粗估和"用 read_file 的 start_line/end_line 分段读"的提示,引导模型按需取回 [tool_output_budget_middleware.py:226](../backend/packages/harness/deerflow/agents/middlewares/tool_output_budget_middleware.py#L226)。

路径选择的决策顺序可以记成一句话:"有沙箱看 mount,无沙箱写 host,都没有就截断"。其中 `uses_thread_data_mounts` 是 provider 的静态属性,不是每次探测出来的,所以判断本身没有 I/O 成本 [tool_output_budget_middleware.py:354](../backend/packages/harness/deerflow/agents/middlewares/tool_output_budget_middleware.py#L354)。host 写入路径所需的 `outputs_path` 来自 `runtime.state["thread_data"]["outputs_path"]`,由 ThreadDataMiddleware 提前写入 state;拿不到(非线程上下文)就跳过 host 外部化 [tool_output_budget_middleware.py:283-295](../backend/packages/harness/deerflow/agents/middlewares/tool_output_budget_middleware.py#L283-L295)。

**Q6.3(边界/异常)** 外部化会不会有路径穿越之类的安全问题?`read_file` 为什么被豁免?阈值配 0 是什么语义?

**参考回答**:三层防护:(1)`storage_subdir` 若是绝对路径或含 `..` 直接拒绝;(2)工具名经 `_sanitize_tool_name` 去掉路径分隔符和 `..`,空名兜底 "unknown";(3)拼出文件路径后校验 `abspath(filepath)` 必须以 `abspath(storage_dir)` 为前缀 [tool_output_budget_middleware.py:101-105](../backend/packages/harness/deerflow/agents/middlewares/tool_output_budget_middleware.py#L101-L105)、[tool_output_budget_middleware.py:128-141](../backend/packages/harness/deerflow/agents/middlewares/tool_output_budget_middleware.py#L128-L141)。`read_file` / `read_file_tool` 在 `exempt_tools` 里默认豁免 [tool_output_config.py:55-58](../backend/packages/harness/deerflow/config/tool_output_config.py#L55-L58),是为了切断"persist→read→persist"死循环:模型读回被外部化的文件时,读到的内容若再次超阈值又被外部化成新文件,循环不止。多模态内容(图片、结构化 block)由 `_message_text` 返回 None 直接跳过预算 [tool_output_budget_middleware.py:51-71](../backend/packages/harness/deerflow/agents/middlewares/tool_output_budget_middleware.py#L51-L71)。

把三层防护抽象一下,其实是安全编码的经典三段式:输入白名单校验(subdir)、每个拼接分量消毒(tool name)、最终结果不变式断言(prefix check)。单靠任何一层都有绕过面(比如符号链接、Unicode 规范化差异),三层叠加才把风险压到可接受。sandbox 路径同样先校验 subdir 再拼接,与 host 路径共用一个命名函数,两条写入路径的安全属性完全一致 [tool_output_budget_middleware.py:152-172](../backend/packages/harness/deerflow/agents/middlewares/tool_output_budget_middleware.py#L152-L172)。阈值语义:`externalize_min_chars = 0` 只关外部化(fallback 仍生效),`fallback_max_chars = 0` 只关截断,两者皆 0 时 `_budget_content` 开头就返回 None,预算系统空转 [tool_output_budget_middleware.py:335-339](../backend/packages/harness/deerflow/agents/middlewares/tool_output_budget_middleware.py#L335-L339);`tool_overrides` 可按工具单独覆盖阈值 [tool_output_config.py:59-62](../backend/packages/harness/deerflow/config/tool_output_config.py#L59-L62)。

fallback 截断本身也有正确性保证:返回串保证不超过 `max_chars`——先算省略标记的开销,如果标记本身比预算还长,就直接硬切 `content[:max_chars]`,绝不超额 [tool_output_budget_middleware.py:242-275](../backend/packages/harness/deerflow/agents/middlewares/tool_output_budget_middleware.py#L242-L275)。这个"保证不超长"的不变式是 fallback 存在的意义:磁盘没了,context 也绝不能被单条工具结果撑爆。

**Q6.4(深挖)** 这个中间件只在工具调用时生效吗?异步路径下沙箱 I/O 会不会卡 event loop?

**参考回答**:不止。它还挂了 `wrap_model_call` / `awrap_model_call`:每次模型调用前扫描历史消息,把漏网的历史超长 ToolMessage 再做一次 inline 截断——历史路径故意不传 sandbox 和 outputs_path,因为历史消息在产生时已被外部化过,这里只剩 fallback 截断要做 [tool_output_budget_middleware.py:531-557](../backend/packages/harness/deerflow/agents/middlewares/tool_output_budget_middleware.py#L531-L557)。性能上两道优化:`_needs_budget` 先做 per-tool 最小阈值预检——`_effective_trigger` 取外部化阈值和 fallback 阈值的较小者,保证预检不产生假阴性 [tool_output_budget_middleware.py:457-470](../backend/packages/harness/deerflow/agents/middlewares/tool_output_budget_middleware.py#L457-L470),小输出完全不走 patch;`_patch_model_messages` 先用 `any()` 扫一遍,没人超预算就不分配新列表,长历史不会每次模型调用都被重建 [tool_output_budget_middleware.py:544-545](../backend/packages/harness/deerflow/agents/middlewares/tool_output_budget_middleware.py#L544-L545)。异步路径里 `_patch_result` 整体被 `asyncio.to_thread` 卸载到 worker 线程,因为沙箱 mkdir/write/test 是同步阻塞 I/O;`_resolve_sandbox` 只碰内存状态,可以安全留在 event loop 上 [tool_output_budget_middleware.py:596-613](../backend/packages/harness/deerflow/agents/middlewares/tool_output_budget_middleware.py#L596-L613)。

为什么历史路径还需要存在?三个来源:旧 checkpoint 里未预算化的历史、`tool_overrides` 调低阈值后历史消息相对新阈值超标、以及豁免名单变更。wrap_model_call 是"兜底一致性",保证送进模型的任何历史都满足当前预算策略,而不是假设写入侧永远无遗漏 [tool_output_budget_middleware.py:531-543](../backend/packages/harness/deerflow/agents/middlewares/tool_output_budget_middleware.py#L531-L543)。

## 问题链 7:Token 计数与用量观测

**Q7.1(基础)** DeerFlow 怎么数 token?为什么不统一用 tiktoken?

**参考回答**:两条路并存。摘要 middleware 用父类 `SummarizationMiddleware` 注入的 `token_counter`,对消息列表整体计数 [summarization_middleware.py:199](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L199);memory 注入侧自己实现 `_count_tokens`,默认 tiktoken `cl100k_base`,失败或配置 `memory.token_counting: char` 时降级为 CJK 感知的字符估计 [prompt.py:263-289](../backend/packages/harness/deerflow/agents/memory/prompt.py#L263-L289)、[memory_config.py:65-73](../backend/packages/harness/deerflow/config/memory_config.py#L65-L73)。不统一用 tiktoken 的原因:tiktoken 首次加载要联网下载 BPE 数据,这是网络依赖 + 阻塞调用,无外网部署里会直接挂;字符估计零网络依赖,作为兜底永远可用。

配置项的注释写得很直白:`'tiktoken' is accurate but the encoding's BPE data may be ...`——精度与可用性的权衡被显式交给了部署方 [memory_config.py:65-73](../backend/packages/harness/deerflow/config/memory_config.py#L65-L73)。这就是"可运维性设计":不假设部署环境有外网,也不为了精度牺牲可用性。

skill 救援的 token 统计也复用同一个 `token_counter`:逐个 ToolMessage 单独计数再求和,而不是对 bundle 整体计数——因为预算判断需要的是"这个 skill 的消息占多少 token",消息级粒度足够且实现简单 [summarization_middleware.py:357-360](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L357-L360)。

**链路解析**:

```
_count_tokens(text)
   │ memory.token_counting == "char"? ──是──> _char_based_token_estimate(零网络)
   │否
   ▼
_get_tiktoken_encoding("cl100k_base")
   │ 模块级缓存 + 锁; 首次可能下载 BPE
   ├─ 成功 → len(encoding.encode(text))
   ├─ 失败 → (None, 失败时刻) 写缓存,冷却后重试 → 本次降级 char 估计
   └─ LOADING 占位 → 并发线程不重复下载
```

**Q7.2(深挖)** 字符估计为什么不是简单的 `len(text) // 4`?CJK 文本差多少?

**参考回答**:英文/代码约 4 字符 1 token,`len // 4` 够用;但中日韩文本接近 1.5~2 字符 1 token,按 4 估会低估近一半,导致 memory 注入预算被 CJK 内容灌爆。`_char_based_token_estimate` 把 CJK 统一表意文字(U+4E00–U+9FFF)、平假名/片假名(U+3040–U+30FF)、韩文音节(U+AC00–U+D7A3)单独计数按 `cjk // 2` 算,其余按 `// 4` 算 [prompt.py:243-260](../backend/packages/harness/deerflow/agents/memory/prompt.py#L243-L260)。tool output 预览引用里的 `~{total // 4} tokens` 只是给模型的量级提示,不参与任何预算判断,所以用粗估即可 [tool_output_budget_middleware.py:226](../backend/packages/harness/deerflow/agents/middlewares/tool_output_budget_middleware.py#L226)。面试时要点是:估算精度要和用途匹配——预算判断要么用真计数,要么用保守方向的估计;给模型的提示语用粗估就够。

还有一个方向性问题值得主动说:CJK 估计是"高估"方向(实际 CJK 更接近 1.5-2 字符/token,按 2 算是保守上限),对"注入预算"这种宁可少塞不可超塞的场景,高估是安全方向;反过来,如果拿它做计费就不可接受——计费必须真计数。DeerFlow 的用量统计(见 Q7.4)走的是 provider 返回的 `usage_metadata` 真实数据,不依赖任何估计,两条链路分工明确。

**Q7.3(边界/异常)** tiktoken 冷启动下载最坏会阻塞多久?线上怎么防?

**参考回答**:最坏阻塞到 OS TCP 超时,约 26 分钟——注释里明确写了这个数字 [dynamic_context_middleware.py:47-51](../backend/packages/harness/deerflow/agents/middlewares/dynamic_context_middleware.py#L47-L51)。防御是四层:(1)网关启动时 `warm_tiktoken_cache` 预热,让首个请求不碰冷下载;(2)加载失败记入缓存并带冷却期,避免每请求重试,冷却后又能自愈 [prompt.py:186-239](../backend/packages/harness/deerflow/agents/memory/prompt.py#L186-L239);(3)`_count_tokens` 任何异常都降级到 CJK 字符估计,计数服务永不失败 [prompt.py:285-289](../backend/packages/harness/deerflow/agents/memory/prompt.py#L285-L289);(4)DynamicContextMiddleware 的异步注入整体包 `asyncio.wait_for`,上限 5.0 秒(`_INJECT_TIMEOUT_SECONDS`),超时后本轮跳过 memory/date 注入、请求继续,而不是挂死;同时同步文件 I/O 和潜在网络下载被 `asyncio.to_thread` 卸载,避免饿死同 event loop 上的其他 HTTP 处理器(issue #3402)[dynamic_context_middleware.py:209-232](../backend/packages/harness/deerflow/agents/middlewares/dynamic_context_middleware.py#L209-L232)。

编码缓存的并发设计也值得展开:模块级 dict + 一把锁,用三态占位(MISSING / LOADING / 就绪)保证并发线程里只有一个真正触发 `tiktoken.get_encoding`,其余要么拿到缓存要么降级,不会重复下载 [prompt.py:192-239](../backend/packages/harness/deerflow/agents/memory/prompt.py#L192-L239)。同步路径(`before_agent`)没有 5 秒超时保护,所以 warm-up 必须在启动时做掉——这是四层防御里启动预热不可省略的原因 [dynamic_context_middleware.py:205-207](../backend/packages/harness/deerflow/agents/middlewares/dynamic_context_middleware.py#L205-L207)。

**Q7.4(深挖)** token 用量怎么统计和归因?subagent 消耗的 token 算在谁头上?

**参考回答**:`TokenUsageMiddleware` 在 `after_model` 里做两件事:(1)读最后一条 AIMessage 的 `usage_metadata` 打日志(含 input/output/total 及 cache 细节);(2)构建 step attribution 写进 `additional_kwargs["token_usage_attribution"]` 供前端渲染"这一步在干嘛" [token_usage_middleware.py:316-350](../backend/packages/harness/deerflow/agents/middlewares/token_usage_middleware.py#L316-L350)。subagent 的 token 回算到派发它的 AIMessage:task 工具完成时 usage 按 tool_call_id 缓存,middleware 从新 AIMessage 前扫连续 ToolMessage 区间,逐个弹出缓存,再向前找"包含该 tool_call 的那条 AIMessage"——多个并发 task 调用来自同一条响应时,累加合并进同一个 `state_updates[dispatch_idx]`,三项字段(`input_tokens` / `output_tokens` / `total_tokens`)分别 `prev.get(..., 0) + ...` 累加 [token_usage_middleware.py:281-314](../backend/packages/harness/deerflow/agents/middlewares/token_usage_middleware.py#L281-L314)。幂等靠先比较再写入:attribution 没变化就不产生状态更新 [token_usage_middleware.py:344-349](../backend/packages/harness/deerflow/agents/middlewares/token_usage_middleware.py#L344-L349);schema 带 `version: 1` 并要求加法式演进,旧前端可安全忽略未知字段 [token_usage_middleware.py:256-264](../backend/packages/harness/deerflow/agents/middlewares/token_usage_middleware.py#L256-L264)。`write_todos` 被特殊处理成细粒度 action(`todo_start` / `todo_complete` / `todo_update` / `todo_remove`),是精确归因的唯一事实来源,缺失时前端才降级为通用 "Update to-do list" 标签 [token_usage_middleware.py:72-132](../backend/packages/harness/deerflow/agents/middlewares/token_usage_middleware.py#L72-L132)。

归因结果里的 `kind` 字段是把 actions 归纳成 step 类型:纯 todo 变更归 `todo_update`,单个 subagent 派发归 `subagent_dispatch`,多 action 归 `tool_batch`,无 tool_call 但有内容归 `final_answer`,否则是 `thinking` [token_usage_middleware.py:206-217](../backend/packages/harness/deerflow/agents/middlewares/token_usage_middleware.py#L206-L217)。配置侧极简:`TokenUsageConfig` 只有一个 `enabled` 开关(默认 True),没有阈值参数 [token_usage_config.py:4-7](../backend/packages/harness/deerflow/config/token_usage_config.py#L4-L7)。

## 问题链 8:memory_flush_hook 与长期记忆联动

**Q8.1(基础)** 摘要发生时,被压掉的消息里的用户偏好、事实信息不就丢了吗?

**参考回答**:这正是 `before_summarization` 钩子存在的意义。摘要 middleware 在调摘要 LLM 之前先 `_fire_hooks`,把 `SummarizationEvent`(含待摘要消息、保留消息、thread_id、agent_name、runtime)分发给所有钩子 [summarization_middleware.py:421-443](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L421-L443)。工厂在 memory 开启时注册 `memory_flush_hook` [agent.py:125-127](../backend/packages/harness/deerflow/agents/lead_agent/agent.py#L125-L127):它把即将被摘要的消息过滤出"用户输入 + 最终助手回复",连同 correction/reinforcement 检测结果 `add_nowait` 进记忆队列,由后台 LLM 异步提炼成长期记忆 [summarization_hook.py:12-34](../backend/packages/harness/deerflow/agents/memory/summarization_hook.py#L12-L34)。闭环的最后一环由 DynamicContextMiddleware 完成:下一轮请求时它把 `<memory>` 段注入首条消息(受 `memory.injection_enabled` 控制 [dynamic_context_middleware.py:111-126](../backend/packages/harness/deerflow/agents/middlewares/dynamic_context_middleware.py#L111-L126))。摘要负责短期上下文"减肥",flush 钩子把减掉的营养存进长期记忆,DynamicContext 下次把记忆摆回桌上——三者拼起来才是完整闭环。

事件对象本身的设计也值得提:`SummarizationEvent` 是 frozen dataclass,消息列表被转成 tuple 再分发,钩子无法反向修改 middleware 内部的列表 [summarization_middleware.py:24-32](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L24-L32)、[summarization_middleware.py:430-436](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L430-L436)。`BeforeSummarizationHook` 是 `runtime_checkable` 的 Protocol,任何可调用对象都能注册,不需要继承基类 [summarization_middleware.py:35-39](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L35-L39)。

**链路解析**:

```
_maybe_summarize
   ├─ _partition_with_skill_rescue
   ├─ _preserve_dynamic_context_reminders
   ├─ _fire_hooks ───────────┐         ┌─ 普通轮次: after_agent
   ▼                         ▼         ▼
_create_summary      memory_flush_hook   MemoryMiddleware.after_agent
                        │ add_nowait      │ add (debounce)
                        └──────┬──────────┘
                               ▼
                        memory queue ──> 后台 LLM 提炼 ──> memory 存储
                               │ (下一轮被 DynamicContextMiddleware 以 <memory> 注入)
```

**Q8.2(深挖)** 这和 MemoryMiddleware 每轮结束后的入队有什么区别?会不会重复入库?

**参考回答**:两个入口抓的语料不同、时序不同。`MemoryMiddleware.after_agent` 在 agent 完整跑完一轮后,把当前 state 消息过滤一遍走防抖队列 `queue.add` [memory_middleware.py:52-108](../backend/packages/harness/deerflow/agents/middlewares/memory_middleware.py#L52-L108)——它抓的是"正常轮次"的完整对话;但一旦发生摘要,被压掉的那段历史会从 state 里物理消失,`after_agent` 再过滤也看不到了。`memory_flush_hook` 在摘要动手前用 `add_nowait` 抢救这一段,两者互补。重复风险靠队列侧防抖/合并以及"至少 1 条 user + 1 条 assistant 才入队"的最低门槛控制 [summarization_hook.py:17-21](../backend/packages/harness/deerflow/agents/memory/summarization_hook.py#L17-L21)。user_id 获取方式也不同:普通路径在请求线程里用 `get_effective_user_id()`——因为 `threading.Timer` 触发的线程不传播 ContextVar,必须在入队时捕获 [memory_middleware.py:96-99](../backend/packages/harness/deerflow/agents/middlewares/memory_middleware.py#L96-L99);flush 路径则用 `resolve_runtime_user_id(event.runtime)` 从 runtime 解析 [summarization_hook.py:25](../backend/packages/harness/deerflow/agents/memory/summarization_hook.py#L25)。

可以把两条路径并排记:

| 维度 | after_agent(普通) | before_summarization(flush) |
| --- | --- | --- |
| 时机 | 一轮完整结束后 | 摘要覆盖历史前 |
| 语料 | 当前 state 全量 | 即将被删除的那一段 |
| 入队方式 | `add`(防抖批量) | `add_nowait`(立即) |
| user_id | 请求线程 ContextVar | runtime 解析 |

flush 用 `add_nowait` 的原因:摘要一完成,这段消息就再也拿不到了,没有"等下一轮一起批"的机会,必须立即入队 [summarization_hook.py:26-34](../backend/packages/harness/deerflow/agents/memory/summarization_hook.py#L26-L34)。

**Q8.3(边界/异常)** 钩子抛异常会阻断摘要吗?thread_id 拿不到怎么办?

**参考回答**:不会阻断。`_fire_hooks` 逐个 try/except,单钩子失败只记 `logger.exception`,摘要流程继续 [summarization_middleware.py:438-443](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L438-L443)——取舍很清晰:丢一次记忆 flush 可接受(下一轮 after_agent 还能补),因记忆系统故障让用户请求失败不可接受。thread_id 解析走双通道:先 `runtime.context`,拿不到再 fallback 到 LangGraph config 的 `configurable.thread_id`;`get_config()` 抛 `RuntimeError`(不在图执行上下文)时返回 None,此时 `memory_flush_hook` 开头就跳过 [summarization_middleware.py:42-51](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L42-L51)、[summarization_hook.py:14-15](../backend/packages/harness/deerflow/agents/memory/summarization_hook.py#L14-L15)。memory 未启用时工厂干脆不注册钩子,`_fire_hooks` 遇空列表直接返回,零开销 [summarization_middleware.py:427-428](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L427-L428)。

同样的双通道解析也用于 `agent_name`(`runtime.context` → `configurable.agent_name`),两个字段的获取逻辑是同构的,便于后续加钩子参数时复制模式 [summarization_middleware.py:54-63](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L54-L63)。失败日志里带钩子名(`__name__` 或类名),排查时能对上号 [summarization_middleware.py:441-443](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L441-L443)。

把整条链 8 的异常处理哲学归纳一句:记忆是"锦上添花"系统,它的任何故障都不允许传导到主对话路径——flush 钩子失败降级、thread_id 缺失降级、memory 关闭时根本不挂钩子,三种降级方式对应三种故障域,各自独立不叠加。

## 面试官最爱追问的 3 个点

这三点几乎覆盖了本模块所有"区分度高"的追问方向:一个考流式系统的副作用隔离,一个考文件系统与工具调用的闭环设计,一个考外部依赖的降级链。每个点都要做到"机制 + 至少一个具体数字 + 一个失败场景"三件套。

1. **"摘要的 LLM 调用会不会被前端看到 / 干扰主流程?"**——应答策略:一口气答出三层:TAG_NOSTREAM 模型副本防流式泄漏、`__init__` 预建副本而非运行时 swap 防并发污染、异常吞成 `"Error generating summary: ..."` 字符串防阻断;并指出 `self.model` 保持无 tag 是为了父类 profile 检查 [summarization_middleware.py:120-161](../backend/packages/harness/deerflow/agents/middlewares/summarization_middleware.py#L120-L161)。
2. **"tool output 外部化后模型怎么读回来?会不会死循环?"**——应答策略:预览里带 `/mnt/user-data/outputs/.tool-results/...` 虚拟路径和分段读取提示,模型用 `read_file` 的 start_line/end_line 取回;`read_file` 在 `exempt_tools` 默认豁免,从机制上切断 persist→read→persist 循环 [tool_output_config.py:55-58](../backend/packages/harness/deerflow/config/tool_output_config.py#L55-L58)。
3. **"tiktoken 没网怎么办?会挂住请求吗?"**——应答策略:启动 warm 缓存 → 失败冷却自愈 → 运行时降级 CJK 感知字符估计(英文 //4、CJK //2)→ 最外层 `asyncio.wait_for` 5 秒超时跳过注入,四道防线层层兜底,单点故障永远降级而非阻断 [dynamic_context_middleware.py:47-51](../backend/packages/harness/deerflow/agents/middlewares/dynamic_context_middleware.py#L47-L51)、[prompt.py:243-289](../backend/packages/harness/deerflow/agents/memory/prompt.py#L243-L289)。

最后给一个复习顺序建议:先按问题链 1→2 把摘要主流程吃透,再用链 4、5 理解两类"救援"分别是救什么(skill 内容 vs reminder 标记),链 6 独立成章可单独准备,链 7、8 是跨模块联动,适合放在最后用来串全局。
