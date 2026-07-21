# 第十四部分：沙箱工具与 LocalSandbox —— 路径映射、工具实现、并发控制

> 本篇是对第 06、13 篇的补充。第 06 篇讲沙箱抽象层和安全策略,第 13 篇讲 AioSandbox 的 Docker 容器实现;本篇聚焦 **LocalSandbox 的路径映射机制、tools.py 的 1926 行工具实现、LocalSandboxProvider 的 LRU 缓存、SandboxMiddleware 的 lazy init、search.py 的忽略规则、file_operation_lock 的并发控制** —— 这些是"agent 真正调用沙箱时发生什么"的完整链路。
>
> 读完本篇你应能回答:LocalSandbox 不用 Docker 怎么做到路径隔离?`bash_tool` 的完整执行流程是什么?为什么 LocalSandboxProvider 用 LRU 缓存而 AioSandboxProvider 用 warm pool?SandboxMiddleware 的 lazy init 怎么和 LangGraph state 交互?

## 1. LocalSandbox:不用 Docker 也能做到路径隔离

LocalSandbox 是"宿主机直接跑"的沙箱,**不启动任何容器**,但它不是"裸奔" —— 它有一套**路径映射(PathMapping)**机制,让 agent 看到的虚拟路径(`/mnt/user-data/workspace`)和宿主机真实路径(`~/.deer-flow/threads/abc123/user-data/workspace`)分离。

### 1.1 核心数据结构:PathMapping

