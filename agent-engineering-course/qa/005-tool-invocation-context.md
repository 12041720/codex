# 问答 005：第三课 ToolInvocation

日期：2026-09-14

## 用户

继续下一节

## 助手

# 第三课：ToolInvocation——一次工具调用为什么需要运行上下文

日期：2026-09-14

源码核验基线：当前 checkout 的 HEAD 为 c222ebe3bbc542ca7698d4eac4c73dbf4dca2e1e。

证据范围：本课根据当前源码静态阅读整理，未启动 Agent、未编译、未运行测试。因此，文中“源码明确做了什么”与“由此推导出的工程含义”分开说明。前两课保留它们各自记录的历史基线，不用当前提交倒写历史。

## 1. 这一课要解决什么问题

上一课已经区分了三件事：

1. 模型生成的是工具调用请求；
2. 工具是否注册，决定是否有执行器可以接手；
3. 参数格式正确，也不等于授权、隔离和执行环境检查已经通过。

这一课继续向下追踪：执行器真正收到的为什么不是只有 tool_name、call_id 和 arguments，而是一个 ToolInvocation。

模型给出的最小请求可以想象成：

~~~text
call_id = C7
tool_name = exec_command
payload = {"cmd":"..."}
~~~

它只能说明“模型想调用什么”。运行时还必须知道：

- 这次调用属于哪个 session 和哪一轮 turn；
- 它使用的是哪一次模型请求所对应的工具列表、配置和环境快照；
- 用户是否取消了这轮调用；
- 本次调用来自普通模型工具、明文协作消息，还是 Code Mode；
- 应该把差异、遥测和最终 Tool Result 记到哪里。

ToolInvocation 就是把这些运行时条件绑定到模型调用上的对象。

## 2. ToolCall 与 ToolInvocation 不是同一个层次

当前源码中，core 层的 ToolCall 主要保存从模型响应中解析出的调用事实：

| 对象 | 主要来源 | 主要字段 | 它能证明什么 |
| --- | --- | --- | --- |
| ToolCall | ToolRouter 从 ResponseItem 构造 | tool_name、call_id、ToolPayload、可选的加密参数 | 模型请求已经被解析成内部调用 |
| ToolInvocation | ToolRouter 在真正 dispatch 前构造 | session、turn、step_context、cancellation_token、tracker、call_id、tool_name、source、payload | 执行器将以哪些运行时上下文处理该调用 |

两者都不等于“副作用已经发生”。ToolCall 更接近模型输出的中间表示；ToolInvocation 是交给执行器的运行时输入。调用对象变成 Invocation，也不表示文件已经写入、命令已经启动或结果已经持久化。

源码位置：

- ToolCall：codex-rs/core/src/tools/router.rs，约第 36—60 行；
- ToolInvocation：codex-rs/core/src/tools/context.rs，约第 60—71 行；
- ToolPayload：codex-rs/core/src/tools/tool_payload.rs。

## 3. 当前调用链：从模型输出到执行器

以一个普通 FunctionCall 为例，当前源码的主要路径是：

~~~text
ResponseItem::FunctionCall
  ↓
ToolRouter::build_tool_call
  ↓
stream_events_utils::handle_output_item_done
  ├─ 先记录模型产生的 FunctionCall
  ├─ 为这次工具调用创建 turn cancellation token 的 child token
  ├─ 创建 ToolCallRuntime::handle_tool_call 返回的 future
  └─ 标记 needs_follow_up，并把 future 放入 in-flight 队列
  ↓
ToolCallRuntime::handle_tool_call
  ├─ 记录执行中的调用
  ├─ 等待 admission/readiness gate
  ├─ 根据工具是否支持并行取得读锁或写锁
  └─ 调用 ToolRouter::dispatch_tool_call_with_terminal_outcome
  ↓
ToolRouter 构造 ToolInvocation
  ↓
ToolRegistry::dispatch_any_with_terminal_outcome
  ├─ 查找工具
  ├─ 检查 payload 类型
  ├─ 执行 pre-tool-use hook
  └─ 调用注册的 ToolExecutor::handle
  ↓
ToolExecutorFuture
  ↓
