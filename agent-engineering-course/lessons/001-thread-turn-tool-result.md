# 第一课：一次输入如何产生多次模型请求

日期：2026-09-11。源码基线：`02a8f038b87ad34d4a1dc5058eda26972ed7aa6c`。
证据范围：静态阅读源码；没有启动 Agent、修改示例文件或执行测试。示例全部为教学构造，不是本机执行日志。

## 先解释你的思考题

你的回答：“我不知道保存工具结果指的是什么，我认为应该先回复修改前的状态，再重新执行工具”。这里按“恢复修改前的状态”理解。

工具执行有两个不同的产物：

1. **环境中的实际变化**：例如 `config.json` 的 `timeout` 从 10 改成 20。
2. **执行后返回的信息**：例如“已修改 config.json”，或者命令的 stdout、stderr、退出码、错误。这叫工具结果（Tool Result）。它也可能包含图片或其他结构化内容。

“保存工具结果”指将第二类信息记录到 Agent 的历史／持久化链路，供后续模型调用和会话恢复使用。它不等于给原文件做备份，不等于保存完整文件快照，也不是面向用户的最终回答。

教学例子：

```text
工具请求：把 config.json 的 timeout 设置为 20
执行影响：文件确实变了
工具结果：已更新 config.json
最终答复：已把超时改为 20；后续模型可能先检查或测试才回复
```

这几件事发生在不同阶段。可能文件已经变了，进程却在结果写入历史前崩溃。

因此，恢复时“没有成功记录”不能直接推出“没有执行成功”。这叫执行状态存在不确定性。

你的回滚再执行方案在特定设计下可以成立，但需要：有可靠旧状态、能识别本次操作修改范围、不会覆盖其他人的新改动、没有遗漏外部副作用。普通 shell 命令不自动满足这些条件。

| 操作 | 恢复时可考虑的策略（通用工程设计，不是本课已证明的 Codex 实现） |
|---|---|
| 读取文件 | 可以重新读取，但文件可能已变化，结果不保证相同 |
| 将某字段设置为固定值 | 检查当前值及版本；在目标仍有效时考虑幂等重试 |
| 计数器加一／追加日志 | 重做可能造成重复，需要操作记录或去重机制 |
| 发邮件／远程提交 | 优先按操作标识查询、核对；无法确认时不要盲目重做 |

`call_id` 用来关联请求和结果；除非执行器额外实现去重，它本身不保证幂等或“恰好一次”。普通文件写入和历史日志写入也不能因为相邻就被当成一个原子事务。

今天先掌握一个判断：**先确认当前状态与操作语义，再决定重试、补记结果、回滚或请求人工处理。**

## 今天的目标

看懂五个概念之间的关系：Thread、Session、Turn、一次模型请求/Response、Tool Call。暂时不要求掌握 Rust 异步语法，也不要求记住所有文件名。

| 概念 | 本课解释 | 不要混淆 |
|---|---|---|
| Thread | 一条有身份和历史的对话，之后可以继续 | 不是 OS 线程 |
| Session | 为这条对话提供服务的运行时对象，持有状态、服务、事件通道等 | 不是一次 HTTP 请求 |
| Turn | Agent 围绕本次输入推进工作的一段执行生命周期；开始、运行、结束或中断 | 一个 Turn 可包含多次模型请求；新输入也可能转向当前 Turn |
| 模型请求 / Response | Runtime 向模型服务发出一次逻辑请求并处理相应输出；网络重试另论 | 一次 Response 可包含多个输出项，不只一段文字 |
| Tool Call | 模型提出的某次工具调用，包含名称、参数及关联标识 | 生成调用请求不等于执行已经成功 |

源码证据：

- [Session](../../codex-rs/core/src/session/session.rs) 包含 `thread_id`、`state`、`active_turn`、`input_queue`、`services` 等字段。
- [client.rs](../../codex-rs/core/src/client.rs) 的文件级说明明确：`ModelClient` 是会话级；`ModelClientSession` 每 Turn 创建，可在该 Turn 内发送一个或多个 Responses 请求。
- [turn_processor.rs](../../codex-rs/app-server/src/request_processors/turn_processor.rs) 通过 `start_or_steer_turn` 提交输入，因此不能将每条新消息机械等同一个全新 Turn。

## 用一条教学轨迹理解 Agent Loop

