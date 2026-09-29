# 问答 006：第三课练习讲解与验收

日期：2026-09-15。

## 用户

1. 不会写对象和新增字段，没看源代码；ToolInvocation；call_id；是两个 Router 阶段
2. 3. 不会，你来解释

## 助手

# 第三课练习讲解与验收

日期：2026-09-15

结论先说：第一题的概念判断部分正确，但没有完成源码字段追踪；第二、三题尚未作答。下面先把答案讲清楚。第三课暂不记为“练习通过”，需要你之后用简化模板重新复述一次，才能进入下一节。

## 1. 第一题：你已经答对的部分

你写出的三个判断是对的：

1. 运行时真正交给执行器的是 ToolInvocation；
2. call_id 主要做调用、结果、轨迹和历史之间的关联，不自动提供 exactly-once；
3. ToolRouter 的两次出现是两个阶段：第一次把模型响应解析成 ToolCall，第二次把 ToolCall 和运行时上下文组合成 ToolInvocation。

“不会写对象和新增字段、没看源代码”说明当前缺口是源码阅读和对象变换，不是概念方向错。现在把这条链展开。

## 2. 第一题：对象和新增字段的完整追踪

假设模型输出：

~~~text
call_id = C7
name = exec_command
arguments = {"cmd":"write file A"}
~~~

| 阶段 | 对象/状态 | 这一阶段得到的字段或事实 |
| --- | --- | --- |
| 1 | ResponseItem::FunctionCall | name、namespace、arguments、call_id、可选的 encrypted_function_args；这是模型响应中的原始工具调用 |
| 2 | ToolCall | tool_name、call_id、payload、encrypted_function_args；其中 payload 变成 ToolPayload::Function { arguments } |
| 3 | handle_output_item_done | 先记录模型产生的 FunctionCall；从 turn 取消令牌创建 child token；创建 in-flight tool future；标记 needs_follow_up=true |
| 4 | ToolCallRuntime | 持有 session、step_context、tracker 和并行执行锁；判断工具是否支持并行；等待 readiness/admission；根据结果派生调用 source |
| 5 | ToolInvocation | 保留 call_id、tool_name、payload；新增 session、turn、step_context、cancellation_token、tracker、source |
| 6 | ToolRegistry | 查找工具、检查 payload 类型、执行 pre-tool-use hook；工具不存在时返回面向模型的错误，不会进入执行器；hook 也可能阻止或重写调用 |
| 7 | ToolExecutorFuture 完成 | 得到 ToolOutput 或 FunctionCallError；封装为带 call_id 的 AnyToolResult/ResponseItemEnvelope；drain_in_flight 再把结果写入 history 和 rollout |

最重要的字段对应关系是：

~~~text
ToolCall:
    tool_name
    call_id
    payload
    encrypted_function_args

ToolInvocation:
    上面仍要执行的调用事实
    + session
    + turn
    + step_context
    + cancellation_token
    + tracker
    + source
~~~

这里的“新增”不是模型突然生成了更多参数，而是 Runtime 在 dispatch 前把执行所需的上下文附加进来。

三个追问的答案：

- 携带产生该调用的 step 环境快照的是 step_context；环境不是 ToolInvocation 的直接 environment_id 字段，而是通过 step_context.environments 里的 TurnEnvironmentSnapshot 取得。
- 只能做关联、不能保证幂等的是 call_id。
- 两个 Router 阶段不是重复执行：build_tool_call 负责 ResponseItem → ToolCall；dispatch 负责 ToolCall + 运行时上下文 → ToolInvocation。真正调用执行器发生在后一个阶段的 Registry/Executor 路径。

还要注意：ToolInvocation 构造时，Function payload 中的 arguments 仍然是字符串。JSON 解析、字段检查、hook 重写和授权判断是后续步骤，不是构造对象本身自动完成的。

## 3. 第二题：取消窗口如何判断

ToolCallRuntime 的关键分支是：

~~~text
等待 dispatch future 和 cancellation_token
    ↓
如果 terminal outcome 已到达，或 dispatch 已经结束
    → 等待已有结果并返回
否则
    → abort dispatch task
    → 生成 aborted response
    → notify_tool_aborted
~~~

三个时间点要分开看：

| 取消时刻 | Handler 是否一定启动 | Tool Result | 外部副作用 | 是否能盲目重试 |
| --- | --- | --- | --- | --- |
| 尚未通过 admission gate | 当前这次 handler 没有开始；execution_started=false | 通常生成 aborted response | 由这次尚未启动的 handler 产生的副作用不存在；但这不是对其他任务或历史重试的证明 | 只有确认没有执行并使用幂等键时才可重试 |
| Handler 已启动但尚未返回 | 已经启动 | 可能是 aborted，也可能与正常完成竞争 | 可能已经写文件、启动进程或发出网络请求，取消不保证回滚 | 不能盲目重试，应先查询状态或使用相同幂等键 |
| Terminal outcome 已到达，正在完成记录 | 已经产生可接受的结果 | Runtime 倾向于保留并返回已有结果，取消不一定抹掉成功结果 | 副作用可能已经完成 | 不能因为客户端超时就重做；先查结果、history、rollout 或外部状态 |

第二行是最容易出错的地方。取消只说明控制流收到了取消信号，不说明 handler 的外部动作被撤销。例如：

~~~text
写文件完成
  → handler 还没返回
  → 用户取消
  → Tool Result 可能变成 aborted
~~~

此时“结果显示 aborted”和“文件没有写入”可以同时不成立。文件可能已经存在，而结果只是没有正常回传。