ToolOutput / FunctionCallError
  ↓
ResponseItemEnvelope
  ↓
Turn::drain_in_flight
  ↓
记录 history、rollout，并发送最终 Tool Result
~~~

这里有两个容易混淆的 Router 阶段：

1. build_tool_call 把模型 ResponseItem 解析成 ToolCall；
2. dispatch 阶段把 ToolCall 和 session、step、取消信号等运行时状态组合成 ToolInvocation。

这不是同一个工具被执行两次，而是“解析模型请求”和“注入运行上下文”两个不同阶段。

源码锚点：

- [handle_output_item_done](../../codex-rs/core/src/stream_events_utils.rs)：约第 300—345 行；
- [ToolCallRuntime](../../codex-rs/core/src/tools/parallel.rs)：约第 43—253 行；
- [ToolRouter 的 dispatch](../../codex-rs/core/src/tools/router.rs)：约第 321—380 行；
- [ToolRegistry 的 dispatch](../../codex-rs/core/src/tools/registry.rs)：约第 495 行以后；
- [drain_in_flight](../../codex-rs/core/src/session/turn.rs)：约第 2356—2384 行。

## 4. ToolInvocation 每个字段的作用

ToolInvocation 当前定义为：

~~~rust
pub struct ToolInvocation {
    pub session: Arc<Session>,
    pub turn: Arc<TurnContext>,
    pub(crate) step_context: Arc<StepContext>,
    pub cancellation_token: CancellationToken,
    pub tracker: SharedTurnDiffTracker,
    pub call_id: String,
    pub tool_name: ToolName,
    pub source: ToolCallSource,
    pub payload: ToolPayload,
}
~~~

逐字段看：

| 字段 | 作用 | 不能误解成 |
| --- | --- | --- |
| session | 让延迟执行的 handler 访问 session 状态、服务和记录接口；用 Arc 保持共享所有权 | 不代表调用一定成功 |
| turn | 当前仍保留的兼容字段；源码注释说明，handler 迁移后应使用 step_context.turn | 一个独立于 step 的新 turn |
| step_context | 绑定这次采样请求的 settings、环境快照、capability roots、MCP、tool router 等 | 只有全局配置的引用 |
| cancellation_token | 由 turn token 派生的 child token，用于取消/终止这次工具 future | 自动回滚已经发生的副作用 |
| tracker | 共享的 turn diff 跟踪器，用于记录执行过程中的差异 | 工具结果本身 |
| call_id | 把模型调用、执行轨迹、Tool Result 和历史记录关联起来 | exactly-once 的事务 ID |
| tool_name | 选择哪个注册的执行器 | 已经通过授权 |
| payload | Function、Tool Search 或 Custom 调用的具体输入 | 参数已经被每个执行器成功解析 |
| source | 标记 Direct、DirectPlaintextMessage 或 CodeMode 等调用来源 | 所有来源都具有相同的审计语义 |

ToolPayload 中的 Function 仍然携带 arguments 字符串。参数 JSON 如何解析、是否满足工具 schema、是否需要重写，发生在后续的 registry/handler/hook 路径；构造 ToolInvocation 本身不是完整的参数验证。

source 还有一个重要细节：CodeMode 里有 cell_id 和 runtime_tool_call_id。runtime_tool_call_id 不是 Codex 普通工具调用的 call_id，而是桥接层自己的运行时标识。因此，看到两个“id”不能直接把它们当成同一个幂等键。

## 5. 为什么必须携带 StepContext

当前 StepContext 的注释把它定义为一次模型采样请求的 request-scoped state。它包含或绑定：

- 本次 step 的 turn；
- 本次请求的 settings 和 token budget；
- TurnEnvironmentSnapshot；
- selected capability roots；
- executor capability discovery；
- MCP binding；
- ToolRouter；
- 已加载的 AGENTS.md 内容。

因此，工具不能只读取“现在全局是什么配置”，而要读取“产生这个工具调用的那次 step 当时向模型声明了什么、允许了什么、绑定了什么环境”。

这点尤其重要于：

