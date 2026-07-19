# 第十二部分：工程可靠性实践 —— 把"约定"变成"会自己报警的测试"

前面十一部分讲的都是"某个功能怎么实现"。最后这一部分反过来:**当一个 agent 系统足够大、团队足够多人协作时,怎么防止那些"没人会故意犯、但迟早会犯"的错误?** DeerFlow 的答案有一个统一的思路——**不靠文档、不靠 code review、不靠人记性,而是把每一条工程约定编码成一个会在 CI 里自己挂红的测试**。这一部分挑三条最有代表性的主线:异步阻塞检测、架构边界防腐、配置热重载边界。它们表面各不相关,内核是同一句话:**可执行的约束,才是真的约束。**

这一部分和前面不同——它不对应单一目录,`learn.md` 第 13 节列的也是三条分散的实践。但正因为分散,更能看出这套"约定即测试"的思路是**贯穿整个仓库的工程文化**,而不是某个模块的局部设计。

## 1. 为什么"文档 + review"不够

先把问题说清楚。这三条约定,如果只写在文档里,会怎么腐化?

- **异步阻塞**:"async 函数里不许调同步阻塞 IO"——这条规则没有任何编译器会帮你检查。一个 `open().read()` 塞进 `async def` 里,代码照跑,测试照过,直到线上并发量上来、事件循环被这个同步读死死卡住,所有请求一起变慢。
- **架构分层**:"harness 不许 import app"——第九部分反复强调的分层规则。但新人不知道、老人手滑,一行 `from app.gateway...` 就能悄悄把可发布的框架包和不可发布的应用层焊死。
- **配置热重载**:"哪些配置改了要重启、哪些立即生效"——这个知识只存在于"写这段启动代码的人"脑子里。加了个新的启动期字段却忘了声明,运维改了配置以为生效了,其实没有。

三者的共性:**违反的成本很高(线上事故/架构腐化/运维踩坑),但违反的动作很轻(一行 import、一个同步调用、一个漏声明),而且当下不报错。** 这正是"靠人自觉"最容易失守的地方。DeerFlow 对每一条都写了一个测试,把"当下不报错"变成"当下就挂红"。

## 2. 异步阻塞检测:运行时钩子,不是静态扫描

这是三条里最硬核的。核心文件 [`blocking_io_runtime.py`](../backend/tests/support/detectors/blocking_io_runtime.py) 出奇地短:

```python
from blockbuster import BlockBuster, BlockBusterFunction, BlockingError

_SCANNED_MODULES: tuple[str, ...] = ("app", "deerflow")

@contextmanager
def detect_blocking_io_strict() -> Iterator[BlockBuster]:
    """Activate Blockbuster scoped to app.* and deerflow.* callers only."""
    bb = BlockBuster(scanned_modules=list(_SCANNED_MODULES))
    _install_project_rules(bb)
    try:
        bb.activate()
        yield bb
    finally:
        bb.deactivate()
```

`BlockBuster` 是一个第三方库,它会 **monkey-patch 掉一批已知的同步阻塞原语**(`open`、`socket`、`os.read`、`time.sleep` 等),让它们在被调用时检查:当前是不是跑在一个正在运行的 asyncio 事件循环上?是的话就抛 `BlockingError`。

最关键的一个设计参数是 `scanned_modules=("app", "deerflow")`。为什么要限定范围?因为如果不限,pytest 自己、langchain、importlib、各种第三方库内部都有大量"在事件循环里做同步 IO"的合法行为,全报出来就是满屏假阳性,没法用。**限定只扫调用栈经过 `app.*`/`deerflow.*` 的阻塞调用**——也就是"只有当阻塞是 DeerFlow 自己的业务代码引发的,才算数"。这是让运行时检测**可用**的关键:精确锁定"我该负责的那部分",忽略"我管不着的框架内部"。

它怎么套到测试上?看 [`blocking_io/conftest.py`](../backend/tests/blocking_io/conftest.py):

```python
@pytest.hookimpl(hookwrapper=True)
def pytest_runtest_protocol(item, nextitem):
    if not _is_blocking_io_item(item) or item.get_closest_marker("allow_blocking_io") is not None:
        yield
        return
    with detect_blocking_io_strict():
        yield
```

用一个 `hookwrapper` 把整个测试执行协议(setup + call + teardown)包在 `detect_blocking_io_strict()` 里——**连异步 fixture、lifespan 代码里的阻塞 IO 都能抓到**,不只是测试函数体。注意那个路径过滤 `_is_blocking_io_item`:pytest 的 conftest hook 一旦加载就全局生效,所以必须显式限定"只对 `tests/blocking_io/` 目录下的测试启用严格门禁",否则跑全量套件时会误伤无关测试。还留了个 `@pytest.mark.allow_blocking_io` 的逃生舱,给确实需要同步阻塞的个别测试用。

