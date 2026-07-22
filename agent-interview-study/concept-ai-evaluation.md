# 概念补充：AI/Agent 评测体系 —— 以 Langfuse 为例

> 本篇是概念补充文档,不是 deer-flow 源码精读。deer-flow 后端没有系统性的 agent 质量评测框架(只有单元/集成测试和 skill 触发评测),而"怎么评估 agent 质量"是 agent 岗位面试高频题。本篇以 Langfuse(GitHub 31.6k star,deer-flow tracing 内置对接的平台)为例,讲清楚生产级 LLM 评测的完整方法论、架构设计和链路流程,以及它和 deer-flow 怎么接。
>
> 资料来源:Langfuse 官方文档 Evaluation 板块(evaluation/overview、concepts、datasets、llm-as-a-judge、scores via UI 等页面)+ GitHub API 实时 star 数据(2026-07 查询)。

## 1. 为什么 agent 评测难

先理解问题本身,才能理解工具为什么这么设计:

- **没有唯一标准答案**:同一个用户问题,十种回答都可能是对的。传统单测的 `assertEqual` 失效
- **质量是主观的**:helpfulness、语气、是否切题——这些维度没有确定性函数能计算
- **多轮轨迹**:agent 的质量不只看最终回答,还要看中间的 tool 调用序列合不合理(该搜的没搜、重复调用、绕远路)
- **分布会漂**:用户输入的分布在变,模型在换,prompt 在改——上个月测好的不代表这个月还好

所以 LLM 评测的答案是:**用 LLM 评 LLM(LLM-as-a-Judge)+ 人工标注做锚点 + 线上持续打分 + 离线回归集**,四件套缺一不可。

## 2. Langfuse 整体框架图

Langfuse 的自我定位是 "AI Engineering Platform"——观测、Prompt 管理、评测、指标四大块共享同一套数据底座:

```
┌─────────────────────────────────────────────────────────────────┐
│                     Langfuse Platform                           │
│                                                                 │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌────────┐  │
│  │ Observability│ │    Prompt    │ │  Evaluation  │ │Metrics │  │
│  │              │ │  Management  │ │              │ │  &     │  │
│  │ traces       │ │ 版本管理     │ │ scores       │ │Dashboards│ │
│  │ observations │ │ 线上拉取     │ │ datasets     │ │ 趋势分析│  │
│  │ sessions     │ │ 灰度发布     │ │ experiments  │ │ 告警   │  │
│  └──────┬───────┘ └──────┬───────┘ └──────┬───────┘ └───┬────┘  │
│         │                │                │             │       │
│         └────────────────┴────────────────┴─────────────┘       │
│                          │                                      │
│              ┌───────────▼────────────┐                         │
│              │   统一数据底座          │                         │
│              │  Trace / Observation   │                         │
│              │  Score / Dataset       │                         │
│              │  (Postgres + ClickHouse│                         │
│              │   + S3 + Redis,自托管) │                         │
│              └───────────▲────────────┘                         │
└──────────────────────────┼──────────────────────────────────────┘
                           │ SDK / API / Integrations
        ┌──────────────────┼──────────────────┐
        │                  │                  │
   ┌────▼─────┐      ┌─────▼──────┐     ┌─────▼─────┐
   │ 你的应用  │      │ 评测脚本    │     │ CI/CD     │
   │(deer-flow│      │(experiment)│     │(回归门禁) │
   │ 埋点上报) │      │            │     │           │
   └──────────┘      └────────────┘     └───────────┘
```

**关键点**:
- 四大块不是四个独立产品,是**同一批数据(trace/score/dataset)上的四个视图**。trace 上能打分,打分的对象能进 dataset,dataset 能跑 experiment,experiment 的结果在 dashboard 看趋势——这就是为什么选平台型工具而不是拼凑单点工具
- 自托管时数据底座是 Postgres(元数据)+ ClickHouse(trace/score 分析)+ S3(大 payload)+ Redis(队列)——这个组合本身就是"OLTP + OLAP + 对象存储"的经典分析型架构

## 3. 核心数据模型关系图