1. 同一轮 turn 里可能有多次模型采样；
2. 工具列表、设置或能力发现结果可能随 step 改变；
3. 工具 future 可能排队后才真正执行；
4. 执行时必须避免误用后来更新的环境或工具计划。

注意，ToolInvocation 没有直接名为 environment_id 或 cwd 的字段。环境信息通过 step_context.environments 和其中的 TurnEnvironmentSnapshot 传递；TurnEnvironment 的调试字段包括 environment_id、cwd、workspace_roots、shell、executor platform 等。这个设计把“调用上下文”与“具体环境字段”分层保存。

## 6. 取消不是回滚：三个时间窗口

ToolCallRuntime 外层用 select 同时等待 dispatch future 和 cancellation token。当前源码至少要区分以下窗口：

| 取消发生的时间 | 源码行为 | 工程结论 |
| --- | --- | --- |
| 尚未通过 admission，handler 还没开始 | 任务可以被中止；测试观察到 execution_started=false，并返回 aborted response | 这次 handler 可能确实没有开始 |
| dispatch 已完成，或 terminal outcome 已经到达 | 运行时会继续等待/采用已有结果 | 收到取消信号，不等于结果一定被抹掉 |
| handler 可能已产生外部副作用，但结果、history 或 rollout 尚未全部完成 | future、结果封装和记录过程是不同阶段 | 不能因为用户看到“取消”就假定副作用不存在 |

源码中还会区分 dispatch duration、handler duration 和 total duration；非 fatal 的工具失败会被包装成面向模型的失败 Tool Result，fatal error 才会继续向上返回。

因此，“取消成功”至少要问清楚它指的是哪一层：

- 停止等待；
- 中止尚未开始或仍可中止的 runtime task；
- handler 收到取消信号；
- Tool Result 被标记为 aborted；
- 外部副作用被撤销。

当前调用链只能直接支持前几层，不能从 cancellation_token 自动推出最后一层。文件写入、网络请求、进程启动等副作用是否可撤销，要看具体 handler 和外部系统。

## 7. call_id 为什么不是 exactly-once

当前流程中，模型的 FunctionCall item 会在 dispatch 前先记录；工具 future 完成后，drain_in_flight 再记录 Tool Result。record_conversation_items 随后会更新 session history，并调用 live_thread.append_items 持久化 rollout。

这些动作有先后关系，但源码没有把“外部副作用、history 更新、rollout append”组成一个跨系统原子事务。因此可能出现不同的中间状态，例如：

~~~text
外部写入完成
  → handler 尚未返回
  → future 被取消或进程崩溃
  → Tool Result 尚未记录
~~~

或者：

~~~text
Tool Result 已经生成
  → history 已更新
  → rollout append 失败
~~~

call_id 很适合做相关性追踪和日志检索，但它本身不保证：

- 网络重试不会再次执行；
- 进程崩溃后重放不会重复写入；
- 外部服务知道这个 call_id；
- history 与外部副作用会一起提交。

要实现 at-least-once 下的安全恢复，必须由具体工具设计幂等键、去重记录、临时文件加原子替换、版本条件写入或可查询的操作状态。这个问题属于工具协议和副作用设计，不是 ToolInvocation 自动提供的能力。

## 8. 为什么这里使用 Arc、Clone 和 Future

ToolCallRuntime 会保存并克隆 Arc<Session>、Arc<StepContext>、tracker 等共享状态，因为工具调用可能等待 readiness、排队、并行执行，或者在稍后才进入 handler。Arc 解决的是异步任务之间的所有权和生命周期问题，不代表这些任务互相隔离。

ToolInvocation 自身实现 Clone；registry 在执行 hook、构造 handler 输入或记录结果时可以保留同一调用上下文的副本。副本仍然带着同一个 call_id 和同一个取消 token，因此 Clone 也不是重新生成一次调用。

ToolExecutor 的接口返回：

~~~rust
Pin<Box<
    dyn Future<Output = Result<Box<dyn ToolOutput>, FunctionCallError>>
        + Send
        + 'a
>>
~~~

这里表达了两个事实：

1. handler 是异步的，结果要等 future 完成后才有；
2. future 借用执行器本身，并要求 invocation 在所需生命周期内有效。

