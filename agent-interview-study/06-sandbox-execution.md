# Module 6：Sandbox 执行环境与安全

**核心文件**：
- [sandbox/sandbox.py](backend/packages/harness/deerflow/sandbox/sandbox.py) / [sandbox/sandbox_provider.py](backend/packages/harness/deerflow/sandbox/sandbox_provider.py)
- [sandbox/middleware.py](backend/packages/harness/deerflow/sandbox/middleware.py)
- [sandbox/tools.py](backend/packages/harness/deerflow/sandbox/tools.py)（1927 行，虚拟路径转换 + 安全校验 + 全部沙箱工具实现）
- [sandbox/local/local_sandbox.py](backend/packages/harness/deerflow/sandbox/local/local_sandbox.py) / [sandbox/local/local_sandbox_provider.py](backend/packages/harness/deerflow/sandbox/local/local_sandbox_provider.py)
- [sandbox/security.py](backend/packages/harness/deerflow/sandbox/security.py) / [sandbox/exceptions.py](backend/packages/harness/deerflow/sandbox/exceptions.py) / [sandbox/file_operation_lock.py](backend/packages/harness/deerflow/sandbox/file_operation_lock.py)
- [agents/middlewares/sandbox_audit_middleware.py](backend/packages/harness/deerflow/agents/middlewares/sandbox_audit_middleware.py)
- [community/aio_sandbox/aio_sandbox_provider.py](backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox_provider.py) / [aio_sandbox.py](backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox.py) / [local_backend.py](backend/packages/harness/deerflow/community/aio_sandbox/local_backend.py)

## 一、抽象接口 + Provider 模式：两层解耦