理解 Langfuse 的数据模型,就理解了评测怎么落地:

```
                        ┌──────────┐
                        │ Project  │
                        └────┬─────┘
          ┌──────────────────┼──────────────────┐
          │                  │                  │
   ┌──────▼───────┐   ┌──────▼──────┐    ┌──────▼──────┐
   │   Session    │   │   Dataset   │    │ ScoreConfig │
   │ (多轮会话)    │   │  (测试集)   │    │ (评分标准)  │
   └──────┬───────┘   └──────┬──────┘    └──────┬──────┘
          │ 1:N              │ 1:N              │ 定义
   ┌──────▼───────┐   ┌──────▼──────┐          │
   │    Trace     │   │ DatasetItem │          │
   │ (一次 run)   │   │ input +     │          │
   │              │   │ expected_   │          │
   │              │   │ output      │          │
   └──────┬───────┘   └─────────────┘          │
          │ 1:N                                │
   ┌──────▼───────┐                            │
   │ Observation  │  (span: LLM 调用/tool 调用/ │
   │              │   chain 节点,树状嵌套)      │
   └──────┬───────┘                            │
          │                                    │
          ▼ 挂载(N:1,四种来源)                ▼
   ┌─────────────────────────────────────────────┐
   │                   Score                     │
   │  name + value + 类型(NUMERIC/CATEGORICAL/  │
   │  BOOLEAN/TEXT) + source                     │
   │  ┌──────────┬──────────┬────────┬─────────┐ │
   │  │LLM-judge │ 人工标注  │代码检查│ 用户反馈 │ │
   │  └──────────┴──────────┴────────┴─────────┘ │
   └──────────────────┬──────────────────────────┘
                      │
   DatasetRun ────────┘  (一次 experiment:Dataset × N items
   (实验运行记录)         → N 条 trace + N 组 scores)
```

**三层关系一句话**:观测侧 `Session → Trace → Observation` 是树状嵌套(一次会话多次 run,一次 run 多个操作);评测侧 `Dataset → DatasetItem` 是测试集;`Score` 是胶水,可以挂在 Trace/Observation/Session/DatasetRun 任意一层上,并且通过 `ScoreConfig` 标准化。

**为什么 Score 要统一**:人工标注、LLM judge、代码检查、用户反馈产生的分数都是同一个数据模型 → 可以在同一视图对比"人 vs judge"的一致性(校准),可以按时间/维度做统一趋势分析。

## 4. 评估闭环总链路(离线实验 → 线上打分 → bad case 回流)

```
                        ┌──────────────────────────┐
                        │  ① 开发:改 prompt/模型   │
                        └────────────┬─────────────┘
                                     ▼
┌──────────────────── 离线(部署前)──────────────────────────┐
│  ② Experiment:Dataset 每条 item 跑一遍应用                 │
│     ┌─────────┐   ┌─────────┐   ┌─────────┐               │
│     │ item 1  │   │ item 2  │...│ item N  │               │
│     │ input→  │   │ input→  │   │ input→  │               │
│     │ app→out │   │ app→out │   │ app→out │               │
│     └────┬────┘   └────┬────┘   └────┬────┘               │
│          ▼             ▼             ▼                     │
│  ③ Evaluator 打分(judge/code)→ 每条 item 一组 scores      │
│          ▼                                                 │
│  ④ 汇总对比:新 prompt vs baseline,看分数分布              │
│     → 满意 → 部署;不满意 → 回到 ①                        │
└────────────────────────────┬───────────────────────────────┘
                             ▼
┌──────────────────── 线上(部署后)─────────────────────────┐
│  ⑤ 生产流量 → 每条 trace 实时上报 Langfuse                 │
│  ⑥ Online judge:抽样/全量 trace 自动打分                  │
│     + 用户反馈(thumbs up/down)+ 人工抽检(annotation)     │
│  ⑦ Dashboard 监控:分数趋势、按 metadata 切片              │
│     (哪个模型版本/哪个 agent/哪类 query 分数掉了)          │
│  ⑧ 发现 bad case:低分 trace / 用户点踩 / 人工发现         │
└────────────────────────────┬───────────────────────────────┘
                             ▼
              ⑨ 回流:bad case 一键加入 Dataset
               (input 来自 trace,expected_output 人工修)
                             │
                             └──► 回到 ②,下次实验自动覆盖这类 case
```

