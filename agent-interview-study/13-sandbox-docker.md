# 第十三部分：沙箱 Docker 实现 —— 容器生命周期与跨进程协调

> 本篇是对第 06 篇(沙箱执行环境与安全)的深入补充。第 06 篇主要讲 `sandbox/` 抽象层、安全策略、工具接口;本篇聚焦 **Docker 容器的具体实现、生命周期管理、多进程协调、部署拓扑**,回答"沙箱 Docker 到底是怎么跑起来的"这个工程问题。
>
> 读完本篇你应能回答:一个 thread 的沙箱容器什么时候被创建、什么时候被复用、什么时候被销毁?多个进程怎么避免重复创建同一个容器?DooD 场景下路径和端口怎么处理?为什么说 `LocalSandboxProvider` 不算真正的沙箱?

## 1. 先纠正一个常见误读

"沙箱 docker" 在 deer-flow 里**不是**本地化的 `docker run` 命令。真正的安全边界是 [AioSandboxProvider](../backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox_provider.py#L108) —— 默认镜像 `enterprise-public-cn-beijing.cr.volces.com/vefaas-public/all-in-one-sandbox:latest`([aio_sandbox_provider.py:45](../backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox_provider.py#L45)),每个 thread 一个容器,通过 **HTTP API**(容器内 8080 端口)执行命令和读写文件,不是 docker exec。

那个叫 `LocalSandboxProvider` 的才是不启动任何容器、直接在宿主机跑 bash 的(为了没有 docker 环境的开发机)。而且 [security.py:10-20](../backend/packages/harness/deerflow/sandbox/security.py#L10-L20) 明确写了本地模式 **默认禁用 bash**,因为它不是安全边界 —— "LocalSandboxProvider 不算沙箱" 这个态度本身就是设计判断。

## 2. 整体架构:三层,每层各自解耦

```
   ┌─────────────────────────────────────────────────────────┐
   │ 第1层 抽象  Sandbox / SandboxProvider / SandboxBackend   │  ← "给什么用什么"
   │           (sandbox.py, sandbox_provider.py, backend.py) │
   └─────────────────────────────────────────────────────────┘
                            ▼
   ┌─────────────────────────────────────────────────────────┐
   │ 第2层 实现  AioSandbox (HTTP 客户端,沙箱使用方)         │  ← "怎么用沙箱"
   │             LocalSandbox (host-local,无隔离)            │
   └─────────────────────────────────────────────────────────┘
                            ▼
   ┌─────────────────────────────────────────────────────────┐
   │ 第3层 供给  LocalContainerBackend (本地 Docker 起容器)   │  ← "沙箱从哪来"
   │             RemoteSandboxBackend (K8s provisioner)      │
   └─────────────────────────────────────────────────────────┘
```

三个抽象类对应三个职责:
- `Sandbox`([sandbox.py](../backend/packages/harness/deerflow/sandbox/sandbox.py)) —— "给什么用什么":8 个抽象方法(execute_command/read_file/download_file/list_dir/write_file/glob/grep/update_file),屏蔽 LocalSandbox(子进程)和 AioSandbox(HTTP)的差异。
- `SandboxProvider`([sandbox_provider.py](../backend/packages/harness/deerflow/sandbox/sandbox_provider.py)) —— "给谁发一个、怎么收回":acquire/get/release 生命周期,单例模式 + 可注入测试 provider。
- `SandboxBackend`([backend.py](../backend/packages/harness/deerflow/community/aio_sandbox/backend.py)) —— "沙箱从哪来":create/destroy/is_alive/discover/list_running,LocalContainerBackend(本地 docker)vs RemoteSandboxBackend(K8s provisioner)。

三层独立意味着:换 K8s 不影响 `AioSandbox` 怎么用沙箱,换沙箱镜像不影响 provider 的生命周期管理。

## 3. 第 3 层 LocalContainerBackend:真正的 docker run 在这里

[local_backend.py:513-593](../backend/packages/harness/deerflow/community/aio_sandbox/local_backend.py#L513-L593) 是唯一拼 `docker run` 命令的地方。每一行命令背后都是一个具体问题:

| 代码位置 | 做什么 | 为什么 |
|---|---|---|
| L233-258 | macOS 优先用 Apple Container(`container` CLI),降级 Docker | 不为 macOS 用户单独开一类,自动适配 |
| L536 | `--security-opt seccomp=unconfined` | docker 默认 seccomp 会拦截部分 syscall,沙箱里跑用户任意 Python 可能踩到,**主动**关掉 —— 功能 vs 安全的明确取舍 |
| L545 | `--rm` | 容器退出自动删除,不留垃圾 |
| L286-310 | 端口分配 + 10 次重试 | `get_free_port` 是宿主机视角的 free,docker 对 0.0.0.0 绑定的端口释放**异步**,可能上一秒 free 下一秒 docker 又告诉你"已被占用",所以报 "port is already allocated" 就换端口重试 |
| L303-307 | 容器名冲突 → adopt 而不是 fail | `docker run` 报 "container name already in use" 时不报错,调 `self.discover(sandbox_id)` 看是不是另一个进程已经为同一 thread 起好了容器 —— 是的话直接复用 |
| L70-87 | docker 用 `--mount type=bind,...`,Apple Container 用 `-v` | docker 的 `-v host:container` 语法在 Windows 下 `D:/path` 会被解析成卷定义的一部分,**docker 用显式 `--mount` 语法**避免歧义 |
| L142-169 | 默认 `-p 127.0.0.1:{port}:8080` 只绑 loopback;DooD 场景绑 `0.0.0.0` | 最低权限原则:裸机/本地只暴露 localhost,容器外访问才需要 0.0.0.0 |
| L90-121 | `_redact_container_command_for_log` | docker run 里带 `-e API_KEY=xxx`,**打到日志前**把 `=` 后面的值替换成 `<redacted>`,防凭证进日志 |

其他关键操作:
- `destroy`(L322-338):`docker stop`,`--rm` 保证 stop 后容器自动删除。
- `discover`(L346-386):给定 sandbox_id,容器名是确定的 `{prefix}-{sandbox_id}`,任何进程都能 `docker inspect` 这个名字看容器在不在、健康检查通不通 —— **不需要共享状态文件**,这是"跨进程发现"的实现方式。
- `list_running`(L388-463):`docker ps --filter name={prefix}-` + 单次 batched `docker inspect` 拿到所有容器的创建时间和端口 —— 用于进程重启后的孤儿容器回收。

## 4. 第 2 层 AioSandbox:怎么用这个 docker 容器

[aio_sandbox.py](../backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox.py) 是 `Sandbox` 抽象类的 AIO 实现,**所有操作都通过 HTTP 调容器里的 agent_sandbox 服务**(容器内 8080 端口):

- `execute_command`(L113-155):HTTP 调 `shell.exec_command`,**带 threading.Lock**(L41) —— AIO 容器维护**单个持久 shell session**,并发 exec_command 会把 session 搞坏(返回 `ErrorObservation` 而不是真输出)。如果加锁后还是检测到损坏,自动**新建一个 session 重试**,重试完清理这个临时 session(L140-150)。
- `read_file`/`write_file`/`download_file`/`update_file`:HTTP 调 `file.*` API。`update_file`(L336-349)走 base64 编码传输二进制。
- `download_file` 在 HTTP 请求**之前**做路径遍历检查(L184-194) —— `LocalSandbox` 是靠路径解析隐式拒绝的,这里显式检查,因为路径会被原样转发给容器 API。
- `grep`/`glob`(L256-334):也是 HTTP API(`file.find_files`/`search_in_file`),但要本地再过滤一次 `should_ignore_path`(比如 `node_modules`),并在本地 `re.compile` 先验证一次正则,避免无效正则到了远端才报错、错误类型不清晰。
- `close()`(L48-97):agent_sandbox SDK 是 Fern 自动生成的,**没有 close() 方法**,要顺着属性链 `_client_wrapper.httpx_client.httpx_client` 一路找到真正持有 socket 的 `httpx.Client` 才能关 —— 深入第三方库内部结构做资源管理。

## 5. 第 1 层抽象 + AioSandboxProvider:生命周期管理的核心

### 5.1 Warm pool(核心优化)

- `release()`([aio_sandbox_provider.py:883-922](../backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox_provider.py#L883-L922))**不销毁容器**,把 sandbox_id 放进 `_warm_pool`,容器继续跑。下一次 `acquire` 同一个 thread 时,`_reclaim_warm_pool_sandbox`(L499-532)直接从 warm pool 拿回来 —— **避免 docker 冷启动**(拉镜像、起进程、挂载卷可能好几秒)。
- 容量管理:`replicas` 默认 3(L49),超出时 `_evict_oldest_warm`(L797-815)销毁最老的 warm 容器腾位置。但**只驱逐 warm 的,活跃的永不驱逐** —— 注释明确说 "The replicas limit is a soft cap; we never forcibly stop a container that is actively serving a thread"(L642)。
- 后台 `_idle_checker_loop`(L357-363)每 60s 扫一次,`idle_timeout` 默认 600s(L48),把超时的 active 和 warm 容器都销毁。

### 5.2 跨进程安全

- `_deterministic_sandbox_id`([aio_sandbox_provider.py:279-285](../backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox_provider.py#L279-L285)):`sha256(thread_id)[:8]` —— gateway 进程和 langgraph 进程算出的 sandbox_id 是一样的,这是跨进程发现的前提。
- **文件锁**(`_lock_file_exclusive`/`_unlock_file`,L56-71):用 `fcntl.flock`(Linux/macOS)或 `msvcrt.locking`(Windows)在 `~/.deer-flow/threads/{thread_id}/{sandbox_id}.lock` 上加排他锁。两个进程同时 acquire 同一个 thread 时,先拿到锁的进程创建容器,后拿到的进程在锁内 `_recheck_cached_sandbox`(L534-536)发现容器已经被对方起好了,直接复用 —— **防止 docker 容器名冲突**。
- `_acquire_thread_lock_async`(L78-90)用 `asyncio.shield` + `run_in_executor` 把 `threading.Lock.acquire` 放到专用线程池(`_THREAD_LOCK_EXECUTOR`,L52)等 —— 不阻塞事件循环。如果被 cancel,`_release_cancelled_lock_acquire`(L93-105)兜底释放可能"acquire 成功但 coroutine 已被 cancel"的孤儿锁。

### 5.3 孤儿容器回收(对 docker 垃圾的兜底)

`_reconcile_orphans`(L233-274)在 provider 启动时,调 `list_running()` 列出所有 `{prefix}-` 开头的容器,**无条件**全部 adopt 进 warm pool —— 因为进程重启后 in-memory 状态丢了,你不知道哪些容器是"上一个进程留下的"哪些是"另一个正在跑的进程还在用的"。注释里写得很清楚:"`idle_timeout` represents inactivity, not uptime"(L245),所以只能靠 idle checker 后续自然回收,不能根据年龄直接杀。

这个设计堵住了"进程崩溃/SIGKILL 导致 docker 容器永远泄漏"这个根本性漏洞(L248)。

### 5.4 Volume 挂载(安全边界的关键)

`_get_thread_mounts`([aio_sandbox_provider.py:304-323](../backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox_provider.py#L304-L323))每个 thread 挂 4 个卷:
- `~/.deer-flow/threads/{thread_id}/sandbox/workspace` → 容器内 `/mnt/user-data/workspace`(读写)
- `.../uploads` → `/mnt/user-data/uploads`(读写)
- `.../outputs` → `/mnt/user-data/outputs`(读写)
- `.../acp_workspace` → `/mnt/acp-workspace`(**只读**,lead agent 只读 ACP 子 agent 的产出)

`_get_skills_mount`(L326-343)把 skills 目录挂进容器,也是**只读** —— 技能文件不允许 agent 在沙箱里改。

DooD(Docker-outside-of-Docker)情况下,**容器内的 docker 客户端连的是宿主机的 docker daemon**,所以挂载源路径必须是宿主机路径,不是容器内路径 —— 代码里通过 `DEER_FLOW_HOST_BASE_DIR`/`DEER_FLOW_HOST_SKILLS_PATH` 环境变量切换(L310、L339)。

## 6. 沙箱容器的真实生命周期:不是"一次对话一个容器"

这是面试高频误解点。**不是每次对话/每个 HTTP 请求都创建容器,而是每个 thread_id 一个容器,整个对话生命周期内复用。**

`acquire()` 的复用优先级([aio_sandbox_provider.py:686-713](../backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox_provider.py#L686-L713))是"能不新建就不新建":

```
1. 先查 in-process cache (_reuse_in_process_sandbox)  → 命中直接用,零开销
2. 再查 warm pool (_reclaim_warm_pool_sandbox)          → 容器还在跑,直接拿回来用
3. 都没有 → 文件锁内 _discover_or_create_with_lock:
   a. 先 _recheck_cached_sandbox 再查一遍
   b. backend.discover(sandbox_id) 看别的进程是不是已经起好了
   c. 都不行才 backend.create() → docker run
```

第 3c 步真正调 `docker run` 的,只在"这个 thread 从未有过容器、或容器已被回收"时发生。

完整生命周期:

```
Thread 第一次提问  → acquire() → 没有缓存 → docker run (冷启动,秒级)
         ↓
同 thread 第 2、3、4... 次提问 → acquire() → warm pool 命中 → 直接用,毫秒级
         ↓
Thread 静默 10 分钟 → idle checker 发现超时 → docker stop (容器销毁)
         ↓
同一 thread 再来提问 → 又得 docker run 冷启动一次
```

容器只有三种情况被销毁:
- 超过 `idle_timeout`(默认 600s)没活动,被后台 `_idle_checker_loop` 回收
- warm pool 超过 `replicas`(默认 3),最老的被 LRU 驱逐腾位置
- 进程 shutdown 时统一清理

如果真是每个 HTTP 请求都 `docker run`,冷启动几秒的开销根本没法用 —— warm pool + deterministic sandbox_id + 跨进程文件锁这一整套设计,就是为了**在"需要强隔离"和"需要低延迟"之间取平衡**。

## 7. docker-compose.yaml:部署拓扑印证这个分层

[docker-compose.yaml](../docker/docker-compose.yaml) 里有几个细节和代码完全对得上:

- gateway 服务**不挂 docker.sock**(L87-90),默认 LocalSandbox 模式。要用 AIO 沙箱需要**显式 opt-in** `docker-compose.dood.yaml` overlay 才会挂 `/var/run/docker.sock` —— 默认不给权限、需要才开的纵深防御姿态。
- `GATEWAY_WORKERS` 默认 1(L81),注释明确说多 worker 会破,因为 RunManager/StreamBridge 是 in-process singleton,而且 sandbox 的 `_thread_sandboxes` 也是 in-process 的 —— 这解释了为什么 `_reconcile_orphans` 和文件锁是必要的:多进程场景下要么靠 K8s provisioner(RemoteSandboxBackend),要么靠这套跨进程兜底机制。
- `DEER_FLOW_SANDBOX_HOST=host.docker.internal`(L111) —— DooD 场景下容器里的 agent 通过宿主机 docker daemon 起的兄弟容器,访问入口要走 `host.docker.internal` 而不是 `localhost`,这和 `_resolve_docker_bind_host` 绑 `0.0.0.0` 是配套的。

## 8. 常见误解澄清(面试高频追问)

### 8.1 Warm pool 不是线程池

**Warm pool 和线程池是两个完全不同的东西**,名字里都有"pool"但缓存的资源完全不同:

| | Warm Pool | 线程池 (ThreadPoolExecutor) |
|---|---|---|
| **缓存什么** | Docker 容器(正在运行的进程) | OS 线程 |
| **为什么存在** | 避免 Docker 冷启动(几秒) | 避免频繁创建/销毁线程(几毫秒) |
| **容量** | 默认 3 个容器 | 默认 4-32 个线程 |
| **生命周期** | 10 分钟 idle timeout | 进程退出才销毁 |
| **在哪一层** | 沙箱 provider 的业务逻辑 | Python asyncio 的基础设施 |

**Warm pool** 就是一个 dict,缓存的是**还在运行的 Docker 容器**的连接信息(`SandboxInfo`),容器真的在跑(`docker ps` 能看到),只是当前没有请求在用它。

**线程池**是 `concurrent.futures.ThreadPoolExecutor`,预先创建一批 OS 线程,`asyncio.to_thread` 把阻塞操作扔到池子里的某个线程跑。

在沙箱代码里两者并存:
```python
# 线程池:给 asyncio.to_thread 用([aio_sandbox_provider.py:52](../backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox_provider.py#L52))
_THREAD_LOCK_EXECUTOR = ThreadPoolExecutor(max_workers=32, ...)

# Warm pool:缓存 Docker 容器([aio_sandbox_provider.py:143](../backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox_provider.py#L143))
self._warm_pool: dict[str, tuple[SandboxInfo, float]] = {}
```

### 8.2 Warm pool 到底是什么

**Warm pool 是"已经不用了但还没销毁、随时能快速拿回来用的容器池"** —— 一个 LRU 缓存层,放在"正在用的容器"和"彻底销毁"之间。

三种状态:
```
状态 1: Active (正在用)
  _sandboxes = {"x7f3a9b2": AioSandbox(...)}
  ↑ 用户正在对话,agent 正在执行命令

状态 2: Warm Pool (闲置但没销毁)
  _warm_pool = {"x7f3a9b2": (SandboxInfo(...), 1690000000.0)}
  ↑ 对话结束了,但容器还在跑,下次直接用

状态 3: Destroyed (彻底没了)
  (容器被 docker stop,什么都不剩)
```

**为什么叫 "warm"**:
- **Cold(冷)**:从零开始,`docker run` 一个新容器,要拉镜像、起进程、挂载卷,**几秒到十几秒**
- **Warm(温)**:容器已经起好了,进程在跑,卷已挂载,**直接用,毫秒级**

**容器怎么进 warm pool**:`release()` 被调用时([aio_sandbox_provider.py:883-922](../backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox_provider.py#L883-L922)),从 `_sandboxes` 挪到 `_warm_pool`,**不执行 `docker stop`**,容器还在跑。

**容器怎么从 warm pool 出来**:下一次同一个 thread 的请求来了,`acquire()` 查 warm pool([aio_sandbox_provider.py:499-532](../backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox_provider.py#L499-L532)),直接从 warm pool 拿出来,**不需要 `docker run`**,毫秒级返回。

**容器怎么从 warm pool 被销毁**:
1. **Idle timeout**(默认 600 秒):后台 `_idle_checker_loop` 每 60 秒扫一次,超时的销毁
2. **Warm pool 满了**(默认 3 个):新的容器 release 时,最老的被驱逐
3. **进程 shutdown**:退出时全部销毁

### 8.3 replicas=3 不是"只能有 3 个容器"

**这是最容易误解的地方**:`replicas=3` 不是"总容器数的上限",是"warm pool 最多保留 3 个闲置容器"。**正在用的(active)容器不受这个限制**。

代码证据([aio_sandbox_provider.py:627-632](../backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox_provider.py#L627-L632)):
```python
def _replica_count(self) -> tuple[int, int]:
    replicas = self._config.get("replicas", DEFAULT_REPLICAS)  # 默认 3
    total = len(self._sandboxes) + len(self._warm_pool)  # active + warm
    return replicas, total
```

驱逐逻辑([aio_sandbox_provider.py:834-837](../backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox_provider.py#L834-L837)):
```python
if total >= replicas:
    evicted = self._evict_oldest_warm()  # 只驱逐 warm 的
```

**只驱逐 warm pool 里的,不驱逐 active 的**。注释明确说([aio_sandbox_provider.py:640-643](../backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox_provider.py#L640-L643)):

> The replicas limit is a soft cap; we never forcibly stop a container that is actively serving a thread.

**实际场景**:

- **5 个用户同时对话** → 5 个 active 容器都在跑,replicas=3 只是触发 warning 日志,不影响创建
- **5 个用户对话结束都 release** → warm pool 只能留 3 个,最老的 2 个被销毁
- **第 1 个用户又回来了** → warm pool 里的容器已被驱逐,要重新 `docker run` 冷启动

**为什么默认 3**:资源 vs 性能的权衡。太大浪费内存/CPU,太小命中率低、用户回来经常冷启动。3 是经验值,假设同时活跃用户不超过 3 个。

## 9. 值得记住的设计判断(面试可以直接用)

1. **Warm pool 是"释放 ≠ 销毁"**:release 把容器留在原地,靠 idle timeout 回收,平衡了冷启动延迟和资源占用。
2. **replicas 是 warm pool 容量,不是总容器数上限**:active 容器不受限制,只驱逐闲置的 —— 这是"资源占用"和"可用性"的权衡,不是硬限制。
3. **"LocalSandbox 不算沙箱"是明确写出来的**:`security.py` 里默认禁用 host bash,因为它不构成安全边界 —— 这是对"沙箱"这个词的严谨定义。
4. **跨进程靠确定性 ID + 文件锁,不靠共享内存**:多进程/多 pod 场景下,`sha256(thread_id)[:8]` 让所有进程算出同一个容器名,文件锁序列化创建过程 —— 这是"无共享状态"的分布式协调。
5. **docker 层的每个坑都有对应处理**:端口异步释放(重试)、容器名冲突(adopt)、Windows 路径(`--mount`)、DooD 路径转换(host path)、凭证进日志(redact)、macOS runtime 切换 —— 这些都不是理论,是踩过的坑。
6. **三层抽象(Sandbox/Provider/Backend)职责清晰**:使用方不用关心沙箱是本地子进程还是 HTTP 容器,提供方不用关心容器是 docker 起还是 K8s 起 —— 每层可以独立替换。

## 10. 小结

沙箱 Docker 在 deer-flow 里不是一个"跑个 docker 命令"的简单功能,而是一套**围绕容器生命周期的工程系统**:三层抽象隔离关注点、warm pool 优化冷启动、确定性 ID + 文件锁实现跨进程协调、孤儿回收兜底资源泄漏、部署拓扑用 docker-compose 表达 DooD/单 worker 约束。面试里讲这一块,重点不是"用了 docker",而是"在分布式、多进程、有状态、有安全约束的场景下,怎么把 docker 容器当成一种需要精细管理的资源"。

**核心要点**:
- Warm pool 是 LRU 缓存,缓存的是**正在运行的 Docker 容器**,不是线程
- `replicas` 是 warm pool 容量(默认 3),不是总容器数上限 —— active 容器不受限制
- "一个用户对话一个 docker" 是对的,但对话结束后容器进 warm pool,不是立刻销毁
- 三种销毁时机:idle timeout(10 分钟)、warm pool 满了(LRU 驱逐)、进程 shutdown