这解释了为什么“构造了 ToolInvocation”与“已经得到 Tool Result”之间存在一段可取消、可排队、可失败的时间窗口。

## 9. 一个未执行的最小模型

下面只是帮助建立时序的伪代码，本课没有运行它：

~~~text
on_model_call(call, step_context, turn_token):
    record_model_call_before_execution()
    token = turn_token.child_token()
    future = tool_runtime.handle_tool_call(call, token)
    result = await future
    record_tool_result_to_history_and_rollout(result)
~~~

阅读真实源码时，要把它展开成：

1. output item 先进入 history/rollout 记录路径；
2. future 经过 runtime gate 和并行锁；
3. router 注入 ToolInvocation；
4. registry 执行查找、payload kind 检查和 hooks；
5. handler 返回 ToolOutput 或面向模型的错误；
6. drain_in_flight 再写入结果。

## 10. 本课练习：比“工具没注册/参数错/授权拦截”更难

请先自己回答，回答时把“源码事实”“根据源码的推断”“你提出的设计方案”分开标记。

### 题 1：调用链字段追踪

给定一个 Direct 来源的 FunctionCall：

~~~text
call_id = C7
name = exec_command
arguments = {"cmd":"write file A"}
~~~

请逐步填写它经过以下节点时的对象和新增字段：

1. ResponseItem；
2. ToolCall；
3. handle_output_item_done 创建的 child cancellation token 和 in-flight future；
4. ToolCallRuntime 通过 gate/并行锁之后；
5. ToolInvocation；
6. ToolRegistry 的查找与 hook 之后；
7. ToolExecutorFuture 完成并进入 drain_in_flight 之后。

特别回答：哪一个对象携带“产生该调用的 step 的环境快照”？哪一个字段只能做关联，不能做幂等保证？为什么 Router 的两次出现不是重复执行？

### 题 2：取消窗口判定

对下面三个时间点分别判断：handler 是否一定启动、外部副作用是否一定不存在、是否会有 Tool Result、是否可以安全重试。不能只回答“会被取消”。

1. 尚未通过 admission gate 时取消；
2. handler 已启动但尚未返回时取消；
3. terminal outcome 已经到达、正在做 history/rollout 记录时取消。

请指出你的判断分别对应 ToolCallRuntime 的哪条分支，以及哪些结论只是“不能由当前源码推出”。

### 题 3：为 write_file 设计 at-least-once 恢复

假设工具要把内容写入路径 A，进程可能在以下任一窗口崩溃：

1. 临时文件已写完但还没有 rename；
2. rename 已完成但 handler 还没返回；
3. handler 已返回但 Tool Result 尚未写入 rollout；
4. rollout 已记录但客户端超时并要求重试。

请设计一个恢复协议，至少说明：

- 幂等键由什么组成，为什么不能只用 call_id；
- 如何记录 operation state；
- 如何避免半文件和错误版本覆盖；
- 重试时如何区分“未执行”“执行中”“已完成但结果丢失”；
- 如果副作用发生在远端，Codex 本地应该查询什么。

本课的通过标准不是背出字段，而是能说明每个状态转换、失败窗口和可重试边界。

## 11. 当前学习边界

已核验：

- ToolCall 是模型输出的内部表示，ToolInvocation 是注入 session/step/取消/追踪上下文后的运行时输入；
- 当前源码从 output item、ToolCallRuntime、ToolRouter、ToolRegistry 到 ToolExecutorFuture 的主要调用链；
- StepContext 为什么必须绑定到产生该调用的 step；
- cancellation_token 能影响 runtime 生命周期，但不自动撤销外部副作用；
- call_id 能做相关性追踪，但不等于 exactly-once。

尚未宣称掌握：

- 具体执行器在不同授权、隔离和环境下的实际行为；
- 每种 handler 的取消响应和外部副作用语义；
- write_file 或远端 API 的真正幂等恢复；
- 上面三道题的解答质量。

请完成练习后再进入下一节：把 ToolInvocation 的静态上下文，落实到执行环境、授权决策与失败恢复的完整状态机。