**官方文档的客户支持 bot 实例**(走一遍闭环):改 prompt 让语气更随意 → ②③④ 实验发现语气好了但回复变长漏链接 → 迭代 prompt 再实验 → 部署 → ⑥ 线上 judge 发现法语 query 用英语回了 → ⑨ 把这条加进 dataset → 改 prompt 支持法语 → 再实验验证。**dataset 从几条长成有代表性的真实测试集**——这回答了"评测集从哪来":不是拍脑袋编的,是从生产流量里沉淀的。

## 5. 链路分析一:LLM-as-a-Judge 线上打分

```
线上 trace 产生(deer-flow 的 agent run 完成)
        │
        ▼
┌─────────────────┐
│ Langfuse 接收    │  trace + observations(完整树)
│ trace 入库       │  metadata: session_id=thread_id, user_id, tags
└────────┬────────┘
         │ 触发(可配置:全量/抽样/按 metadata 过滤)
         ▼
┌─────────────────────────────────────────┐
│ 组装 Judge Prompt(四要素):             │
│                                         │
│  ┌─────────────────────────────────┐    │
│  │ ① Rubric(评分标准)             │    │
│  │  "1分=事实错误,5分=完全准确     │    │
│  │   且有依据"                     │    │
│  ├─────────────────────────────────┤    │
│  │ ② Input context                 │    │
│  │  用户原始 query(从 trace 提取) │    │
│  ├─────────────────────────────────┤    │
│  │ ③ Output to evaluate            │    │
│  │  应用的回答(从 trace 提取)     │    │
│  ├─────────────────────────────────┤    │
│  │ ④ Reference(可选)              │    │
│  │  ground truth / 期望输出        │    │
│  └─────────────────────────────────┘    │
└────────────────┬────────────────────────┘
                 ▼
        ┌─────────────────┐
        │  Judge 模型调用  │ (通常用比被评应用更强的模型)
        │  返回 score +    │
        │  reasoning       │
        └────────┬────────┘
                 ▼
┌─────────────────────────────────────────┐
│ Score 落库,挂到 trace 上:              │
│  name="helpfulness", value=0.8,        │
│  type=NUMERIC, source=llm-judge,       │
│  comment="回答准确但缺少引用..."         │
└────────────────┬────────────────────────┘
                 ▼
┌─────────────────────────────────────────┐
│ Score Analytics:                        │
│  - 按时间看趋势(发版后分数掉没掉)      │
│  - 按 metadata 切片(哪个模型/agent)   │
│  - 低分 trace 列表 → 人工 review → 回流 │
└─────────────────────────────────────────┘
```

**三种 score 类型的适用场景**(面试加分细节):

| 类型 | 适用 | 例子 |
|---|---|---|
| **NUMERIC**(0~1 连续) | 程度判断 | helpfulness、流畅度 |
| **CATEGORICAL**(分档) | 明确分档 | correct / partially_correct / incorrect |
| **BOOLEAN**(二值) | 是否判断 | 是否违规、是否超范围、用户是否在反驳 |

**Judge 的三种挂载点**:Observations(单个操作,评某次 LLM/tool 调用)、Traces(完整工作流,评整个 run 的最终结果)、Experiments(受控测试集,离线评)。开发期用 Experiments 挂载点,生产监控用 Traces/Observations。

## 6. 链路分析二:Experiment 离线回归

