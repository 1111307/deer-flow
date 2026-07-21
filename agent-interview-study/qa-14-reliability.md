# 工程可靠性实践 —— 面试问题链

> 本文档按"大厂面试追问链"组织:每条问题链从基础问题开始,层层追问到实现细节、设计权衡、异常边界。配套深读笔记:[12-reliability-practices.md](12-reliability-practices.md)(深读笔记讲"怎么实现",本文档讲"怎么被问、怎么答")。
> 代码引用格式:[文件名.py:起始行-结束行](../backend/仓库相对路径#L起始行-L结束行)。只引用实际读过的行。

本模块覆盖 DeerFlow 的四道可靠性防线:

- Blockbuster 阻塞 IO 运行时闸门(`tests/blocking_io/`);
- 门禁自身的元测试(`test_gate_smoke.py`);
- harness/app 分层边界守卫(`test_harness_boundary.py`);
- 配置热重载边界注册表(`reload_boundary.py` + `test_reload_boundary.py`)。

共同主题是:**把人容易违反、违反成本高、当下不报错的约定,转成违反即失败的自动化检查**。

---

## 问题链 1:Blockbuster 闸门与 scanned_modules 的作用域设计

**Q1.1(基础)** 你们的测试套件是怎么防止"在 async 事件循环里跑同步阻塞 IO"这类 bug 回归的?

**参考回答**:用 Blockbuster 库做运行时检测。`detect_blocking_io_strict()` 创建一个 `BlockBuster(scanned_modules=["app", "deerflow"])` 实例,在 `activate()`/`deactivate()` 之间,任何调用栈经过 `app.*` 或 `deerflow.*` 模块的阻塞同步 IO 都会抛 `BlockingError`([blocking_io_runtime.py:32-41](../backend/tests/support/detectors/blocking_io_runtime.py#L32-L41))。整个 `backend/tests/blocking_io/` 目录的测试都跑在这个上下文里,形成一道回归闸门。修复侧的标准动作是把阻塞调用包进 `await asyncio.to_thread(...)`(问题链 4 的锚点测试会展开)。值得注意的是闸门判定的是"调用栈是否经过业务模块",而不是"阻塞调用本身长什么样",所以业务代码新用的任何阻塞原语都自动落在监控范围内,无需逐个登记。

**链路解析**:

```
pytest 收集 blocking_io/ 下用例
  -> conftest.pytest_runtest_protocol (hookwrapper)
       -> detect_blocking_io_strict()
            -> BlockBuster(scanned_modules=("app","deerflow"))
            -> bb.activate()   # monkeypatch os/io/socket 等阻塞原语
            -> yield           # 跑 setup + call + teardown
                 -> 业务代码调用 os.mkdir / os.walk / sqlite connect ...
                      -> Blockbuster 检查调用栈帧:
            +--------------------+--------------------+
            | 栈帧经过           | 栈帧只经过         |
            | app.* / deerflow.* | pytest/importlib/  |
            | (业务代码)         | langchain/三方库   |
            +--------------------+--------------------+
            | raise BlockingError| 放行               |
            | -> 测试失败        | (不算违规)         |
            +--------------------+--------------------+
            -> finally: bb.deactivate()  # 还原补丁
                (异常也保证还原,test_gate_smoke 钉住)
```

**Q1.2(深挖)** `scanned_modules` 为什么限定成 `("app", "deerflow")`?直接全局扫描所有模块不行吗?

**参考回答**:`_SCANNED_MODULES = ("app", "deerflow")` 是有意收窄的([blocking_io_runtime.py:19](../backend/tests/support/detectors/blocking_io_runtime.py#L19))。模块 docstring 写得很明确:pytest、langchain、importlib 和第三方库都在扫描范围之外,否则会产生大量假阳性([blocking_io_runtime.py:1-9](../backend/tests/support/detectors/blocking_io_runtime.py#L1-L9))。比如 pytest 自己的 fixture 机制、importlib 首次导入模块时读 `.pyc` 文件,这些都是同步 IO 且跑在事件循环线程上,但它们不是业务代码、修不了也不该修。全局扫描会让闸门天天红,最后团队学会无视红灯——**一个总是误报的闸门比没有闸门更糟**。举个具体的假阳性例子:测试里第一次 `import` 某模块时,importlib 会同步读盘加载字节码,这件事发生在事件循环线程上却完全合法;把它排除后,红灯就只代表"业务代码真的在阻塞"这一种含义,信号纯度是闸门可信度的来源。

**Q1.3(边界/异常)** 闸门上下文里如果测试自己抛了异常,Blockbuster 的 monkeypatch 会不会泄漏、污染后面的测试?

**参考回答**:不会。`detect_blocking_io_strict()` 用 `try/finally` 保证 `bb.deactivate()` 一定执行([blocking_io_runtime.py:37-41](../backend/tests/support/detectors/blocking_io_runtime.py#L37-L41))。而且这个保证本身被元测试钉死了:`test_gate_restores_blockbuster_patches_after_exceptions` 在闸门内主动 `raise RuntimeError("boom")`,出来后断言 `os.stat is original_stat`,即补丁被完整还原([test_gate_smoke.py:36-43](../backend/tests/blocking_io/test_gate_smoke.py#L36-L43))。没有这条,一次失败的测试会让后续所有测试运行在"半打补丁"的脏环境里,出现极难排查的级联失败。另一个细节是闸门用 `@contextmanager` 实现([blocking_io_runtime.py:32-33](../backend/tests/support/detectors/blocking_io_runtime.py#L32-L33)),让 activate/deactivate 的配对在语法层面就不可能缺半边——这也是"把约定变成结构"思路的体现。

---

## 问题链 2:hookwrapper 为什么包住 setup + call + teardown

**Q2.1(基础)** 闸门是怎么挂到 pytest 上的?直接写个 fixture 不行吗?

**参考回答**:用的是 `pytest_runtest_protocol` 的 hookwrapper,不是 fixture([conftest.py:26-33](../backend/tests/blocking_io/conftest.py#L26-L33))。`@pytest.hookimpl(hookwrapper=True)` 让 `yield` 包住整个测试条目协议——setup、call、teardown 三个阶段全部在 `detect_blocking_io_strict()` 上下文内执行。选 `pytest_runtest_protocol` 而不是更细的 `pytest_runtest_call`,是因为前者是 pytest 条目生命周期的最外层钩子,位置刚好能罩住全部三个阶段。

**链路解析**:

```
pytest_runtest_protocol(item, nextitem)
  |
  |-- [过滤 1] item.path 不在 tests/blocking_io/ 下? -> 直接 yield 放行
  |-- [过滤 2] 带 @pytest.mark.allow_blocking_io?      -> 直接 yield 放行
  |
  |-- 两道过滤都通过:打开 detect_blocking_io_strict()
  |
  |-- hookwrapper yield 内部(pytest 框架继续执行):
  |      setup      -> async fixture 初始化、lifespan 启动   [被监控]
  |      call       -> 测试体                                 [被监控]
  |      teardown   -> fixture 清理、连接关闭                 [被监控]
  |
  |-- yield 返回:with 块退出,bb.deactivate()
  |
  |  对比:autouse fixture 方案的监控窗口
  |      setup( fixture 激活前,盲区 ) -> call( 监控 ) -> teardown( 盲区 )
  |      => #1912 恰好发生在盲区里
```

**Q2.2(深挖)** 为什么必须包三个阶段?只监控测试体(call 阶段)会漏什么?

**参考回答**:conftest 的 docstring 明确回答了:包住整个协议是为了让"async fixtures 和 lifespan 代码里的阻塞 IO 也被抓到,而不只是测试体内的"([conftest.py:1-6](../backend/tests/blocking_io/conftest.py#L1-L6))。这不是理论顾虑——真实 bug #1912 就是 `ensure_sqlite_parent_dir` 里的 `Path.mkdir`/`os.mkdir` 跑在 FastAPI **lifespan** 事件循环线程上阻塞了启动([test_sqlite_lifespan.py:1-15](../backend/tests/blocking_io/test_sqlite_lifespan.py#L1-L15))。**反例分析**:如果用 fixture 方案(比如 autouse fixture 只在 call 前后激活),fixture 和 lifespan 的 setup/teardown 阶段恰恰落在监控之外——而那里正是建连、建目录、初始化 checkpointer 这类重 IO 最密集的地方,等于把最容易出事的阶段留成盲区。另外 async fixture 在 pytest-asyncio 下通常跑在与测试体同一个事件循环上,fixture 里一次同步建连就足以卡住同循环上的全部并发任务——这正是阻塞 IO 最危险也最隐蔽的形态,只监控 call 阶段等于默许它存在。

**Q2.3(深挖)** 全量跑测试套件时,这个闸门会不会误伤 `blocking_io/` 目录之外的普通测试?

**参考回答**:不会,有两道过滤。第一道是路径过滤:`_is_blocking_io_item()` 用 `Path(item.path).resolve().is_relative_to(_BLOCKING_IO_TEST_ROOT)` 判断,不在目录下直接 `yield` 放行([conftest.py:36-37](../backend/tests/blocking_io/conftest.py#L36-L37))。docstring 解释了为什么必须有这道过滤:conftest 一旦被加载,hookwrapper 是**全局注册**的,不显式过滤就会对无关测试开火([conftest.py:7-10](../backend/tests/blocking_io/conftest.py#L7-L10))。实现上两处 `resolve()` 也不是多余:`item.path` 可能带符号链接或相对段,双方都 resolve 之后 `is_relative_to` 的前缀比较才稳定。第二道是 opt-out 标记,见下一问。

**Q2.4(边界/异常)** 如果某个测试确实需要合法的阻塞 IO(比如验证同步工具函数本身),怎么办?硬绕吗?

**参考回答**:有正式的逃生舱:给测试打 `@pytest.mark.allow_blocking_io` 标记,hookwrapper 里 `item.get_closest_marker("allow_blocking_io") is not None` 时直接放行([conftest.py:28-30](../backend/tests/blocking_io/conftest.py#L28-L30))。注意判断顺序:路径过滤和标记检查做在同一个条件里,两道闸门任一命中都走"裸 yield"的快路径,不产生任何 Blockbuster 开销。逃生舱本身也被测试钉住:`test_allow_blocking_io_marker_opts_out_of_gate` 带着标记直接调用会触发 `BlockingError` 的 `ensure_sqlite_parent_dir`,验证标记确实关掉了闸门([test_gate_smoke.py:46-55](../backend/tests/blocking_io/test_gate_smoke.py#L46-L55))。设计要点是逃生舱**显式、可审计**——grep 一下标记就能列出所有豁免项,而不是散落在代码里的隐式绕过。从评审视角看,新增一个豁免标记在 diff 里是一行显眼的 decorator,远比藏在测试体里的 `try/except BlockingError` 之类绕法容易审查——逃生舱的"显眼"是故意设计的,让每次豁免都成为需要论证的决定。

---

## 问题链 3:test_gate_smoke —— 守卫者也要被守卫

**Q3.1(基础)** `test_gate_smoke.py` 是测什么的?它不测任何业务代码,存在的意义是什么?

**参考回答**:它是元测试(meta-test):不验证业务行为,而是验证**闸门机制本身还在工作**。docstring 说得很直白:"a green gate that no longer catches anything is worse than no gate at all"(一个不再抓东西的绿闸门比没有闸门更糟)([test_gate_smoke.py:1-13](../backend/tests/blocking_io/test_gate_smoke.py#L1-L13))。第一个用例从 `deerflow` 模块的 async 函数里直接调用已知阻塞的 `ensure_sqlite_parent_dir`,断言必须抛 `BlockingError`([test_gate_smoke.py:27-33](../backend/tests/blocking_io/test_gate_smoke.py#L27-L33))。整个文件三个用例恰好对应闸门机制的三条关键不变量:能抓阻塞、异常后补丁还原、逃生舱生效——一个文件把闸门自检闭环了。

**链路解析**:

```
业务回归测试(锚点)         test_gate_smoke(元测试)
  | 依赖                        | 依赖
  v                             v
Blockbuster 闸门正常干活  <---  闸门失效时,谁报警?
  ^                             |
  |__ 锚点全绿,但可能是 ________|
     "闸门死了"造成的假绿

元测试三个用例 = 闸门三条不变量:
  [1] 直接调用 ensure_sqlite_parent_dir(无 to_thread)
        -> 必须抛 BlockingError          => "还能抓"
  [2] 闸门内 raise RuntimeError("boom")
        -> 出来后 os.stat is original    => "补丁能还原"
  [3] 带 allow_blocking_io 标记调同一函数
        -> 不抛且目录建成                => "逃生舱有效"

闸门典型死因(全是静默的):
  scanned_modules 配错 / blockbuster 依赖被移除 / hookwrapper 不触发
  => 表现都是"测试变绿",唯有元测试会变红
```

**Q3.2(深挖)** 闸门会以哪些方式"悄悄死掉"?元测试具体防的是哪几种?

**参考回答**:docstring 列了三类典型死因([test_gate_smoke.py:5-8](../backend/tests/blocking_io/test_gate_smoke.py#L5-L8)):

- `scanned_modules` 配错——比如业务包改名后没同步,扫描范围落空;
- blockbuster 这个 dev 依赖被意外移除,闸门上下文直接失效;
- conftest 的 hookwrapper 不再触发——比如 pytest 大版本升级改了 hook 语义。

这三种故障的共同点是**全部表现为测试变绿**——没有元测试的话,团队会以为防线还在,实际早已裸奔。这也解释了元测试为什么必须放进常驻测试套件而不是一次性验证脚本:这三种死因全是"后来的变更"引入的,只有每次 CI 都跑,才能在引入的那一刻报警。

**Q3.3(深挖)** 为什么不把元测试的断言直接塞进某个业务锚点测试里,顺便验证闸门?

**参考回答**:因为锚点测试绿有两种解释:"业务代码没阻塞"或"闸门根本没生效",两者无法区分。元测试的价值恰恰在于它**独立于任何具体生产路径**([test_gate_smoke.py:2-4](../backend/tests/blocking_io/test_gate_smoke.py#L2-L4)),用的是一个永远阻塞的已知函数做活体探针。如果闸门失效,业务锚点全绿、只有元测试红,故障定位一步到位;混写则红一片,分不清是业务回归还是闸门故障。这也是"反向锚点"思路:不仅锚住"好行为还在",还要锚住"坏行为仍然会被抓"。工程上这相当于给监控系统本身配监控(类似 SRE 里的 dead man's switch):报警器沉默时,必须有另一个机制区分"无事发生"和"报警器已死"。

---

## 问题链 4:回归锚点哲学 —— 每修一个 bug 加一个锚点

**Q4.1(基础)** 看 `blocking_io/` 目录下的测试,文件名都对应具体生产模块,这是一种什么测试策略?

**参考回答**:这是"回归锚点"(regression anchor)策略:每修一个阻塞 IO 的 bug,就写一个把**生产真实调用路径**跑在严格闸门下的测试,把这个修复永久钉住。比如 `test_sqlite_lifespan.py` 锚的是 `_async_checkpointer` 里 `asyncio.to_thread` 卸载(修 #1912)([test_sqlite_lifespan.py:1-15](../backend/tests/blocking_io/test_sqlite_lifespan.py#L1-L15));`test_skills_load.py` 锚的是 `_load_skills` 里 `get_or_new_skill_storage` 和 `storage.load_skills` 两处 `asyncio.to_thread` 卸载(修 #1917,`os.walk` 阻塞 LangGraph 事件循环)([test_skills_load.py:1-12](../backend/tests/blocking_io/test_skills_load.py#L1-L12))。整个目录就是这种哲学的产物:`test_agents_router`、`test_jsonl_run_event_store`、`test_persistence_engine_sqlite`、`test_uploads_middleware` 等文件一一对应各自修过的阻塞点,目录本身就是一份"阻塞 IO 事故编年史"。

**链路解析**:

```
线上 bug (#1912: lifespan 里同步 mkdir / #1917: os.walk 阻塞)
  |
  |-- 修复:阻塞调用包进 await asyncio.to_thread(...)
  |
  |-- 写锚点测试(每修一个 bug 加一个):
  |     真实生产函数(_async_checkpointer / _load_skills)
  |       -> 在 detect_blocking_io_strict() 下执行
  |       -> 卸载被人删了?  -> BlockingError      -> 测试红(回归被抓)
  |       -> 卸载还在?      -> 正常返回            -> 测试绿
  |
  |-- 双向锚定:
  |     下界:Blockbuster      -> 不许阻塞(防止回到 bug)
  |     上界:业务断言          -> 必须真的干活
  |           (db_file.parent.exists() / from_conn_string 调了一次
  |            / saver.setup 被 await / skills 列表里有 demo)
  |
  |-- mock 边界:隔离外部副作用(数据库连接),
  |     保留被锚行为(路径解析 + 建目录走真实逻辑)
```

**Q4.2(深挖)** 锚点测试和"直接单测阻塞函数本身"有什么区别?为什么强调走生产路径?

**参考回答**:区别在于锚点测的是**卸载决策点**,不是阻塞原语本身。#1912 的测试直接调用生产函数 `_async_checkpointer(CheckpointerConfig(type="sqlite", ...))`,让真实的 `Path.mkdir`/`os.mkdir` 都执行(父目录故意不存在)([test_sqlite_lifespan.py:29-50](../backend/tests/blocking_io/test_sqlite_lifespan.py#L29-L50))。如果将来有人把 `await asyncio.to_thread(...)` 改回直接调用,Blockbuster 立刻抛 `BlockingError`。**反例分析**:如果只单测 `ensure_sqlite_parent_dir` 本身,它本来就是同步函数、在闸门下必然报错,什么信息量都没有;而绕过生产路径写"模拟"测试,则测不出"有人删了 to_thread"这种回归——锚点必须钉在真实调用链上才有效力。注意 mock 的边界划得很讲究:`from_conn_string` 返回 mock 的 async context manager 以隔离外部依赖([test_sqlite_lifespan.py:35-44](../backend/tests/blocking_io/test_sqlite_lifespan.py#L35-L44)),但路径解析和建目录走的是真实逻辑——隔离的是副作用,保留的是被锚的行为。

**Q4.3(边界/异常)** 锚点测试只验证"不抛 BlockingError"就够了吗?看代码好像还有别的断言?

**参考回答**:不够,还要验证业务行为本身没被破坏。#1912 的测试除了跑通闸门外,还有三条业务断言([test_sqlite_lifespan.py:50-52](../backend/tests/blocking_io/test_sqlite_lifespan.py#L50-L52)):

- `db_file.parent.exists()` —— 目录真的建了;
- `from_conn_string.assert_called_once_with(str(db_file.resolve()))` —— 连接串是 resolve 后的路径;
- `mock_saver.setup.assert_awaited_once()` —— 协程语义正确。

#1917 的测试断言返回的 skills 列表里真有 `demo`([test_skills_load.py:99-102](../backend/tests/blocking_io/test_skills_load.py#L99-L102))。否则一个"既不阻塞、但也不干活"的空实现也能让测试变绿,锚点就失去意义了。可以把这组断言理解为双向锚定:Blockbuster 管"不许阻塞"的下界,业务断言管"必须真的干活"的上界,中间的合法区间才是修复想要的行为。

---

## 问题链 5:harness 边界守卫 —— 为什么用 AST 不用正则

**Q5.1(基础)** `test_harness_boundary.py` 在守什么约定?这个约定为什么值得用测试来守?

**参考回答**:守的是分层架构约定:`packages/harness/deerflow/` 是一个可独立发布的 agent 框架包,**绝不能 import app 层**。测试扫描 harness 包下全部 `.py` 文件,发现任何 `from app.` 或 `import app.` 就失败([test_harness_boundary.py:1-8](../backend/tests/test_harness_boundary.py#L1-L8),[test_harness_boundary.py:37-46](../backend/tests/test_harness_boundary.py#L37-L46))。这类约定口头约定几乎必破——harness 里想用 app 层某个工具函数是最高频的诱惑,而一旦破了,独立发布就名存实亡。实现上 `sorted(HARNESS_ROOT.rglob("*.py"))` 全量递归扫描([test_harness_boundary.py:40](../backend/tests/test_harness_boundary.py#L40)),违规先收集再一次性断言,报错列出全部违规文件与行号,修复者一趟清完,而不是修一个冒一个。

**链路解析**:

```
HARNESS_ROOT = packages/harness/deerflow   # 可独立发布的框架包
BANNED_PREFIXES = ("app.",)                # 违禁上层
       |
       v
HARNESS_ROOT.rglob("*.py")                 # 全量递归扫描(sorted,结果可复现)
  -> 每个文件 ast.parse 成语法树
       |-- SyntaxError? -> 返回空列表(不炸,宽容解析)
       v
  ast.walk 全树遍历(含函数体内延迟 import / TYPE_CHECKING 块)
       |-- ast.Import     -> 收集每个 alias.name
       |-- ast.ImportFrom -> 收集 node.module(非空时)
       v
  逐个比对:module == "app"  or  module.startswith("app.")
       |-- 命中 -> violations.append("rel_path:lineno  imports module")
       v
  assert not violations   # 一次性报告全部违规(文件:行号 精确定位)
```

**Q5.2(深挖)** 这个检查用正则 `grep "from app\."` 一行就能做,为什么要搬出 `ast` 模块?

**参考回答**:三个理由:

1. **假阳性**:正则会匹配注释、docstring、字符串字面量里的 "from app."(比如文档示例),AST 只认真实的 `Import`/`ImportFrom` 节点([test_harness_boundary.py:27-33](../backend/tests/test_harness_boundary.py#L27-L33));
2. **精确报错**:AST 节点自带 `node.lineno`,报错能给出 `文件:行号 imports 模块` 的精确定位([test_harness_boundary.py:30-33](../backend/tests/test_harness_boundary.py#L30-L33)),正则做不到可靠行号;
3. **健壮解析**:`ast.parse` 遇到语法错误返回空列表而不是炸掉([test_harness_boundary.py:21-24](../backend/tests/test_harness_boundary.py#L21-L24))。

代价是 AST 方案约 20 行 vs 正则 3 行,但边界守卫是长期运行的基础设施,误报一次就会消耗团队信任。再补一点:`ast.walk` 全树遍历意味着函数体内的延迟 import、`TYPE_CHECKING` 块里的 import 同样被收集——这其实是特性,延迟 import 并不改变 harness 依赖 app 的事实,只会把问题推迟到运行时暴露。

**Q5.3(深挖)** 匹配逻辑 `module == prefix.rstrip(".") or module.startswith(prefix)` 为什么要写成两段?

**参考回答**:因为要同时接住两种导入形态:

- `import app` 时收集到的 `module` 恰好是 `"app"`(不带点);
- `from app.foo import x` 时是 `"app.foo"`(带前缀)。

单写 `startswith("app.")` 会漏掉裸的 `import app`;单写相等判断会漏掉子模块([test_harness_boundary.py:42](../backend/tests/test_harness_boundary.py#L42))。这是边界检查里典型的"枚举攻击面"思维:把违禁模式的全部语法变体都列出来,逐一封堵。`BANNED_PREFIXES` 本身是元组而非单个字符串([test_harness_boundary.py:15](../backend/tests/test_harness_boundary.py#L15)),也为未来扩展(禁止 import 其他上层包)留了形状上的余地。

**Q5.4(边界/异常)** 如果有人用 `importlib.import_module("app.xxx")` 动态导入,这个 AST 守卫不就瞎了吗?

**参考回答**:会,静态 AST 只能抓字面 import 语句,动态导入的模块名是运行时字符串、甚至可能拼接而成,确实能绕过。这是该守卫的已知能力边界:它防的是"无意违规"和"图省事违规",防不了蓄意绕过。工程上的缓解是——动态导入在代码评审中本身就是高可见度行为,而 AST 守卫把违规成本压到极低(违反即失败、报错带行号),足以让 95% 的"习惯成自然"式违规在 PR 阶段被拦住;剩下 5% 的蓄意绕过留给评审和运行时手段。完美防线不存在,**把最高频的违规路径自动化**才是可靠性工程的常态取舍。

---

## 问题链 6:reload_boundary —— 单一事实来源与双向漂移检测

**Q6.1(基础)** 配置热重载的边界(哪些字段改完必须重启进程)是怎么管理的?写文档里不行吗?

**参考回答**:不行,静态文档会漂移——写完那一刻起就开始和代码脱节。DeerFlow 把它做成了机器可读的注册表:`reload_boundary.py` 里的 `STARTUP_ONLY_FIELDS` 字典,把 8 个 restart-required 字段路径各映射到一段解释"为什么"的文本([reload_boundary.py:45-62](../backend/packages/harness/deerflow/config/reload_boundary.py#L45-L62)):

- `database` —— SQLAlchemy 引擎与连接池,启动时建一次;
- `checkpointer` —— 持久化 checkpointer,含 SQLite WAL / busy_timeout;
- `run_events` —— memory vs SQL 实现的选型,冻结在 app.state 上;
- `stream_bridge` —— stream-bridge 单例;
- `sandbox` —— provider 单例缓存(`_default_sandbox_provider`);
- `log_level` —— `apply_logging_level()` 只在启动时跑;
- `channels` / `channel_connections` —— IM 客户端(Feishu/Slack/Telegram/DingTalk)启动时接线,不在 AppConfig schema 内。模块 docstring 明确定位:这是单一事实来源(single source of truth),未来任何"needs restart"扫描器、运维工具、文档生成器都应该消费这个注册表,而不是重新解析散文([reload_boundary.py:22-25](../backend/packages/harness/deerflow/config/reload_boundary.py#L22-L25))。背景是 issue #3144:网关请求依赖每请求都经 `get_app_config()` 解析 `AppConfig`,per-run 字段下一条消息即生效;注册表覆盖的是启动时只捕获一次快照的基础设施子集——引擎、单例、IM 客户端、日志 handler([reload_boundary.py:1-9](../backend/packages/harness/deerflow/config/reload_boundary.py#L1-L9))。"哪些改了立即生效、哪些要重启"如果不做成数据,就会以运维事故的形式反复被重新发现。

**链路解析**:

```
                 reload_boundary.py (单一事实来源)
                 STARTUP_ONLY_FIELDS: dict[path, reason]  (8 个字段)
                 STARTUP_ONLY_PREFIX = "startup-only:"
                          |
      +-------------------+------------------------+
      |                                            |
      v                                            v
AppConfig 侧                                  测试侧 (test_reload_boundary.py)
format_field_description(path, field_doc=?)     [正向] 注册表字段
  -> KeyError(未注册,故意)                        -> schema description 必须
  -> "startup-only: <reason>"                        startswith("startup-only:")
  -> + "\n\n" + field_doc(可选)                  [反向] schema 带前缀的字段
      |                                            -> 必须存在于注册表
      v                                            [辅助] iter/membership 与字典同步
IDE hover: "需重启 + 哪段代码捕获了快照"           [辅助] reason 非空且 > 20 字符
      |                                            [辅助] IDE hover 来源
      |                                                 (model_fields[...].description)
      |                                                 运行时可读
      +---------------- 双向一夹,两侧只能同步演进 ----+
```

**Q6.2(深挖)** 每个字段的 reason 文本有什么讲究?写个 "needs restart" 不行吗?

**参考回答**:不行,测试强制了质量标准:`test_registry_has_a_reason_for_every_field` 要求每条 reason 非空且**长度必须大于 20 字符**,否则判定"太短没用"([test_reload_boundary.py:25-33](../backend/tests/test_reload_boundary.py#L25-L33))。设计意图写在注释里:reason 必须解释**是哪段代码在启动时捕获了快照**,而不只是声明"需要重启",这样运维改了值才知道要重启哪个子系统([reload_boundary.py:39-44](../backend/packages/harness/deerflow/config/reload_boundary.py#L39-L44))。看实际内容,比如 `database` 的 reason 点名 `init_engine_from_config() runs once during langgraph_runtime() startup`、`sandbox` 点名 `_default_sandbox_provider` 单例缓存([reload_boundary.py:46-50](../backend/packages/harness/deerflow/config/reload_boundary.py#L46-L50))——全是具体的函数名和机制。还有一层容易忽略的考量:reason 文本会直接进入 `Field(description=...)`,也就是 IDE hover 和 schema 文档的展示面,所以它本质上面向运维的 UI 文案,不是写给自己看的注释——这也是"太短没用"判定成立的根本原因。

**Q6.3(深挖)** "双向漂移检测"具体是哪两个方向?各防什么?

**参考回答**:正向:`test_appconfig_schema_marks_registered_fields_with_prefix` 遍历注册表,凡顶层 AppConfig 字段的 `Field(description=)` 必须以 `"startup-only:"` 前缀开头,保证注册表到 schema 的落实([test_reload_boundary.py:101-114](../backend/tests/test_reload_boundary.py#L101-L114))。反向:`test_no_appconfig_field_uses_prefix_without_registration` 遍历 schema,凡是 description 带前缀的字段必须已在注册表中——防的是"有人在 schema 里标了 restart-required 却忘了更新注册表,导致运维扫描器漏报"([test_reload_boundary.py:117-129](../backend/tests/test_reload_boundary.py#L117-L129))。单边检查都会有盲区:只查正向,schema 侧新增标记会逃逸;只查反向,注册表条目可能没在 schema 落实。双向一夹,两侧只能同步演进。配套还有两个小的同步锚:`test_iter_startup_only_field_paths_matches_registry` 钉住迭代器与字典一致([test_reload_boundary.py:36-38](../backend/tests/test_reload_boundary.py#L36-L38));`test_is_startup_only_field_recognises_registered_fields` 用 `memory`、`models`、`nonexistent_field` 三个负例确认 membership helper 不会把热加载字段误判成 restart-required([test_reload_boundary.py:41-47](../backend/tests/test_reload_boundary.py#L41-L47))。

**Q6.4(边界/异常)** 注册表里有些条目(比如 `channels`)不在 AppConfig schema 里,正向检查不就报错了吗?`format_field_description` 收到未注册字段会怎样?

**参考回答**:两个边界都被显式处理了:

1. 正向测试里 `if field_path not in schema_fields: continue`,并注释说明 `channels` 这类条目活在 schema 之外、注册表仍是其唯一权威位置,schema 前缀断言不适用([test_reload_boundary.py:107-112](../backend/tests/test_reload_boundary.py#L107-L112);[reload_boundary.py:19-21](../backend/packages/harness/deerflow/config/reload_boundary.py#L19-L21));
2. `format_field_description` 对未注册字段直接 `raise KeyError`——这是**故意**的,docstring 写明"silently returning a placeholder would let a typo bypass the drift coverage"(静默返回占位符会让拼写错误绕过漂移覆盖)([reload_boundary.py:98-103](../backend/packages/harness/deerflow/config/reload_boundary.py#L98-L103)),且有专门测试 `pytest.raises(KeyError)` 钉住这个行为([test_reload_boundary.py:61-63](../backend/tests/test_reload_boundary.py#L61-L63))。**反例分析**:如果改成返回空串或占位符,一个手滑的字段名拼写会让 schema 描述悄悄丢失前缀,反向漂移测试又只看已有前缀的字段——错误就彻底隐身了。让错误吵起来,是这里的一贯手法。`is_startup_only_field` 的 docstring 还交代了粒度边界:只接受顶层路径,`database.url` 这类嵌套键不建模,因为边界是 per-section 而非 per-leaf([reload_boundary.py:70-77](../backend/packages/harness/deerflow/config/reload_boundary.py#L70-L77))——粒度取舍同样被写进了契约。

---

## 问题链 7:统一设计哲学 —— 违反即失败

**Q7.1(基础)** 这一组测试看起来技术各异(Blockbuster、AST、Pydantic 内省),背后有没有统一的设计原则?

**参考回答**:有:**把人容易违反、违反成本高、当下不报错的约定,转成违反即失败的自动化检查**。三个守卫各自命中一类:阻塞 IO 是"当下不报错、生产上拖垮事件循环";harness 分层是"import 时完全合法、发布时才发现耦合";reload 边界是"改了配置不生效、运维半夜排查"。它们的共同点是无法靠编译器/解释器在编写时拦截,又贵到不能靠评审肉眼守,于是全部下沉为 CI 里的硬失败。可以拿三条入选标准反查:阻塞 IO 三条全中——async 代码里写同步调用是肌肉记忆级易犯,事件循环一堵影响全部在线请求,本地单跑一次根本看不出来;这也解释了为什么它无法降级为"评审时注意一下"。

**链路解析**:

```
              约定(写在 Wiki/口头,几乎必破)
                        |
      +-----------------+----------------+----------------+
      |                 |                |                |
      v                 v                v                v
   阻塞 IO          分层边界         reload 边界       闸门自身
   "async 里混      "harness 不      "这个字段改完     "守卫还活着吗"
    同步调用"        许 import app"   要不要重启"
      |                 |                |                |
      v                 v                v                v
   Blockbuster      AST 扫描         注册表 + schema   元测试
   运行时闸门       (静态分析)        双向漂移检测      (活体探针)
   (运行时信息)     (纯静态可行)      (运行时内省)      (依赖闸门本身)
      |                 |                |                |
      +-----------------+-------+--------+----------------+
                                |
                      违反即 CI 失败(附精确位置/字段名)
                                |
              入选标准:人易违反 + 违反成本高 + 当下不报错
```

**Q7.2(深挖)** 这些检查为什么不放在 pre-commit 或 lint 阶段,而要放测试套件里?

**参考回答**:看各检查的性质:AST 边界扫描是纯静态的,确实可以放 lint;但 Blockbuster 必须**运行时**执行才能基于真实调用栈判断([blocking_io_runtime.py:5-7](../backend/tests/support/detectors/blocking_io_runtime.py#L5-L7));reload 双向检测要内省 `AppConfig.model_fields` 的运行时 Pydantic schema([test_reload_boundary.py:106](../backend/tests/test_reload_boundary.py#L106)),`test_pydantic_field_descriptions_are_introspectable_at_runtime` 还专门钉住"`model_fields[name].description` 运行时可读"这个前提,防 Pydantic 升级让内省失效([test_reload_boundary.py:132-141](../backend/tests/test_reload_boundary.py#L132-L141))。三个守卫里两个依赖运行时信息,统一放测试套件反而是最简方案;而且测试失败能带业务断言上下文,比 lint 报错信息量大。另一个现实因素是依赖管理:Blockbuster 是 dev 依赖、闸门由 conftest 加载,放进 pytest 体系等于复用现成的依赖与发现机制,不必再发明一套 runner 和 CI 入口。

**Q7.3(深挖)** 这套守卫自身的演化纪律是什么?比如 Blockbuster 漏了业务新用的阻塞原语、配置新增 restart-required 字段?

**参考回答**:两条演化通道都被"守卫"着。Blockbuster 侧预留 `_PROJECT_BLOCKING_RULES` 扩展点,注释写清添加纪律——只在默认规则集漏了生产代码在用的通用阻塞原语时才加;"路径没被任何测试覆盖"的盲区则靠新增生产路径锚点解决,而不是扩大检测器([blocking_io_runtime.py:21-24](../backend/tests/support/detectors/blocking_io_runtime.py#L21-L24))。reload 侧由反向漂移测试强制同步:schema 加了前缀就必须更新注册表,否则 CI 红([test_reload_boundary.py:125-129](../backend/tests/test_reload_boundary.py#L125-L129))。即**守卫的演化本身也被守卫着**,不存在"忘了同步"的自由。这套"给演化通道也加测试"的做法是元测试哲学的推广:不仅当前状态要被钉住,状态迁移的合法路径也要被钉住。

**Q7.4(边界/异常)** 这套体系的成本在哪?什么团队/项目不值得照搬?

**参考回答**:成本有三块:

1. **基建成本**——hookwrapper、作用域过滤、逃生舱标记这套机制本身需要维护和元测试守护([conftest.py:26-37](../backend/tests/blocking_io/conftest.py#L26-L37));
2. **锚点成本**——每修一个 bug 要写一个走真实生产路径的测试,有时还得像 `_real_subagent_executor` 那样和 `sys.modules` 搏斗约 35 行([test_skills_load.py:45-79](../backend/tests/blocking_io/test_skills_load.py#L45-L79));
3. **纪律成本**——reason 必须写满 20 字符讲清机制([test_reload_boundary.py:32-33](../backend/tests/test_reload_boundary.py#L32-L33)),注册表/schema 双向同步由 CI 强制执行,没有"先合后补"的自由。

短周期、单人、无长期维护预期的项目照搬会入不敷出;这套体系的回报曲线依赖两个前提:**代码活得足够久**(回归才会真实发生)和**协作者足够多**(口头约定才会真实失效)——恰恰是大厂长寿命平台团队的处境。反过来若被问"这套体系最大的风险是什么",答案应是:豁免机制被滥用导致闸门名存实亡——所以 `allow_blocking_io` 要可 grep、锚点要走真实生产路径,两处设计都在刻意提高"绕过"的可见度。

---

## 面试官最爱追问的 3 个点

1. **"闸门失效了你怎么知道?"** —— 应答核心:元测试 `test_gate_smoke.py` 独立于业务路径做活体探测,直接抛已知阻塞调用断言 `BlockingError`([test_gate_smoke.py:27-33](../backend/tests/blocking_io/test_gate_smoke.py#L27-L33));主动说出"a green gate that no longer catches anything is worse than no gate at all"这句原文,并列举三种死因(scanned_modules 配错、依赖被移除、hook 不触发)。
   - 加分项:主动补一句"元测试的三个用例对应闸门的三条不变量——能抓、能还原、能豁免",展示你把文件读到了用例粒度。
2. **"为什么用 AST 不用正则?"** —— 应答核心:三句话——注释/字符串里的假阳性、`node.lineno` 精确报错定位、语法错误时优雅降级返回空列表([test_harness_boundary.py:21-33](../backend/tests/test_harness_boundary.py#L21-L33));再补一刀主动暴露边界:动态 `importlib.import_module("app.x")` 能绕过,静态守卫防无意违规、蓄意绕过留给评审。
   - 加分项:提到 `ast.walk` 连函数体内的延迟 import 也能抓到,说明理解遍历语义而不只是"用了 AST"。
3. **"双向漂移检测防的到底是什么场景?"** —— 应答核心:正向防"注册表写了但 schema 没落实"(IDE hover 失信),反向防"schema 标了但注册表没更新"(运维扫描器漏报)([test_reload_boundary.py:101-129](../backend/tests/test_reload_boundary.py#L101-L129));并指出 `format_field_description` 对未注册字段故意 `raise KeyError`——让拼写错误吵起来而不是静默绕过([reload_boundary.py:98-103](../backend/packages/harness/deerflow/config/reload_boundary.py#L98-L103))。
   - 加分项:补一句"reason 必须超过 20 字符且点名捕获快照的函数"([test_reload_boundary.py:32-33](../backend/tests/test_reload_boundary.py#L32-L33)),说明文案质量本身也被测试管着。