第三行也不能简化为“取消失败”。如果 terminal outcome 已经到达，当前源码会优先等待/采用已经形成的结果；但结果进入 history、rollout append 仍是后续阶段，持久化仍可能失败。

所以四个问题的严谨回答是：

1. Handler 是否启动，要看 cancellation 发生在 admission 前还是之后；
2. 外部副作用是否存在，不能只看取消结果，要看 handler 和外部系统；
3. Tool Result 是否存在，要区分 aborted response、正常结果和结果持久化；
4. 是否安全重试，必须先确认外部状态，或让工具用同一个幂等键把重试合并。

当前源码可以证明 runtime 如何中止任务，不能单独证明每个工具的外部副作用都可撤销。

## 4. 第三题：write_file 的 at-least-once 恢复方案

### 4.1 先定义稳定的操作身份

不能只用 call_id，原因有四个：

1. 同一个用户意图被模型重新生成时，可能得到新的 call_id；
2. 进程重放时可能重复使用旧 call_id；
3. 外部文件系统或远端 API 未必收到 Codex 的 call_id；
4. call_id 的职责是关联当前调用，不是声明外部副作用的生命周期。

应当为写操作建立独立的 operation_id，并在第一次副作用前持久化。它可以由稳定的任务意图、规范化目标路径、内容 hash、期望的旧版本和客户端生成的操作序号组合而成。实际设计还要决定“用户明确要求重复写同一内容”是否算新的操作，因此不能机械地只用 path + hash。

一个可行的记录是：

~~~text
operation_id
canonical_target = A
content_hash
expected_version
temporary_path
state
created_at
~~~

### 4.2 状态机

建议至少有这些状态：

~~~text
PREPARED
  → TEMP_WRITTEN
  → COMMITTING
  → COMMITTED
  → RESULT_RECORDED
~~~

崩溃后不能判断时，增加 UNKNOWN，而不是猜测为失败：

~~~text
UNKNOWN
  → 查询临时文件、目标文件、版本或远端 operation status
  → COMMITTED / NOT_COMMITTED / CONFLICT
~~~

状态记录必须独立于最终 Tool Result。因为 rollout 可能还没有写入，但文件副作用已经完成；如果只把状态放在 Tool Result 里，最需要恢复的窗口反而没有证据。

### 4.3 写入流程

1. 规范化路径，读取或记录 expected_version，计算内容 hash；
2. 先持久化 PREPARED 和 operation_id；
3. 在目标文件同一目录创建临时文件；
4. 写入完整内容，flush/fsync，并重新计算 hash；
5. 使用同一 operation_id 标识临时文件；
6. 在版本仍等于 expected_version 时执行原子提交；避免无条件覆盖并发修改；
7. 持久化 COMMITTED 和最终版本/hash；
8. 生成带 operation_id、最终 hash 和 committed 状态的 Tool Result；
9. 即使 rollout 记录失败，也能从操作记录重新返回相同结果。

原子 rename 解决的是“目标文件不要暴露半写内容”，不单独解决并发覆盖、重复执行和结果丢失。因此还需要 expected_version 或 compare-and-swap 语义。

### 4.4 四个崩溃窗口如何恢复

| 崩溃窗口 | 恢复动作 |
| --- | --- |
| 临时文件写完但未 rename | 校验临时文件 hash；目标未改变则继续提交或清理；不要直接把半文件当成成功 |
| rename 已完成但 handler 未返回 | 比对目标文件 hash、版本和 operation_id；吻合则标记 COMMITTED，返回已有结果，不重复写 |
| handler 已返回但 Tool Result 未进 rollout | 从 operation ledger 读取 COMMITTED，重建并补发结果；不要再次执行写入 |
| rollout 已记录但客户端超时重试 | 用 operation_id 查询 ledger/history/目标版本；若已完成，返回同一结果；若冲突，报告冲突，不盲目覆盖 |

这就是 at-least-once 下的核心：执行可能重复到达，但相同 operation_id 的最终效果和返回结果可以收敛。它仍然不是跨文件系统、Codex history 和远端服务的 exactly-once 分布式事务。

### 4.5 如果副作用在远端

远端服务需要支持：

- 请求中的幂等键；
- 可查询的 operation status；
- 已提交结果的重放；
- 版本或条件写入；
- 对 UNKNOWN 状态的查询。

Codex 本地在超时后不应该直接重新发送“写入请求”，而应该先用 operation_id 查询远端：NOT_FOUND、IN_PROGRESS、COMMITTED、CONFLICT 或 UNKNOWN。只有确认未提交，才允许按协议继续。

## 5. 这次验收结果

第一题：部分通过。你已经理解 ToolInvocation、call_id 和两个 Router 阶段；缺少源码对象字段追踪。

第二题：本次没有作答，现已解释 cancellation、admission、terminal outcome、Tool Result 和外部副作用的区别。

第三题：本次没有作答，现已给出 operation_id、状态机、临时文件、原子提交、版本检查和远端查询方案。

因此当前状态是：第三课已完成讲解，但练习尚未通过，暂不进入下一节。

请下一条只用下面三句话重新回答，不需要再写长文：

1. ToolCall 到 ToolInvocation 时，至少列出三个新增的运行时字段，并指出环境快照在哪里。
2. “Handler 已启动但未返回时取消”为什么不能直接安全重试？
3. write_file 为什么需要独立 operation_id？恢复时如何判断“已写入但 Tool Result 丢失”？