```
Dataset(测试集)
  ├─ item 1: {input: "查一下苹果股价", expected_output: "包含实时价格..."}
  ├─ item 2: {input: "analyze this CSV", expected_output: "生成图表+结论"}
  └─ ...N 条
        │
        ▼ 对每条 item(可并行)
┌──────────────────────────────┐
│ ① 用 item.input 调你的应用    │ ←  deer-flow:POST /threads/{id}/runs
│   得到 output + 新 trace      │     (新 trace 也进 Langfuse,和
└──────────────┬───────────────┘      DatasetRun 关联)
               ▼
┌──────────────────────────────┐
│ ② Evaluator 逐个打分          │
│   - judge:output vs expected │
│   - code:格式/关键词/能跑吗   │
└──────────────┬───────────────┘
               ▼
┌──────────────────────────────┐
│ ③ DatasetRun 汇总            │
│   平均分/分布/每条对比        │
└──────────────┬───────────────┘
               ▼
┌──────────────────────────────────────────┐
│ ④ Compare View:本次 run vs baseline     │
│                                          │
│   item │ baseline │ 新prompt │ Δ         │
│   ─────┼──────────┼──────────┼───        │
│   1    │ 0.9      │ 0.7      │ -0.2 ⚠️   │
│   2    │ 0.6      │ 0.8      │ +0.2 ✅   │
│   3    │ 0.8      │ 0.8      │  0.0      │
│   avg  │ 0.77     │ 0.77     │  0.0      │
│                                          │
│  → 平均分没变但 item 1 退化 → 点开看     │
│    具体输出差异 → 决定:接受/回滚/继续改 │
└──────────────┬───────────────────────────┘
               ▼
┌──────────────────────────────┐
│ ⑤ CI/CD 集成(可选)          │
│   分数低于阈值 → block 部署   │
└──────────────────────────────┘
```

**关键洞察**:看**分数分布**不只看平均分——平均分不变可能掩盖"一半变好一半变坏"。这就是为什么 Compare View 是按 item 逐条对比的。

## 7. 链路分析三:人工标注校准 Judge

```
问题:LLM 评 LLM 会漂(judge 和被评模型同构,犯相似错误;
     judge 有位置偏见/长度偏见)
        │
        ▼
① 从生产 trace 抽一批样本(100 条左右,分层抽样:
   不同分数段/不同 query 类型都覆盖)
        │
        ├──► 路径 A:人工标注(Annotation Queue)
        │     ┌────────────────────────────────┐
        │     │ 先建 Score Config:              │
        │     │  name="correctness"             │
        │     │  1=错误,2=部分对,3=完全对      │
        │     │  (标准化,保证多人标注一致)      │
        │     │          │                      │
        │     │  多人协作在 trace 页打分        │
        │     │  → 得到 100 条"人工 ground truth"│
        │     └────────────────────────────────┘
        │
        └──► 路径 B:LLM judge 评同一批
              ┌────────────────────────────────┐
              │ 用当前 rubric 对这 100 条打分   │
              │ → 得到 100 条"judge 分"        │
              └────────────────────────────────┘
        │
        ▼
② 对比:人 vs judge 一致性(Cohen's Kappa / 简单一致率)
        │
        ├─ 一致率高(>80%)→ judge 可信,放心扩大自动打分
        │
        └─ 一致率低 → 分析分歧 case:
           ├─ rubric 太模糊?("评价好不好"→ 改成具体行为描述)
           ├─ judge 缺信息?(把更多 context 塞进 judge prompt)
           └─ 这个维度就不适合 judge?→ 改用 code evaluator 或放弃自动评
        │
        ▼
③ 迭代 rubric → 重跑一致性 → 直到 judge 可信
```

**工程原则**:能不用 judge 的地方不用——格式/长度/关键词/代码能否运行,用 Code Evaluator(便宜、确定、零偏差);judge 只负责主观维度(helpfulness、正确性、语气)。

## 8. 链路分析四:deer-flow 接入 Langfuse 评测

deer-flow 已内置 Langfuse 对接([tracing/factory.py](../backend/packages/harness/deerflow/tracing/factory.py) 的 `build_tracing_callbacks()`,详见 [qa-15](qa-15-observability-guardrails-misc.md)),完整接入链路:

