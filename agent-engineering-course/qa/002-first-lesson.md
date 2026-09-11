# 问答 002：归档、思考题与第一课

日期：2026-09-11。

## 用户

1.把 [CODEX_AGENT_LEARNING.md](../CODEX_AGENT_LEARNING.md) 和以后的问答内容都放在一个文件夹里，持久化
2.思考题：我不知道保存工具结果指的是什么，我认为应该先回复修改前的状态，再重新执行工具
3.开始第一课

## 助手

已整理到统一的[课程文件夹](C:/Users/fyfjz/Desktop/fyf/codex/agent-engineering-course/README.md)：

```text
agent-engineering-course/
├── CODEX_AGENT_LEARNING.md   路线与进度
├── sources/                 原始课程要求
├── qa/                      用户问题、你的作答与我的正式答复
└── lessons/                 每课完整笔记
```

已保存前一轮问答、本轮问答和[第一课完整笔记](C:/Users/fyfjz/Desktop/fyf/codex/agent-engineering-course/lessons/001-thread-turn-tool-result.md)。后续课程问答也按这个结构保存。

**第一课：一次用户输入，为什么会产生多次模型请求？**

先解决你卡住的地方：**“工具结果”是执行后返回给 Agent 的信息。它与文件本身的修改是两回事。**

例如，你要求把 `timeout` 从 10 改为 20：

| 内容 | 例子 |
|---|---|
| 工具请求 | “把 config.json 的 timeout 设置为 20” |
| 实际影响 | 磁盘里的文件发生变化 |
| 工具结果 | “已更新 config.json”，或失败原因 |
| 最终答复 | Agent 检查后告诉你：“已修改并确认” |

“保存工具结果”，就是把执行返回的信息记入 Agent 的历史和持久化流程，让后续模型请求以及恢复过程能利用它。**这不等于备份修改前的文件。**

你说的“先恢复修改前状态，再重新执行”，在有可靠快照、能够准确撤销本次修改的情况下可以作为方案。但不能普遍采用：可能有人在文件上又做了新修改，也可能工具还发送了网络请求，这些都不能靠恢复一个文件撤销。

所以更合适的顺序是：

> **先确认操作是否已经生效，再根据操作性质决定重做、补记结果或回滚。**

例如“设置为 20”和“在原值上加 10”就不同。后者重复执行会产生不同结果。这个问题涉及恢复与幂等，后面再深入；现在先分清“实际变化”和“关于变化的记录”。

**1. 先认识五个概念**

| 概念 | 你可以怎样理解 |
|---|---|
| Thread | 一条可以持续追加内容的对话；不是操作系统线程 |
| Session | 运行这条对话的对象，持有状态、服务和事件通道 |
| Turn | Agent 推进本次工作的一段执行过程 |
| 模型请求／Response | Runtime 请求模型，模型返回一组输出 |
| Tool Call | 模型输出中的一次工具调用请求 |

核心关系是：**一个 Turn 可以包含多次模型请求，每次模型输出又可能包含工具调用。**

这不是根据经验猜测：[client.rs 的说明](C:/Users/fyfjz/Desktop/fyf/codex/codex-rs/core/src/client.rs:1)明确写到，`ModelClientSession` 在一个 Turn 内可以发送一个或多个 Responses 请求。

**2. 看一次具体过程**

下面是教学例子，不是实际运行日志：

```text
用户：把 timeout 改为 20，并检查是否正确。

Turn 开始
  │
  ├─ 第一次请求模型
  │    模型：需要先读取文件 → 发出工具调用 C1
  │    工具执行：读取文件
  │    工具结果：timeout 当前为 10
  │
  ├─ 第二次请求模型，带上刚才的调用和结果
  │    模型：修改文件 → 发出工具调用 C2
  │    工具执行：完成修改
  │    工具结果：修改成功
  │
  ├─ 第三次请求模型
  │    模型：重新读取检查 → 发出工具调用 C3
  │    工具结果：timeout 当前为 20
  │
  └─ 第四次请求模型
       模型：向用户报告结果
       Runtime 处理结束条件
```

这里是一个 Turn、四次模型请求、三次工具调用。真实执行可以采用其他步骤，不规定必须四次。

为什么要再次请求模型？因为模型第一次生成调用时，还没有这次执行的结果。**Runtime 执行工具，再把结果放进后续输入，模型才能据此继续处理。**

**3. 请求和结果怎么对应？**

用普通 function 工具作简化示例，工具名仅为教学构造：

```text
工具调用：
  name: read_file
  arguments: {"path": "config.json"}
  call_id: call_001

工具结果：
  call_id: call_001
  output: {"timeout": 10}
```

两边的 `call_id` 把“做什么”和“做完得到什么”对应起来。相关数据结构见 [FunctionCall / FunctionCallOutput](C:/Users/fyfjz/Desktop/fyf/codex/codex-rs/protocol/src/models.rs:1061)。

注意：**有 call_id 不代表自动防重复执行。**是否去重，还要看执行器有没有实现。

**4. 在 Codex 源码里，只抓三个动作**

第一，**从历史构造模型输入**。[run_turn](C:/Users/fyfjz/Desktop/fyf/codex/codex-rs/core/src/session/turn.rs:510) 中：

```rust
sess.clone_history()
    .await
    .for_prompt(&step_context.settings.model_info.input_modalities)
```

先不用纠结 Rust 语法。它表达的是“取出当前历史，转换为模型可用的输入”。这只是完整请求构造的一部分。

第二，**识别调用并安排执行**。[handle_output_item_done](C:/Users/fyfjz/Desktop/fyf/codex/codex-rs/core/src/stream_events_utils.rs:302) 解析模型的工具调用，交给工具运行层，并设置：

```rust
output.needs_follow_up = true;
output.tool_future = Some(tool_future);
```

含义是：工具已安排执行，后面还需要继续处理。一次模型输出完成，不代表整个 Turn 完成。

第三，**接收结果并记录历史**。[drain_in_flight](C:/Users/fyfjz/Desktop/fyf/codex/codex-rs/core/src/session/turn.rs:2356) 等待工具结果，再调用记录方法。进一步追到 [record_prepared_conversation_items](C:/Users/fyfjz/Desktop/fyf/codex/codex-rs/core/src/session/mod.rs:3523)，可以看到：

```text
工具返回项
  → 写入 state.history
  → 转成 RolloutItem
  → 进入持久化流程
```

下一次请求再从历史取输入，闭环就接上了。

这也解释了原思考题：**工具修改文件和 Runtime 保存结果，处在不同步骤。**仅看到这里，不能声称它们是一个“要么一起成功、要么一起失败”的原子事务。

**5. 本课先掌握到什么程度？**

今天先做到：能分清用户要求、模型调用、工具执行、工具结果和最终答复。源码入口的完整链路已放进课文，暂时不用背函数名。

本轮只做静态核验，没有运行上述示例；你的进度记录为“第一课学习中，等待概念练习验收”。

最后做一个比上一题更基础的小练习：

> 用户问：“这个项目用了什么语言？”
> 模型先调用工具读取 README，工具返回文本，然后模型根据文本回答用户。
> 假设没有中断或新输入：
>
> 1. 这是几个 Turn？
> 2. 请求了几次模型？
> 3. 读取 README 返回的文本，是 Tool Call、Tool Result，还是最终答复？
