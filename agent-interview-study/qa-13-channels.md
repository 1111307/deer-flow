# IM Channels 多平台接入 —— 面试问题链

> 本文档按"大厂面试追问链"组织:每条问题链从基础问题开始,层层追问到实现细节、设计权衡、异常边界。配套深读笔记:无(本模块尚无深读笔记,本文档是第一份)。
> 代码引用格式:[文件名.py:起始行-结束行](../backend/仓库相对路径#L起始行-L结束行)。只引用你实际读过的行,禁止编造行号。

## 问题链 1:整体架构与 MessageBus pub/sub

**Q1.1(基础)** 这套 IM 多平台接入的整体架构是什么样的?Channel、MessageBus、ChannelManager 各自承担什么角色?

**参考回答**:三层结构,职责严格单向。Channel 是平台适配层,ABC 只定义 `start/stop/send` 三个抽象方法,外加 `send_file/receive_file` 两个可选钩子([base.py:47-71](../backend/app/channels/base.py#L47-L71) 覆盖 start/stop/send/send_file;`receive_file` 默认实现单独在 [base.py:181-185](../backend/app/channels/base.py#L181-L185))。MessageBus 是进程内 pub/sub 中枢,入站用一个无界 `asyncio.Queue`,出站用 listener 回调列表([message_bus.py:142-144](../backend/app/channels/message_bus.py#L142-L144))。

ChannelManager 是唯一的消费者/调度器:从队列取消息、通过 langgraph_sdk 调 Gateway 创建 thread 和 run,再把响应 publish 回 bus([manager.py:775-781](../backend/app/channels/manager.py#L775-L781))。ChannelService 按 `config.yaml` 的 `channels` 段懒加载各平台类并启动,注册表是"名字 → import path"的字符串映射([service.py:23-31](../backend/app/channels/service.py#L23-L31))。

关键设计点:Channel 之间互不感知,新增一个平台只需实现 Channel 接口、注册表加一行,Manager 和 bus 零改动——这是典型的"总线 + 插件"解耦。

**链路解析**:

```
 ┌─ 平台适配层(每个 Channel 一个实例,各自跑独立线程/事件循环)─────────┐
 │ FeishuChannel    lark WS 线程 ──┐                                 │
 │ TelegramChannel  polling 线程 ──┤ InboundMessage                  │
 │ DingTalkChannel  stream 线程 ───┤ (channel/chat/user/text/topic…) │
 │ SlackChannel     socket 线程 ───┘        │                        │
 └──────────────────────────────────────────┼────────────────────────┘
                                            ▼ bus.publish_inbound()
                                asyncio.Queue(无界 FIFO)
                                            ▼ bus.get_inbound()
 ┌─ 调度层 ChannelManager ────────────────────────────────────────────┐
 │ _dispatch_loop: wait_for(get_inbound, 1.0s) → dedupe → create_task │
 │ _handle_message: bound-identity 准入 → Semaphore(5)                │
 │   ├─ COMMAND → _handle_command(/new /status /models /memory …)     │
 │   └─ CHAT    → _handle_chat → runs.wait / runs.stream (Gateway)    │
 └──────────────────────────────────────────┬─────────────────────────┘
                                            ▼ bus.publish_outbound()
 ┌─ 出站扇出:listener 列表逐个 await ──────────────────────────────────┐
 │ ch._on_outbound: msg.channel_name 匹配? → send() → send_file()     │
 └────────────────────────────────────────────────────────────────────┘
```

**Q1.2(深挖)** 出站为什么用"回调列表逐个 await"而不是也放一个队列?一个 listener 抛异常会影响其他 channel 吗?

**参考回答**:出站语义是"定向投递"。每个 Channel 在 `_on_outbound` 里先按 `msg.channel_name == self.name` 过滤,不属于自己的消息直接 return([base.py:158-166](../backend/app/channels/base.py#L158-L166)),所以"广播 + 过滤"比按 channel 建 N 个队列简单;订阅关系管理只需 `subscribe_outbound/unsubscribe_outbound` 两个方法([message_bus.py:169-175](../backend/app/channels/message_bus.py#L169-L175))。

容错上,`publish_outbound` 对每个 listener 单独 try/except,一个 channel 挂掉不会阻断其他 listener([message_bus.py:186-190](../backend/app/channels/message_bus.py#L186-L190))。代价是串行 await:某个平台发送慢会拖住同一条出站消息的后续 listener——但实践中一个 run 的出站只对应一个目标 channel,其余 listener 都是 O(1) 过滤,影响可忽略。

**不这样设计会怎样**:若出站也走"每 channel 一个队列 + 各自消费协程",channel 动态启停(`restart_channel/remove_channel`)时就要同步管理队列和消费者 task 的生命周期;而回调模型下 channel 停止只需 `unsubscribe_outbound` 一行([feishu.py:217-219](../backend/app/channels/feishu.py#L217-L219)),故障面小得多。

**Q1.3(边界/异常)** 入站队列是无界的,突发流量打进来会怎样?dispatch 循环的 `timeout=1.0` 是干什么的?

**参考回答**:`asyncio.Queue()` 默认无界([message_bus.py:143](../backend/app/channels/message_bus.py#L143)),背压靠下游 `ChannelManager` 的 `asyncio.Semaphore(max_concurrency=5)` 限制并发处理数([manager.py:788](../backend/app/channels/manager.py#L788),[manager.py:936](../backend/app/channels/manager.py#L936))。但注意 `_dispatch_loop` 是无条件 `create_task(self._handle_message(msg))`([manager.py:980](../backend/app/channels/manager.py#L980)),信号量在 `_handle_message` 内部才获取——极端情况下 task 对象会堆积,硬背压并不存在。

这是刻意选择:平台侧已经给了用户即时反馈(Feishu 的 OK reaction 和"Working on it"卡片在 publish 之前就发出,[feishu.py:730-736](../backend/app/channels/feishu.py#L730-L736)),排队比丢消息体验好。

`asyncio.wait_for(self.bus.get_inbound(), timeout=1.0)`([manager.py:958](../backend/app/channels/manager.py#L958))的作用是让循环每秒检查一次 `self._running`,使 `stop()` 能优雅退出,而不是永久阻塞在 `queue.get()` 上([manager.py:954-962](../backend/app/channels/manager.py#L954-L962))。

---

## 问题链 2:channel:chat[:topic] → thread_id 映射与持久化

**Q2.1(基础)** IM 会话是怎么映射到 DeerFlow thread 的?key 的结构是什么?

**参考回答**:两层 key。Legacy 模式用 `ChannelStore`,key 是 `"<channel>:<chat_id>"`,带话题时是 `"<channel>:<chat_id>:<topic_id>"`([store.py:74-78](../backend/app/channels/store.py#L74-L78));绑定过 connection 的消息走数据库 `ChannelConversationRow`,按 `(connection_id, external_conversation_id, external_topic_id)` 三元组查找([sql.py:537-549](../backend/packages/harness/deerflow/persistence/channel_connections/sql.py#L537-L549))。

选择逻辑在 `_lookup_thread_id`:有 `connection_id` 且 repo 可用就查库,否则查 JSON store([manager.py:1172-1179](../backend/app/channels/manager.py#L1172-L1179))。`topic_id` 为 `None` 时(如 Telegram 私聊)整个 chat 共享一个 thread,实现"单会话持续对话"。

各平台的 topic 语义自己定:Telegram 群聊用被回复消息的 id、没回复就用自己 msg_id 开新话题([telegram.py:536-543](../backend/app/channels/telegram.py#L536-L543));Feishu 用 root_id/parent_id/thread_id 候选逐个查 store([feishu.py:693-716](../backend/app/channels/feishu.py#L693-L716));DingTalk 单聊 None、群聊 msg_id([dingtalk.py:414-417](../backend/app/channels/dingtalk.py#L414-L417))。

**链路解析**:

```
 InboundMessage(channel="feishu", chat_id, topic_id?, connection_id?)
      │
      ▼ _lookup_thread_id()                          [manager.py:1172]
 connection_id 存在且 repo 可用?
      │ yes                            │ no(legacy)
      ▼                                ▼
 DB: ChannelConversationRow            JSON: ChannelStore._key()
 WHERE connection_id=?                 "feishu:<chat_id>" 或
   AND external_conversation_id=?      "feishu:<chat_id>:<topic_id>"
   AND external_topic_id=?                   [store.py:74-78]
      └──────────────┬────────────────┘
                     ▼ 命中?
              是 → 复用 thread_id(并回填 channel_source 元数据,每进程一次)
              否 → _create_thread()
                     ├─ Gateway threads.create(metadata=channel_source)
                     └─ _store_thread_id() 沿同一路径写回(DB 或 JSON)
 JSON 落盘: NamedTemporaryFile → json.dump → replace(原子 rename)
```

**Q2.2(深挖)** `ChannelStore` 是单 JSON 文件且每次全量重写,崩溃安全、并发、性能这三件事分别怎么交代?

**参考回答**:崩溃安全靠"临时文件 + 原子替换":`tempfile.NamedTemporaryFile(dir=同目录, delete=False)` 写完再 `Path(fd.name).replace(self._path)`,异常时删临时文件([store.py:56-70](../backend/app/channels/store.py#L56-L70))——同目录保证 replace 落在同一文件系统,POSIX 下 rename 原子,崩溃后要么旧文件要么新文件,不存在半写状态。启动时 JSON 损坏则 warn 并空库重建([store.py:48-54](../backend/app/channels/store.py#L48-L54))。

并发用 `threading.Lock` 包住整个 read-modify-write([store.py:44](../backend/app/channels/store.py#L44),[store.py:97-107](../backend/app/channels/store.py#L97-L107))——必须是线程锁而不是 asyncio.Lock,因为 Feishu/Telegram 的回调跑在各自的工作线程里。性能上,条目数等于"活跃会话×话题数",典型几十到几百条,全量 dump 是微秒级;类注释明确说这是刻意简单,高并发可换数据库后端([store.py:31-33](../backend/app/channels/store.py#L31-L33))。

**不这样设计会怎样**:改 append-only 日志,启动要 replay、要压缩,多一类"日志损坏"故障;上 SQLite,则在无数据库的纯本地部署里引入依赖。同一套模式被复用到存凭据的 runtime-config.json,还额外加了 `chmod 0o600` 防同机其他用户读凭据([runtime_config_store.py:54-64](../backend/app/channels/runtime_config_store.py#L54-L64))——这个文件就是为 "local/private deployments" 设计的([runtime_config_store.py:19-23](../backend/app/channels/runtime_config_store.py#L19-L23))。

**Q2.3(边界/异常)** Feishu 里用户直接回复 bot 的澄清卡片,这条消息的 root_id 指向卡片消息而不是原对话,thread 映射会不会断?

**参考回答**:不会,有三层兜底。第一,`_resolve_topic_id` 按 `root_id → parent_id → thread_id` 顺序逐个查 store,任何一个平台消息 id 曾被记录过映射就复用([feishu.py:702-716](../backend/app/channels/feishu.py#L702-L716))。

第二,bot 发出站消息时 `_remember_thread_mapping` 会把卡片 message_id、root_id、parent_id、metadata 里的多个候选 topic 全部写入 store([feishu.py:595-627](../backend/app/channels/feishu.py#L595-L627))——所以"回复卡片"时那个新 root_id 早就被登记过了。

第三,纯文本追问(不点回复)走内存 pending-clarification 表,TTL 30 分钟(`PENDING_CLARIFICATION_TTL_SECONDS = 30 * 60`,[feishu.py:30](../backend/app/channels/feishu.py#L30)),命中后恢复 topic 并补写 store([feishu.py:660-691](../backend/app/channels/feishu.py#L660-L691))。注释明确说内存表只是 "short-lived hint",显式回复仍靠持久化映射兜底([feishu.py:648-651](../backend/app/channels/feishu.py#L648-L651))——进程重启丢的是 30 分钟内未显式回复的追问连续性,不丢主映射。

---

## 问题链 3:流式输出的平台差异化策略

**Q3.1(基础)** 各平台"边生成边显示"是怎么做的?为什么不是统一方案?

**参考回答**:Manager 侧是统一的:`_channel_supports_streaming` 为真就走 `runs.stream`,以 0.35 秒最小间隔 publish 非 final 出站(`STREAM_UPDATE_MIN_INTERVAL_SECONDS = 0.35`,[manager.py:57](../backend/app/channels/manager.py#L57),节流判断在 [manager.py:1410-1412](../backend/app/channels/manager.py#L1410-L1412))。能力查询优先问运行中的 channel 实例,服务没起来才查静态表([manager.py:819-828](../backend/app/channels/manager.py#L819-L828))。

平台侧分化:Feishu/Telegram/WeCom 声明 `supports_streaming=True`,Slack/Discord/WeChat 为 False 走 `runs.wait` 一次性返回([manager.py:73-81](../backend/app/channels/manager.py#L73-L81));DingTalk 是条件式——配置了 `card_template_id` 才支持流式([dingtalk.py:141-143](../backend/app/channels/dingtalk.py#L141-L143))。

不统一的原因是平台 API 能力不同:飞书有"原地 patch 卡片",Telegram 有 `editMessageText` 但有严格限频,Slack/Discord 在此实现里没做原地编辑。统一抽象只能统一到"非 final / final 两类出站消息"这一层——这正是 `OutboundMessage.is_final` 字段的作用([message_bus.py:119](../backend/app/channels/message_bus.py#L119))。

**链路解析**:

```
 LangGraph runs.stream ──► chunk(event, data)
      │
      ├─ event ∈ {"messages-tuple","messages"}
      │     └─ _accumulate_stream_text: 按 message_id 分桶, delta/累计 归并
      ├─ event == "values"
      │     └─ 快照兜底: _extract_response_text(最后一条 AI 消息)
      ▼
 latest_text 有变化 且 距上次 publish ≥ 0.35s? ──否──► skip
      │ 是
      ▼ bus.publish_outbound(is_final=False)
 ┌──────────┬─────────────────────┬─────────────────────┐
 ▼          ▼                     ▼                     ▼
Feishu    Telegram              DingTalk              Slack/Discord/WeChat
patch 卡片 editMessageText      AI Card               supports_streaming=False
update_multi 私聊≥1s/群聊≥3s     async_streaming       → Manager 走 runs.wait
失败→final  429→丢帧等 final     append=False 全量覆盖  → 等 is_final 一次发全量
降级新回复  >4096 截断加"…"      卡片缺失→丢中间帧保 final
      └──────────┴─────────────────────┴─────────────────────┘
                           ▼
              is_final=True 的出站消息(所有平台必有,
              保证完整文本至少送达一次)
```

**Q3.2(深挖)** 飞书卡片"原地更新"具体怎么实现的?`update_multi` 是什么?patch 失败了怎么办?

**参考回答**:收到用户消息时先异步建一张"Working on it..."运行卡片,并缓存 `source_msg_id → card_msg_id` 映射([feishu.py:505-534](../backend/app/channels/feishu.py#L505-L534));后续每个出站 chunk 调 `im.v1.message.patch` 改写这张卡的内容([feishu.py:477-484](../backend/app/channels/feishu.py#L477-L484))。

卡片 JSON 里 `config.update_multi=True` 是关键:它让卡片内容更新对会话里**所有成员**可见,而不是只对触发者生效([feishu.py:438-442](../backend/app/channels/feishu.py#L438-L442))。没有它,群聊里其他人看到的是旧内容。

失败处理分档:非 final chunk patch 失败直接抛给 `_send_with_retry` 走指数退避;final 消息 patch 失败则降级为重新 reply 一张新卡,保证完整文本一定送达([feishu.py:556-568](../backend/app/channels/feishu.py#L556-L568))。final 之后清理 `_running_card_ids` 并给原消息加 DONE 表情([feishu.py:586-588](../backend/app/channels/feishu.py#L586-L588))。

**不这样设计会怎样**:若每条流式 chunk 都发新消息,一个长回答会刷几十条气泡,群聊完全不可用——原地 patch 把"流式"从 N 条消息压成 1 条;若没有 final 降级,一次 patch 抖动就会丢掉整个回答。

**Q3.3(深挖)** Telegram 为什么不能用同样激进的编辑策略?它的节流数字是怎么定的?

**参考回答**:Telegram Bot API 对群聊有约 20 条/分钟的硬限制,代码注释直接写明了这个来源([telegram.py:19-21](../backend/app/channels/telegram.py#L19-L21))。所以做两级节流:私聊最小编辑间隔 1.0 秒,群聊(chat_id 为负数)3.0 秒,判断在 `_send_stream_update`([telegram.py:169-172](../backend/app/channels/telegram.py#L169-L172))。

被 429 限流时静默丢弃该次更新——因为 manager 之后一定发 final 消息兜底,完整文本不会丢,丢的只是中间帧([telegram.py:182-184](../backend/app/channels/telegram.py#L182-L184))。内容没变就不发请求,"message is not modified" 错误也被识别为正常([telegram.py:173-174](../backend/app/channels/telegram.py#L173-L174),[telegram.py:330-331](../backend/app/channels/telegram.py#L330-L331))。

文本超过 4096 字符(`TELEGRAM_MAX_MESSAGE_LENGTH = 4096`,[telegram.py:17](../backend/app/channels/telegram.py#L17)):流式显示截断为前 4095 字符加省略号([telegram.py:149-151](../backend/app/channels/telegram.py#L149-L151)),final 时按 4096 切块分多条发送([telegram.py:196-213](../backend/app/channels/telegram.py#L196-L213))。跟踪表 `_stream_messages` 上限 256 条,防 final 永远不到达导致泄漏(`MAX_TRACKED_STREAM_MESSAGES`,[telegram.py:24](../backend/app/channels/telegram.py#L24))。

**Q3.4(边界/异常)** DingTalk 的 AI Card 流式如果卡片创建失败了会怎样?为什么 `supports_streaming` 要依赖配置?

**参考回答**:DingTalk 流式依赖 `AICardReplier` 先创建 AI 卡片拿到 `out_track_id`,后续 chunk 调 `async_streaming(append=False)` 全量覆盖卡片内容([dingtalk.py:746-765](../backend/app/channels/dingtalk.py#L746-L765))。卡片 key 由 `conversation_type:sender:conversation:message` 四元组构成,入站和出站用同一规则生成才能对上([dingtalk.py:705-712](../backend/app/channels/dingtalk.py#L705-L712))。

卡片创建失败(没有 track id)时,`send()` 对非 final 消息直接 return 丢弃,避免 final 再发一遍造成重复([dingtalk.py:225-228](../backend/app/channels/dingtalk.py#L225-L228));流式更新中途失败且是 final,降级走 sampleMarkdown 一次性发送([dingtalk.py:237-243](../backend/app/channels/dingtalk.py#L237-L243))。

`supports_streaming` 返回 `bool(self._card_template_id)`([dingtalk.py:141-143](../backend/app/channels/dingtalk.py#L141-L143)),因为钉钉普通机器人消息没有"编辑"能力,没有 AI 卡片模板时流式 chunk 无处安放,只能退回 `runs.wait`。另外钉钉 markdown 渲染能力弱,代码块要转引用块、管道表格要逐行转键值对([dingtalk.py:80-117](../backend/app/channels/dingtalk.py#L80-L117))。

---

## 问题链 4:connect code 绑定流

**Q4.1(基础)** 浏览器端发起的"绑定 IM 账号"流程是怎么闭环的?code 的生成和消费路径?

**参考回答**:已登录 DeerFlow 的用户在设置页点"连接",后端 `_new_binding_code()` 用 `secrets.token_urlsafe(16)` 生成一次性 code(128 bit 熵),以 SHA-256 哈希为主键存 `ChannelOAuthStateRow`,TTL 600 秒(`_STATE_TTL_SECONDS = 600`,[channel_connections.py:27](../backend/app/gateway/routers/channel_connections.py#L27),生成与落库在 [channel_connections.py:310-337](../backend/app/gateway/routers/channel_connections.py#L310-L337))。

用户把 `/connect <code>` 发给 bot(Feishu/DingTalk/Slack),或点 `t.me/<bot>?start=<code>` 深链(Telegram,[channel_connections.py:349-352](../backend/app/gateway/routers/channel_connections.py#L349-L352))。平台适配器在**任何鉴权之前**先截获 code:Feishu 在 `_on_message` 里 `_pending_connect_code` 命中后直接走绑定并 return([feishu.py:861-876](../backend/app/channels/feishu.py#L861-L876));Telegram 在 `_cmd_start` 里先试 `_bind_connection_from_start_token` 再查 `allowed_users`([telegram.py:463-474](../backend/app/channels/telegram.py#L463-L474))。

绑定动作 = `consume_oauth_state` 核销 code + `upsert_connection` 建立 `(owner_user_id, provider, external_account_id, workspace_id)` 记录([feishu.py:746-771](../backend/app/channels/feishu.py#L746-L771))。之后每条入站消息由 `attach_connection_identity` 按平台身份查库,挂上 `connection_id/owner_user_id`([connection_identity.py:30-42](../backend/app/channels/connection_identity.py#L30-L42))。

**链路解析**:

```
 Browser(已登录 owner A)                 IM 平台用户 X
     │ POST /connect                        │
     ▼                                      │
 secrets.token_urlsafe(16)                 │
   → sha256(state) 为主键入库               │
   → TTL 600s                              │
   → 每 (owner,provider) ≤ 5 个 pending     │
     │ 返回 code 或 t.me/<bot>?start=code    │
     └──────────────► 用户转发 ────────────►│ /connect <code> / /start <code>
                                            ▼
                         adapter 截获(在 allowed_users 门之前!)
                                            ▼
                  consume_oauth_state:
                    DELETE expired
                    UPDATE … SET consumed_at=now
                      WHERE state_hash=? AND consumed_at IS NULL
                    rowcount==1 ? 放行 : None(重放/并发拒绝)
                                            ▼
                  upsert_connection(owner=A, provider, ext_id=X, workspace)
                                            ▼
            后续每条消息 attach_connection_identity → 带 owner 进 Manager
```

**Q4.2(深挖)** code 为什么是"一次性"的?并发两个 worker 同时核销同一个 code 会发生什么?

**参考回答**:`consume_oauth_state` 用条件 UPDATE 保证只有一个赢家:`UPDATE ... SET consumed_at=now WHERE state_hash=? AND consumed_at IS NULL`,然后检查 `rowcount == 1`,不是 1 就返回 None([sql.py:458-471](../backend/packages/harness/deerflow/persistence/channel_connections/sql.py#L458-L471))。即使两个平台 worker(或用户双击、平台重投)同时拿到同一 code,数据库层也只放行一个,另一个得到 "invalid or expired"。

存储的是哈希不是明文([sql.py:283-284](../backend/packages/harness/deerflow/persistence/channel_connections/sql.py#L283-L284)),DB 泄漏不能直接冒用 code。核销前还会先删过期行、再查 `consumed_at` 和 `expires_at`,三重校验([sql.py:447-456](../backend/packages/harness/deerflow/persistence/channel_connections/sql.py#L447-L456))。

发放侧也有限流:每个 `(owner, provider)` 最多 5 个 pending code(`_MAX_PENDING_CONNECT_CODES_PER_PROVIDER = 5`,[channel_connections.py:28](../backend/app/gateway/routers/channel_connections.py#L28)),超限直接 429;delete-expired + count + insert 在同一事务里,PostgreSQL 用 `pg_advisory_xact_lock` 按 (owner,provider) 序列化,SQLite 靠先发的 DELETE 拿写锁天然序列化([sql.py:314-404](../backend/packages/harness/deerflow/persistence/channel_connections/sql.py#L314-L404))。

**Q4.3(深挖)** 为什么说 `allowed_users` 不是绑定的防线?那真正的防线在哪?

**参考回答**:`allowed_users` 是"平台侧静态白名单",而绑定流程恰恰要允许一个**平台从未见过的新用户**完成首次绑定。`_pending_connect_code` 的 docstring 明确要求适配器在 `allowed_users`/`_check_user` 之前检查 connect code([base.py:122-133](../backend/app/channels/base.py#L122-L133)),否则浏览器发起的绑定永远过不了白名单。一句话:白名单管"日常谁能用",绑定码管"谁能成为谁"。

真正的防线有四层:一是 code 本身(128 bit 随机、600s TTL、一次性、哈希存储);二是绑定后身份由服务端数据库记录,`attach_connection_identity` 只按 `(provider, external_account_id, workspace_id)` 查库,不信客户端字段([connection_identity.py:30-42](../backend/app/channels/connection_identity.py#L30-L42));三是 Manager 的 `require_bound_identity` 准入——非 command 消息在占用信号量之前就要过 `_get_bound_identity_rejection`,且**不信任** InboundMessage 自带的 `connection_id/owner_user_id`,而是按平台身份重新查库比对,不一致时只把服务端读到的值当出站路由提示([manager.py:1110-1147](../backend/app/channels/manager.py#L1110-L1147));四是数据库单 owner 唯一索引(见问题链 5)。

**不这样设计会怎样**:若 allowed_users 挡在绑定之前,新用户永远无法自助绑定,白名单得手工维护;若只信消息里的 identity 字段不复查库,伪造一条 InboundMessage 就能冒用别人的 DeerFlow 账号跑 agent、读别人的文件桶——Manager 是 run 创建的安全边界,这是它复查库的根本原因。

---

## 问题链 5:单 owner 转移语义

**Q5.1(基础)** 同一个飞书账号,先被用户 A 绑定、后来又被用户 B 绑定,系统怎么处理?

**参考回答**:后者赢,前者被撤销。`upsert_connection` 在 flush 新行之前先执行 `_revoke_other_active_owners`:把同 `(provider, external_account_id, workspace_id)` 但 owner 不同且未 revoked 的行全部置 `revoked`,并级联删掉对应 credential 行([sql.py:137-154](../backend/packages/harness/deerflow/persistence/channel_connections/sql.py#L137-L154))。

即身份转移是显式的"revoke 旧 + upsert 新",旧 owner 的连接立即失效、凭据立即销毁,且发生在同一事务里。之后 A 再发消息,`find_connection_by_external_identity` 只查 `status == "connected"` 的行([sql.py:487-500](../backend/packages/harness/deerflow/persistence/channel_connections/sql.py#L487-L500)),查到的是 B 的连接。

同一 owner 重复绑定同一身份则走普通唯一约束 `uq_channel_connection_owner_provider_identity` 的 update 分支([model.py:42-47](../backend/packages/harness/deerflow/persistence/channel_connections/model.py#L42-L47)),是幂等刷新,不是转移。

**链路解析**:

```
 用户 B 完成 /connect ──► upsert_connection(owner=B, feishu, open_id=X, ws)
        │
        ▼ 同一事务内
 SELECT id WHERE provider=? AND ext_id=? AND ws=?
           AND owner != B AND status != 'revoked'
        │
        ▼ 命中 A 的行
 UPDATE status='revoked' ; DELETE FROM channel_credentials WHERE …
        │
        ▼
 INSERT/UPDATE B 的 connected 行
        │
        ▼ commit
 部分唯一索引校验:(provider, ext_id, ws) 在 status != 'revoked' 行中唯一
        │
        ▼ IntegrityError? → rollback → 重读 → 再试(至多 3 次)
```

**Q5.2(深挖)** "并发两个 owner 同时绑定同一外部身份"这个竞态,应用层 revoke-then-insert 不够吧?数据库层怎么兜底?

**参考回答**:对,应用层先查后写存在 TOCTOU 窗口。所以表上建了部分唯一索引 `uq_channel_connection_active_identity`:`(provider, external_account_id, workspace_id)` 在 `status != 'revoked'` 的行上唯一,SQLite ≥3.8 和 PostgreSQL 都支持 partial index([model.py:49-62](../backend/packages/harness/deerflow/persistence/channel_connections/model.py#L49-L62))。

顺序很关键:revoke 必须先于新行 flush,否则 commit 时索引冲突——代码注释明确写了这个约束([sql.py:168-171](../backend/packages/harness/deerflow/persistence/channel_connections/sql.py#L168-L171))。

仍撞上 `IntegrityError` 就回滚重试,最多 `_UPSERT_MAX_ATTEMPTS = 3` 次;每次重试重读已提交的对方状态、撤掉对方再写自己的([sql.py:32](../backend/packages/harness/deerflow/persistence/channel_connections/sql.py#L32),[sql.py:185-192](../backend/packages/harness/deerflow/persistence/channel_connections/sql.py#L185-L192))。3 次是有界重试,注释说明在 realistic contention 下会收敛——因为每次重试都能看到对方已提交的行并撤销它。

**Q5.3(边界/异常)** 为什么用"部分唯一索引"而不是普通唯一约束?revoked 的历史行还要留着干嘛?

**参考回答**:因为 revoked 行要保留审计轨迹(谁绑定过、何时解绑),而普通唯一约束会阻止同一外部身份出现第二行——包括 revoked 历史行,那样重新绑定就必须物理删除历史。`sqlite_where=text("status != 'revoked'")` 让索引只覆盖活跃行,历史行不参与唯一性([model.py:60-61](../backend/packages/harness/deerflow/persistence/channel_connections/model.py#L60-L61))。

**不这样设计会怎样**:若不用部分索引而靠应用层检查,并发绑定会让两个 owner 都写成功;之后 `find_connection_by_external_identity` 按 `updated_at desc` 取最新([sql.py:496-498](../backend/packages/harness/deerflow/persistence/channel_connections/sql.py#L496-L498))——消息路由变成"谁后更新归谁",owner 之间可以互相"抢"会话。这不是小 bug 而是安全事故:被抢者的文件桶、memory、thread 全部暴露给另一个用户。

---

## 问题链 6:身份解析与 owner-scoped 文件存储

**Q6.1(基础)** 一条 IM 消息进来,agent run 用哪个 `user_id`?和平台 user_id 是什么关系?

**参考回答**:单一事实来源是 `_channel_storage_user_id`:优先用绑定的 DeerFlow owner(经 `make_safe_user_id` 消毒),没绑定时回退到消毒后的平台 user_id([manager.py:528-553](../backend/app/channels/manager.py#L528-L553))。

这个值同时驱动两个东西:run 身份(`run_context["user_id"]`)和文件桶(`receive_file`、入站附件落盘、出站 artifact 解析),保证"agent 读写的目录"和"channel 暂存文件的目录"是同一个桶([manager.py:861-871](../backend/app/channels/manager.py#L861-L871))。原始平台 user 保留在 `channel_user_id` 里供平台侧查询和审计。

注意 `_owner_headers` 走的是相反策略:刻意发**未消毒**的 owner id 给 Gateway 重新解析。docstring 明确区分两者——一个是 "in-process, sanitized, filesystem-facing identity",一个是线上传输给 gateway 再解析的身份([manager.py:544-547](../backend/app/channels/manager.py#L544-L547))。

**链路解析**:

```
 InboundMessage(user_id=平台ID, owner_user_id=绑定的DeerFlow用户?)
        │
        ▼ _channel_storage_user_id()                [manager.py:528]
   owner 存在? ──yes──► make_safe_user_id(owner) ──┐
        │ no                                       │
        ▼                                          ├──► run_context["user_id"]
   msg.user_id 存在? ──► make_safe_user_id(平台ID) ─┤      (agent run 身份)
        │                                          ├──► receive_file(user_id=…)
        ▼ 都没有                                    │      (入站文件落盘桶)
   None → 退回 contextvar/default                   ├──► _ingest_inbound_files
                                                    │      (uploads 目录)
                                                    └──► _resolve_attachments
                                                           (outputs 目录,出站 artifact)
   三者永远指向同一个 users/<safe_id>/ 桶 —— 这是该函数存在的全部理由
```

**Q6.2(深挖)** 为什么要有"回退到平台 user_id"这一支?直接拒绝未绑定用户不行吗?

**参考回答**:docstring 说得很直白:如果没有这个回退,未绑定场景下 run 会跑在 `safe(msg.user_id)` 下,但入站文件会落到 dispatcher task 的 contextvar 默认值 `users/default/...`——agent 读 `users/{平台ID}/`、文件却在 `users/default/`,上传直接"消失"([manager.py:535-543](../backend/app/channels/manager.py#L535-L543))。所以回退是"桶一致性"的兜底,不是功能施舍。

至于"直接拒绝"——那是 `require_bound_identity` 开关的职责:开启时未绑定消息在占用信号量之前就被拒([manager.py:1068-1073](../backend/app/channels/manager.py#L1068-L1073)),且该开关默认跟随 connections enabled([service.py:136-142](../backend/app/channels/service.py#L136-L142))。

auth 关闭的本地部署里,`_auth_disabled_owner_user_id` 让所有消息归到统一 owner([manager.py:491-497](../backend/app/channels/manager.py#L491-L497)),此时绑定检查也整体跳过([manager.py:1120-1121](../backend/app/channels/manager.py#L1120-L1121))——本地单人场景不增加任何摩擦。

**Q6.3(边界/异常)** agent 产出的文件要发回 IM,路径安全怎么保证?用户能不能诱导 agent 把服务器上任意文件发出来?

**参考回答**:`_resolve_attachments` 只接受 `/mnt/user-data/outputs/` 前缀的虚拟路径,其他直接拒绝并告警;前缀过了之后 resolve 成宿主路径,再做一次 `relative_to(outputs_dir)` 防 `..` 逃逸;不存在于磁盘的也跳过([manager.py:598-614](../backend/app/channels/manager.py#L598-L614))。

artifact 列表本身也有边界:只从最后一条 human 消息之后的 `present_files` tool call 提取,历史轮次的 artifact 不会被重发([manager.py:417-449](../backend/app/channels/manager.py#L417-L449))。入站方向同样设防:文件名经 `normalize_filename` + `claim_unique_filename`,写入用 `write_upload_file_no_symlink` 防符号链接攻击,文件下载 HTTP 超时 20 秒([manager.py:684](../backend/app/channels/manager.py#L684),[manager.py:716-734](../backend/app/channels/manager.py#L716-L734))。

平台侧还有体积上限:Feishu 图片 10MB/文件 30MB([feishu.py:254-259](../backend/app/channels/feishu.py#L254-L259)),Telegram 文档 50MB/图片 10MB([telegram.py:264-272](../backend/app/channels/telegram.py#L264-L272)),DingTalk 20MB([dingtalk.py:30](../backend/app/channels/dingtalk.py#L30))。另外文本发送失败时附件直接跳过,杜绝"只有文件没有上下文"的部分投递([base.py:166-179](../backend/app/channels/base.py#L166-L179))。

---

## 问题链 7:并发、去重与可靠性边界

**Q7.1(基础)** 同一个会话用户连发两条消息,agent 还在处理第一条,第二条会怎样?

**参考回答**:run 创建用 `multitask_strategy="reject"`([manager.py:1303](../backend/app/channels/manager.py#L1303)),LangGraph 拒绝同 thread 并发 run;Manager 捕获后识别 `ConflictError` 或 "already running a task" 字符串,回一条固定文案 `THREAD_BUSY_MESSAGE`([manager.py:175-180](../backend/app/channels/manager.py#L175-L180),[manager.py:1313-1317](../backend/app/channels/manager.py#L1313-L1317))。

流式路径同样处理,在 finally 里把 busy 转成面向用户的提示([manager.py:1429-1452](../backend/app/channels/manager.py#L1429-L1452))。全局并发上限是 `max_concurrency=5` 的信号量([manager.py:788](../backend/app/channels/manager.py#L788))。

另一个容易混的数字:lead agent 的 recursion budget 是 100 个 super-step(`DEFAULT_RUN_CONFIG`,[manager.py:51](../backend/app/channels/manager.py#L51)),注释明确说它和 subagent 的 `max_turns` 是两本账,不许混淆([manager.py:46-50](../backend/app/channels/manager.py#L46-L50))。

**链路解析**:

```
 msg ──► _dispatch_loop
        │
        ├─ _is_duplicate_inbound? ──yes──► 记日志并忽略(10min TTL / 4096 条窗口)
        │ no
        ▼ create_task(_handle_message)        ← 注意:此处不取信号量,task 可堆积
        │
        ├─ msg_type != COMMAND 且 require_bound_identity?
        │      └─ 复查库失败 ──► _reject_unbound_channel_message(不占信号量)
        ▼
   async with Semaphore(5):
        ├─ COMMAND → _handle_command(自带同一道 bound-identity 门)
        └─ CHAT    → _handle_chat
                        │
                        ▼ runs.wait / runs.stream(multitask_strategy="reject")
                   ConflictError / "already running a task"
                        │
                        ▼
              THREAD_BUSY_MESSAGE 回复用户(非流式直接回;流式在 finally 回)
```

**Q7.2(深挖)** 平台会重投同一条消息(比如 Slack 重试事件),怎么去重?为什么缺 workspace_id 就放弃去重?

**参考回答**:`_inbound_dedupe_key` 取 metadata 里 `event_id/message_id/msg_id` 中第一个非空值——刻意只用服务端稳定 id,客户端生成的 id 在重投时可能变化,注释写明了这点([manager.py:68-71](../backend/app/channels/manager.py#L68-L71))。找不到还会去 `raw_message` 里再翻一层([manager.py:993-999](../backend/app/channels/manager.py#L993-L999))。

key 形如 `(channel, workspace_id, chat_id, message_id)`;缓存是 OrderedDict,TTL 10 分钟、上限 4096 条(`INBOUND_DEDUPE_TTL_SECONDS = 10 * 60`、`INBOUND_DEDUPE_MAX_ENTRIES = 4096`,[manager.py:66-67](../backend/app/channels/manager.py#L66-L67)),利用插入序即时间序,过期和溢出都从头部 O(k) 驱逐([manager.py:1016-1026](../backend/app/channels/manager.py#L1016-L1026))。

没有 workspace_id 时 fail-closed 不去重([manager.py:1003-1009](../backend/app/channels/manager.py#L1003-L1009))——因为 Slack channel id 不是全局唯一的,缺了 workspace 维度会把两个不同工作区的消息误判为重复。处理失败时 `_release_inbound_dedupe_key` 释放 key,让平台重投能恢复,避免"可恢复错误变成 10 分钟黑洞"([manager.py:1040-1050](../backend/app/channels/manager.py#L1040-L1050))。

注意边界:manager 级去重只保护 agent run 和最终答案;平台侧的"Working on it"占位和 reaction 在 publish 前已发出,刻意不去重([manager.py:964-970](../backend/app/channels/manager.py#L964-L970))。

**Q7.3(边界/异常)** 出站发送失败的重试策略是什么?重试期间用户会看到什么?

**参考回答**:统一走 `_send_with_retry`,指数退避 `delay = 2**attempt`(1s、2s、4s),默认 `max_retries=3`,全部失败抛出最后一次异常([base.py:75-107](../backend/app/channels/base.py#L75-L107))。

关键顺序:先发文本,文本失败就**直接 return,不再发附件**——避免"文件到了但正文丢了"的部分投递([base.py:166-179](../backend/app/channels/base.py#L166-L179))。Telegram 对 final 编辑单独给一次 `retry_after` 重试,按平台返回的秒数 sleep([telegram.py:215-229](../backend/app/channels/telegram.py#L215-L229))。

相关基础设施:DingTalk 的 access token 有缓存层,提前 300 秒刷新(`_TOKEN_REFRESH_MARGIN_SECONDS = 300`,默认 expireIn 7200 秒),双检锁防并发刷新([dingtalk.py:25](../backend/app/channels/dingtalk.py#L25),[dingtalk.py:582-612](../backend/app/channels/dingtalk.py#L582-L612))。

用户在重试期间看到的是之前已发出的占位:Feishu 的运行卡片、Telegram 注册进 stream 表的"Working on it..."消息([telegram.py:337-356](../backend/app/channels/telegram.py#L337-L356))——不会看到报错,除非最终也失败。

---

## 问题链 8:ChannelService 生命周期与配置热更新

**Q8.1(基础)** 七个平台 channel 是怎么被加载和启动的?新增一个平台要改哪些地方?

**参考回答**:懒加载注册表 `_CHANNEL_REGISTRY` 把名字映射到 `module:Class` 字符串,启动时 `resolve_class` 动态 import([service.py:23-31](../backend/app/channels/service.py#L23-L31),[service.py:310-323](../backend/app/channels/service.py#L310-L323))——没启用的平台连模块都不加载,lark_oapi 这类重依赖只有 Feishu 启用时才 import,import 失败只记 error 不 crash 进程([feishu.py:111-131](../backend/app/channels/feishu.py#L111-L131))。

启动流程:`ChannelService.__init__` 建 bus/store/manager([service.py:93-121](../backend/app/channels/service.py#L93-L121))→ `start()` 先启 manager 再 `ensure_ready_channels(attempts=2)`([service.py:145-155](../backend/app/channels/service.py#L145-L155))。`_start_channel` 实例化时会注入共享的 `channel_store` 和 `connection_repo`([service.py:325-330](../backend/app/channels/service.py#L325-L330))。

新增一个平台要动三处:实现 Channel 子类、注册表加一行、`_CHANNEL_CREDENTIAL_KEYS` 加凭据键([service.py:34-42](../backend/app/channels/service.py#L34-L42))——后者用于"配了凭据但 enabled=false"的告警提示([service.py:163-167](../backend/app/channels/service.py#L163-L167))。

**链路解析**:

```
 config.yaml channels 段 + runtime-config.json(UI 凭据 overlay)
        │ merge_runtime_channel_configs(_runtime_disabled → 直接剔除)
        ▼
 ChannelService.from_app_config
        ├─ ChannelStore(JSON) / ChannelConnectionRepository(DB,可选)
        └─ ChannelManager(bus, store, repo, require_bound_identity)
        ▼ start()
 manager.start() ──► ensure_ready_channels(attempts=2)
        │ 逐 channel: enabled? 有凭据但 disabled → warning
        ▼
 ensure_channel_ready(name)  ── 每 channel 一把 asyncio.Lock 串行化
        │
        ▼ _start_channel
 resolve_class("app.channels.feishu:FeishuChannel")  ← 懒 import
        │ config 注入 channel_store / connection_repo
        ▼
 channel_cls(bus, config) → await channel.start()
        │
        ▼ is_running? (Feishu 还要求 WS 线程存活)
   否 → 从 _channels 摘除,记 error,按 attempts 重试
```

**Q8.2(深挖)** readiness 是从 HTTP 请求处理里轮询触发的,并发两个请求同时发现 channel 挂了会怎样?

**参考回答**:`ensure_channel_ready` 用每 channel 一把 `asyncio.Lock` 串行化([service.py:190-193](../backend/app/channels/service.py#L190-L193)),注释明确写了动机:readiness 会被请求handler并发轮询,不能允许同一 channel worker 被 stop/start 两次。

拿到锁后先查 `channel.is_running`,已跑就直接返回;处于非 running 状态的实例先 stop 再摘除,然后按 `attempts` 重试启动([service.py:201-218](../backend/app/channels/service.py#L201-218))。Feishu 的 `is_running` 不只是看标志位,还要求 WS 工作线程存活([feishu.py:90-94](../backend/app/channels/feishu.py#L90-L94))——能识别"标志位没来得及清但线程已死"的中间态。

`_start_channel` 里 `start()` 返回后 `is_running` 仍为 False 也算失败并摘除([service.py:330-336](../backend/app/channels/service.py#L330-L336)),防止"start 静默失败但实例留在注册表"的僵尸状态。

**Q8.3(边界/异常)** 浏览器里改凭据,和直接改 config.yaml,两条热更新路径有什么区别?为什么 `configure_channel` 刻意不做 file reload?

**参考回答**:文件路径走 `restart_channel(reload_config=True)` → `_load_channel_config`:用带签名检测的 `get_app_config()` 读盘,并重新叠加 runtime overlay,磁盘 IO 放 `asyncio.to_thread`([service.py:234-259](../backend/app/channels/service.py#L234-L259),[service.py:270-273](../backend/app/channels/service.py#L270-L273))。

UI 路径走 `configure_channel` → `restart_channel(reload_config=False)`。注释说明原因:调用方刚给的 config 是权威值(浏览器录入的凭据从不写回 config.yaml),此时 reload 会用磁盘上的旧值把新凭据覆盖掉([service.py:286-294](../backend/app/channels/service.py#L286-L294))。

UI 凭据持久化在 runtime-config.json(`chmod 0o600`);用户在 UI 点"断开"时用 `_runtime_disabled` 标志位表达,合并时命中该标志直接从 channels_config 删除整个 provider([runtime_config_store.py:84-90](../backend/app/channels/runtime_config_store.py#L84-L90),[runtime_config_store.py:110-131](../backend/app/channels/runtime_config_store.py#L110-L131))——**不这样设计会怎样**:没有 disabled 标志的话,文件里的旧配置会在下次 reload 时把 UI 里已断开的 channel"复活",凭据删除变成不可达操作。

---

## 面试官最爱追问的 3 个点

1. **"流式更新在三个平台上分别怎么落地,限频数字哪来的?"** —— 应答策略:一句话分层:Manager 侧统一 0.35s 节流,平台侧各显神通:Feishu `update_multi=true` 原地 patch 卡片(final patch 失败降级为新回复),Telegram 私聊 1s/群聊 3s 编辑 + 4096 截断 + 429 静默丢帧等 final 兜底,Slack/Discord 直接 `runs.wait` 一次性,DingTalk 靠 `card_template_id` 开 AI Card 全量覆盖流、卡片缺失时丢中间帧保 final。数字全部来自平台官方限频(群聊约 20 msg/min)。

2. **"绑定的安全模型:code 会不会被重放/爆破/冒用?"** —— 应答策略:四层答案背熟:128 bit `token_urlsafe(16)` + SHA-256 哈希存储、600s TTL + 一次性(条件 UPDATE rowcount 判胜)、每 (owner,provider) ≤5 pending(事务内 count+insert / PG advisory lock)、Manager 侧不信消息字段、按平台身份复查库比对。再补一句 allowed_users 刻意放行绑定码的原因——白名单管"日常谁能用",绑定码管"谁能成为谁"。

3. **"同一外部身份被两个 owner 绑定,并发下怎么保证只有一个活跃?"** —— 应答策略:应用层 revoke-other-owners(先撤旧再插新,同事务)+ 数据库部分唯一索引 `uq_channel_connection_active_identity`(status != 'revoked' 才参与唯一性)+ IntegrityError 回滚重试至多 3 次。强调 revoked 历史行保留用于审计,credential 随行级联删除。