```
┌──────────────── deer-flow 侧(已有)─────────────────┐
│                                                    │
│  用户请求 → POST /threads/{id}/runs/stream         │
│       │                                            │
│       ▼                                            │
│  LangGraph run(agent.py 根部挂 tracing 回调)      │
│  ┌──────────────────────────────────────────┐     │
│  │ metadata:                                 │     │
│  │  session_id = thread_id    ← 关联会话    │     │
│  │  user_id                   ← 关联用户    │     │
│  │  tags = [agent_name, model_name, ...]    │     │
│  └──────────────────────────────────────────┘     │
│       │ 每个 LLM 调用/tool 调用 = 一个 observation │
└───────┼────────────────────────────────────────────┘
        │ 实时上报
        ▼
┌──────────────── Langfuse 侧(接上即用)─────────────┐
│                                                    │
│  ⑤ 线上打分:LLM-as-a-Judge 对 trace 自动评分      │
│     按 tags 过滤(只评某个 model 的/某个 agent 的) │
│       │                                            │
│       ▼                                            │
│  ⑧ 低分 trace → 一键加进 Dataset                   │
│     (同时可导 deer-flow 的 run_events 表做补充,   │
│      append-only 事件流,见 qa-11)                 │
│       │                                            │
│       ▼                                            │
│  ② Experiment:改了 prompt.py 或 config.yaml       │
│    的模型配置 → 对 dataset 跑回归 → 对比 baseline  │
│       │                                            │
│       ▼                                            │
│  满意 → 部署 → 回到线上打分,闭环运转              │
└────────────────────────────────────────────────────┘
```

