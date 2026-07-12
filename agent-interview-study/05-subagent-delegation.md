# Module 5：Sub-Agent 多智能体委派

**核心文件**：
- [tools/builtins/task_tool.py](backend/packages/harness/deerflow/tools/builtins/task_tool.py)
- [subagents/executor.py](backend/packages/harness/deerflow/subagents/executor.py)
- [subagents/registry.py](backend/packages/harness/deerflow/subagents/registry.py)

## 一、入口：`task_tool`——模型看到的唯一接口

[task_tool.py:187-322](backend/packages/harness/deerflow/tools/builtins/task_tool.py#L187-L322)，模型调用时只传 `description`/`prompt`/`subagent_type` 三个参数，但工具内部做了大量"继承 + 隔离"的工作：

```python
sandbox_state = runtime.state.get("sandbox")      # 继承父 agent 的 sandbox
thread_data = runtime.state.get("thread_data")    # 继承父 agent 的线程目录
parent_model = metadata.get("model_name")         # 子 agent 默认沿用父模型（除非 config 里指定）
parent_available_skills = metadata.get("available_skills")
overrides["skills"] = _merge_skill_allowlists(list(parent_available_skills), config.skills)
...
tools = get_available_tools(..., subagent_enabled=False)  # 关键：子 agent 拿不到 task 工具
```

**两处安全相关的细节**：
1. **`_merge_skill_allowlists`**（[task_tool.py:176-184](backend/packages/harness/deerflow/tools/builtins/task_tool.py#L176-L184)）：如果父 agent 本身有技能白名单限制，子 agent 配置里声明的技能列表要**取交集**，不能超出父 agent 被允许的范围——防止"子任务反而拿到了比主任务更大的权限"这种权限升级漏洞。
2. **`subagent_enabled=False`**（[task_tool.py:299](backend/packages/harness/deerflow/tools/builtins/task_tool.py#L299)）：子 agent 拿到的工具列表**永远不包含 `task` 工具本身**，从根上防止"子 agent 又调 task 委派孙 agent"这种无限递归。

## 二、执行引擎：两级调度设计

`task_tool` 拿到 `executor.execute_async(prompt, task_id=tool_call_id)`（[executor.py:794-855](backend/packages/harness/deerflow/subagents/executor.py#L794-L855)）：

```python
def execute_async(self, task, task_id=None):
    ...
    def run_task():
        ...
        execution_future = _submit_to_isolated_loop_in_context(
            parent_context, lambda: self._aexecute(task, result_holder),
        )
        try:
            execution_future.result(timeout=self.config.timeout_seconds)   # ← 真正的硬超时
        except FuturesTimeoutError:
            result_holder.cancel_event.set()
            result_holder.try_set_terminal(SubagentStatus.TIMED_OUT, ...)
            execution_future.cancel()
        ...
    _scheduler_pool.submit(run_task)   # 提交到"调度池"（3 worker 线程）
    return task_id
```

这里有**两层**，容易被面试问到"为什么不直接 `asyncio.create_task`"：

- **第一层（`_scheduler_pool`，普通线程池）**：跑 `run_task`，它要做一件 asyncio 原生做不好的事——**用 `concurrent.futures.Future.result(timeout=...)` 实现真正意义上的硬超时阻塞等待**。这是同步阻塞调用，只能放在独立线程里，不能占用 event loop 的线程。
- **第二层（`_isolated_subagent_loop`，一个常驻的独立事件循环+专属线程）**：`run_task` 把真正的协程 `self._aexecute(...)` 通过 `asyncio.run_coroutine_threadsafe` 扔进这个常驻循环去跑。

**为什么要有一个"常驻"的事件循环，而不是每次 `asyncio.run()` 起一个新的**（[executor.py:145-150](backend/packages/harness/deerflow/subagents/executor.py#L145-L150) 的注释说得很直接）：

> Reusing one long-lived loop avoids creating a fresh loop per execution and then closing async resources bound to it.

子 agent 内部可能用到 MCP 工具的 async client（比如 httpx 长连接），这些 client 一旦绑定到某个 event loop，这个 loop 关掉了它们也就废了。如果每次子任务执行都新建一个 loop、跑完就关掉，绑定在这个 loop 上的连接池/client 也要跟着重建——常驻 loop 避免了这种反复重建的开销。

## 三、一个"文档过时"的真实案例

`CLAUDE.md` 和 `docs/task_tool_improvements.md` 里都写着"双线程池设计：`_scheduler_pool`（3 workers）+ `_execution_pool`（3 workers）"，但 `grep -n "_execution_pool\|_scheduler_pool" executor.py` 只能命中：

```
143:_scheduler_pool = ThreadPoolExecutor(max_workers=3, thread_name_prefix="subagent-scheduler-")
854:        _scheduler_pool.submit(run_task)
```

`_execution_pool` 这个符号在当前源码里已经不存在了。也就是说架构已经从"双线程池"演进成了"一个调度线程池 + 一个常驻事件循环"，但两份文档都没跟着更新——这是个很典型的"文档描述的是某个历史版本的架构，代码已经往前走了"的真实案例。回答面试里"你怎么核实一个系统的实际架构"这类问题时，这个例子比空谈"要读源码"更有说服力。

## 四、协作式取消（cooperative cancellation）

[executor.py:584-596](backend/packages/harness/deerflow/subagents/executor.py#L584-L596)：

```python
async for chunk in agent.astream(state, config=run_config, context=context, stream_mode="values"):
    if result.cancel_event.is_set():
        result.try_set_terminal(SubagentStatus.CANCELLED, error="Cancelled by user", ...)
        return result
    final_state = chunk
    ...
```

`cancel_event` 是一个 `threading.Event`，取消操作只是"设置一个标志位"，真正生效要等到 **`astream` 的下一次 chunk 产出**（也就是子 agent 内部下一次 model/tool 调用完成的间隙）才会被检测到并退出循环。代码注释也承认了这个局限（[L585-588](backend/packages/harness/deerflow/subagents/executor.py#L585-L588)）：如果某次工具调用本身特别慢（比如一次卡住的网络请求），在它返回之前取消是不会立刻生效的——这是"协作式取消"天生的代价：**不能强制打断一个正在执行的同步/长阻塞操作，只能在检查点上响应**。

## 五、`try_set_terminal`——防止两条并发路径互相覆盖结果

[executor.py:102-135](backend/packages/harness/deerflow/subagents/executor.py#L102-L135)：

```python
def try_set_terminal(self, status, *, result=None, error=None, ...) -> bool:
    with self._state_lock:
        if self.status.is_terminal:
            return False              # 已经终结过了，后来者写不进去
        ...
        self.status = status
        return True
```

**为什么需要这把锁和这个"只成功一次"的语义**：`SubagentResult` 会被两条独立的执行路径同时touch——① `_aexecute` 协程本身跑完/出异常时想把状态标成 `COMPLETED`/`FAILED`；② `run_task` 里的 `FuturesTimeoutError` 分支，在等待超时后想把状态标成 `TIMED_OUT`。这两件事可能几乎同时发生（比如子任务正好在超时那一瞬间完成）。`try_set_terminal` 用锁 + "已终结就直接返回 False，不再改写"保证**谁先到谁说了算**，后到的写入被静默丢弃，而不是覆盖掉已经产生的正确结果（类似 Module 2 讲过的"first write wins"思路，但这里是用真正的锁而不是消息排队）。

## 六、`registry.py`——配置解析的优先级链

[registry.py:50-116](backend/packages/harness/deerflow/subagents/registry.py#L50-L116)，解析顺序是：**内置配置 → config.yaml 的 custom_agents → config.yaml 的 agents 段（per-agent override）**，但有一处容易忽略的不对称：

```python
if agent_override is not None and agent_override.timeout_seconds is not None:
    overrides["timeout_seconds"] = agent_override.timeout_seconds
elif is_builtin and subagents_config.timeout_seconds != config.timeout_seconds:
    overrides["timeout_seconds"] = subagents_config.timeout_seconds   # 全局默认值
```

**全局默认超时/轮次只会应用到内置 agent（`general-purpose`/`bash`），不会应用到自定义 agent**——因为自定义 agent 在 `custom_agents` 里已经声明了"自己的默认值"，如果全局默认再去覆盖它，等于用户在 `custom_agents.my_agent.timeout_seconds` 里精心配的值被一个跟这个 agent 毫无关系的全局开关悄悄覆盖掉。只有"per-agent 显式 override"（`agents.my_agent.timeout_seconds`）才有资格覆盖自定义 agent 的默认值——这是"谁的默认值就该谁的优先级链条覆盖"的一个具体设计取舍。

## 七、总结表

| 组件 | 角色 |
|---|---|
| `task_tool` | 模型侧唯一入口；负责继承父 agent 状态、做技能白名单交集、禁止 `task` 工具递归 |
| `SubagentExecutor._scheduler_pool` | 独立线程池，专门负责阻塞等待硬超时（`Future.result(timeout=...)`） |
| `_isolated_subagent_loop` | 常驻单例事件循环，真正跑子 agent 的 `astream`，避免反复创建/销毁 async 资源 |
| `cancel_event` | 协作式取消标志，只在 `astream` 迭代边界被检查，不能打断正在进行的单次工具调用 |
| `try_set_terminal` + `_state_lock` | 防止"正常完成"和"超时"两条路径竞态覆盖终态结果 |
| `registry.py` 解析优先级 | 内置 → custom_agents → 显式 per-agent override；全局默认值不覆盖自定义 agent 自己的默认值 |
| 文档 vs 代码 | `CLAUDE.md`/`task_tool_improvements.md` 仍写"双线程池"，当前代码已是"调度池+常驻循环"——文档滞后的真实样本 |
