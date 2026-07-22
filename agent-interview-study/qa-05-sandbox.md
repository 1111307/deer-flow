# Sandbox 执行环境与安全 —— 面试问题链

> 本文档按"大厂面试追问链"组织:每条问题链从基础问题开始,层层追问到实现细节、设计权衡、异常边界。配套深读笔记:[06-sandbox-execution.md](06-sandbox-execution.md)、[13-sandbox-docker.md](13-sandbox-docker.md)、[14-sandbox-tools-and-local.md](14-sandbox-tools-and-local.md)(深读笔记讲"怎么实现",本文档讲"怎么被问、怎么答")。
> 代码引用格式:[文件名.py:起始行-结束行](../backend/仓库相对路径#L起始行-L结束行)。只引用实际读过的行。

## 问题链 1:三层抽象 —— Sandbox / SandboxProvider / SandboxBackend

**Q1.1(基础)** 你们 Agent 的执行环境是怎么抽象的?为什么要分层,直接 subprocess 不行吗?

**参考回答**:系统分三层:

- 第一层 `Sandbox` 抽象类定义 Agent 可见的能力面:`execute_command`、`read_file`、`write_file`、`download_file`、`list_dir`、`glob`、`grep`、`update_file` 共 8 个抽象方法 [sandbox.py:18-112](../backend/packages/harness/deerflow/sandbox/sandbox.py#L18-L112)。
- 第二层 `SandboxProvider` 管生命周期:`acquire(thread_id)` / `get(sandbox_id)` / `release(sandbox_id)`,并声明 `uses_thread_data_mounts`、`needs_upload_permission_adjustment` 两个能力位 [sandbox_provider.py:9-50](../backend/packages/harness/deerflow/sandbox/sandbox_provider.py#L9-L50)。
- 第三层 `SandboxBackend` 只管"怎么供给",接口是 `create/destroy/is_alive/discover/list_running`,其中 `list_running` 默认返回空列表——不管容器的后端(如纯远程)无需实现 [backend.py:68-144](../backend/packages/harness/deerflow/community/aio_sandbox/backend.py#L68-L144)。

两个 `Sandbox` 实现:`LocalSandbox` 直接操作宿主机文件系统,`AioSandbox` 通过 HTTP 连一个 all-in-one Docker 容器。不这样设计会怎样:如果工具直接调 subprocess,换隔离方案(本地→Docker→K8s)就要改所有工具;三层抽象让 `config.yaml` 里 `sandbox.use` 一行配置经 `resolve_class` 动态加载即可整体切换执行后端,单例由 `get_sandbox_provider()` 惰性创建并缓存 [sandbox_provider.py:60-74](../backend/packages/harness/deerflow/sandbox/sandbox_provider.py#L60-L74)。

**链路解析**:
```
config.yaml: sandbox.use ──► resolve_class() ──► Provider 单例
                                                       │ acquire(thread_id)
   ┌───────────────────────────────────────────────────┼────────────────────┐
   ▼ LocalSandboxProvider                               ▼ AioSandboxProvider
 LocalSandbox(路径映射)                     ┌──────────┴───────────┐
                                            ▼ LocalContainerBackend ▼ RemoteSandboxBackend
                                          docker run 本地容器      POST provisioner (k3s)
                                            └──────────┬───────────┘
                                                       ▼
                                            AioSandbox(HTTP client :8080)
```

**Q1.2(深挖)** Provider 是单例,`acquire` 是同步阻塞的(Docker 操作),async runtime 里不会卡 event loop 吗?

**参考回答**:会卡,所以基类专门提供 `acquire_async`,默认实现是 `asyncio.to_thread(self.acquire, thread_id)`,把阻塞的 Docker/provisioner 调用丢到 worker 线程 [sandbox_provider.py:24-32](../backend/packages/harness/deerflow/sandbox/sandbox_provider.py#L24-L32)。`AioSandboxProvider` 进一步覆写了真正的异步版:文件锁、`ensure_thread_dirs`、`backend.create` 全部 `asyncio.to_thread`,就绪轮询用 `httpx.AsyncClient` 的 `wait_for_sandbox_ready_async`(deadline 驱动,单次请求超时取 `min(5s, 剩余时间)`)[aio_sandbox_provider.py:669-684](../backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox_provider.py#L669-L684)、[backend.py:40-65](../backend/packages/harness/deerflow/community/aio_sandbox/backend.py#L40-L65)。连 `threading.Lock` 的等待都有专门设计:

- 放进专用 `ThreadPoolExecutor`,worker 数 `min(32, cpu+4)`,不占默认 executor [aio_sandbox_provider.py:51-53](../backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox_provider.py#L51-L53);
- 用 `asyncio.shield` 包住 acquire future——若协程被取消,后台线程仍可能拿到锁,通过 done-callback 补一次 `lock.release()`,防止锁永久泄漏 [aio_sandbox_provider.py:78-105](../backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox_provider.py#L78-L105);
- executor 自身也注册 `atexit` 关闭(`wait=False, cancel_futures=True`),避免退出时悬挂 [aio_sandbox_provider.py:53](../backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox_provider.py#L53)。

**Q1.3(边界/异常)** 运行中改了 `sandbox.use` 配置,单例怎么失效?`reset` 和 `shutdown` 有什么区别?

**参考回答**:`reset_sandbox_provider()` 只清缓存实例并调用 provider 的 `reset()` 钩子——注释明确警告"如果 provider 还有活跃 sandbox,它们会变成孤儿" [sandbox_provider.py:77-95](../backend/packages/harness/deerflow/sandbox/sandbox_provider.py#L77-L95)。`shutdown_sandbox_provider()` 会先调 provider 的 `shutdown()` 释放所有沙箱再清单例 [sandbox_provider.py:98-109](../backend/packages/harness/deerflow/sandbox/sandbox_provider.py#L98-L109)。两个 provider 的语义也不同:`LocalSandboxProvider.reset()` 清空 `_thread_sandboxes` LRU 缓存和模块级 `_singleton` 别名,且 `shutdown` 直接复用 `reset`(本地无外部资源),保证 mount 配置变更下次 `acquire` 生效 [local_sandbox_provider.py:327-345](../backend/packages/harness/deerflow/sandbox/local/local_sandbox_provider.py#L327-L345);`AioSandboxProvider` 还额外在构造时注册 `atexit` 与信号处理器,shutdown 要停 idle checker 线程并销毁 active + warm 全部容器 [aio_sandbox_provider.py:132-160](../backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox_provider.py#L132-L160)。

## 问题链 2:LocalSandbox 路径映射与防逃逸

**Q2.1(基础)** Local 模式下 Agent 看到的 `/mnt/user-data/...` 是怎么落到宿主机真实路径的?

**参考回答**:核心是 `PathMapping(container_path, local_path, read_only)` 三元组 [local_sandbox.py:21-27](../backend/packages/harness/deerflow/sandbox/local/local_sandbox.py#L21-L27)。正向解析走 `_resolve_path_with_mapping`:先按 container_path 最长前缀匹配 mapping(嵌套挂载时最具体的赢),把相对部分拼到 `local_root` 上再 `.resolve()` [local_sandbox.py:159-200](../backend/packages/harness/deerflow/sandbox/local/local_sandbox.py#L159-L200)。每个 thread 的 mapping 由 provider 在 `acquire` 时构建:`/mnt/user-data/{workspace,uploads,outputs}` 和 `/mnt/acp-workspace` 都指向 `{base_dir}/users/{user_id}/threads/{thread_id}/...`,其中还有一个 `/mnt/user-data` 父级聚合映射,让 `ls /mnt/user-data` 的行为与 AIO 容器内一致(那里父目录是真实存在的)[local_sandbox_provider.py:187-233](../backend/packages/harness/deerflow/sandbox/local/local_sandbox_provider.py#L187-L233)。静态映射(skills 目录、config.yaml 自定义 mounts)在 provider 构造时 `_setup_path_mappings` 一次完成,skills 恒为 `read_only=True` [local_sandbox_provider.py:82-113](../backend/packages/harness/deerflow/sandbox/local/local_sandbox_provider.py#L82-L113)。

**链路解析**:
```
Agent: write_file("/mnt/user-data/workspace/a.py", ...)
            │
            ▼ _find_path_mapping (最长 container_path 前缀优先)
   mapping = (/mnt/user-data/workspace → D:\deerflow\...\threads\t1\user-data\workspace)
            │
            ▼ (local_root / "a.py").resolve()
            ▼ resolved.relative_to(local_root)  ← 防逃逸闸
            ▼ _is_resolved_path_read_only?      ← 只读闸
            ▼ open(resolved, "w")
```

**Q2.2(深挖)** 防逃逸具体怎么做的?`../..` 或者 symlink 能逃出去吗?

**参考回答**:逃逸防线分四层叠加:

1. **解析层**:`_resolve_path_with_mapping` 把容器路径拼到 `local_root` 后调 `.resolve()`,再用 `resolved_path.relative_to(local_root)` 校验,逃逸即抛 `PermissionError(EACCES, "path escapes mounted directory")` [local_sandbox.py:175-200](../backend/packages/harness/deerflow/sandbox/local/local_sandbox.py#L175-L200)。关键在 `.resolve()` 会先规范化 `..` 并解开 symlink,再做前缀校验,所以 `/mnt/user-data/workspace/../../etc` 在 resolve 阶段就落到 root 外被拦。
2. **工具层**:`_reject_path_traversal` 逐段检查 `..`;`_validate_resolved_user_data_path` 对 resolve 后的路径再做一次 `relative_to` 三个 allowed_roots(workspace/uploads/outputs)[tools.py:631-637](../backend/packages/harness/deerflow/sandbox/tools.py#L631-L637)、[tools.py:694-719](../backend/packages/harness/deerflow/sandbox/tools.py#L694-L719)。
3. **遍历层**:`list_dir` 对 symlink 条目 resolve 后确认仍在 root 内,否则直接跳过 [list_dir.py:42-55](../backend/packages/harness/deerflow/sandbox/local/list_dir.py#L42-L55)。
4. **下载层**:`download_file` 先强制路径必须位于 `VIRTUAL_PATH_PREFIX` 之下(与 mapping 无关的独立闸),再限单文件 **100 MB**;注释坦承 `getsize()` 与 `read()` 之间的 TOCTOU 窗口是"受控沙箱环境下的可接受折衷" [local_sandbox.py:405-425](../backend/packages/harness/deerflow/sandbox/local/local_sandbox.py#L405-L425)。

只读判定同样按"最长 local 前缀"选 mapping,嵌套挂载时最具体的只读标记生效 [local_sandbox.py:134-157](../backend/packages/harness/deerflow/sandbox/local/local_sandbox.py#L134-L157)。

**Q2.3(边界/异常)** 命令执行的输出里有宿主机路径会泄漏给模型吗?另外为什么 `read_file` 只对"agent 写过的文件"做反向替换?

**参考回答**:`execute_command` 返回前会跑 `_reverse_resolve_paths_in_output`,把输出里的宿主机绝对路径按"最长 local_path 优先"替换回容器路径 [local_sandbox.py:330-375](../backend/packages/harness/deerflow/sandbox/local/local_sandbox.py#L330-L375)、[local_sandbox.py:233-254](../backend/packages/harness/deerflow/sandbox/local/local_sandbox.py#L233-L254)。`read_file` 则只对该 sandbox `_agent_written_paths` 集合里的文件做反向替换——因为 write_file 时内容里的容器路径已被正向替换成本地路径,读回要还原;而用户上传的文件、外部工具产出不该被静默改写(注释引用 PR #1935)[local_sandbox.py:389-403](../backend/packages/harness/deerflow/sandbox/local/local_sandbox.py#L389-L403)、[local_sandbox.py:442-445](../backend/packages/harness/deerflow/sandbox/local/local_sandbox.py#L442-L445)。这个集合也是 `LocalSandboxProvider.release` 故意做成 no-op 的原因:释放就丢了这个反向解析提示 [local_sandbox_provider.py:316-325](../backend/packages/harness/deerflow/sandbox/local/local_sandbox_provider.py#L316-L325)。缓存本身是一个以 `threading.Lock` 保护的 LRU,上限默认 **256** 个 thread 沙箱,逐出只丢 `_agent_written_paths` 提示,下次 acquire 重建即可,`get()` 命中也会 `move_to_end` 防止活跃线程被逐出 [local_sandbox_provider.py:23-31](../backend/packages/harness/deerflow/sandbox/local/local_sandbox_provider.py#L23-L31)、[local_sandbox_provider.py:282-314](../backend/packages/harness/deerflow/sandbox/local/local_sandbox_provider.py#L282-L314)。

**链路解析**(正向写入 / 反向读回的对称数据流):
```
write_file 路径:  虚拟路径 ──► _resolve_path_with_mapping ──► 宿主路径 open("w")
                  内容里的 /mnt/... ──► _resolve_paths_in_content(转正斜杠)
                  完成后 resolved_path 记入 _agent_written_paths
read_file 路径:   宿主路径 open ──► 内容
                  ├─ 在 _agent_written_paths 内 ──► _reverse_resolve_paths_in_output 还原 /mnt/...
                  └─ 不在(用户上传/外部产出)──► 原样返回,不做静默改写
bash 输出路径:    stdout/stderr ──► _reverse_resolve_paths_in_output(最长 local 前缀优先)
```

## 问题链 3:tools.py 的安全流水线 —— 验证→替换→执行→掩码

**Q3.1(基础)** 描述一下 local 模式下一次 `bash` 工具调用的完整安全流程。

**参考回答**:五步流水线,全部在 `bash_tool` 里可见:① `ensure_sandbox_initialized` 惰性获取沙箱;② `is_host_bash_allowed()` 门控,默认直接拒绝;③ `validate_local_bash_command_paths` 静态扫描命令里的危险路径;④ `replace_virtual_paths_in_command` 把虚拟路径替换成宿主机路径,再 `_apply_cwd_prefix` 加 `cd <workspace> &&` 锚定相对路径;⑤ 执行后 `mask_local_paths_in_output` 把输出里的宿主机路径掩码回虚拟路径,最后按 `bash_output_max_chars`(默认 **20000** 字符)中段截断 [tools.py:1394-1425](../backend/packages/harness/deerflow/sandbox/tools.py#L1394-L1425)。写类工具另有互斥保障:`write_file` 与 `str_replace`(读-改-写)都在 `get_file_operation_lock(sandbox, path)` 临界区内执行,锁的粒度是 `(sandbox.id, path)` 二元组——不同沙箱的同名虚拟路径互不阻塞;锁表用 `WeakValueDictionary` 存放,外层再加一把 guard lock,长进程里不引用的锁条目会被自动回收、防内存泄漏 [tools.py:1822-1823](../backend/packages/harness/deerflow/sandbox/tools.py#L1822-L1823)、[tools.py:1885-1895](../backend/packages/harness/deerflow/sandbox/tools.py#L1885-L1895)、[file_operation_lock.py:6-27](../backend/packages/harness/deerflow/sandbox/file_operation_lock.py#L6-L27)。

**链路解析**:
```
LLM tool call: bash("cat /mnt/user-data/workspace/x.log")
   │ ① ensure_sandbox_initialized → provider.acquire(thread_id)
   │ ② is_host_bash_allowed()? ── 默认 False → 直接返回禁用提示
   │ ③ validate_local_bash_command_paths ── file://? ".."? 非法绝对路径?
   │ ④ replace_virtual_paths_in_command → cd <ws> && cat D:\...\x.log
   │ ⑤ sandbox.execute_command (subprocess, timeout=600s)
   ▼ mask_local_paths_in_output → _truncate_bash_output(≤20000 chars) → ToolMessage
```

**Q3.2(深挖)** `validate_local_bash_command_paths` 具体怎么扫一条命令?为什么说自己是 "best-effort, not a security boundary"?

**参考回答**:三层扫描:先 `_FILE_URL_PATTERN` 封杀 `file://`(它绕过绝对路径正则但可读本地文件外泄)[tools.py:1012-1015](../backend/packages/harness/deerflow/sandbox/tools.py#L1012-L1015);再用 shlex 分词做 `_validate_local_bash_shell_tokens`——逐 token 查 `..` 段、先用正则拦 `$(... cd ...)` 形式的命令替换逃逸、追踪 `cd/pushd` 目标(拒绝 `-`、`$`、反引号、`~` 开头)、对 `cat/rm/cp` 等 17 个根路径敏感命令拒绝对 `/` 裸引用,还能识别 `command`/`builtin` 包装器和 `if/for/while` 等 shell 关键字占位 [tools.py:881-938](../backend/packages/harness/deerflow/sandbox/tools.py#L881-L938)、[tools.py:62-97](../backend/packages/harness/deerflow/sandbox/tools.py#L62-L97);最后用 `_ABSOLUTE_PATH_PATTERN` 扫所有 `/xxx` 绝对路径,逐个过白名单(MCP allowed paths、`/mnt/user-data`、skills、acp-workspace、custom mounts,以及 `/bin/`、`/dev/` 等 6 个系统前缀)[tools.py:36-43](../backend/packages/harness/deerflow/sandbox/tools.py#L36-L43)、[tools.py:792-820](../backend/packages/harness/deerflow/sandbox/tools.py#L792-L820)。自称 best-effort 是因为字符串分析无法覆盖运行时展开——命令替换、变量、进程替换都可能重组出宿主路径,真正的边界是"默认禁止 host bash",这个校验只是显式 opt-in 后的减伤网 [tools.py:994-1008](../backend/packages/harness/deerflow/sandbox/tools.py#L994-L1008)。

**链路解析**(扫描漏斗,任一关拒绝即 PermissionError):
```
command 字符串
  ├─ 关卡0: file:// URL?                ──► 拒(本地文件外泄通道)
  ├─ 关卡1: shlex 分词
  │     ├─ $(... cd ...) 命令替换?      ──► 拒
  │     ├─ 任一 token 含 ".." 路径段?    ──► 拒
  │     ├─ cd/pushd 目标非法?           ──► 拒(-、$、`、~、未授权绝对路径)
  │     └─ cat/rm/cp 等 17 命令引用 "/"? ──► 拒
  └─ 关卡2: _ABSOLUTE_PATH_PATTERN 逐个 /xxx
        ├─ 命中 URL span / 文本片段?     ──► 豁免(误伤保护)
        ├─ 命中白名单(MCP/user-data/skills/acp/mounts/系统前缀)? ──► 放行
        └─ 其余                          ──► 汇总拒: "Unsafe absolute paths in command: ..."
```

**Q3.3(边界/异常)** 错误消息也会泄漏宿主机路径吗?掩码和截断的数字策略是什么?

**参考回答**:会。`_sanitize_error` 在 local 模式下对异常消息跑 `mask_local_paths_in_output` 再返回 [tools.py:441-452](../backend/packages/harness/deerflow/sandbox/tools.py#L441-L452)。掩码按"最长 actual_base 优先"对 skills、acp-workspace、user-data 三类宿主机根做正反斜杠变体替换;custom mount 的宿主机路径则由 `LocalSandbox._reverse_resolve_paths_in_output` 在沙箱层掩码 [tools.py:557-628](../backend/packages/harness/deerflow/sandbox/tools.py#L557-L628)。数字策略一览:

- bash/ls 输出默认 **20000** 字符,read_file 默认 **50000** 字符,均可被 `sandbox.*_output_max_chars` 配置覆盖 [tools.py:1418-1425](../backend/packages/harness/deerflow/sandbox/tools.py#L1418-L1425)、[tools.py:1707-1714](../backend/packages/harness/deerflow/sandbox/tools.py#L1707-L1714)。
- glob 默认 200 条、硬上限 **1000** 条;grep 默认 100 条、硬上限 **500** 条;请求值与配置值经 `_resolve_max_results` 取较小者 [tools.py:47-50](../backend/packages/harness/deerflow/sandbox/tools.py#L47-L50)、[tools.py:373-386](../backend/packages/harness/deerflow/sandbox/tools.py#L373-L386)。
- bash 输出**中段**截断(头尾各 50%,因为 stderr/stdout 顺序不定,两头都可能有错误);截断标记长度预先计入预算,返回值保证不超 max_chars [tools.py:1318-1343](../backend/packages/harness/deerflow/sandbox/tools.py#L1318-L1343)。
- write_file 单次非 append 上限 **80 KB**(约 20K token),可用 `DEERFLOW_WRITE_FILE_MAX_BYTES` 调整、设 0 关闭;动机是 LLM 流式 tool-call JSON 超长会触发 chunk-gap 超时(issue #3189),错误详情本身也按 2000 字符中段截断 [tools.py:51-61](../backend/packages/harness/deerflow/sandbox/tools.py#L51-L61)、[tools.py:1799-1811](../backend/packages/harness/deerflow/sandbox/tools.py#L1799-L1811)。

## 问题链 4:security.py —— 为什么默认禁 host bash

**Q4.1(基础)** Local sandbox 不就是图个方便吗,为什么默认把 bash 禁了?

**参考回答**:因为 `LocalSandboxProvider` 不是安全边界——命令直接在宿主机以 gateway 进程权限跑,路径映射只是视图层的便利,挡不住 `cat ~/.ssh/id_rsa` 这类直读。`is_host_bash_allowed` 的判定逻辑分三步:

1. 非 local provider → 一律放行(容器自己就是边界);
2. local provider 且 `sandbox.allow_host_bash: true` → 放行(显式 opt-in);
3. local provider 默认 → 拒绝 [security.py:35-45](../backend/packages/harness/deerflow/sandbox/security.py#L35-L45)。

被禁时 `bash_tool` 返回固定提示,引导用户换 `AioSandboxProvider` 或确认"完全可信的本地环境"再开 [security.py:10-14](../backend/packages/harness/deerflow/sandbox/security.py#L10-L14)、[tools.py:1409-1411](../backend/packages/harness/deerflow/sandbox/tools.py#L1409-L1411)。连 bash 类 subagent 也有对应的禁用文案,提示词层与工具层双保险 [security.py:16-20](../backend/packages/harness/deerflow/sandbox/security.py#L16-L20)。文件类工具(read/write/ls/glob/grep)不受此限——它们走的不是 shell,而是 `_resolve_path` 的 resolve+relative_to 硬约束,仍然可用。

**链路解析**:
```
bash_tool ──► is_local_sandbox(runtime)?   (sandbox_id == "local" 或 "local:{thread_id}")
                 │ yes
                 ▼ is_host_bash_allowed()
                     │ provider 是 LocalSandboxProvider? ── no ──► allow
                     │ yes
                     ▼ sandbox.allow_host_bash == true? ── no ──► Error: LOCAL_HOST_BASH_DISABLED_MESSAGE
                     ▼ yes
              validate_local_bash_command_paths (best-effort 减伤)
```

**Q4.2(深挖)** 判定"当前是不是 local provider"怎么做的?为什么不用 isinstance?

**参考回答**:用配置字符串匹配,不走类型判定:

- 命中两个完整 marker:`deerflow.sandbox.local:LocalSandboxProvider` 与 `deerflow.sandbox.local.local_sandbox_provider:LocalSandboxProvider`;
- 或以 `:LocalSandboxProvider` 结尾且包含 `deerflow.sandbox.local`(兜底模块路径写法差异)[security.py:5-8](../backend/packages/harness/deerflow/sandbox/security.py#L5-L8)、[security.py:23-32](../backend/packages/harness/deerflow/sandbox/security.py#L23-L32)。

不用 isinstance 是因为判定发生在配置层(`get_app_config()`),此时 provider 类可能还没被实例化/导入,字符串判定零依赖、可在任意模块使用,也不会因为延迟导入引入循环依赖。工具侧判断"本次调用是不是 local 沙箱"则看 state 里的 sandbox_id:`"local"` 或 `"local:{thread_id}"` 前缀——前者是无 thread 上下文的旧式单例,后者是 per-thread 实例 [tools.py:1111-1128](../backend/packages/harness/deerflow/sandbox/tools.py#L1111-L1128)、[local_sandbox_provider.py:235-255](../backend/packages/harness/deerflow/sandbox/local/local_sandbox_provider.py#L235-L255)。

**Q4.3(边界/异常)** 如果管理员开了 `allow_host_bash: true`,是不是就安全了?

**参考回答**:不是,这正是 `validate_local_bash_command_paths` docstring 里写死的立场:该校验"only a best-effort guard for the explicit opt-in... must not be treated as isolation from the host filesystem" [tools.py:994-999](../backend/packages/harness/deerflow/sandbox/tools.py#L994-L999)。即使放行,`LocalSandbox.execute_command` 也只是 `subprocess.run(shell -c command, timeout=600)`——600 秒超时、stderr 拼进输出、非零退出码追加 `Exit Code: N`,全程没有 chroot/namespace/seccomp 任何内核级隔离 [local_sandbox.py:330-375](../backend/packages/harness/deerflow/sandbox/local/local_sandbox.py#L330-L375)。shell 探测顺序是 `/bin/zsh → /bin/bash → /bin/sh → PATH 上的 sh`,Windows 再降级 pwsh/powershell/cmd;Git Bash 类 MSYS shell 还要设 `MSYS_NO_PATHCONV=1` 防止路径被自动转换 [local_sandbox.py:304-328](../backend/packages/harness/deerflow/sandbox/local/local_sandbox.py#L304-L328)、[local_sandbox.py:335-357](../backend/packages/harness/deerflow/sandbox/local/local_sandbox.py#L335-L357)。不这样设计会怎样:如果把字符串扫描当成安全边界,攻击面是字符串分析无法覆盖的运行时展开(变量、命令替换、动态脚本),一次漏判就是宿主机任意读写。所以架构上把"隔离"职责完全交给容器(AIO),local 模式的价值仅是开发便利,bash 默认关、文件操作靠 `_resolve_path` 的 resolve+relative_to 硬约束兜底。

## 问题链 5:AioSandboxProvider —— warm pool、确定性 ID、跨进程锁

**Q5.1(基础)** `acquire(thread_id)` 的完整查找路径是什么样的?

**参考回答**:四层级联,全部在 `_acquire_internal` 里 [aio_sandbox_provider.py:686-713](../backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox_provider.py#L686-L713):

1. **Layer 1 进程内缓存**:`_thread_sandboxes` 命中后过 `is_alive` 健康检查;不健康走 `_drop_unhealthy_sandbox` 销毁再重来。
2. **Layer 1.5 warm pool**:已 release 但容器还在跑的沙箱按确定性 ID 回收,零冷启动;回收前同样做健康检查。
3. **Layer 2 跨进程协调**:在文件锁内先 re-check 缓存(等锁期间别人可能建好了)、再 `backend.discover`(别的进程可能已建好同名容器)。
4. **Layer 3 创建**:`_create_sandbox` 起容器并轮询就绪。

整个 `acquire` 外层还套了 per-thread 的进程内 `threading.Lock` 串行化同线程并发 [aio_sandbox_provider.py:647-667](../backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox_provider.py#L647-L667)。调用入口是 `SandboxMiddleware`:默认 `lazy_init=True`,`before_agent` 不抢建,第一次工具调用时 `ensure_sandbox_initialized` 才真正 acquire [middleware.py:40-49](../backend/packages/harness/deerflow/sandbox/middleware.py#L40-L49)、[tools.py:1160-1216](../backend/packages/harness/deerflow/sandbox/tools.py#L1160-L1216)。这里有个易踩的坑:`ensure_sandbox_initialized` 直接改 `runtime.state["sandbox"]`,该变更不会经过 LangGraph 的 channel reducer,后续 graph step 看不到;所以 middleware 用 `wrap_tool_call` 在 handler 前后 diff state 快照,发现新的 sandbox_id 就把结果包装成 `Command(update={"sandbox": ..., "messages": [...]})` 持久化进图状态 [middleware.py:135-148](../backend/packages/harness/deerflow/sandbox/middleware.py#L135-L148)、[middleware.py:189-202](../backend/packages/harness/deerflow/sandbox/middleware.py#L189-L202)。

**链路解析**:
```
acquire(thread_id)
  │ thread_lock(thread_id) 进程内串行
  ▼
Layer1  _thread_sandboxes 命中? ── is_alive? ──► return
  │ miss
Layer1.5 _warm_pool[sha256(thread_id)[:8]]? ── 健康? ──► pop 回 active, return
  │ miss
Layer2  flock({sandbox_id}.lock) 跨进程串行
  │   ├─ re-check Layer1/1.5 (等锁期间别人可能建好了)
  │   ├─ backend.discover(deterministic name) ──► adopt
  │   └─ backend.create + wait_for_sandbox_ready(60s) ──► register
  ▼
return sandbox_id
```

**Q5.2(深挖)** 为什么 sandbox_id 要用 `sha256(thread_id)[:8]` 做确定性 ID?

**参考回答**:注释写明目的:"all processes derive the same sandbox_id for a given thread, enabling cross-process sandbox discovery without shared memory" [aio_sandbox_provider.py:278-285](../backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox_provider.py#L278-L285)。容器名是 `{prefix}-{sandbox_id}`,任何进程拿到 thread_id 都能算出同一个容器名,于是 gateway 重启、多 worker、多 pod 场景下 `docker inspect` 按名发现即可接管,不需要共享状态文件。无 thread 上下文的匿名 acquire 退化为 `uuid4()[:8]` 随机 ID [aio_sandbox_provider.py:458-460](../backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox_provider.py#L458-L460)。发现/创建的结果统一装进 `SandboxInfo` dataclass(`sandbox_id`、`sandbox_url`、`container_name`、`container_id`、`created_at`),它带 `to_dict/from_dict`,设计上就是"跨进程重连一个已有沙箱所需的全部元数据" [sandbox_info.py:9-41](../backend/packages/harness/deerflow/community/aio_sandbox/sandbox_info.py#L9-L41)。

**Q5.3(深挖)** 跨进程文件锁具体怎么实现?为什么锁内还要 re-check?

**参考回答**:锁文件是 thread 目录下的 `{sandbox_id}.lock`,Unix 用 `fcntl.flock(LOCK_EX)`,Windows 降级 `msvcrt.locking(LK_LOCK)`;async 路径里 flock 本身也是阻塞调用,同样经 `asyncio.to_thread` 挪出 event loop [aio_sandbox_provider.py:56-76](../backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox_provider.py#L56-L76)、[aio_sandbox_provider.py:767-795](../backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox_provider.py#L767-L795)。锁内 re-check 是经典 double-checked 模式,防三类竞态:

1. 等锁期间,**同进程**其他线程可能已建好 → re-check in-process 缓存;
2. 等锁期间,**另一进程**可能已建好 → `backend.discover` 按确定性容器名收养;
3. 都 miss 才真正 `create`,且 local_backend 里还有第三道兜底:`docker run` 报 "name already in use" 时回头 `discover` 收养那个容器 [local_backend.py:303-307](../backend/packages/harness/deerflow/community/aio_sandbox/local_backend.py#L303-L307)。

锁内完整流程见 `_discover_or_create_with_lock`:先 `ensure_thread_dirs` 保证锁文件所在目录存在,再 open → flock → re-check → discover → create → finally 解锁 [aio_sandbox_provider.py:735-765](../backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox_provider.py#L735-L765)。

**Q5.4(边界/异常)** `release` 为什么不销毁容器?放进 warm pool 时关了 HTTP client 会不会导致回收时出错?

**参考回答**:`release` 只把 `SandboxInfo` 连同 release 时间戳停进 `_warm_pool`,容器继续跑,同一线程下一轮可秒级回收、免去冷启动 [aio_sandbox_provider.py:883-922](../backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox_provider.py#L883-L922)。关闭 client 是修 #2872 的资源泄漏:agent_sandbox SDK 是 Fern 生成的、没有公开 `close()`,代码沿 `_client_wrapper.httpx_client.httpx_client` 属性链找到真正的 `httpx.Client` 关掉以释放连接池 socket;回收时 warm pool 只存 `SandboxInfo`,会新建 `AioSandbox`(新 client),所以无影响 [aio_sandbox.py:48-97](../backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox.py#L48-L97)、[aio_sandbox_provider.py:519-532](../backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox_provider.py#L519-L532)。值得注意的是 `AioSandbox` 实例级还有一把 `threading.Lock`,serialize 该容器内单持久 shell 会话上的所有操作;且 `write_file(append=True)` 是先 `read_file` 再拼接重写的 read-modify-write,并非原子追加——所以 append 场景更要依赖 tools 层的 per-path 文件锁防并发 [aio_sandbox.py:41-42](../backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox.py#L41-L42)、[aio_sandbox.py:237-254](../backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox.py#L237-L254)。下载侧同样先拒 `..` 段和 `VIRTUAL_PATH_PREFIX` 之外的路径,再分块流式累加、累计超 **100 MB** 即抛 `EFBIG` [aio_sandbox.py:173-214](../backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox.py#L173-L214)。

**链路解析**:
```
acquire ──► active(_sandboxes + _last_activity)
              │ release()                     │ idle>600s / destroy()
              ▼                               ▼
         warm pool(_warm_pool: info+ts) ──► backend.destroy (docker stop, --rm 自动删除)
              │ 同 thread 再次 acquire: pop 回 active(免冷启动)
              │ replicas 超限: _evict_oldest_warm 淘汰最旧
              ▼
         shutdown(): active 与 warm 全部 destroy
```

## 问题链 6:docker run 细节与 replicas 软上限

**Q6.1(基础)** `_start_container` 拼出来的 docker 命令有哪些关键点?

**参考回答**:`docker run --security-opt seccomp=unconfined --rm -d -p {bind_host}:{port}:8080 --name {prefix}-{sandbox_id} -e KEY=VAL --mount type=bind,src=...,dst=... <image>` [local_backend.py:532-581](../backend/packages/harness/deerflow/community/aio_sandbox/local_backend.py#L532-L581)。要点:

- `--rm`:stop 后自动删除,不留尸体——这是孤儿容器总量的第一道保险。
- `seccomp=unconfined`:AIO 沙箱内部要跑浏览器/多样工具链,Docker 默认 seccomp profile 会拦掉其 syscall(仅 docker 运行时添加)。
- 挂载分两段拼接:先 config.yaml 的静态 mounts,再 thread 级 extra mounts(skills、workspace 等)[local_backend.py:559-579](../backend/packages/harness/deerflow/community/aio_sandbox/local_backend.py#L559-L579)。
- 日志安全:输出启动命令前用 `_redact_container_command_for_log` 把 `-e` / `--env=` 的值打码成 `<redacted>`,防止 API_KEY 进日志 [local_backend.py:90-121](../backend/packages/harness/deerflow/community/aio_sandbox/local_backend.py#L90-L121)。

销毁路径对称:`destroy` 优先用 container_id、退化用 name 去 `docker stop`(兼容只有名字的 list_running 收养场景),再从 `sandbox_url` 解析端口调 `release_port` 归还 [local_backend.py:322-338](../backend/packages/harness/deerflow/community/aio_sandbox/local_backend.py#L322-L338)。

**链路解析**:
```
_create_sandbox
  │ replicas 检查: total(active+warm) >= 3? ──► _evict_oldest_warm()
  ▼
LocalContainerBackend.create
  │ for attempt in range(10):
  │   port = get_free_port(start=_next_start)
  │   try _start_container ──► docker run --rm -d -p bind:port:8080 ...
  │   except "port is already allocated" ──► release_port, _next_start=port+1, retry
  │   except "name already in use" ──► discover() 收养已有容器
  ▼
wait_for_sandbox_ready(url, timeout=60) ──► GET /v1/sandbox 每秒轮询
  ▼
_register_created_sandbox
```

**Q6.2(深挖)** 端口分配为什么既用 `get_free_port` 又要重试 10 次?这不是重复吗?

**参考回答**:注释解释了:bind 检查和 Docker 的端口释放存在时间差——进程重启后旧容器可能还持有绑定,`get_free_port` 的 socket 探测与 Docker 的 0.0.0.0 bind 行为并不完全同步,所以采用"乐观选口 + 被拒重试"的反应式兜底,`_next_start = port + 1` 跳过被拒口,最多 **10 次**,全失败才抛 "all candidate ports are already allocated" [local_backend.py:278-310](../backend/packages/harness/deerflow/community/aio_sandbox/local_backend.py#L278-L310)。不这样设计会怎样:只靠 `get_free_port` 的 TOCTOU 窗口在并发 acquire 或 Docker 异步释放时必然偶发失败,而失败对 Agent 表现为"沙箱起不来"的硬错误;重试把竞态转化为可恢复抖动。

**Q6.3(深挖)** `replicas: 3` 是硬上限吗?超过会怎样?

**参考回答**:是**软上限**,且只约束闲置容器。`_create_sandbox` 里 `total = len(_sandboxes) + len(_warm_pool)` 达到 replicas(默认 **3**)时,只调 `_evict_oldest_warm` 销毁 warm pool 里最久未用的那个;如果 3 个槽全被活跃线程占用,代码只打 warning("All 3 replica slots are in active use; creating sandbox beyond the soft limit")然后照建 [aio_sandbox_provider.py:627-643](../backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox_provider.py#L627-L643)、[aio_sandbox_provider.py:832-837](../backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox_provider.py#L832-L837)。设计理由写在注释里:"Active sandboxes are in use by live threads and must not be forcibly stopped"——容量治理不能杀掉正在服务用户的容器,只能回收闲置的。

**链路解析**(replicas 预算的决策树):
```
_create_sandbox 入口
   │ total = len(active) + len(warm)
   ▼
total >= replicas(3)?
   │ no  ──────────────────────► backend.create 直接建
   │ yes
   ▼
_evict_oldest_warm()
   ├─ warm 非空: 销毁最旧(release_ts 最小)──► 腾出席位 ──► create
   └─ warm 为空: 槽位全被活跃线程占用
        ──► warning 日志"beyond the soft limit"──► create 照建
             (活跃容器永不被强杀;idle checker 之后按 600s 闲置再收)
```

## 问题链 7:DooD 与挂载

**Q7.1(基础)** 什么是 DooD?这个项目里哪里要处理它?

**参考回答**:DooD(Docker-outside-of-Docker)指 gateway 自己跑在容器里、通过挂载宿主 Docker socket 操控**宿主机** daemon。此时 `docker run -v` 的源路径由宿主 daemon 解释,必须用宿主机路径而非 gateway 容器内路径。代码里两处体现:thread 数据挂载用 `paths.host_sandbox_work_dir(...)` 系列(host 视角路径)绑到容器内 `/mnt/user-data/{workspace,uploads,outputs}`,注释明说"when running inside Docker with a mounted Docker socket (DooD), the host Docker daemon can resolve the paths" [aio_sandbox_provider.py:304-323](../backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox_provider.py#L304-L323);skills 挂载优先取环境变量 `DEER_FLOW_HOST_SKILLS_PATH` [aio_sandbox_provider.py:325-343](../backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox_provider.py#L325-L343)。网络侧对应 `DEER_FLOW_SANDBOX_HOST`(DooD 下沙箱容器在宿主 daemon 上,gateway 要经 `host.docker.internal` 访问)[local_backend.py:312-320](../backend/packages/harness/deerflow/community/aio_sandbox/local_backend.py#L312-L320)。端口发布也按场景收紧:`_resolve_docker_bind_host` 在 loopback sandbox host 下只绑 `127.0.0.1`(IPv6 则 `[::1]`),不把沙箱 HTTP API 暴露到局域网;只有非 loopback(DooD 经 `host.docker.internal` 访问)才放宽 `0.0.0.0`,还可用 `DEER_FLOW_SANDBOX_BIND_HOST` 显式覆盖 [local_backend.py:142-169](../backend/packages/harness/deerflow/community/aio_sandbox/local_backend.py#L142-L169)。

**链路解析**:
```
make dev (裸机)                          make up (DooD)
gateway on host                          gateway container ──► /var/run/docker.sock
   │ docker run -v /host/data:...            │ docker run -v /HOST/data:...  (宿主 daemon 解释)
   │ curl http://localhost:PORT              │ curl http://host.docker.internal:PORT
   ▼                                         ▼            (DEER_FLOW_SANDBOX_HOST)
sandbox container on host daemon ◄───────────┘ 同一个宿主 daemon
```

**Q7.2(深挖)** Docker 的 `-v` 语法有什么坑?这里怎么绕的?

**参考回答**:Windows 盘符路径 `D:/...` 里的 `:` 和 `-v` 的 `host:container` 分隔符冲突,解析有歧义。所以 Docker 运行时统一用 `--mount type=bind,src=...,dst=...`(只读加 `,readonly`),Apple Container 才保留 `-v host:container[:ro]` [local_backend.py:70-87](../backend/packages/harness/deerflow/community/aio_sandbox/local_backend.py#L70-L87)。运行时探测在 macOS 上优先 Apple Container(`container --version`,5 秒超时),失败回退 Docker;其他平台直接 Docker [local_backend.py:233-258](../backend/packages/harness/deerflow/community/aio_sandbox/local_backend.py#L233-L258)。只读语义上,skills 挂载恒为只读(注释:"Read-only for security"),`/mnt/acp-workspace` 也按只读挂入——因为 lead agent 只在容器内读结果,写动作由宿主侧的 ACP 子进程完成 [aio_sandbox_provider.py:320-322](../backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox_provider.py#L320-L322)、[aio_sandbox_provider.py:340](../backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox_provider.py#L340)。

**Q7.3(边界/异常)** config.yaml 里用户自定义 mount 配错了会怎样?静默跳过吗?

**参考回答**:分三档处理:

- **warning 跳过**:host_path 非绝对路径、container_path 非绝对路径、container_path 撞保留前缀(`/mnt/user-data`、`/mnt/acp-workspace`、skills 路径)——配置错误但无副作用 [local_sandbox_provider.py:116-149](../backend/packages/harness/deerflow/sandbox/local/local_sandbox_provider.py#L116-L149)。
- **ERROR 升级**:host_path 不存在。因为 `make up` 场景下 gateway 容器看不到宿主路径,静默跳过会让 sandbox 读到空目录,成为"高调试成本的静默失败"(注释引用 #3244),日志里直接给出操作建议:去 docker-compose 的 `services.gateway.volumes` 补对应 volume,或改用 `make dev` 本地模式 [local_sandbox_provider.py:150-180](../backend/packages/harness/deerflow/sandbox/local/local_sandbox_provider.py#L150-L180)。
- **整体容错**:整个 `_setup_path_mappings` 外层 try/except,配置加载失败只 warning 不致命,skills 目录不存在也只是不生成对应 mapping [local_sandbox_provider.py:98-113](../backend/packages/harness/deerflow/sandbox/local/local_sandbox_provider.py#L98-L113)、[local_sandbox_provider.py:181-185](../backend/packages/harness/deerflow/sandbox/local/local_sandbox_provider.py#L181-L185)。

## 问题链 8:孤儿 reconcile 与 idle 回收

**Q8.1(基础)** 进程崩溃/重启后,之前起的 Docker 容器怎么办?

**参考回答**:`AioSandboxProvider.__init__` 里 `_reconcile_orphans` 调 `backend.list_running()` 枚举所有前缀匹配的存活容器,无条件**收养进 warm pool**,之后由 idle checker 按闲置时长决定去留 [aio_sandbox_provider.py:233-274](../backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox_provider.py#L233-L274)。为什么不按容器年龄杀:`idle_timeout` 度量的是"不活跃"而非" uptime",凭年龄无法区分"真孤儿"和"另一进程正在用",收养+闲置判定避免误杀别人正在用的容器。本地后端 `list_running` 优化成 2 次 subprocess(一次 `docker ps` + 一次批量 `docker inspect`),而非朴素 2N+1 次;因 Docker 的 `--filter name=` 是子串匹配,还要再 `startswith(prefix + "-")` 二次过滤防误收养 [local_backend.py:388-463](../backend/packages/harness/deerflow/community/aio_sandbox/local_backend.py#L388-L463)。容器年龄来自 `_parse_docker_timestamp`:Docker 时间戳是纳秒精度带 `Z`,先截断到微秒再把 `Z` 换成 `+00:00` 才能被 `fromisoformat` 接受,解析失败返回 `0.0` 作为"未知年龄"哨兵 [local_backend.py:24-50](../backend/packages/harness/deerflow/community/aio_sandbox/local_backend.py#L24-L50)。

**链路解析**:
```
进程启动
  ▼
_reconcile_orphans: docker ps --filter name=deer-flow-sandbox-
  │ 对每个容器: 已在 _sandboxes/_warm_pool? ── skip
  │             否则 _warm_pool[id] = (info, now)   ← 无条件收养
  ▼
idle checker 线程 (每 60s 醒一次, idle_timeout=600s)
  ├─ active: now - _last_activity > 600s? ── 加锁二次确认 ──► destroy
  └─ warm:   now - release_ts      > 600s? ──► backend.destroy
```

**Q8.2(深挖)** idle checker 销毁前为什么要"二次确认"?

**参考回答**:典型 TOCTOU 防护。快照阶段在锁内扫描 `_last_activity` 与 `_warm_pool`,挑出闲置超 **600s**(默认 `DEFAULT_IDLE_TIMEOUT` [aio_sandbox_provider.py:48](../backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox_provider.py#L48))的候选;但从快照到真正 destroy 之间,该沙箱可能被重新 acquire(`get` 会刷新 `_last_activity` [aio_sandbox_provider.py:868-881](../backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox_provider.py#L868-L881))或已被别的路径释放。所以 destroy 前在锁内重读 `_last_activity`:

- 为 `None` → 已被释放/销毁,跳过;
- 重新计算闲置时长不足 600s → 被重新 acquire 过,跳过;
- 仍然超时 → 才真正 `destroy` [aio_sandbox_provider.py:386-405](../backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox_provider.py#L386-L405)。

同理 `_remove_tracked_sandbox` 支持 `expected_info` 参数:健康检查结果过期时,只有当前跟踪的还是那个旧 info 对象才允许删除,防止"陈旧的健康检查删掉同名新建沙箱"——确定性 ID 复用场景下这是一个真实的竞态 [aio_sandbox_provider.py:571-604](../backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox_provider.py#L571-L604)。

**Q8.3(边界/异常)** 还有哪些兜底清理路径?信号、atexit、单容器会话损坏分别怎么处理?

**参考回答**:四层兜底:

1. **退出钩子**:`atexit.register(self.shutdown)` 加 SIGTERM/SIGINT/SIGHUP 处理器,终端关闭也能清;非主线程注册失败时降级为 debug 日志 [aio_sandbox_provider.py:152-153](../backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox_provider.py#L152-L153)、[aio_sandbox_provider.py:417-447](../backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox_provider.py#L417-L447)。
2. **幂等 shutdown**:`_shutdown_called` 标志防重入,先停 idle checker 线程(`join(timeout=5)`)再逐个 destroy active 与 warm [aio_sandbox_provider.py:952-981](../backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox_provider.py#L952-L981)。
3. **容器自清理**:`docker run --rm` 让 stop 即删;`docker stop` 失败只 warning 不阻断后续清理。
4. **运行时自愈**:AIO 容器是单持久 shell 会话,并发 exec 会返回 `ErrorObservation`;`AioSandbox` 用 `threading.Lock` 串行化,检测到损坏签名时创建一次性新 session 重试,finally 里 best-effort `cleanup_session` 防 session 堆积 [aio_sandbox.py:113-155](../backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox.py#L113-L155)。

exec 的 `no_change_timeout` 提到 **600s**,避免长命令被内置 120s 默认值误杀 [aio_sandbox.py:107-111](../backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox.py#L107-L111)。远程模式下清理由 provisioner 代理:`RemoteSandboxBackend` 是个薄 HTTP 客户端,create `POST /api/sandboxes`(30s 超时)、destroy `DELETE`(15s)、is_alive/discover `GET`(10s,404 即不存在);`list_running` 的存在同样是为了让进程重启后能收养上一个生命周期留下的 k3s Pod [remote_backend.py:89-99](../backend/packages/harness/deerflow/community/aio_sandbox/remote_backend.py#L89-L99)、[remote_backend.py:135-207](../backend/packages/harness/deerflow/community/aio_sandbox/remote_backend.py#L135-L207)。

## 面试官最爱追问的 3 个点

1. **"你的安全边界到底在哪一层?"** —— 应答策略:先给结论"容器是唯一硬边界,local 模式默认禁 bash",再用 `is_host_bash_allowed` 的判定顺序 [security.py:35-45](../backend/packages/harness/deerflow/sandbox/security.py#L35-L45) 和 `_resolve_path_with_mapping` 的 resolve+relative_to [local_sandbox.py:193-198](../backend/packages/harness/deerflow/sandbox/local/local_sandbox.py#L193-L198) 说明"视图层映射 + 默认拒绝 + 显式 opt-in + best-effort 扫描"的分层减伤。主动承认 `validate_local_bash_command_paths` 的 docstring 自述"不是安全边界"反而加分——说明读过源码且理解威胁模型,而不是把正则当隔离。

2. **"多进程/多 pod 下同一 thread 的 acquire 竞态怎么解?"** —— 应答策略:按"进程内 per-thread lock → 确定性 ID(`sha256(thread_id)[:8]`)→ 跨进程 flock(`{sandbox_id}.lock`)→ 锁内 re-check 缓存与 discover → docker 容器名冲突再 discover 收养"的五层防线递进回答 [aio_sandbox_provider.py:735-765](../backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox_provider.py#L735-L765)、[local_backend.py:303-307](../backend/packages/harness/deerflow/community/aio_sandbox/local_backend.py#L303-L307),强调每一层防的是不同的竞态窗口:线程间、进程间、以及与 Docker daemon 状态之间。

3. **"replicas=3 超了会发生什么?为什么不硬限?"** —— 应答策略:软上限只驱逐 warm pool 最旧闲置容器,活跃容器绝不强杀,超限仅 warning 照建 [aio_sandbox_provider.py:627-643](../backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox_provider.py#L627-L643);一句话点破本质——容量治理的敌人是闲置资源,不是正在服务的请求;再补一句兜底:超建的容器最终仍受 `idle_timeout=600s` 的 idle checker 回收,规模会自然收敛回水位线。
