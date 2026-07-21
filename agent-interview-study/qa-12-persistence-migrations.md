# 持久化与 Schema 迁移 —— 面试问题链

> 本文档按"大厂面试追问链"组织:每条问题链从基础问题开始,层层追问到实现细节、设计权衡、异常边界。配套深读笔记:无(本模块尚无深读笔记,本文档是第一份)。
> 代码引用格式:[文件名.py:起始行-结束行](../backend/仓库相对路径#L起始行-L结束行)。只引用你实际读过的行,禁止编造行号。

本模块回答一个核心问题:**一个同时支持 SQLite 和 Postgres、还夹带着 LangGraph 私有表的 FastAPI 服务,如何在启动时安全地把数据库 Schema 演进到最新版本?** 答案是一套"create_all + Alembic stamp/upgrade"的混合 bootstrap 状态机,配合方言级的并发锁、幂等列迁移和方言感知的 JSON 匹配。

模块文件地图(全部逐行读过):

```
persistence/
  base.py            Base(DeclarativeBase) + to_dict, @cache 的列反射
  engine.py          init_engine / PRAGMA / 连接池 / 自动建库重试
  bootstrap.py       三分支状态机 + 两种后端锁 + alembic in-process 配置
  json_compat.py     JsonMatch 方言可移植 JSON 过滤
  migrations/
    env.py           alembic 环境(online/offline, 自建 engine)
    _env_filters.py  include_object: 排除 LangGraph 4 表
    _helpers.py      safe_add_column / safe_drop_column + 漂移检测
    versions/
      0001_baseline.py          9 张基线表 DDL + 完整 downgrade
      0002_runs_token_usage.py  runs.token_usage_by_model 幂等加列
```

关键数字速查(面试时被问"具体数值"直接答):

| 数字 | 含义 | 出处 |
|---|---|---|
| 30s | `PRAGMA busy_timeout=30000`,生产引擎和 alembic 自建引擎都设 | engine.py:130 / env.py:91 |
| 5s | Python sqlite3 驱动默认 busy_timeout,不够跨进程 bootstrap 用 | engine.py:115-116 |
| 9 张 | `_BASELINE_TABLE_NAMES` 基线表白名单 | bootstrap.py:128-140 |
| 4 张 | LangGraph 自有表,`include_object` 排除 | _env_filters.py:14-21 |
| 2 个 | 当前 versions/ 下的 revision 数(0001 基线 + 0002 加列) | versions/ |
| 5 | Postgres `pool_size` 默认值 | engine.py:63 |
| 1-10 min | 托管 PG `idle_in_transaction_session_timeout` 常见默认 | bootstrap.py:334-335 |
| ±2^63 | `JsonMatch` int 值的 signed 64-bit 合法区间 | json_compat.py:24-25 |
| 0x0DEE12F10BEE3682 | PG advisory lock 固定 key,改动等于释放旧锁 | bootstrap.py:110-113 |

---

## 问题链 1:三分支决策状态机

**Q1.1(基础)** Gateway 启动时怎么决定数据库该走哪条初始化路径?讲讲这个状态机。

**参考回答**:入口是 `bootstrap_schema(engine, backend)`,先反射 DB 状态再分发到三个分支:empty(没有任何 DeerFlow 表)→ `create_all` + `alembic stamp head`;legacy(有 DeerFlow 表但没有 `alembic_version` 表)→ 受限 `create_all` 回填 + `stamp 0001_baseline` + `upgrade head`;versioned(已有 `alembic_version` 行)→ 直接 `alembic upgrade head`。分发逻辑在 [_decide_state](../backend/packages/harness/deerflow/persistence/bootstrap.py#L252-L267),顶层编排在 [bootstrap.py:399-452](../backend/packages/harness/deerflow/persistence/bootstrap.py#L399-L452)。设计动机写在模块 docstring:`create_all` 保留为 empty 快路径,因为它能忠实地按方言渲染 JSON vs JSONB、server default、索引名;而 **baseline 之后的一切变更必须属于某个 revision**([bootstrap.py:5-11](../backend/packages/harness/deerflow/persistence/bootstrap.py#L5-L11))。

**链路解析**:

```
Gateway lifespan
      |
      v
init_engine(backend, url)                      [engine.py:58]
      |
      v
bootstrap_schema(engine, backend=backend)      [bootstrap.py:399]
      |
      +-- _bootstrap_lock (PG advisory / SQLite asyncio.Lock)
      |
      v
_reflect_state: {has_alembic_version,          [bootstrap.py:225]
                 has_deerflow_tables}
      |
      v
_decide_state                                  [bootstrap.py:252]
      |-- empty     -> create_all + stamp head
      |-- legacy    -> create_all(baseline 9 表) + stamp 0001 + upgrade head
      +-- versioned -> upgrade head (已是 head 则 alembic 语义上 no-op)
      |
      v
logger.info("bootstrap: complete (backend=%s)")   [bootstrap.py:452]
```

**Q1.2(深挖)** `_reflect_state` 具体怎么判定"有 DeerFlow 表"?如果硬编码表名会有什么问题?

**参考回答**:它用 `reflected ∩ Base.metadata.tables` 求交集——反射出 DB 里所有表名,和 ORM 元数据里注册的表名取交,非空即"有 DeerFlow 表",见 [bootstrap.py:243-249](../backend/packages/harness/deerflow/persistence/bootstrap.py#L243-L249)。这样设计的好处是**bootstrap 层永不硬编码任何表名/列名**:新增一个 ORM 模型只改 `Base.metadata`,判定逻辑零改动。两个细节:① 判定前先 `import deerflow.persistence.models` 强制注册全部 9 个模型类([bootstrap.py:238-241](../backend/packages/harness/deerflow/persistence/bootstrap.py#L238-L241),注册清单见 [models/__init__.py:17-39](../backend/packages/harness/deerflow/persistence/models/__init__.py#L17-L39)),否则未 import 的子模块不会出现在 metadata 里,交集会漏表;② import 失败只降级为 `logger.debug` 而不是炸掉,因为 Alembic-only 场景(裸 `alembic` CLI)可能没有完整应用环境。若硬编码表名(比如判断 `runs` 表存在),某个只建了 `users` 的部署就会被误判成 empty,stamp head 跳过本应执行的中间 revision,留下静默漂移。

**Q1.3(边界/异常)** 两个方向的状态污染各自怎么路由:(a) 库里只有 LangGraph 的 4 张 checkpointer 表;(b) `alembic_version` 在,但业务表被人误删了一半?

**参考回答**:(a) 走 empty 分支。`has_deerflow_tables` 是"反射表 ∩ DeerFlow metadata"的交集,LangGraph 的 `checkpoints`/`checkpoint_blobs`/`checkpoint_writes`/`checkpoint_migrations` 不在 `Base.metadata` 里,交集为空;`_decide_state` 注释里明确写了这个 case:"a DB containing only tables we don't own (e.g. LangGraph's checkpointer tables on a fresh deployment)"([bootstrap.py:263-266](../backend/packages/harness/deerflow/persistence/bootstrap.py#L263-L266))。判断标准是"有没有**我的**表",不是"库是不是空"——因为库是共享的。(b) `has_alembic_version` 优先级最高,直接返回 `"versioned"`([bootstrap.py:260-261](../backend/packages/harness/deerflow/persistence/bootstrap.py#L260-L261)),只跑 `upgrade head`;而幂等 helper 遇到缺表会**静默跳过**(`safe_add_column` 第一行就是 `if table not in insp.get_table_names(): return`,见 [_helpers.py:169-171](../backend/packages/harness/deerflow/persistence/migrations/_helpers.py#L169-L171)),升级"成功"但表依然缺失,运行时第一个 SELECT 才报 `no such table`。这是刻意取舍:bootstrap 信任 `alembic_version` 作为单一事实源,不做启动时全量 schema diff(成本与所有权边界考虑,见 Q2.3)。

---

## 问题链 2:Legacy 分支的顺序之谜

**Q2.1(基础)** 对一个"alembic 引入之前就在跑"的老库,迁移步骤的顺序是什么?每一步做什么?

**参考回答**:三步,顺序固定:① `_run_baseline_create_all_sync`——只对 `_BASELINE_TABLE_NAMES` 里的 **9 张表**做 `checkfirst=True` 的 `create_all` 回填;② `alembic stamp 0001_baseline`——写入版本行,告诉 alembic"基线已经在了";③ `alembic upgrade head`——跑基线之后的所有 revision(当前就是 `0002_runs_token_usage`)。代码在 [bootstrap.py:424-443](../backend/packages/harness/deerflow/persistence/bootstrap.py#L424-L443),回填实现本身在 [bootstrap.py:283-300](../backend/packages/harness/deerflow/persistence/bootstrap.py#L283-L300)。

**链路解析**:

```
legacy DB (有 runs/users 等, 无 alembic_version)
      |
      v
(1) create_all(tables=_BASELINE_TABLE_NAMES, checkfirst=True)
      |     只补"基线时代缺失的表", 已存在的跳过
      v
(2) alembic stamp 0001_baseline
      |     写入 alembic_version='0001_baseline'
      |     => 后续 upgrade 会跳过 0001 自身的 create_table DDL
      v
(3) alembic upgrade head
      |     0002: safe_add_column(runs.token_usage_by_model)
      |     列已在 => no-op + 漂移检查; 列不在 => batch_alter 加列
      v
schema == 全新库的 schema
(test_token_usage_column_parity / test_create_all_and_alembic_upgrade_
 produce_same_schema 两个测试专门验证这个等价性)

对照另外两条分支:
empty     -> create_all(全量 metadata) + stamp head  (0001/0002 的 upgrade 都不跑)
versioned -> upgrade head                            (at head 时 alembic 语义 no-op)
```

**Q2.2(深挖)** 为什么第一步的 `create_all` 必须存在?直接 `stamp 0001` + `upgrade head` 不行吗?

**参考回答**:不行,这是整个模块最经典的一道坑题。`stamp 0001_baseline` 之后,alembic 认为基线已应用,**后续 upgrade 只执行严格位于 stamp 点之后的 revision**——`0001_baseline.upgrade()` 里的 `create_table` DDL 永远不会跑(0001 自己的 docstring 确认了这点,[0001_baseline.py:17-19](../backend/packages/harness/deerflow/persistence/migrations/versions/0001_baseline.py#L17-L19))。于是问题来了:用户的库是 alembic 引入前建的,此后 DeerFlow 又往 `Base.metadata` 里加过**新的基线表**——docstring 里的真实案例是 PR #1930 的 `channel_*` 系列表([bootstrap.py:25-28](../backend/packages/harness/deerflow/persistence/bootstrap.py#L25-L28))。跨多个版本升级的用户库里根本没有这些表,而它们属于基线、不属于任何后续 revision,所以没有任何 DDL 会建它们,第一个打到这些表的请求直接 500 `no such table`。先跑受限 `create_all` 就是把这些"后加的基线表"回填上。也就是说:**stamp 是"声明过去",create_all 是"补齐过去"**,两者缺一不可。

**Q2.3(反例 + 深挖)** 回填为什么要用 `_BASELINE_TABLE_NAMES` 白名单限制?全量 `create_all` 不是更简单?这个手维护的常量怎么保证不和 0001 漂移?

**参考回答**:**不这样设计会怎样**:全量 `create_all` 会把"未来 revision 才引入的表"也提前建出来。假设 revision 0003 有 `op.create_table("new_table")`,而 `create_all` 已经在回填阶段建了 `new_table`,upgrade 走到 0003 时就 `relation already exists` 直接炸掉——docstring 原文就是这个推理([bootstrap.py:29-33](../backend/packages/harness/deerflow/persistence/bootstrap.py#L29-L33),实现处注释 [bootstrap.py:430-439](../backend/packages/harness/deerflow/persistence/bootstrap.py#L430-L439))。列级变更安全(有 `safe_add_column` 幂等),但**目前没有 `safe_create_table`**,所以表级安全性被刻意留在 bootstrap 层,用白名单解决,而不是推给每个未来 revision 自己处理([bootstrap.py:117-123](../backend/packages/harness/deerflow/persistence/bootstrap.py#L117-L123))。防漂移靠守卫测试 pinning:常量固定在 [bootstrap.py:128-140](../backend/packages/harness/deerflow/persistence/bootstrap.py#L128-L140)(4 张 `channel_*` + `feedback`/`run_events`/`runs`/`threads_meta`/`users`);`test_baseline_table_names_constant_matches_0001` 会真的在临时库上执行 `0001_baseline.upgrade()`,反射出它实际创建的表集合,和常量做双向 diff,任何一边漂移都失败并报出 `only-in-0001` / `only-in-constant` 两个差集。`_BASELINE_REVISION = "0001_baseline"` 这个字符串([bootstrap.py:107](../backend/packages/harness/deerflow/persistence/bootstrap.py#L107))同样有测试断言它是 script tree 里真实存在的 revision id——防止重命名 revision 文件后 stamp 在运行时才炸。这是"常量 + 可执行规范互相钉死"的模式。

---

## 问题链 3:并发安全 —— 两种后端的两种锁

**Q3.1(基础)** 两个 Gateway 实例同时启动、同时跑 bootstrap,会发生什么?怎么防?

**参考回答**:分后端,整体是三层防线(docstring 总述 [bootstrap.py:44-75](../backend/packages/harness/deerflow/persistence/bootstrap.py#L44-L75))。Postgres 用 **session 级 advisory lock**,key 是固定常量 `0x0DEE_12F1_0BEE_3682`——注释强调这两个随机 32-bit 半字是一次性选定的,**改 key 等于释放旧锁**,要改必须协调一次性迁移([bootstrap.py:109-113](../backend/packages/harness/deerflow/persistence/bootstrap.py#L109-L113))。整个"反射+建表+stamp+upgrade"序列都在锁内,第二个实例排队,拿到锁后观察到 head 直接 no-op——真跨进程串行化。SQLite 没有 advisory lock,用**每引擎一把 `asyncio.Lock`** 做单进程串行([bootstrap.py:360-383](../backend/packages/harness/deerflow/persistence/bootstrap.py#L360-L383)),跨进程退化为"SQLite 文件写锁 + `PRAGMA busy_timeout=30000`(**30 秒**)"的 best-effort 等待。第三层兜底是幂等 revision:即使锁被绕过,重跑也安全。锁选择在 [_bootstrap_lock](../backend/packages/harness/deerflow/persistence/bootstrap.py#L386-L391),不支持的 backend 直接 `ValueError`。另外 alembic 自身语义也帮忙:`upgrade head` 在已 at head 的库上是 no-op,所以第 2 到第 N 个 actor 只是观察一下 head 就退出([bootstrap.py:74-75](../backend/packages/harness/deerflow/persistence/bootstrap.py#L74-L75))。

**链路解析**:

```
Postgres:                          SQLite:
gateway A ──pg_advisory_lock──┐    task A ──asyncio.Lock(per engine)──┐
gateway B ──排队等待──────────┤    task B ──排队(同进程)─────────────┤
                              v                                       v
                    reflect -> decide -> act                  reflect -> decide -> act
                              |                                       |
                    unlock(显式+session断开兜底)            跨进程: file lock + busy_timeout=30s
                                                                    |
                                                              超时仍失败? -> 幂等 revision 兜底
```

**Q3.2(深挖)** Postgres 为什么用 session 级而不是 transaction 级 advisory lock?`SET LOCAL idle_in_transaction_session_timeout = 0` 是干什么的?unlock 失败会泄漏锁吗?

**参考回答**:三个递进点。① 锁必须**活得比 alembic 的内部事务久**:`stamp`/`upgrade` 自己开事务,用 `pg_advisory_xact_lock` 会在事务提交时提前放锁,DDL 还没跑完,所以选 session 级([bootstrap.py:319-325](../backend/packages/harness/deerflow/persistence/bootstrap.py#L319-L325))。② 更阴险的坑:持锁连接 `engine.connect()` 后第一次 `execute` 就 auto-begin 了一个事务,然后它就**闲在那里**——真正的 DDL 跑在 `asyncio.to_thread` 里 alembic 自己开的另一条连接上。托管 PG(RDS/Cloud SQL/Supabase)默认 `idle_in_transaction_session_timeout` 为 **1-10 分钟**,alembic 跑得比这久,宿主会杀掉这个 idle-in-transaction 会话;而 advisory lock 是 session 作用域,**锁被静默释放**,第二个 Gateway 立刻拿锁并发跑 DDL,锁完全失效。防御:拿锁之前先 `SET LOCAL idle_in_transaction_session_timeout = 0`,只对本事务生效,自建 PG 默认关闭所以是无害 no-op([bootstrap.py:327-349](../backend/packages/harness/deerflow/persistence/bootstrap.py#L327-L349))。③ 不会泄漏:`finally` 里对 unlock 全捕获异常,失败只 warning,并依赖"session 断开时 Postgres 自动释放其全部 advisory lock"的服务端语义兜底——进程崩溃、`kill -9`、网络分区都一样([bootstrap.py:353-357](../backend/packages/harness/deerflow/persistence/bootstrap.py#L353-L357))。对比 Redis 分布式锁还要自己管租约续期,这里把生命周期外包给了 PG 会话管理。

**Q3.3(深挖 + 边界)** SQLite 的锁为什么做成 per-engine 的 `WeakKeyDictionary`?用 `id(engine)` 当 key 行不行?跨进程为什么不用 `BEGIN IMMEDIATE` 或 OS 文件锁?

**参考回答**:锁存储的三层理由全在 [bootstrap.py:143-157](../backend/packages/harness/deerflow/persistence/bootstrap.py#L143-L157) 注释里:① **per-engine 而非模块全局**——`asyncio.Lock` 绑定到它第一次见到的 event loop,pytest 给每个 async 测试一个独立 loop,全局锁被跨 loop 复用直接 `RuntimeError`;引擎和锁一一配对,生产上一个进程一个引擎,dict 实际只有一项。② **不能用 `id(engine)`**——CPython 回收内存地址,死引擎的 `id -> Lock` 残留可能被复用同地址的新引擎拿到,而旧锁绑在死引擎的 loop 上,`async with` 抛 "bound to a different event loop"。③ **WeakKeyDictionary**——以引擎对象为 key,GC 后条目自动消失,dict 大小永不超过活引擎数。获取逻辑在 [bootstrap.py:160-165](../backend/packages/harness/deerflow/persistence/bootstrap.py#L160-L165)。跨进程方案两个都被明确否决([bootstrap.py:365-374](../backend/packages/harness/deerflow/persistence/bootstrap.py#L365-L374)):`BEGIN IMMEDIATE` 会让 sentinel 连接持有写锁,alembic 内部自己开的连接**和我们死锁**(它等我们的写锁,我们等它跑完);OS 文件锁可行但要为"多进程 SQLite"这种官方就不鼓励的部署形态引入平台相关 `fcntl`/`msvcrt` 硬依赖。**不这样设计会怎样**的反面取舍:接受 best-effort——30 秒 `busy_timeout` 覆盖现实场景,病态重叠下 30 秒后仍可能 `database is locked`,此时幂等 revision 保证重试正确;真多实例就该用 Postgres,这是部署约束而非代码缺陷。

---

## 问题链 4:LangGraph 表的隔离

**Q4.1(基础)** 这个库里还有 LangGraph 的 checkpointer 表,Alembic 怎么做到不管它们?

**参考回答**:靠 `include_object` 过滤器。`env.py` 在 `context.configure` 时传入 `include_object=include_object`,offline 和 online 两条路都传([env.py:52-58](../backend/packages/harness/deerflow/persistence/migrations/env.py#L52-L58) 和 [env.py:63-69](../backend/packages/harness/deerflow/persistence/migrations/env.py#L63-L69))。过滤器对两类对象返回 False:名字在 `LANGGRAPH_OWNED_TABLES` 里的表,以及父表在其中的索引/约束;4 张表名固定在 [_env_filters.py:14-21](../backend/packages/harness/deerflow/persistence/migrations/_env_filters.py#L14-L21),实现本体在 [_env_filters.py:24-36](../backend/packages/harness/deerflow/persistence/migrations/_env_filters.py#L24-L36)。注意这个集合和 bootstrap 的判定逻辑是**两份独立机制**:bootstrap 靠 metadata 交集天然忽略它们(Q1.3),autogenerate 靠这个显式黑名单。另外 offline 模式(`run_migrations_offline`,`literal_binds=True` 把值渲染成 SQL 字面量,见 [env.py:50-60](../backend/packages/harness/deerflow/persistence/migrations/env.py#L50-L60))同样过这个过滤器——过滤是 configure 层的,与运行模式无关,所以 `alembic upgrade head --sql` 生成的离线脚本也不会出现 LangGraph 表。

**链路解析**:

```
alembic revision --autogenerate
      |
      v
reflect DB schema  <────────  checkpoints / checkpoint_blobs /
      |                        checkpoint_writes / checkpoint_migrations
      v
include_object(obj, name, type_, reflected, compare_to)
      |-- type_=='table' and name in LANGGRAPH_OWNED_TABLES -> False
      |-- obj.table.name in LANGGRAPH_OWNED_TABLES          -> False (索引/约束)
      +-- otherwise                                         -> True
      |
      v
只对 DeerFlow 9 张表生成 diff
```

**Q4.2(深挖)** 不加这个过滤器会怎样?这只是个"洁癖"问题吗?

**参考回答**:**不这样设计会怎样**:不是洁癖,是数据丢失级事故。autogenerate 的工作方式是"反射 DB vs metadata 求 diff"——LangGraph 的表在 DB 里但不在 `Base.metadata` 里,autogenerate 会把它们解释为"metadata 删除了这些表",于是**每个新 revision 都自动生成 `op.drop_table("checkpoints")` 等 DDL**,一旦有人不看直接 upgrade,所有会话 checkpoint 被删光。模块 docstring 原话就是 "would emit spurious `drop_table` ops every revision"([_env_filters.py:4-6](../backend/packages/harness/deerflow/persistence/migrations/_env_filters.py#L4-L6)),`env.py` 头部的责任划分也写明这些表 "have their own schema lifecycle and must not be touched"([env.py:6-12](../backend/packages/harness/deerflow/persistence/migrations/env.py#L6-L12))。本质是**共享数据库里的所有权边界**:同一个 DB 里住了两个"房东"(DeerFlow 的 9 张表 + LangGraph 的 4 张表),Alembic 只对自己名下的财产出 diff。回答时主动把"数据丢失"说出来,比只说"会产生多余 DDL"更能体现你理解后果的严重性等级。

**Q4.3(深挖)** 过滤器为什么单独放 `_env_filters.py` 而不是内联进 `env.py`?索引为什么要靠 `object_.table` 再过滤一层?

**参考回答**:① 单测性:`env.py` 一 import 就拉起 alembic 的 `context.config`、fileConfig 等 import-time 机制([env.py:43-45](../backend/packages/harness/deerflow/persistence/migrations/env.py#L43-L45)),把纯函数抽到独立模块可以不拖 alembic 环境直接单测([_env_filters.py:7-9](../backend/packages/harness/deerflow/persistence/migrations/_env_filters.py#L7-L9));`env.py` 再用 `__all__` re-export,兼容以 `env.LANGGRAPH_OWNED_TABLES` 寻址的旧调用方([env.py:30-32](../backend/packages/harness/deerflow/persistence/migrations/env.py#L30-L32))。② 索引/约束对象自身的 `name` 不是表名,`name in LANGGRAPH_OWNED_TABLES` 对它们不命中,必须回查 `object_.table.name`——否则出现"表被排除了、但它上面的索引还被 autogenerate 盯上"的漏网之鱼,实现见 [_env_filters.py:31-36](../backend/packages/harness/deerflow/persistence/migrations/_env_filters.py#L31-L36)。函数签名严格匹配 alembic 的 `include_object` callable 契约 `(object, name, type_, reflected, compare_to)`,未用的参数用 `# noqa: ARG001` 压住 lint。

---

## 问题链 5:幂等列迁移与形状漂移

**Q5.1(基础)** `0002_runs_token_usage` 这个 revision 为什么不直接 `op.add_column`,而要用 `safe_add_column`?

**参考回答**:为了幂等。背景是 issue #3682:e7a03e52 之前建的库缺 `runs.token_usage_by_model` 列,所有 SELECT `runs` 的端点都会 `no such column`([0002_runs_token_usage.py:7-10](../backend/packages/harness/deerflow/persistence/migrations/versions/0002_runs_token_usage.py#L7-L10))。但有两类库"列已经在了":① 用户按 issue 里的 workaround 手工 `ALTER TABLE ... ADD COLUMN` 过;② 并发 bootstrap 锁被绕过的重试场景。裸 `op.add_column` 对这两类库直接 `duplicate column` 炸掉;`safe_add_column` 先反射,列在就 no-op,不在才走 `batch_alter_table` 加列([_helpers.py:159-177](../backend/packages/harness/deerflow/persistence/migrations/_helpers.py#L159-L177))。`_helpers.py` 的模块 docstring 把动机归纳成两条:一是 bootstrap 锁之上的 **defence-in-depth**(手工 ALTER、配置错误、SQLite 跨进程竞争都可能绕过锁),二是与 `create_all` "跳过已存在表"的宽容姿态对齐——列迁移也该跳过已在目标状态的列([_helpers.py:1-17](../backend/packages/harness/deerflow/persistence/migrations/_helpers.py#L1-L17))。revision 本体只有这一个调用([0002_runs_token_usage.py:56-65](../backend/packages/harness/deerflow/persistence/migrations/versions/0002_runs_token_usage.py#L56-L65)),downgrade 对称地用 `safe_drop_column`([0002_runs_token_usage.py:68-69](../backend/packages/harness/deerflow/persistence/migrations/versions/0002_runs_token_usage.py#L68-L69))。

**链路解析**:

```
upgrade(): safe_add_column("runs", Column("token_usage_by_model", JSON,
                                          nullable=False, server_default='{}'))
      |
      v
inspector: table 存在?  --否--> return (静默; 只支持已有基线表的 legacy 库)
      |是
      v
column 已存在?  --是--> _check_column_drift(nullable/server_default/type)
      |                      |-- 一致: 静默 no-op
      |                      +-- 漂移: logger.warning, 不自动修
      |否
      v
op.batch_alter_table -> add_column (SQLite 也安全)
```

**Q5.2(深挖)** 列已存在时的 "drift warning" 检测什么?为什么 `JSON` vs `JSONB` 不算漂移、`TEXT` vs `JSON` 算?检测到漂移为什么不自动修?

**参考回答**:`_check_column_drift` 比对三个维度:`nullable`、`server_default`(经 `_normalize_default` 归一化:去外层括号、去 Postgres 的 `'{}'::jsonb` 类型 cast,归一化实现 [_helpers.py:51-75](../backend/packages/harness/deerflow/persistence/migrations/_helpers.py#L51-L75))、`type`(经 `_type_equivalent`)([_helpers.py:123-156](../backend/packages/harness/deerflow/persistence/migrations/_helpers.py#L123-L156))。类型比较先归一化成大写类名、去参数(`VARCHAR(255)`→`VARCHAR`,见 [_helpers.py:78-91](../backend/packages/harness/deerflow/persistence/migrations/_helpers.py#L78-L91)),再查等价族白名单 `_EQUIVALENT_TYPE_FAMILIES = (frozenset({"JSON", "JSONB"}),)`([_helpers.py:104](../backend/packages/harness/deerflow/persistence/migrations/_helpers.py#L104))——Postgres 上 `sa.JSON` 建的列反射回来可能是 `JSONB`,方言同义词不该报;而 #3682 的真实事故是有人手工 `ALTER TABLE ... ADD COLUMN token_usage_by_model TEXT NOT NULL DEFAULT '{}'`,`TEXT` 对 `JSON` 是批发级类型错误必须报。两个防误报设计:反射侧信息缺失时 `_type_equivalent` 直接返回 True,不给"没有数据"的维度报警([_helpers.py:107-120](../backend/packages/harness/deerflow/persistence/migrations/_helpers.py#L107-L120));白名单注释规定只有真实部署证明的误报才能加新等价对,不能预防性添加,否则重新打开静默漂移的洞([_helpers.py:94-103](../backend/packages/harness/deerflow/persistence/migrations/_helpers.py#L94-L103))。不自动修是刻意保守:docstring 明说 "We do not auto-repair -- a warning is enough for operators to notice and decide"([_helpers.py:33-34](../backend/packages/harness/deerflow/persistence/migrations/_helpers.py#L33-L34))——`TEXT`→`JSON` 的自动 ALTER 撞上存量非法 JSON 字符串会当场失败或静默截断,`nullable` 收紧会撞存量 NULL 行,这些决策必须人看着数据做。warning 里 reflected 和 desired 两侧 type repr 都打出来,方便运维一眼定位([_helpers.py:148-156](../backend/packages/harness/deerflow/persistence/migrations/_helpers.py#L148-L156))。

**Q5.3(边界/异常)** `ALTER TABLE runs ADD COLUMN ... NOT NULL` 在一张有几百万行的老表上为什么能成功?SQLite 上跑 ALTER 有什么特殊处理?

**参考回答**:① 能成功靠 `server_default=sa.text("'{}'")`:带 server default 的 `ADD COLUMN ... NOT NULL`,存量行在 ALTER 时刻直接取默认值 `'{}'`,不触发 NOT NULL 冲突——revision docstring 专门解释了这一点([0002_runs_token_usage.py:22-26](../backend/packages/harness/deerflow/persistence/migrations/versions/0002_runs_token_usage.py#L22-L26)),ORM 侧声明在 [run/model.py:42](../backend/packages/harness/deerflow/persistence/run/model.py#L42)。这也解释了为什么列要带 server_default 而不只是 Python 侧 `default=dict`:Python 默认值只在 INSERT 时生效,救不了 ALTER。② SQLite 的 `ALTER TABLE` 能力极弱(原生基本只支持 `ADD COLUMN`/`RENAME`),所以 alembic 全程用 `render_as_batch=True` + `op.batch_alter_table`,把变更翻译成"建新表-拷数据-换名"的 batch 模式;`env.py` 里 online/offline 两处 configure 都开了 `render_as_batch`([env.py:56](../backend/packages/harness/deerflow/persistence/migrations/env.py#L56) 和 [env.py:67](../backend/packages/harness/deerflow/persistence/migrations/env.py#L67)),helper 里也是 `batch.add_column` 而非裸 `op.add_column`([_helpers.py:176-177](../backend/packages/harness/deerflow/persistence/migrations/_helpers.py#L176-L177)),0001 里建索引同样全部走 batch([0001_baseline.py:73-77](../backend/packages/harness/deerflow/persistence/migrations/versions/0001_baseline.py#L73-L77))。

**Q5.4(深挖)** downgrade 路径也做了幂等吗?为什么 0001 要写一份"几乎不会执行"的完整 `downgrade()`?

**参考回答**:做了。`0002` 的 downgrade 用 `safe_drop_column`,同样先反射、表或列不在就 no-op,在才走 `batch_alter_table` 删列([_helpers.py:180-189](../backend/packages/harness/deerflow/persistence/migrations/_helpers.py#L180-L189) 和 [0002_runs_token_usage.py:68-69](../backend/packages/harness/deerflow/persistence/migrations/versions/0002_runs_token_usage.py#L68-L69))。`0001_baseline` 则写满了 9 张表的 `drop_table` + `drop_index` 对称 DDL([0001_baseline.py:240-291](../backend/packages/harness/deerflow/persistence/migrations/versions/0001_baseline.py#L240-L291)),尽管它的 docstring 明说生产路径上这个 `upgrade()` 都"几乎不会执行"——empty 分支 stamp head、legacy 分支 stamp 0001,都会跳过它([0001_baseline.py:10-22](../backend/packages/harness/deerflow/persistence/migrations/versions/0001_baseline.py#L10-L22))。写全的原因是让 0001 充当**可执行的基线规范**:① 测试 fixture 里 `alembic upgrade base -> head` 能完整 round-trip;② 守卫测试(见 Q2.3)正是靠真的执行 `0001.upgrade()` 再反射,拿到"基线到底建哪些表"的 ground truth 去 pin `_BASELINE_TABLE_NAMES`。降级到对称的 downgrade 是这个规范的另一半。换句话说:baseline revision 的首要身份是 **stamp target + chain root + 可执行文档**,DDL 被生产执行只是附带场景。

---

## 问题链 6:SQLite vs Postgres 的 JSON 方言差异

**Q6.1(基础)** `json_compat.py` 解决什么问题?为什么不能用 SQLAlchemy 自带的 JSON 操作符?

**参考回答**:解决"对 JSON 列做 `column[key] == value` 过滤"的方言可移植性与类型安全。内置操作符会踩一串坑:SQLite `json_type` 返回 `'integer'/'real'/'text'/'true'/'false'/'null'`;Postgres `json_typeof` 对 int 和 float **都返回 `'number'`** 不区分,而 `->>` 永远返回 text,比 int 必须 CAST,直接对 float 值 CAST BIGINT 会报错;NULL 和"键不存在"也要区分。`JsonMatch` 是自定义 `ColumnElement`,按方言编译:SQLite 走 `json_type`/`json_extract`,Postgres 走 `json_typeof`/`->>`([json_compat.py:60-69](../backend/packages/harness/deerflow/persistence/json_compat.py#L60-L69))。差异集中在 `_Dialect` 配置表:SQLite `num_types=('integer','real')`、`int_guard=None`;PG `num_types=('number',)`、`int_guard='^-?[0-9]+$'`——先用正则挡住 float 再 CAST,`CASE WHEN ... THEN CAST(...) END` 的结构保证不匹配时返回 NULL 而不是报错([json_compat.py:111-131](../backend/packages/harness/deerflow/persistence/json_compat.py#L111-L131),PG 分支生成逻辑 [json_compat.py:157-160](../backend/packages/harness/deerflow/persistence/json_compat.py#L157-L160))。

**链路解析**:

```
json_match(RunRow.metadata_json, "key", 42)
      |
      v
JsonMatch(column, key, value)  -- 构造时校验 key/value 合法性
      |
      +-- sqlite:  json_type(col,'$."key"') IN ('integer')
      |            AND CAST(json_extract(...) AS INTEGER) = :p1
      |
      +-- postgres: CASE WHEN json_typeof(col->'key')='number'
      |               AND (col->>'key') ~ '^-?[0-9]+$'
      |               THEN CAST(col->>'key' AS BIGINT) END = :p1
      |
      +-- 其他方言: NotImplementedError (显式拒绝, 不是静默错译)
                    [json_compat.py:189-191]

值分派 (_build_clause, json_compat.py:146-165):
  None  -> typeof = 'null'                        (区分 NULL 与键缺失)
  bool  -> sqlite: typeof='true'/'false'
           pg:     typeof='boolean' AND extract = 'true'/'false'
  int   -> 类型守卫 + CAST AS INTEGER/BIGINT + bindparam(BigInteger)
  float -> num_types 守卫 + CAST AS REAL/DOUBLE PRECISION + bindparam(Float)
  str   -> typeof='text'/'string' AND extract = bindparam(String)
```

**Q6.2(深挖)** key 和 value 的校验规则是什么?int 为什么要限制在 signed 64-bit?bool 的检查顺序为什么重要?

**参考回答**:key 必须匹配 `^[A-Za-z0-9_\-]+$`([json_compat.py:17](../backend/packages/harness/deerflow/persistence/json_compat.py#L17)),因为 key 是**插值进编译后 SQL 文本**的(JSONPath `$."key"` / `->` 字面量),不走 bind param,放开字符集就是 SQL/JSONPath 注入面;两个编译函数入口还会再校验一次兜底([json_compat.py:170-171](../backend/packages/harness/deerflow/persistence/json_compat.py#L170-L171))。value 只允许 `None/bool/int/float/str` 五种([json_compat.py:20](../backend/packages/harness/deerflow/persistence/json_compat.py#L20)),list/dict/bytes 刻意拒绝而不是 `str()` 静默强转——静默强转会 (a) 匹配结果错,(b) value 不可哈希时破坏 SQLAlchemy `inherit_cache` 不变量([json_compat.py:39-47](../backend/packages/harness/deerflow/persistence/json_compat.py#L39-L47))。int 限制 `[-2**63, 2**63 - 1]`([json_compat.py:24-25](../backend/packages/harness/deerflow/persistence/json_compat.py#L24-L25)):SQLite 驱动绑定超范围 Python int 直接 OverflowError,Postgres 在 `CAST AS BIGINT` 时溢出,所以在**校验期**前置成明确的 `TypeError`,报错信息里直接给出合法区间([json_compat.py:84-87](../backend/packages/harness/deerflow/persistence/json_compat.py#L84-L87))——fail fast at the API boundary。bool 必须先于 int 检查,因为 Python 里 `bool` 是 `int` 子类,`True` 走错分支会被当 1 处理([json_compat.py:150-151](../backend/packages/harness/deerflow/persistence/json_compat.py#L150-L151));而数值绑定统一走 `bindparam` + 显式类型(BigInteger/Float/String),值本身永远参数化([json_compat.py:134-136](../backend/packages/harness/deerflow/persistence/json_compat.py#L134-L136))。

**Q6.3(深挖)** `inherit_cache = True` 和 `_traverse_internals` 是干什么的?写错了会怎样?

**参考回答**:这是 SQLAlchemy 自定义 `ColumnElement` 参与查询编译缓存的契约。`inherit_cache = True` 声明"此元素编译结果可缓存",SQLAlchemy 用 `_traverse_internals` 声明的字段计算缓存 key:column 按 `dp_clauseelement`、key 按 `dp_string`、value 按 `dp_plain_obj`([json_compat.py:71-79](../backend/packages/harness/deerflow/persistence/json_compat.py#L71-L79))。如果漏声明 value 或标错 traversal 类型,两个不同 value 的过滤可能命中同一条编译缓存——**A 请求的过滤条件用到 B 请求的 SQL 上**,这是正确性事故不是性能问题。这也是为什么 value 必须是可哈希标量,和 Q6.2 的校验形成闭环。自定义表达式三块拼图——遍历声明、缓存标志、按方言 `@compiles`(sqlite/postgresql 各一个,默认实现 raise `NotImplementedError`,[json_compat.py:189-191](../backend/packages/harness/deerflow/persistence/json_compat.py#L189-L191))——缺一就退化为"每次重编译"或"错误缓存";手写 SQL 拼字符串则完全没有这层保护,这也是为什么值类型校验要顺带守护缓存不变量。

---

## 问题链 7:Engine 初始化、PRAGMA 与环境差异

**Q7.1(基础)** `init_engine` 在 SQLite 模式下给每条连接设置了哪些 PRAGMA?为什么用 event listener 而不是启动时执行一次?

**参考回答**:四条,挂在 `connect` 事件上([engine.py:123-132](../backend/packages/harness/deerflow/persistence/engine.py#L123-L132)):`journal_mode=WAL`(读写并发不互斥,生产级 SQLite 标配)、`synchronous=NORMAL`(WAL 的安全-速度配对,只在 checkpoint 边界 fsync)、`foreign_keys=ON`(SQLite 默认**关**外键!)、`busy_timeout=30000`(30 秒等文件锁)。必须每条连接重放的原因:SQLite 的 PRAGMA 是**连接级**状态,连接池里每条新建 DBAPI 连接都是全新的,启动时跑一次只影响当时那一条,池里后续 checkout 的连接全部回到默认值——`foreign_keys` 悄悄关掉、`busy_timeout` 回到 Python sqlite3 驱动默认的 **5 秒**。注释原话:"SQLite PRAGMA settings are per-connection, so we wire the listener instead of running PRAGMA once at startup"([engine.py:108-111](../backend/packages/harness/deerflow/persistence/engine.py#L108-L111));5s 对行级瞬时竞争够用,但跨进程 bootstrap 时第二个 Gateway 要等第一个跑完 `CREATE TABLE`/`ALTER TABLE`,所以加宽到 30s([engine.py:115-122](../backend/packages/harness/deerflow/persistence/engine.py#L115-L122))。另外 `json_serializer` 用 `ensure_ascii=False` 的自定义 dumps 保证中文不转义([engine.py:20-22](../backend/packages/harness/deerflow/persistence/engine.py#L20-L22))。

**链路解析**:

```
init_engine("sqlite", url)
   |
   +-- asyncio.to_thread(os.makedirs, sqlite_dir)   # 不阻塞 lifespan 事件循环
   +-- create_async_engine(url, json_serializer=...)
   +-- event.listens_for(sync_engine, "connect")
   |       每条新 DBAPI 连接:
   |         PRAGMA journal_mode=WAL
   |         PRAGMA synchronous=NORMAL
   |         PRAGMA foreign_keys=ON
   |         PRAGMA busy_timeout=30000
   +-- async_sessionmaker(expire_on_commit=False)
   |
   v
bootstrap_schema(_engine, backend="sqlite")

teardown (对称路径):
close_engine() -> engine.dispose() -> 双全局(_engine/_session_factory)置 None
                  [engine.py:197-205]; backend="memory" 时 init_engine 直接 return
```

**Q7.2(深挖)** `migrations/env.py` 里为什么又设了一遍 `busy_timeout=30000`?生产引擎不是设过了吗?

**参考回答**:因为 **alembic 自己 spawn 了一个独立 engine**:`run_migrations_online` 里 `create_async_engine(config.get_main_option("sqlalchemy.url"))` 是全新引擎([env.py:75](../backend/packages/harness/deerflow/persistence/migrations/env.py#L75)),它的连接不从生产引擎的池里借;event listener 挂在引擎对象上,不跨引擎继承。不重复设置的话,alembic 的连接在另一个进程持锁时用默认 5s 超时直接 `database is locked`。注释原话:"alembic spawns its OWN engine here -- those connections wouldn't inherit anything unless we wire the same hook on this one"([env.py:77-93](../backend/packages/harness/deerflow/persistence/migrations/env.py#L77-L93))。**不这样设计会怎样**:生产路径和迁移路径连接行为不一致,出现"应用跑得好好的、启动迁移却锁超时"的灵异故障,而且只在有第二进程竞争时复现,极难排查。这体现一个普遍原则:**连接级配置必须跟着每个建连接的点走**,不能假设有全局生效位置。

**Q7.3(边界/异常)** Postgres 模式下目标数据库本身还不存在,启动会怎样?session factory 和连接池还有哪些关键配置?

**参考回答**:`bootstrap_schema` 抛出的异常字符串里带 `"does not exist"` 时,`init_engine` 捕获后走自动建库 + 重试一次:`_auto_create_postgres_db` 连接同服务器的 `postgres` 维护库,用 `AUTOCOMMIT` 隔离级别执行 `CREATE DATABASE "<db_name>"`(CREATE DATABASE 不能在事务里跑),然后 **dispose 旧引擎、用同样参数重建引擎和 session factory、再跑一次 bootstrap**([engine.py:157-169](../backend/packages/harness/deerflow/persistence/engine.py#L157-L169),建库细节 [engine.py:31-55](../backend/packages/harness/deerflow/persistence/engine.py#L31-L55));重建这步不能省,旧引擎的池缓存着"连不上那个库"的状态。其他关键配置:Postgres `pool_size=5`(默认值,[engine.py:63](../backend/packages/harness/deerflow/persistence/engine.py#L63))+ `pool_pre_ping=True`([engine.py:133-140](../backend/packages/harness/deerflow/persistence/engine.py#L133-L140))——checkout 时先发轻量探测剔除被服务端/NAT 静默断开的死连接,避免"半夜 RDS 重启、早上第一批请求全挂";`async_sessionmaker(_engine, expire_on_commit=False)`([engine.py:144](../backend/packages/harness/deerflow/persistence/engine.py#L144))——关掉 commit 后属性过期,避免"读个属性却触发隐式 SELECT"在 async 下抛 `MissingGreenlet`;`backend == "memory"` 时 engine 干脆不建,`get_session_factory()` 返回 `None`,仓库层自行判空回退内存实现([engine.py:77-79](../backend/packages/harness/deerflow/persistence/engine.py#L77-L79) 和 [engine.py:188-190](../backend/packages/harness/deerflow/persistence/engine.py#L188-L190))。

---

## 问题链 8:Alembic 配置的 in-process 化

**Q8.1(基础)** bootstrap 调 alembic 时 `AlembicConfig` 怎么构造?为什么不读 `alembic.ini`?

**参考回答**:完全 in-process:空 `AlembicConfig()` + 两个 `set_main_option`——`script_location` 锚到包内 `migrations/` 的磁盘绝对路径(`Path(__file__).resolve().parent / "migrations"`,[bootstrap.py:98](../backend/packages/harness/deerflow/persistence/bootstrap.py#L98)),`sqlalchemy.url` 从活引擎渲染——不读任何 ini([_get_alembic_config](../backend/packages/harness/deerflow/persistence/bootstrap.py#L198-L208))。理由:生产运行时不能依赖"工作目录相对路径能找到 alembic.ini"这种脆弱假设;锚包路径后 pip 安装到任何环境都对。head revision 也从 script tree 进程内算出并缓存一次(`_HEAD_REVISION` 全局,[bootstrap.py:101](../backend/packages/harness/deerflow/persistence/bootstrap.py#L101) 和 [bootstrap.py:211-222](../backend/packages/harness/deerflow/persistence/bootstrap.py#L211-L222));`versions/` 为空直接 `RuntimeError` 防爆。缓存会 stale 但被刻意接受:revision 随代码发布,进程生命周期内代码不变,发新版必然重启。ini 在 `env.py` 里只剩配日志一个用途,且判了 `is not None`([env.py:43-45](../backend/packages/harness/deerflow/persistence/migrations/env.py#L43-L45))。

**链路解析**:

```
engine (活的 AsyncEngine, 密码正确)
   |
   v
_alembic_safe_url(engine)
   |-- render_as_string(hide_password=False)   # 坑1: str(url) 会把密码掩成 ***
   |-- replace("%", "%%")                       # 坑2: ConfigParser 的 %(..)s 插值
   v
AlembicConfig.set_main_option("sqlalchemy.url", safe_url)
   |
   v
alembic_command.stamp/upgrade  (同步阻塞 -> asyncio.to_thread 包裹)
   |
   v
env.py 用该 url 自建 engine 跑迁移 (见 Q7.2)
```

**Q8.2(深挖)** URL 传递里有哪两个坑?各是怎么解决的?

**参考回答**:坑一:**密码掩码**。`str(engine.url)` 和不带参数的 `render_as_string()` 把密码渲染成 `***`,alembic 拿这个 URL 自己开连接,凭证是垃圾,运行时才认证失败——而活引擎明明连得好好的,故障极具迷惑性。修法:`render_as_string(hide_password=False)`([bootstrap.py:186-194](../backend/packages/harness/deerflow/persistence/bootstrap.py#L186-L194))。坑二:**ConfigParser 插值**。`set_main_option` 底层走 `ConfigParser.set`,对值做 `%(name)s` 风格插值;URL 编码的密码比如 `p%40ss`(`@`→`%40`)会抛 `InterpolationSyntaxError`。修法:把所有字面 `%` 翻倍成 `%%`,ConfigParser 反转义回单个 `%`([_escape_url_for_alembic](../backend/packages/harness/deerflow/persistence/bootstrap.py#L168-L178));且这个规则和 autogen 脚本共享,round-trip 规则只住在一个地方。这两个坑的共同教训:**把连接信息从"活对象"序列化到"配置文本"再喂给另一个子系统时,每一层的默认行为(掩码、插值)都是地雷**。

**Q8.3(边界/异常)** alembic 的 `stamp`/`upgrade` 是同步阻塞调用,在 async 的 FastAPI lifespan 里直接调会怎样?

**参考回答**:会阻塞事件循环——启动阶段整个进程的所有请求处理都卡住。所以两个同步包装函数 `_stamp`/`_upgrade` 的 docstring 都明确要求调用方用 `asyncio.to_thread` 包裹([bootstrap.py:303-310](../backend/packages/harness/deerflow/persistence/bootstrap.py#L303-L310)),实际调用点如 `await asyncio.to_thread(_stamp, cfg, head)`([bootstrap.py:422](../backend/packages/harness/deerflow/persistence/bootstrap.py#L422))。同样的"同步 IO 不进事件循环"纪律还出现在 SQLite 建目录上:`os.makedirs` 是 stat+mkdir 系统调用,也走 `await asyncio.to_thread(os.makedirs, ...)`([engine.py:101-105](../backend/packages/harness/deerflow/persistence/engine.py#L101-L105)),注释里还引用了 checkpointer 侧的同类修复 #1912。这是全模块一致的纪律:lifespan 里任何同步阻塞调用都必须过 `to_thread`;配套地,`env.py` 自己跑迁移时用 `asyncio.run(run_migrations_online())` 起独立循环([env.py:100-103](../backend/packages/harness/deerflow/persistence/migrations/env.py#L100-L103)),因为它运行在 `to_thread` 拉起的线程里,那里没有事件循环。

---

## 面试官最爱追问的 3 个点

1. **"legacy 分支为什么必须先 create_all 再 stamp?顺序反了/少了会怎样?"** —— 应答策略:一句话锁定因果链:stamp 0001 之后 alembic 只跑 stamp 点之后的 revision,0001 自己的 `create_table` 永远跳过,而"后加的基线表"(channel_* 案例)不属于任何后续 revision,于是永远不会被建,首个请求 500 `no such table`。再补一刀:所以回填必须限制在 `_BASELINE_TABLE_NAMES` 白名单(9 张表)内,否则抢占未来 revision 的表导致 `relation already exists`,且有守卫测试把这个常量和 `0001_baseline.upgrade()` 的实际产出双向 pin 死。

2. **"两个实例同时启动跑迁移,你的锁方案在什么情况下会失效?"** —— 应答策略:主动暴露弱点最加分。Postgres:advisory lock 是 session 级,托管 PG 的 `idle_in_transaction_session_timeout`(默认 1-10 分钟)会杀掉持锁的 idle 事务连接导致锁静默释放——所以拿锁前先 `SET LOCAL ... = 0`;unlock 失败有 session 断开兜底。SQLite:承认跨进程只是 best-effort(文件锁 + 30s busy_timeout,驱动默认只有 5s),病态重叠会 `database is locked`,兜底是幂等 revision;真多实例就该用 Postgres,这是写在 docstring 里的部署约束。

3. **"autogenerate 和这套混合 bootstrap 怎么共存?新加一列/一张表的完整流程是什么?"** —— 应答策略:列:写新 revision 用 `safe_add_column`(幂等 + nullable/server_default/type 三维漂移 warning,JSON/JSONB 等价族豁免),不需要改 bootstrap;表:写 revision + `op.create_table`,如果是基线级表还必须同步更新 `_BASELINE_TABLE_NAMES`,否则守卫测试会火。autogenerate 侧有 `include_object` 过滤器保证不会给 LangGraph 的 4 张表生成 `drop_table`。再带一句基线原则:"create_all 只管 empty 快路径,baseline 之后每个变更都属于某个 revision"——这句话同时解释了为什么列形状漂移的答案放在 revision 里而不是 bootstrap 里。