**为什么运行时检测比静态扫描/code review 强?** 因为"async 里有没有阻塞 IO"这个问题,静态分析很难精确回答——同步函数 A 被谁调用了?如果 A 被一个 async 函数间接调到,它就有问题;如果只被同步路径调到,就没问题。这个"可达性"靠读代码很容易看漏(CLAUDE.md 里也提到静态 AST 扫描会因为同名 helper 而过度上报)。而运行时检测是**用真实执行的调用栈**来判断的——测试真的跑到了那行阻塞代码、且真的在事件循环上,才报。它把"隐性的性能地雷"变成了"CI 里一个确定性的红叉"。

这套东西在 CLAUDE.md 里对应一批"回归锚点"测试(`test_skills_load.py` 锁 `LocalSkillStorage.load_skills` 的 `asyncio.to_thread` offload、`test_uploads_middleware.py` 锁上传目录扫描的 offload 等)。每修好一个阻塞 IO bug,就在 `tests/blocking_io/` 加一个锚点,**锁住这个 offload 不能被后人改回去**——这是回归测试的本质:不只是"证明现在对了",而是"防止将来又错"。它有一个明确的覆盖边界(CLAUDE.md 也诚实写了):门禁只能看到**测试执行真正触及**的代码,没测到的路径它看不见——所以"加锚点"这个动作本身很重要。

## 3. 架构边界防腐:一个 AST 扫描把分层规则焊死

第九、十一部分反复提到 harness/app 分层:`deerflow.*`(可发布框架)不能 import `app.*`(应用层)。这条规则怎么保证不腐化?[`test_harness_boundary.py`](../backend/tests/test_harness_boundary.py) 全文就 47 行:

```python
def test_harness_does_not_import_app():
    violations: list[str] = []
    for py_file in sorted(HARNESS_ROOT.rglob("*.py")):
        for lineno, module in _collect_imports(py_file):
            if any(module == prefix.rstrip(".") or module.startswith(prefix) for prefix in BANNED_PREFIXES):
                rel = py_file.relative_to(HARNESS_ROOT.parent.parent.parent)
                violations.append(f"  {rel}:{lineno}  imports {module}")
    assert not violations, "Harness layer must not import from app layer:\n" + "\n".join(violations)
```

它用 `ast.parse` 把 harness 目录下每个 `.py` 文件解析成语法树,遍历所有 `import` / `from ... import` 节点,只要有一个模块路径是 `app.` 开头,就把"文件:行号"记进 violations,最后 `assert not violations`。