[local_sandbox.py:21-27](../backend/packages/harness/deerflow/sandbox/local/local_sandbox.py#L21-L27):

```python
@dataclass(frozen=True)
class PathMapping:
    container_path: str      # 虚拟路径,如 /mnt/user-data/workspace
    local_path: str          # 宿主机真实路径,如 /Users/ryan/.deer-flow/threads/abc/user-data/workspace
    read_only: bool = False  # 是否只读
```

每个 LocalSandbox 持有一个 `path_mappings` 列表,所有的文件操作(read/write/execute_command)都要经过**正向解析**(虚拟路径 → 真实路径)和**反向解析**(真实路径 → 虚拟路径)。

### 1.2 正向解析:agent 给虚拟路径,转成真实路径

[local_sandbox.py:159-200](../backend/packages/harness/deerflow/sandbox/local/local_sandbox.py#L159-L200) 的 `_resolve_path_with_mapping`:

```python
def _resolve_path_with_mapping(self, path: str) -> ResolvedPath:
    # path = "/mnt/user-data/workspace/test.py"
    
    # 1. 找匹配的 PathMapping(按 container_path 长度排序,最长优先)
    mapping, relative = self._find_path_mapping(path)
    # mapping = PathMapping(container_path="/mnt/user-data/workspace", local_path="/Users/.../workspace")
    # relative = "test.py"
    
    # 2. 拼接真实路径
    local_root = Path(mapping.local_path)
    resolved_path = (local_root / relative).resolve()
    # resolved_path = /Users/ryan/.deer-flow/threads/abc/user-data/workspace/test.py
    
    # 3. 防路径逃逸:resolved_path 必须在 local_root 之下
    resolved_path.relative_to(local_root)  # 如果 ../../.. 逃逸了,这里抛 ValueError
    
    return ResolvedPath(str(resolved_path), mapping)
```

**关键**:
- **最长前缀匹配**:`/mnt/user-data/workspace` 比 `/mnt/user-data` 更具体,优先匹配([L162-171](../backend/packages/harness/deerflow/sandbox/local/local_sandbox.py#L162-L171))
- **防路径逃逸**:`resolved_path.relative_to(local_root)` 确保 `../../../etc/passwd` 这种路径逃逸会被拒绝([L196-198](../backend/packages/harness/deerflow/sandbox/local/local_sandbox.py#L196-L198))

### 1.3 反向解析:工具输出里的真实路径,转回虚拟路径

[local_sandbox.py:208-254](../backend/packages/harness/deerflow/sandbox/local/local_sandbox.py#L208-L254) 的 `_reverse_resolve_path` 和 `_reverse_resolve_paths_in_output`:

```python
def _reverse_resolve_paths_in_output(self, output: str) -> str:
    # output = "total 8\n-rw-r--r-- 1 user staff 123 Jan 1 /Users/ryan/.deer-flow/threads/abc/workspace/test.py"
    
    # 用正则匹配所有真实路径,替换成虚拟路径
    for pattern in self._reverse_output_patterns:
        result = pattern.sub(replace_match, output)
    
    # result = "total 8\n-rw-r--r-- 1 user staff 123 Jan 1 /mnt/user-data/workspace/test.py"
```

**为什么要反向解析**:`ls -la` 这种命令的输出会包含真实路径,如果直接返回给 agent,agent 会看到宿主机的真实路径(`/Users/ryan/.deer-flow/...`),破坏了"虚拟路径"的抽象。所以要**把输出里的真实路径替换回虚拟路径**。

### 1.4 内容里的路径替换:write_file 时替换虚拟路径

[local_sandbox.py:276-302](../backend/packages/harness/deerflow/sandbox/local/local_sandbox.py#L276-L302) 的 `_resolve_paths_in_content`:

```python
def _resolve_paths_in_content(self, content: str) -> str:
    # content = "import pandas as pd\ndf = pd.read_csv('/mnt/user-data/uploads/data.csv')"
    
    # 把内容里的虚拟路径替换成真实路径
    pattern = self._content_pattern
    resolved = pattern.sub(replace_match, content)
    
    # resolved = "import pandas as pd\ndf = pd.read_csv('/Users/ryan/.deer-flow/threads/abc/uploads/data.csv')"
```

**为什么需要**:agent 写 Python 脚本时,代码里会引用虚拟路径(`pd.read_csv('/mnt/user-data/uploads/data.csv')`),但这个脚本要在宿主机上跑,必须把内容里的虚拟路径替换成真实路径。

### 1.5 只替换 agent 写的文件:`_agent_written_paths`

[local_sandbox.py:84-86](../backend/packages/harness/deerflow/sandbox/local/local_sandbox.py#L84-L86):

```python
# Track files written through write_file so read_file only
# reverse-resolves paths in agent-authored content.
self._agent_written_paths: set[str] = set()
```

**为什么需要**:`read_file` 读文件时,如果文件里有真实路径(比如用户上传的文件里写了 `/Users/ryan/data.csv`),要不要替换成虚拟路径?

- **agent 写的文件**:要替换(因为 write_file 时替换过虚拟路径 → 真实路径,read 时要替换回来)
- **用户上传的文件**:不替换(用户写的是真实路径,不该改)

所以用 `_agent_written_paths` 记录"哪些文件是 agent 写的",`read_file` 只对这些文件做反向解析([L398-400](../backend/packages/harness/deerflow/sandbox/local/local_sandbox.py#L398-L400))。

## 2. tools.py:agent 真正调用的 5 个工具

[tools.py](../backend/packages/harness/deerflow/sandbox/tools.py) 有 **1926 行**,实现了 7 个工具:

| 工具 | 行数 | 干什么 |
|---|---|---|
| `bash_tool` | L1394-1440 | 执行 bash 命令 |
| `ls_tool` | L1450-1497 | 列目录 |
| `glob_tool` | L1504-1554 | 按 pattern 找文件 |
| `grep_tool` | L1576-1646 | 搜索文件内容 |
| `read_file_tool` | L1672-1733 | 读文件 |
| `write_file_tool` | L1763-1843 | 写文件 |
| `str_replace_tool` | L1856-1907 | 字符串替换(编辑文件) |

### 2.1 bash_tool 的完整流程

[tools.py:1394-1440](../backend/packages/harness/deerflow/sandbox/tools.py#L1394-L1440):

```python
@tool("bash", parse_docstring=True)
def bash_tool(runtime: Runtime, description: str, command: str) -> str:
    try:
        # 1. 确保沙箱已初始化(lazy init)
        sandbox = ensure_sandbox_initialized(runtime)
        
        # 2. 如果是 LocalSandbox,检查是否允许 bash
        if is_local_sandbox(runtime):
            if not is_host_bash_allowed():
                return f"Error: {LOCAL_HOST_BASH_DISABLED_MESSAGE}"
            
            # 3. 验证命令里的路径是否合法
            thread_data = get_thread_data(runtime)
            validate_local_bash_command_paths(command, thread_data)
            
            # 4. 把命令里的虚拟路径替换成真实路径
            command = replace_virtual_paths_in_command(command, thread_data)
            
            # 5. 执行命令
            output = sandbox.execute_command(command)
            
            # 6. 把输出里的真实路径替换回虚拟路径
            output = mask_local_paths_in_output(output, thread_data)
            
            # 7. 截断输出(防止超长)
            return _truncate_bash_output(output, max_chars=20000)
        
        # AioSandbox 直接执行
        return _truncate_bash_output(sandbox.execute_command(command), max_chars)
    
    except SandboxError as e:
        return f"Error: {e}"
    except PermissionError as e:
        return f"Error: {e}"
```

**关键步骤**:
1. **lazy init**:第一次调用时才 acquire 沙箱(`ensure_sandbox_initialized`)
2. **安全检查**:LocalSandbox 默认禁用 bash(`is_host_bash_allowed`)
3. **路径验证**:命令里的路径必须在允许范围内(`/mnt/user-data`、`/mnt/skills` 等)
4. **正向替换**:命令里的虚拟路径 → 真实路径
5. **执行**:调 `sandbox.execute_command`
6. **反向替换**:输出里的真实路径 → 虚拟路径
7. **截断**:防止输出超长(默认 20000 字符)

### 2.2 路径验证:validate_local_bash_command_paths

[tools.py:994-1036](../backend/packages/harness/deerflow/sandbox/tools.py#L994-L1036):

```python
def validate_local_bash_command_paths(command: str, thread_data: ThreadDataState | None) -> None:
    # 解析 bash 命令,提取所有路径
    tokens = _split_shell_tokens(command)
    
    for token in tokens:
        # 跳过 URL、环境变量、选项等
        if _is_non_file_url_token(token):
            continue
        
        # 检查是否包含 .. 路径穿越
        if _has_dotdot_path_segment(token):
            raise PermissionError("Path traversal detected")
        
        # 检查是否在允许的路径范围内
        if token.startswith("/"):
            validate_local_tool_path(token, thread_data, read_only=False)
```

**防什么**:
- `cat ../../../etc/passwd` → 路径穿越,拒绝
- `rm -rf /` → 绝对路径不在允许范围,拒绝
- `cat /mnt/user-data/workspace/file.txt` → 允许

### 2.3 虚拟路径替换:replace_virtual_paths_in_command

[tools.py:1038-1084](../backend/packages/harness/deerflow/sandbox/tools.py#L1038-L1084):

```python
def replace_virtual_paths_in_command(command: str, thread_data: ThreadDataState | None) -> str:
    # command = "cat /mnt/user-data/workspace/test.py"
    
    # 用正则匹配所有 /mnt/user-data/... 路径
    # 替换成真实路径
    # result = "cat /Users/ryan/.deer-flow/threads/abc/workspace/test.py"
```

**为什么要替换**:agent 看到的是虚拟路径(`/mnt/user-data/...`),但 LocalSandbox 在宿主机上跑,必须用真实路径。

### 2.4 write_file 的 80KB 限制

[tools.py:53-60](../backend/packages/harness/deerflow/sandbox/tools.py#L53-L60):

```python
# Maximum bytes accepted in a single non-append write_file call (issue #3189).
# Oversized single-shot writes correlate with LLM streaming chunk-gap timeouts
# because the tool-call JSON payload (which the model must emit as one
# continuous stream) grows past the safe window. 80 KB ≈ 20K tokens, a
# comfortable headroom under the factory-default 240s stream_chunk_timeout.
_WRITE_FILE_CONTENT_MAX_BYTES = 80 * 1024
```

**为什么限制**:LLM 生成 tool_call 的 JSON payload 是**一次性流式输出**的,如果 content 太大(比如 1MB),LLM 可能在生成 JSON 的中途超时,导致 tool_call 失败。80KB 约等于 20K tokens,在默认 240 秒超时内是安全的。

## 3. LocalSandboxProvider 的 LRU 缓存

[local_sandbox_provider.py](../backend/packages/harness/deerflow/sandbox/local/local_sandbox_provider.py) 和 AioSandboxProvider 的 warm pool 不同,它用的是 **LRU 缓存**:

```python
self._thread_sandboxes: OrderedDict[str, LocalSandbox] = OrderedDict()
self._max_cached_threads = 256  # 默认最多缓存 256 个 thread 的沙箱
```

**为什么是 LRU 不是 warm pool**:
- LocalSandbox 不是容器,是**轻量级 Python 对象**(只有 path_mappings 和 _agent_written_paths)
- 创建成本低(不需要 docker run),所以不需要 warm pool
- 但 thread 数量可能无限增长,所以要 LRU 淘汰最老的

**LRU 实现**([local_sandbox_provider.py:258-280](../backend/packages/harness/deerflow/sandbox/local/local_sandbox_provider.py#L258-L280)):

```python
with self._lock:
    cached = self._thread_sandboxes.get(thread_id)
    if cached is not None:
        # 标记为最近使用
        self._thread_sandboxes.move_to_end(thread_id)
        return cached.id

# 缓存满了,淘汰最老的
while len(self._thread_sandboxes) > self._max_cached_threads:
    evicted_thread_id, _ = self._thread_sandboxes.popitem(last=False)  # last=False = 最老的
```

**和 AioSandboxProvider 的对比**:

| | LocalSandboxProvider | AioSandboxProvider |
|---|---|---|
| 缓存什么 | LocalSandbox 对象(轻量) | Docker 容器(重量) |
| 缓存策略 | LRU(满了淘汰最老的) | Warm pool(idle timeout + LRU) |
| 默认容量 | 256 个 thread | 3 个容器 |
| 创建成本 | 低(new 一个对象) | 高(docker run) |

**release 是 no-op**([local_sandbox_provider.py:316-325](../backend/packages/harness/deerflow/sandbox/local/local_sandbox_provider.py#L316-L325)):

```python
def release(self, sandbox_id: str) -> None:
    # LocalSandbox has no resources to release; keep the cached instance so
    # that ``_agent_written_paths`` (used to reverse-resolve agent-authored
    # file contents on read) survives between turns.
    pass
```

因为 LocalSandbox 没有资源要释放(不是容器),所以 `release` 什么都不做,缓存的实例保留在 LRU 里,下次同一个 thread 直接用。

## 4. SandboxMiddleware:lazy init 和状态持久化

[middleware.py](../backend/packages/harness/deerflow/sandbox/middleware.py) 是 agent middleware,负责在 agent 启动时 acquire 沙箱、结束时 release。

### 4.1 Lazy init:第一次工具调用时才 acquire

[middleware.py:40-49](../backend/packages/harness/deerflow/sandbox/middleware.py#L40-L49):

```python
def __init__(self, lazy_init: bool = True):
    self._lazy_init = lazy_init
```

**为什么 lazy**:
- 如果 agent 只是聊天,不调工具,就不需要沙箱
- Lazy init 可以避免"每个请求都 acquire 沙箱"的开销

**两种模式**:
- `lazy_init=True`(默认):第一次工具调用时才 acquire(`ensure_sandbox_initialized` in tools.py)
- `lazy_init=False`:agent 启动时就 acquire(`before_agent` hook)

### 4.2 状态持久化:把 sandbox_id 写进 LangGraph state

[middleware.py:190-217](../backend/packages/harness/deerflow/sandbox/middleware.py#L190-L217) 的 `wrap_tool_call`:

```python
def wrap_tool_call(self, request: ToolCallRequest, handler: Callable) -> ToolMessage | Command:
    # 1. 工具调用前,读 sandbox_id
    prev_sandbox_id = self._read_sandbox_id_from_request(request)
    
    # 2. 执行工具
    result = handler(request)
    
    # 3. 工具调用后,再读 sandbox_id
    curr_sandbox_id = self._read_sandbox_id_from_request(request)
    
    # 4. 如果之前没有、现在有了(lazy init 发生了),把 sandbox_id 写进 state
    if prev_sandbox_id is None and curr_sandbox_id is not None:
        return self._attach_sandbox_update(result, curr_sandbox_id)
    
    return result
```

**为什么需要**:lazy init 是在工具调用内部发生的(`ensure_sandbox_initialized`),LangGraph 的 state reducer 不知道 sandbox_id 变了,所以要**显式包一层 Command,把 sandbox_id 写进 state**。

### 4.3 Release:对话结束时 release 沙箱

[middleware.py:100-133](../backend/packages/harness/deerflow/sandbox/middleware.py#L100-L133):

```python
def after_agent(self, state: SandboxMiddlewareState, runtime: Runtime) -> dict | None:
    sandbox = state.get("sandbox")
    if sandbox is not None:
        sandbox_id = sandbox["sandbox_id"]
        get_sandbox_provider().release(sandbox_id)
```

**注意**:LocalSandboxProvider 的 `release` 是 **no-op**(因为 LocalSandbox 没有资源要释放),AioSandboxProvider 的 `release` 才会把容器放进 warm pool。

## 5. search.py:glob/grep 的忽略规则

[search.py](../backend/packages/harness/deerflow/sandbox/search.py) 实现了 `find_glob_matches` 和 `find_grep_matches`,都有**忽略规则**:

```python
IGNORE_PATTERNS = [
    ".git", "node_modules", "__pycache__", ".venv", "dist", "build", ...
]
```

**为什么需要**:agent 调 `glob("**/*.py")` 或 `grep("TODO")` 时,不该搜 `node_modules`、`.git` 这些目录(太大、没用)。

**性能优化**([search.py:70-85](../backend/packages/harness/deerflow/sandbox/search.py#L70-L85)):

```python
# 大部分 pattern 是字面量(node_modules),用 set 查找 O(1)
_EXACT_IGNORE_NAMES = frozenset(os.path.normcase(p) for p in IGNORE_PATTERNS if not any(c in p for c in "*?["))

# 少数是 glob(*.log),预编译成正则
_GLOB_IGNORE_PATTERNS = [p for p in IGNORE_PATTERNS if any(c in p for c in "*?[")]
_GLOB_IGNORE_RE = re.compile("|".join(fnmatch.translate(p) for p in _GLOB_IGNORE_PATTERNS))

def should_ignore_name(name: str) -> bool:
    # 先查 set(O(1)),再查正则
    if name in _EXACT_IGNORE_NAMES:
        return True
    return _GLOB_IGNORE_RE.match(name) is not None
```

**为什么这样优化**:`should_ignore_name` 在 `os.walk` 遍历目录时会被调用**几千次**,如果每次都跑 50 个 `fnmatch`,太慢。所以把字面量和 glob 分开,字面量用 set,glob 预编译成正则。

## 6. file_operation_lock:同一个文件的并发写锁

[file_operation_lock.py](../backend/packages/harness/deerflow/sandbox/file_operation_lock.py) 只有 27 行,但解决了一个关键问题:**多个工具同时写同一个文件**。

```python
_FILE_OPERATION_LOCKS: weakref.WeakValueDictionary[tuple[str, str], threading.Lock] = weakref.WeakValueDictionary()

def get_file_operation_lock(sandbox: Sandbox, path: str) -> threading.Lock:
    lock_key = (sandbox.id, path)
    with _FILE_OPERATION_LOCKS_GUARD:
        lock = _FILE_OPERATION_LOCKS.get(lock_key)
        if lock is None:
            lock = threading.Lock()
            _FILE_OPERATION_LOCKS[lock_key] = lock
        return lock
```

**为什么需要**:agent 可能同时调 `write_file("test.py", "...")` 和 `str_replace("test.py", "old", "new")`,如果不加锁,两个写操作会交错,文件内容损坏。

**为什么用 WeakValueDictionary**:锁的数量可能无限增长(每个 `(sandbox_id, path)` 一个锁),用 weakref 可以自动清理不再被引用的锁,防止内存泄漏。

**使用方式**(tools.py 里):

```python
lock = get_file_operation_lock(sandbox, path)
with lock:
    sandbox.write_file(path, content)
```

## 7. 完整链路:agent 调 bash_tool 时发生了什么

```
Agent 调 bash_tool("ls /mnt/user-data/workspace")
  ↓
SandboxMiddleware(lazy init)→ acquire 沙箱(AioSandbox 或 LocalSandbox)
  ↓
validate_local_bash_command_paths → 检查路径是否合法
  ↓
replace_virtual_paths_in_command → 虚拟路径替换成真实路径
  ↓
sandbox.execute_command("ls /Users/.../workspace")
  ↓
mask_local_paths_in_output → 输出里的真实路径替换回虚拟路径
  ↓
返回 "/mnt/user-data/workspace/test.py" 给 agent
```

**关键机制**:
- **路径映射**:LocalSandbox 用 PathMapping 把虚拟路径 → 真实路径
- **路径验证**:防止路径穿越、限制访问范围
- **路径替换**:命令里的虚拟路径 → 真实路径,输出里的真实路径 → 虚拟路径
- **并发控制**:file_operation_lock 防止同一文件并发写
- **缓存策略**:LocalSandboxProvider 用 LRU,AioSandboxProvider 用 warm pool
- **Lazy init**:SandboxMiddleware 延迟到第一次工具调用才 acquire 沙箱

## 8. 面试里怎么讲

**如果被问"LocalSandbox 和 AioSandbox 的区别"**:

> LocalSandbox 是宿主机直接跑,不启动容器,用 PathMapping 做路径隔离(虚拟路径 → 真实路径),默认禁用 bash(因为不是安全边界)。AioSandbox 是 Docker 容器,真隔离,通过 HTTP API 执行命令。
>
> LocalSandbox 的优势是快(不需要 docker run),适合开发环境;劣势是不安全(代码在宿主机上跑)。AioSandbox 的优势是安全(容器隔离),适合生产;劣势是慢(docker 冷启动)。
>
> 缓存策略也不同:LocalSandboxProvider 用 LRU 缓存 LocalSandbox 对象(轻量,默认 256 个),AioSandboxProvider 用 warm pool 缓存 Docker 容器(重量,默认 3 个)。

**如果被问"agent 调 bash_tool 时发生了什么"**:

> 完整流程是:lazy init acquire 沙箱 → 验证命令路径是否合法(防路径穿越) → 虚拟路径替换成真实路径 → 执行命令 → 输出里的真实路径替换回虚拟路径 → 截断返回。
>
> 关键设计是**路径映射**:agent 看到的是虚拟路径(/mnt/user-data/...),宿主机/容器里是真实路径(~/.deer-flow/threads/.../...),tools.py 负责正反向转换。这让 agent 不需要关心沙箱是 Docker 还是宿主机,统一用虚拟路径。

## 9. 小结

本篇讲的是"agent 真正调用沙箱时发生了什么"的完整链路:LocalSandbox 的 PathMapping 机制、tools.py 的 7 个工具实现、LocalSandboxProvider 的 LRU 缓存、SandboxMiddleware 的 lazy init、search.py 的忽略规则、file_operation_lock 的并发控制。

**核心要点**:
- LocalSandbox 用 PathMapping 做路径隔离,不需要 Docker 也能让 agent 看到虚拟路径
- tools.py 的每个工具都有"验证路径 → 替换路径 → 执行 → 反向替换 → 截断"的完整流程
- LocalSandboxProvider 用 LRU(轻量对象),AioSandboxProvider 用 warm pool(重量容器)
- SandboxMiddleware 的 lazy init 避免"每个请求都 acquire 沙箱"的开销
- file_operation_lock 用 WeakValueDictionary 防止内存泄漏

这些机制加起来,让 agent 可以在"看起来像 Linux 沙箱"的环境里跑代码,实际上可能是在 Docker 容器里(AioSandbox)或宿主机上(LocalSandbox),但 agent 不需要关心。