用户请求：“把 timeout 改为 20，并确认修改正确。”

```text
Thread A
└── Turn T1
    ├── 模型请求 R1
    │   输入：用户要求、指令、已有历史、可用工具定义
    │   输出：工具调用 C1——读取 config.json
    ├── 执行 C1 → 结果：timeout 当前为 10
    ├── 模型请求 R2
    │   输入增加：C1 调用及对应结果
    │   输出：工具调用 C2——修改 timeout
    ├── 执行 C2 → 结果：修改成功
    ├── 模型请求 R3
    │   输入增加：C2 调用及对应结果
    │   输出：工具调用 C3——重新读取以检查
    ├── 执行 C3 → 结果：timeout 当前为 20
    └── 模型请求 R4
        输出：给用户的最终答复
        Runtime 处理结束条件
```

这是便于学习的一种轨迹，不规定真实模型必须四次请求、必须先读后写或每次只能调用一个工具。模型服务负责产出调用意图；Runtime 负责路由、校验和执行，执行后端真正操作环境。

从这张图看，一个用户请求可以带来多次模型交互。后一次交互能利用前一次执行的结果，形成“观察—行动—再观察”的闭环。

## 工具请求和结果长什么样

下面采用普通 function 工具作教学示意，`read_file` 不是对当前会话已注册工具清单的断言；也省略了完整协议字段。

```json
{
  "type": "function_call",
  "name": "read_file",
  "arguments": "{\"path\":\"config.json\"}",
  "call_id": "call_001"
}
```

执行器返回与之对应的结果：

```json
{
  "type": "function_call_output",
  "call_id": "call_001",
  "output": "{\"timeout\":10}"
}
```

关键是两边的 `call_id`。如果同一轮读了两个文件，它帮助系统确定每个结果对应哪个调用。

在 [protocol/src/models.rs](../../codex-rs/protocol/src/models.rs) 中，`ResponseItem::FunctionCall` 包含 `name`、`arguments: String`、`call_id`；`FunctionCallOutput` 有对应关联字段及输出负载。当前内部输出类型的 `call_id` 可选，不能把此示例误读为所有内部事件都具有完全相同的字段约束。普通工具配对时仍应保留关联。

Codex 还有 Custom Tool 等变体，第一课先学一种，不能把所有工具输出都说成纯文本 JSON。

## 真实入口：先选非交互 exec 路径

下面的源码链省略了鉴权、配置、状态校验和事件回送，但关键派发已定位：

```text
cli/src/main.rs::main
  → exec/src/lib.rs::run_main
  → InProcessAppServerClient::start
  → ClientRequest::TurnStart
  → app-server/src/message_processor.rs 的 TurnStart 分支
  → request_processors/turn_processor.rs::turn_start / turn_start_inner
  → CodexThread::start_or_steer_turn
  → SessionIo::submit_turn_input → Op::TurnInput
  → session/handlers.rs::submission_loop
  → session/turn_input.rs::handle
  → 输入接纳和任务启动分支 → RegularTask
  → tasks/regular.rs::run → run_turn
```

定位索引（当前基线行号）：CLI 1126；exec 973、1137；message_processor 1620；turn_processor 521、644 附近；session/mod.rs 929；handlers 598；turn_input 202、328、465；regular 77。

注意两种循环的责任不同：`submission_loop` 接收用户输入、取消、设置等运行控制；`run_turn` 推进当前工作的模型/工具交互。它们不能合称一个无差别的 while 循环。

## 只读三处关键代码

### 1. 请求模型前，取出当前历史

[session/turn.rs](../../codex-rs/core/src/session/turn.rs) 约 510 行：

```rust
sess.clone_history()
    .await
    .for_prompt(&step_context.settings.model_info.input_modalities)
```

解释：从当前历史构造适合模型输入的内容。这是请求构造的一部分；完整请求还有指令和工具定义等，不能认为这三行就是整个 Prompt Builder。

### 2. 模型提出工具调用，Runtime 安排执行

[stream_events_utils.rs](../../codex-rs/core/src/stream_events_utils.rs) 的 `handle_output_item_done` 中，`ToolRouter::build_tool_call` 识别工具调用；调用项先记录，再把执行交给 `ToolCallRuntime`。关键状态是：