为什么用 AST 而不是正则搜 `import app`?因为 AST 精确——它只认真正的 import 语句,不会把注释里、字符串里、变量名里出现的 "app" 误报。[`_collect_imports`](../backend/tests/test_harness_boundary.py#L18-L34) 分别处理 `ast.Import`(`import app.x`)和 `ast.ImportFrom`(`from app.x import y`)两种节点,拿到准确的行号,报错信息直接给出 `文件:行号  imports app.xxx`,违规者一眼就能定位。

这条测试的价值在于:**它把"架构规则"从文档里一句"App imports deerflow, but deerflow never imports app"变成了一个 CI gate。** 任何 PR 只要在 harness 里手滑写了一行 `from app...`,CI 直接挂红,连 review 都不用等。这是"可执行的约束"最纯粹的形态——规则本身就是代码,违反它的定义就是让代码失败。

## 4. 配置热重载边界:一份注册表 + 双向漂移检测

第七、九部分提过 DeerFlow 的配置热重载:大部分配置改了下一条消息就生效(因为 gateway 每个请求都重新 `get_app_config()`),但**基础设施类字段**(数据库连接、checkpointer、sandbox provider、日志级别、IM channel)是启动时一次性抓取的快照,改了必须重启。问题来了:**哪些字段属于"必须重启"?这个清单怎么不腐化?**

[`reload_boundary.py`](../backend/packages/harness/deerflow/config/reload_boundary.py) 把它做成一份**单一事实来源**的注册表:

```python
STARTUP_ONLY_FIELDS: dict[str, str] = {
    "database": "init_engine_from_config() runs once during langgraph_runtime() startup; ...",
    "checkpointer": "make_checkpointer() binds the persistent checkpointer once at startup, ...",
    "sandbox": "get_sandbox_provider() caches the provider singleton ...",
    "log_level": "apply_logging_level() runs only during app.py startup; ...",
    "channels": "start_channel_service() is invoked once during startup; ...",
    ...
}
```

注意每个字段的值不是简单的 `True`,而是一段**解释"哪段代码抓了这个快照"**的理由文本。为什么?注释里写得很直白:这段文本会出现在 `Field(description=...)` 里,让运维在 IDE hover 时不仅看到"这个字段要重启",还看到"要重启哪个子系统才能让它生效"。这个理由文本是给人看的,但它的**位置**(注册表)是给机器用的。

真正把这份注册表"焊死"的是 [`test_reload_boundary.py`](../backend/tests/test_reload_boundary.py) 里的**双向漂移检测**:

```python
# 正向:注册表里的字段,schema 描述必须带 startup-only: 前缀
def test_appconfig_schema_marks_registered_fields_with_prefix():
    for field_path in STARTUP_ONLY_FIELDS:
        if field_path not in schema_fields: continue
        assert description.startswith(STARTUP_ONLY_PREFIX)

# 反向:schema 里带了 startup-only: 前缀的字段,注册表必须列出
def test_no_appconfig_field_uses_prefix_without_registration():
    for name, info in AppConfig.model_fields.items():
        if not description.startswith(STARTUP_ONLY_PREFIX): continue
        assert name in STARTUP_ONLY_FIELDS
```

**双向**是关键。正向防的是"注册表里写了、但 schema 忘了标"(操作工具读注册表,IDE hover 读 schema,两边得一致);反向防的是"schema 里标了 startup-only、但注册表忘了收录"——如果只有正向检查,有人在 schema 里给某字段加了 `startup-only:` 前缀却没更新注册表,那些消费注册表的运维脚本、文档生成器就会漏掉这个字段。两个方向都锁,才能保证"注册表"和"schema 描述"这两处永远同步,谁也别想单方面漂移。

这跟第十部分 Skills 的 `_BASELINE_TABLE_NAMES` 守卫测试(CLAUDE.md 里的 schema migration 一节)、第九部分 MCP 的配置别名归一化,是同一类"用测试锁住一个必须手工维护、又极易忘记同步的清单"的手法。**凡是"两处必须保持一致、但没有机制强制"的地方,就写个测试把它们钉在一起。**

## 5. 三条主线的共同思想:约束的"可执行化"

把三条放在一起看,DeerFlow 的工程可靠性哲学非常清晰:

| 约定 | 如果只靠文档 | 可执行化的形态 | 检测时机 |
|---|---|---|---|
| async 里不许阻塞 IO | 线上并发才暴露 | Blockbuster 运行时钩子 + 回归锚点 | 测试执行时(真实调用栈) |
| harness 不许 import app | review 漏看就腐化 | AST 扫描 import 语句 | CI(静态语法树) |
| 哪些配置要重启 | 只在作者脑子里 | 注册表 + 双向漂移测试 | CI(注册表 vs schema) |

三者的机制不同(运行时 monkey-patch / 静态 AST / 一致性断言),但都遵循同一个原则:**把一个"人容易违反、违反成本高、当下不报错"的约定,转化成一个"违反即失败"的确定性检查。** 这也回答了面试里一类高频开放题——"你怎么保证一个大型系统的工程质量不随迭代腐化":答案不是"加强 code review"或"写更详细的文档",而是**尽可能把每一条隐性约定变成一条显性的、会自己报警的测试**。文档会过时、review 会疲劳、人会忘记,但一个挂红的 CI 不会。

还值得一提的是这三条测试本身的"元设计":`blocking_io` 有个 `test_gate_smoke.py`(CLAUDE.md 提到)专门验证"门禁真的能抓到未 offload 的阻塞 IO、且 `allow_blocking_io` 逃生舱真的能放行"——**连"检测机制本身有没有失效"都要测**。一个不会失败的测试等于没有测试;所以要有一个 smoke test 证明门禁真的会在该失败的时候失败。这是可靠性工程里最容易被忽略、但最能体现成熟度的一层:**守卫者也需要被守卫。**

## 6. 小结:可靠性不是功能,是"防止功能坏掉的机制"

```
工程约定 ──编码成──▶ 可执行检查 ──运行于──▶ 违反即失败
─────────────────────────────────────────────────────────

① async 不许阻塞 IO
   blocking_io_runtime.py    BlockBuster(scanned_modules=("app","deerflow"))
   blocking_io/conftest.py   hookwrapper 包住 setup+call+teardown
   tests/blocking_io/*.py    每修一个 bug 加一个回归锚点(锁住 offload)
   test_gate_smoke.py        元测试:验证门禁本身真的会抓/会放行
        │
        └─▶ 精确锁定"业务代码引发的阻塞",忽略框架内部,避免假阳性

② harness 不许 import app
   test_harness_boundary.py  ast.parse 每个 .py → 遍历 import 节点
        │                    → 命中 app. 前缀 → assert 失败(带文件:行号)
        └─▶ 把分层规则从文档一句话变成 CI gate

③ 哪些配置要重启
   reload_boundary.py        STARTUP_ONLY_FIELDS 单一事实来源(字段→理由)
   test_reload_boundary.py   双向漂移:注册表⇄schema 前缀必须一致
        │                    正向(注册表→schema) + 反向(schema→注册表)
        └─▶ 锁住"两处必须同步、又极易忘记"的清单

共同原则:约束的可执行化
   人容易违反 + 违反成本高 + 当下不报错  ──▶  违反即失败的确定性检查
   文档会过时、review 会疲劳、人会忘记,但挂红的 CI 不会
```

这一部分是整份学习笔记的收尾,也是最能体现"工程成熟度"的一部分。前十一部分讲的是"DeerFlow 实现了哪些能力",这一部分讲的是"DeerFlow 怎么保证这些能力在多人、多次迭代之后不悄悄坏掉"。面试里如果被问到"你觉得一个生产级 agent 系统和一个 demo 的本质区别是什么",这三条就是最好的素材:**demo 关心"能不能跑通一次",生产系统关心"怎么保证一百个人改一年之后还跑得通"。** 而 DeerFlow 给出的答案始终如一——**别相信人的自觉,把约束写成测试。**