[sandbox.py:6-112](backend/packages/harness/deerflow/sandbox/sandbox.py#L6-L112) 定义了一份纯接口 `Sandbox(ABC)`：`execute_command`/`read_file`/`download_file`/`list_dir`/`write_file`/`glob`/`grep`/`update_file`。工具层（`tools.py`）永远只跟这个接口打交道，完全不知道背后是本地文件系统还是一个 Docker 容器。

**`download_file` 的文档字符串里藏着一个接口设计原则**（[L52-58](backend/packages/harness/deerflow/sandbox/sandbox.py#L52-L58)）：无论本地还是远程实现，路径穿越/越界都必须抛 `PermissionError`，文件读取失败都必须抛**同一个** `OSError`——不是各自发明一个 `SandboxFileError` 之类的自定义类型。这样调用方只需要 `except PermissionError` / `except OSError` 两种 catch，不用关心当前跑的是哪种沙箱实现。

真正的"选哪种实现"由 [sandbox_provider.py](backend/packages/harness/deerflow/sandbox/sandbox_provider.py) 这个**独立于 `Sandbox` 接口**的第二层抽象负责：`SandboxProvider.acquire/get/release` 描述"生命周期"，而不是"怎么执行命令"。[L60-74](backend/packages/harness/deerflow/sandbox/sandbox_provider.py#L60-L74) 的 `get_sandbox_provider()` 是个全局单例工厂，通过 `resolve_class(config.sandbox.use, SandboxProvider)`（复用 Module 里没细讲但贯穿全项目的 `reflection.resolve_class` 反射加载机制）在运行时决定具体是 `LocalSandboxProvider` 还是 `AioSandboxProvider`——**换沙箱后端只需要改一行 config，不用改一行调用代码**，这是"面向接口编程"在这个项目里最直接的体现。

## 二、`SandboxMiddleware`：懒加载状态怎么"逃出"本次调用

[middleware.py:27-49](backend/packages/harness/deerflow/sandbox/middleware.py#L27-L49)，`lazy_init=True`（默认）时不在 `before_agent` 抢先拿沙箱，而是等第一次真正需要工具时才在 `tools.py` 的 `ensure_sandbox_initialized()` 里惰性获取。问题是这个函数直接改的是 `runtime.state["sandbox"]`——**这是当前工具调用私有的局部字典**，LangGraph 的 channel reducer 根本看不到这次修改，下一步 graph 继续跑时状态又变回没有 sandbox。

[middleware.py:190-202](backend/packages/harness/deerflow/sandbox/middleware.py#L190-L202) 的 `wrap_tool_call` 就是补这个洞的地方：

```python
def wrap_tool_call(self, request, handler):
    prev_sandbox_id = self._read_sandbox_id_from_request(request)   # 调用前
    result = handler(request)                                       # 执行工具，可能内部惰性初始化了
    if prev_sandbox_id is not None:
        return result                                                # 调用前已有 sandbox，不是这次新获取的，不用管
    curr_sandbox_id = self._read_sandbox_id_from_request(request)   # 调用后
    if curr_sandbox_id is None:
        return result
    return self._attach_sandbox_update(result, curr_sandbox_id)     # 前面没有、后面有 -> 这次是首次懒加载
```

**"前后各读一次、做差"是这里唯一能判断"这次调用是不是刚做了懒加载"的办法**——因为 `ensure_sandbox_initialized` 本身不会告诉调用方"我刚刚初始化了"，middleware 只能通过状态快照的前后对比自己推断出来。判断成立后，[L161-179](backend/packages/harness/deerflow/sandbox/middleware.py#L161-L179) 的 `_attach_sandbox_update` 把原本要返回的 `ToolMessage` 包进 `Command(update={"sandbox": ..., "messages": [result]})`——只有 `Command.update` 才会真正走 LangGraph 的 reducer 合并进永久 state，这次获取到的 `sandbox_id` 才能被下游（`ToolOutputBudgetMiddleware`、子 agent 的 `task_tool`）看见。

## 三、虚拟路径系统：三层独立防御

模型只看到 `/mnt/user-data/{workspace,uploads,outputs}`、`/mnt/skills`、`/mnt/acp-workspace` 这几个"看起来像绝对路径"的虚拟前缀。三层防御各自独立，缺一层都会出问题：

**第一层——`validate_local_tool_path`（`tools.py`）：只判断"能不能碰"，不做路径转换**：
```python
def validate_local_tool_path(path, thread_data, *, read_only=False) -> None:
    _reject_path_traversal(path)                 # 先拒绝任何 ".." 段
    if _is_skills_path(path):
        if not read_only: raise PermissionError(...)   # skills 目录永远只读
        return
    if _is_acp_workspace_path(path):
        if not read_only: raise PermissionError(...)   # ACP 工作区永远只读
        return
    if path.startswith(f"{VIRTUAL_PATH_PREFIX}/"):
        return                                          # user-data 下允许读写
    ...
    raise PermissionError(f"Only paths under {VIRTUAL_PATH_PREFIX}/ ... allowed")
```

**第二层——`replace_virtual_path`：真正做前缀替换**，把 `/mnt/user-data/workspace/foo.txt` 换成这个 thread 在主机上的实际目录。这一层**故意不做安全校验**，它假设第一层已经拦过了。

**第三层——`LocalSandbox._resolve_path_with_mapping` 里的 `relative_to()` 二次校验**（[local_sandbox.py](backend/packages/harness/deerflow/sandbox/local/local_sandbox.py)）：
```python
try:
    resolved_path.relative_to(local_root)
except ValueError as exc:
    raise PermissionError(errno.EACCES, "Access denied: path escapes mounted directory", path_str) from exc
```
即使前两层都被绕过（比如一个精心构造的符号链接，或者转换逻辑本身有 bug），**解析出来的最终物理路径**如果 `relative_to()` 校验发现不在挂载根目录之内，还是会被拦下来。这是"输入校验 + 输出校验各做一次"的经典纵深防御思路，跟 Module 4 讲过的"隐藏 schema + 拦截调用"是同一种直觉：**任何单点防御都可能被绕过，所以关键操作前后都要设卡**。

反方向的输出侧还有 `mask_local_paths_in_output`（[tools.py](backend/packages/harness/deerflow/sandbox/tools.py)）：把 bash 命令的 stdout/stderr 里出现的物理路径**逆向替换**回虚拟路径，防止 `backend/.deer-flow/users/xxx/threads/xxx/...` 这样的主机文件系统布局泄露给模型看到——这跟前面三层"防止越界写"的方向正好相反，是"防止越界看"。

## 四、`LocalSandbox`：懒缓存 + 非对称路径还原

`local_sandbox.py` 里一批 `@cached_property`（`_command_pattern`、`_content_pattern`、`_mappings_by_container_specificity` 等）都在缓存"从 `path_mappings` 编译出来的正则/排序结果"。这样设计成立的前提是 `path_mappings` 在 `__init__` 之后**绝不再变**——一份挂载映射对应一个 `LocalSandbox` 实例，不用每次 bash/read_file 调用都重新排序、重新编译正则，这是 agent 热路径上一处很直接的性能优化。

**`_agent_written_paths`（一个 `set[str]`）** 是这个文件里最容易被忽略、但设计意图很明确的一处细节：
```python
def read_file(self, path: str) -> str:
    resolved_path = self._resolve_path(path)
    content = open(resolved_path, encoding="utf-8").read()
    if resolved_path in self._agent_written_paths:      # 只有"agent 自己写过"的文件才做反向替换
        content = self._reverse_resolve_paths_in_output(content)
    return content
```
`read_file` 只对**之前由 agent 自己通过 `write_file` 写入过**的文件内容做路径逆向替换（把文件内容里可能出现的物理路径换回虚拟路径）；用户上传的文件、外部工具产出的文件内容**故意不做**这个替换。这是 PR #1935 讨论后定下的设计——不应该悄悄改写不是 agent 自己产出的内容，即使改写的目的是"看起来更干净"。

## 五、`LocalSandboxProvider`：按线程隔离 + LRU 缓存

`LocalSandboxProvider` 的演进史本身就是一个面试可以讲的故事（[local_sandbox_provider.py:34-63](backend/packages/harness/deerflow/sandbox/local/local_sandbox_provider.py#L34-L63) 的类文档字符串直接写明了）：**早期版本是一个进程级单例**，`id="local"`，但这满足不了"`/mnt/user-data/...` 应该指向这个 thread 自己的目录"这个契约——因为物理目录本身就是 per-thread 的（`.../threads/{thread_id}/user-data/`）。现在改成 `acquire(thread_id)` 返回 `id=f"local:{thread_id}"` 的**每线程专属实例**，`path_mappings` 里带上这个 thread 的四个目录映射（workspace/uploads/outputs/acp-workspace）。

**为什么不是"无限增长的 dict"，而是 `OrderedDict` + LRU**（[L57-80](backend/packages/harness/deerflow/sandbox/local/local_sandbox_provider.py#L57-L80)）：一个长期运行的 gateway 进程里 thread_id 的总数是无界的，每个 `LocalSandbox` 实例本身很轻（一份 mapping 列表 + 一个 set），但攒够了也是内存泄露。`DEFAULT_MAX_CACHED_THREAD_SANDBOXES=256`，超过就 `popitem(last=False)` 驱逐最久未用的那个（[L282-293](backend/packages/harness/deerflow/sandbox/local/local_sandbox_provider.py#L282-L293)）。**被驱逐的代价很小**：下次这个 thread 再来，`acquire` 简单地重建一个新实例，唯一丢失的是 `_agent_written_paths`（上一节讲的"非对称路径还原"提示），退化成"跟全新 session 一样不做反向替换"，功能上不会出错，只是体验上稍微降级——这也是为什么这个 cap 敢设得比看起来"应该无限大"更小的原因：**驱逐的失败模式是优雅退化，不是数据丢失**。

`release()` 被特意留空（[L316-325](backend/packages/harness/deerflow/sandbox/local/local_sandbox_provider.py#L316-L325)）：`LocalSandbox` 没有需要释放的外部资源（不像 Docker 容器），所以"释放"这个动作在这里等价于"什么都不做，继续留在缓存里"——这样才能让 `_agent_written_paths` 之类的状态跨多轮对话存活，直到真正被 LRU 淘汰或 `reset()`。

## 六、输出体积治理：写入上限 + 两种截断策略

[SandboxConfig](backend/packages/harness/deerflow/config/sandbox_config.py) 里三个字符数上限（`bash_output_max_chars=20000`、`read_file_output_max_chars=50000`、`ls_output_max_chars=20000`）分别对应 `tools.py` 里两种**不同方向**的截断策略：

- **bash 输出 / write_file 错误详情 → 中间截断**（保留首尾各一半）：命令的有效信息可能在开头（命令回显）也可能在结尾（最终报错），顺序不可预知，只能两头都留。
- **read_file / ls 输出 → 头部截断**：源码是从上往下读的，文件的 import/类定义、目录的顶层结构在最前面最重要，只保留开头符合阅读习惯。

`write_file` 还有一道单独的**写入前**大小闸门（issue #3189）：
```python
_WRITE_FILE_CONTENT_MAX_BYTES = 80 * 1024
if not append and content_bytes > max_bytes:
    return f"Error: write_file content ({content_bytes} bytes) exceeds the {max_bytes}-byte single-call limit. ..."
```
根因是观测到的真实故障模式——模型一次性塞一个很大的 Markdown 文件，`content` 参数里的引号/转义没处理好，直接整段 JSON 解析失败（这正是 Module 3 `DanglingToolCallMiddleware` 里那条专属 `write_file` 恢复提示要处理的场景）。这里选择**从源头限流**：超过 80KB 就直接拒绝，引导模型用 `str_replace` 做增量编辑，或者用 `append=True` 分段追加——把"防止畸形大 payload"这件事提前到写入之前，而不是等它失败了再补救。

## 七、Host Bash 的安全开关：两层独立防御

`sandbox/security.py` 定义了一个很直白的原则——**`LocalSandboxProvider` 不是一个安全边界**（[L10-14](backend/packages/harness/deerflow/sandbox/security.py#L10-L14) 的错误文案直接写明）：本地沙箱执行的命令跟运行 gateway 进程的用户权限完全相同，没有任何隔离。所以：

```python
def is_host_bash_allowed(config=None) -> bool:
    if not uses_local_sandbox_provider(config):
        return True                                    # AioSandboxProvider(Docker) 天然隔离，默认允许
    return bool(getattr(sandbox_cfg, "allow_host_bash", False))   # Local 必须显式 opt-in
```
用的是哪种 provider 决定了默认策略是否安全——**AIO/Docker 默认允许**（有容器边界），**Local 默认拒绝**，只有运营者在 `config.yaml` 里显式打开 `sandbox.allow_host_bash: true` 才放行，这是"危险操作默认关闭，需要有意识选择才能打开"的典型 fail-closed 设计。

**即便打开了 `allow_host_bash`**，`SandboxAuditMiddleware`（[sandbox_audit_middleware.py](backend/packages/harness/deerflow/agents/middlewares/sandbox_audit_middleware.py)）还会在每次 `bash` 调用前做第二层过滤，这是**独立于**"是否允许 host bash"的另一条防线：

```python
def _classify_command(command: str) -> str:
    normalized = " ".join(command.split())
    for pattern in _HIGH_RISK_PATTERNS:            # 第一遍：整条原始命令扫描高危模式
        if pattern.search(normalized):
            return "block"                          # 命中 rm -rf /、fork bomb、LD_PRELOAD 劫持等直接拒绝
    sub_commands = _split_compound_command(command)  # 第二遍：按 ; && || 拆分子命令，各自分类
    worst = "pass"
    for sub in sub_commands:
        verdict = _classify_single_command(sub)
        if verdict == "block": return "block"
        if verdict == "warn": worst = "warn"        # chmod 777 / sudo / PATH= 修改等只警告不拦截
    return worst
```

**为什么先对整条原始字符串扫一遍，再拆分扫第二遍**：像 fork bomb `:(){ :|:& };:` 这类攻击本身就是靠多个语句拼接才成立的，如果先拆分成子命令再逐个判断，拆分动作本身就把攻击模式破坏了，扫不到。所以第一遍必须在"没有被拆分"的完整字符串上跑。

**`_split_compound_command` 的 fail-closed 兜底**：这是一个手写的、追踪单/双引号状态和转义字符的逐字符解析器，如果扫到结尾发现引号没闭合或者转义符悬空，直接**放弃拆分，把整条字符串当成一个不可分割的命令**去分类——宁可牺牲"精确定位到具体是哪个子命令危险"的能力，也不能因为解析失败而把一部分子命令漏检。

`_validate_input`（拒绝空命令 / 超过 10000 字符 / 含 null byte）**在风险分类之前先跑**，理由是这类畸形输入本身就是可疑信号（多半是 payload 注入或 base64 编码攻击串），不管它是否恰好命中某条正则都值得记录。**每一次 bash 调用无论 block/warn/pass 都会被结构化日志记录**——这跟"是否放行这次调用"是两件独立的事：审计永远发生，拦截只是审计后的一个可能结果。

## 八、AioSandboxProvider：Docker 沙箱的生命周期编排

如果 Local 沙箱的关键词是"路径映射 + 权限校验"，AIO（Docker-based）沙箱的关键词就是**跨进程一致性**——因为一个 gateway 可能有多个 worker 进程，甚至跑在多个 Pod 上，它们必须能对"这个 thread 的沙箱容器在哪"达成一致，而不依赖共享内存。

**确定性容器命名**（[aio_sandbox_provider.py:278-285](backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox_provider.py#L278-L285)）：
```python
def _deterministic_sandbox_id(thread_id: str) -> str:
    return hashlib.sha256(thread_id.encode()).hexdigest()[:8]
```
`sandbox_id` 是 `thread_id` 的哈希，不是随机 UUID——**这一个设计决定就解决了跨进程发现问题**：任何进程只要知道 `thread_id`，就能推算出同一个容器名（`{prefix}-{sandbox_id}`），不需要查任何共享状态存储，直接 `docker ps --filter name=...` 就能找到别的进程已经启动的容器（[local_backend.py:346-386](backend/packages/harness/deerflow/community/aio_sandbox/local_backend.py#L346-L386) 的 `discover()`）。

**三层查找顺序**（[L686-713](backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox_provider.py#L686-L713) `_acquire_internal`）：
1. **进程内缓存**（`_thread_sandboxes` dict）——最快，覆盖同进程重复访问
2. **Warm pool**（`_warm_pool`）——容器还在跑但没被任何 thread 占用，不用冷启动
3. **跨进程发现 + 创建**——用文件锁序列化，靠 Docker 层面的 `discover()` 检测别的进程是否已经建好了

**为什么 release() 不真正停止容器**（[L883-922](backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox_provider.py#L883-L922)）：`release` 只是把 `SandboxInfo` 挪进 warm pool，容器继续跑。同一个 thread 下一轮再来能直接复用，避免冷启动（拉镜像/挂载卷/等待 health check 这些都要秒级时间）。只有 warm pool 里的容器**空闲超过 `idle_timeout`**（默认 600 秒，由后台的 `_idle_checker_loop` 每 60 秒扫一次），或者 `replicas` 容量不够时被强制腾位置，才会真正 `destroy`。

**`replicas` 是软上限，只淘汰 warm pool，绝不杀活跃容器**（[L627-643](backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox_provider.py#L627-L643)）：
```python
def _log_replicas_soft_cap(self, replicas, sandbox_id, evicted):
    if evicted:
        logger.info(f"Evicted warm-pool sandbox {evicted} to stay within replicas={replicas}")
        return
    logger.warning(f"All {replicas} replica slots are in active use; creating sandbox {sandbox_id} beyond the soft limit")
```
如果 warm pool 里没有可淘汰的容器（所有槎位都被活跃 thread 占着），**代码选择继续超额创建**，只打一条 warning 日志，而不是拒绝服务——因为这里的核心不变量是"绝不能强行踢掉一个正在被某个 thread 使用的容器"，`replicas` 的约束力必须让位于"不能弄坏正在跑的会话"这条更硬的规则。

**孤儿回收（`_reconcile_orphans`，[L233-274](backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox_provider.py#L233-L274)）**：进程重启（崩溃/SIGKILL/滚动升级）会让 Docker 容器"没有对应的内存状态"却继续跑着。启动时枚举所有匹配前缀的运行中容器，**不加区分地全部放进 warm pool**——代码注释直接承认了这里的判断局限：无法仅凭"运行时长"区分"这是孤儿"还是"这是另一个进程正在用的活跃容器"，所以干脆都当成 warm pool 候选，交给 idle checker 按空闲时长决定是否回收，而不是贸然销毁可能还在被用的容器。

**跨进程文件锁**（[L735-765](backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox_provider.py#L735-L765)，`fcntl.flock`/`msvcrt.locking` 双平台实现）：两个进程同时给同一个 `thread_id` 创建沙箱时，谁都可能先跑到"发现容器不存在 → 决定创建"这一步，如果不加锁，Docker 会报"容器名已存在"的冲突。文件锁把这段"发现-创建"临界区序列化，第二个进程拿到锁之后重新检查一遍缓存/发现流程，大概率会看到第一个进程刚建好的容器，直接复用而不是报错——这跟 `LocalSandboxProvider` 用 `threading.Lock` 做进程内一致性是同一类问题的两个不同尺度（**同进程用内存锁，跨进程用文件锁**）。

## 九、`AioSandbox`：单一持久 Shell 会话的腐化与恢复

AIO 容器内部维护的是**一个持久的 shell session**（不是每次命令都开一个新进程），这样才能支持 `cd` 之后状态延续、环境变量累积等交互式 shell 语义。但这意味着并发是致命的：

```python
def execute_command(self, command: str) -> str:
    with self._lock:      # 用锁把并发请求序列化到这一个 session 上
        result = self._client.shell.exec_command(command=command, ...)
        output = result.data.output if result.data else ""
        if output and _ERROR_OBSERVATION_SIGNATURE in output:
            # 即便加了锁，仍然检测到腐化迹象 -> 创建一次性恢复 session 重试
            fresh_id = str(uuid.uuid4())
            self._client.shell.create_session(id=fresh_id)
            result = self._client.shell.exec_command(command=command, id=fresh_id, ...)
            self._client.shell.cleanup_session(fresh_id)
        return output
```
`self._lock` 挡住了同一个 `AioSandbox` 实例内部的并发（比如同一个 thread 里 subagent 和主 agent 都想跑 bash），但注释也承认了：**如果多个进程共享同一个沙箱容器**，锁挡不住跨进程并发，session 仍可能被并发请求打乱返回 `ErrorObservation`。这里的兜底是"检测到损坏信号就用一个全新的一次性 session 重跑，跑完立刻清理"——**不修复被污染的持久 session，直接绕开它**，这是"检测已知失败模式 + 换一条干净路径重试"而不是"试图诊断并修复"的选择，因为诊断一个已经损坏的交互式 shell session 状态成本远高于简单重开一个。

`close()`（[L48-97](backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox.py#L48-L97)）是个很典型的"三方 SDK 没给你需要的钩子，只能沿着私有属性链爬"的真实案例：`agent_sandbox`（Fern 自动生成的 SDK）没暴露 `close()`，代码只能顺着 `Sandbox._client_wrapper.httpx_client.httpx_client` 这条属性链，找到真正持有 socket 的那个 `httpx.Client` 手动关闭——写法上做了"最具体优先、逐级降级"（先找最底层的真实 httpx client，找不到就退到外层 wrapper），并且假设未来 SDK 版本可能会加上标准 `close()`，那时这段代码会自动优先命中新接口而不需要改动。

## 十、总结表

| 组件 | 角色 |
|---|---|
| `Sandbox` 抽象接口 | 定义统一异常契约（`PermissionError`/`OSError`），让调用方不用关心具体实现 |
| `SandboxProvider` + `get_sandbox_provider()` | acquire/get/release 生命周期抽象；通过 `resolve_class` 反射切换 Local/AIO 实现 |
| `SandboxMiddleware.wrap_tool_call` | 用前后状态差异检测懒加载，通过 `Command(update=...)` 把局部 state 修改回写进 graph 状态 |
| 虚拟路径三层防御 | `validate_local_tool_path`（判断能不能碰）→ `replace_virtual_path`（转换）→ `relative_to()`（解析后二次校验越界） |
| `_agent_written_paths` | 只对 agent 自己写过的文件做路径反向替换，不改写用户/外部内容 |
| `LocalSandboxProvider` LRU 缓存 | 每线程专属 `LocalSandbox`，256 上限防内存泄露，驱逐后优雅退化而非报错 |
| `write_file` 80KB 上限 + 中间/头部截断 | 从源头限制畸形大 payload；输出截断策略按"信息在哪端"区分两种策略 |
| `allow_host_bash` + `SandboxAuditMiddleware` | 两层独立防御：Provider 级默认拒绝 host bash；命令级两遍扫描（整体优先于拆分）分类拦截/警告 |
| `AioSandboxProvider` 确定性 ID | `sha256(thread_id)[:8]` 让任意进程独立推算出同一容器名，无需共享状态即可跨进程发现 |
| Warm pool + `replicas` 软上限 | release 不销毁容器，只挪进 warm pool；容量不足只淘汰 warm pool，绝不误杀活跃容器 |
| 跨进程文件锁 + 孤儿回收 | 文件锁序列化"发现或创建"临界区；启动时无差别把所有匹配容器纳入 warm pool，交给 idle checker 甄别 |
| `AioSandbox` session 腐化恢复 | 锁只挡本进程内并发；检测到 `ErrorObservation` 后用一次性新 session 重跑，而非修复被污染的持久 session |