**接入时的关键陷阱**(deer-flow [agent.py:3-18](../backend/packages/harness/deerflow/agents/lead_agent/agent.py#L3-L18) 开头的 INVARIANT):Langfuse handler 只在 `on_chain_start(parent_run_id=None)` 时把 `langfuse_session_id`/`langfuse_user_id` 提升到 trace 上,所以回调**必须挂 graph 根**,图内所有 `create_chat_model` 必须 `attach_tracing=False`。违反的后果不只是双重 span——**session/user 维度丢失,线上"按用户/会话切片打分"直接做不了**,评测体系的第一公里就断了。

## 9. 评测工具全景(GitHub API 实时 star,2026-07)

### 平台型(观测+评测一体)

| 项目 | ⭐ | 定位 |
|---|---|---|
| [langfuse/langfuse](https://github.com/langfuse/langfuse) | 31.6k | 开源 AI 工程平台:观测+评测+prompt 管理。**deer-flow tracing 内置对接** |
| [promptfoo/promptfoo](https://github.com/promptfoo/promptfoo) | 23.5k | prompt/agent/RAG 测试 + 红队扫描,CLI+yaml,最接地气 |
| [comet-ml/opik](https://github.com/comet-ml/opik) | 20.8k | LLM 应用调试/评测/监控,主打 agentic workflow |
| [Arize-ai/phoenix](https://github.com/Arize-ai/phoenix) | 10.7k | AI 观测+评测,trace 分析见长 |

### 评测框架型(写评测代码的库)

| 项目 | ⭐ | 定位 |
|---|---|---|
| [openai/evals](https://github.com/openai/evals) | 19.0k | OpenAI 官方框架 + benchmark 注册表,名声大但更新放缓 |
| [confident-ai/deepeval](https://github.com/confident-ai/deepeval) | 17.0k | "LLM 界的 pytest",断言式 API,手感像写单测 |
| [vibrantlabsai/ragas](https://github.com/vibrantlabsai/ragas) | 15.0k | **RAG 评测事实标准**:faithfulness/answer relevancy/context precision/recall 四指标 |
| [truera/trulens](https://github.com/truera/trulens) | 3.5k | 评测+实验追踪,学术味重 |

### 模型 benchmark 型(评模型本身,不是评应用)

| 项目 | ⭐ | 定位 |
|---|---|---|
| [EleutherAI/lm-evaluation-harness](https://github.com/EleutherAI/lm-evaluation-harness) | 13.4k | 基础模型 benchmark 标准工具,HF 排行榜背后 |
| [modelscope/evalscope](https://github.com/modelscope/evalscope) | 3.1k | 阿里 ModelScope 出品,国内大厂常用 |
| [stanford-crfm/helm](https://github.com/stanford-crfm/helm) | 2.9k | 斯坦福整体评估框架 |

**agent 应用岗优先级**:promptfoo(覆盖 prompt+agent+RAG+红队)> deepeval(pytest 风格,和 deer-flow"约定即测试"哲学相通)> ragas(RAG 场景必背四指标)> langfuse(不用额外学,deer-flow 已对接)。

## 10. 面试问答

**Q:你们怎么评估 agent 的质量?**

> 分三层。第一层,代码行为正确性:单元/集成测试 + "约定即测试"的门禁(deer-flow 的 Blockbuster 阻塞检测、架构边界 AST 测试)。第二层,线上质量监控:tracing 平台(Langfuse)收集 trace,LLM-as-a-Judge 按 rubric 自动打分,人工 annotation 做 baseline 校准 judge;线上发现的 bad case 回流进 dataset。第三层,离线回归:用不断增长的 dataset 跑 experiment,任何 prompt/模型变更先过评测再上线,接 CI block 回归。"线上打分 → bad case 回流 → 离线回归"的闭环是大厂主流做法。

**Q:LLM-as-a-Judge 可靠吗?**

> 不完全可靠,所以要校准。两个已知问题:judge 和被评模型可能同构犯相似错误;judge 有位置偏见和长度偏见(偏好长回答)。工程上的解法:(1) 用人工标注一批做 baseline,算 judge 和人的一致性,不一致就调 rubric;(2) rubric 要具体——"1 分=事实错误,5 分=完全准确且有引用"比"评价好不好"稳定得多;(3) 能不用 judge 的地方不用——确定性检查(格式、长度、关键词、代码能否运行)用 Code Evaluator,便宜且稳定。judge 只负责主观维度。

**Q:评测集从哪来?**

> 三条路:(1) 冷启动时人工编一批典型 case(几十条就够跑 experiment);(2) 主力来源是生产流量——线上打分低的 trace、用户点踩的、人工 review 发现的 edge case,回流进 dataset,这样 dataset 跟着真实分布长,不会偏离;(3) 对抗性补充——红队扫描(promptfoo 那类)生成的注入/越狱 case,单独建一个安全评测集,不和质量评测集混。

**Q:RAG 场景怎么评?(如果岗位涉及知识库)**

> 检索侧和生成侧分开评。检索侧:context precision(检回来的有多少是相关的)、context recall(相关的有多少被检回来了)。生成侧:faithfulness(回答有没有编造检索资料里没有的内容——这是 RAG 最重要的指标,防"忠实地基于错误资料生成")、answer relevancy(回答是否切题)。RAGAS 这四个指标是事实标准。两侧分开评的原因:"检索到了但生成没用好"和"检索本身就没找对"是两个不同问题,合在一起看会掩盖瓶颈。

**Q:agent 的轨迹(tool 调用序列)怎么评?**

> 这是 agent 评测和普通 LLM 评测的核心区别。方法:(1) 确定性检查优先——该调的工具调了没有、参数对不对、有没有重复调用(deer-flow 的循环检测中间件本质上就是线上的轨迹异常检测);(2) 用 judge 评轨迹合理性——把完整 trace(含 tool calls)给 judge,问"这个解决路径合理吗";(3) 成本指标也算质量——token 消耗、工具调用次数、延迟,同样的结果谁花得少谁好。deer-flow 的 run_events 表持久化了完整事件流,天然是轨迹评测的数据源。

## 11. 小结

- 评测难在"没有标准答案",所以方案是 **LLM judge + 人工锚点 + 线上打分 + 离线回归**四件套
- Langfuse 的核心抽象:一切皆 Score(统一存储),dataset 从生产流量里长出来(回流闭环)
- 数据模型三层:观测侧 Session→Trace→Observation 树状嵌套,评测侧 Dataset→DatasetItem,Score 是胶水挂任意层
- 四条核心链路:judge 线上打分(rubric 四要素)、experiment 离线回归(按 item 对比不只看平均分)、人工标注校准 judge(一致性检验)、deer-flow 接入(graph 根回调是评测的第一公里)
- 面试答"怎么评估 agent",按"三层 + 闭环"的结构讲,比罗列工具名更有说服力