```rust
output.needs_follow_up = true;
output.tool_future = Some(tool_future);
```

解释：有了工具请求，后续通常还需要把结果送给模型继续处理。此时“模型的一次输出完成”和“用户任务完成”是不同事件。

### 3. 工具执行结束，把结果记入历史

[session/turn.rs](../../codex-rs/core/src/session/turn.rs) 的 `drain_in_flight` 约 2356 行，等待工具 future 并记录返回项：

```rust
sess.record_annotated_conversation_items(
    turn_context,
    &step_context.settings.model_info,
    vec![envelope],
)
.await;
```

[session/inject.rs](../../codex-rs/core/src/session/inject.rs) 与 [session/mod.rs](../../codex-rs/core/src/session/mod.rs) 约 3499–3567 行进一步连接到：

```text
record_annotated_conversation_items
  → record_conversation_items / record_prepared_conversation_items
  → state.history.record_annotated_items
  → RolloutItem::ResponseItem
  → persist_rollout_items
```

所以“保存工具结果”在这里有两个观察层面：更新内存历史，以及进入 rollout 持久化流程。下一次构造模型输入时即可利用历史里的工具结果。

严谨边界：本课没有据此证明每次返回都已获得断电安全保障。队列、写入、flush、崩溃一致性要到持久化课程继续核验；更不能由此推出文件修改与历史持久化是原子的。

## 什么时候结束

`run_turn` 约 565 行组合 `model_needs_follow_up` 和 `has_pending_input`。约 642 行进入无 follow-up 分支后还处理 Stop hooks。流处理约 2825 行还会考虑 `end_turn`。

今天掌握：Runtime 根据协议与运行状态决定继续还是结束。不能看到一句助手文字就认定 Turn 完成，也不能只看有没有 tool call。取消、错误和其他调度分支后续逐一讲。

## 为什么采用这类分层

以下是根据已见结构做的工程分析，不冒充作者的完整设计动机：

- 模型负责生成调用意图，执行器集中处理校验、权限和结果，可以让不同模型共享执行基础设施。
- 结果作为历史项进入后续请求，使模型能够针对真实输出继续行动；调用失败也能成为下一步决策依据。
- 用户输入循环与任务执行分离，为等待工具时接收取消等控制信号提供结构基础。
- 代价是状态协调更复杂：尤其是外部操作完成与历史记录完成之间的失败窗口。

横向比较只引入一个点：[LangGraph 官方概览](https://docs.langchain.com/oss/python/langgraph/overview) 强调状态化编排、持久执行和人工介入。我们后续会比较“显式图状态”与 Codex 此处“命令式异步控制流”如何表达继续、暂停和恢复。本课没有运行 LangGraph，也不据此宣称哪一种普遍更好。

## Mini 实现思路

以下为未运行的概念伪代码，不是 Codex 源码，也不是生产安全实现：

```python
history = [user_message]
for step in range(max_steps):
    response = await model.generate(history, tool_definitions)
    history.extend(response.items)
    if response.tool_calls:
        for call in response.tool_calls:
            result = await executor.execute(call)
            history.append(tool_result(call.call_id, result))
    elif response.is_final:
        return response.final_text
raise StepLimitExceeded()
```

先从中辨认三件事：模型输出进入历史、执行工具、工具结果进入历史。错误分类、持久化、取消、权限、并发和非 final 无工具输出的处理均尚未补齐；不能直接用于执行不可信命令。

## 岗位能力与练习

本课对应：协议数据流、状态生命周期、工具调用与恢复风险的工程解释能力。

面试练习均为自拟题：

1. 基础：Tool Call 和 Tool Result 分别由谁产生？
2. 源码：哪段代码决定工具结果进入下一次模型输入？
3. 系统设计：工具执行成功，但历史写入失败，如何避免重复副作用？第三题只要求识别风险，暂不要求设计完整方案。

本节小练习：用户问“这个项目用了什么语言”，模型调用读取 README 工具，工具返回文本，然后模型回答用户。请判断：

- 这是一还是两个 Turn？（假定期间没有中断、新输入或其他恢复行为。）
- 调用了几次模型？
- 读取 README 返回的文本是 Tool Call、Tool Result 还是最终答复？

进度：第一课材料已讲授；概念区分等待学习者作答验收。尚不能将“已提供课文”记为“已掌握”。
