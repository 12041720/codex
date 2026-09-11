三题都正确：**1 个 Turn、2 次模型请求，README 文本是 Tool Result。**这说明你已能区分第一课简单场景中的 Turn、模型请求和工具结果；恢复与幂等还未验收。

从本节起，**课文文件与对话框正式答复使用完全相同的正文**，旧课文和旧问答保持不变。本节保存于[第二课](C:/Users/fyfjz/Desktop/fyf/codex/agent-engineering-course/lessons/002-model-runtime-tool-contract.md)。

**第二课：模型提出调用，Runtime 如何把它变成执行？**

第一课学的是“如何循环”，这一节补上“谁负责什么”。这是进入 Rust 接口和异步代码前的基础。

本课仍依据源码 `02a8f038b87ad34d4a1dc5058eda26972ed7aa6c`，只做静态阅读；下面的调用是教学示例，没有实际执行。

**1. 模型看到工具说明，不等于模型拥有执行器**

假设用户说：“查看当前目录里的文件。”

Runtime 可以向模型提供一个工具的说明：

```text
名称：exec_command
用途：执行 shell 命令
参数：cmd、workdir 等
```

模型根据请求和说明，可能生成：

```json
{
  "type": "function_call",
  "name": "exec_command",
  "arguments": "{\"cmd\":\"Get-ChildItem\"}",
  "call_id": "call_001"
}
```

这个对象表达的是：**“请调用这个工具，传入这些参数。”**

到这里，目录还没有被读取。Runtime 必须把请求交给已有的工具实现，后者再通过执行后端操作环境。

所以，工具有两个需要对应起来的部分：

| 部分 | 面向谁 | 作用 |
|---|---|---|
| 工具说明／Spec | 模型 | 告诉模型用途和参数结构 |
| 工具实现／Executor | Runtime | 接收调用，执行逻辑，返回结果或错误 |

这是“模型会使用工具”背后的工程连接点。

**2. Codex 怎样把说明和实现放在一起？**

在 [ToolExecutor](C:/Users/fyfjz/Desktop/fyf/codex/codex-rs/tools/src/tool_executor.rs:106) 中，先关注两个真实的方法声明：

```rust
fn spec(&self) -> ToolSpec;

fn handle<'a>(&'a self, invocation: Invocation) -> ToolExecutorFuture<'a>
where
    Invocation: 'a;
```

先翻译成直白的话：

- `spec()`：给出这个工具的说明。
- `handle(...)`：处理一次实际调用，并返回一个可以等待结果的异步任务。

这里第一次遇到的 Rust **trait**，可以先理解为“一组实现者必须遵守的接口要求”。`ToolExecutor` 要求工具提供这些能力；具体工具各自实现它。

`&self` 表示借用当前工具对象。返回类型里的 `Future` 与等待执行结果有关。生命周期 `'a` 今天先认得即可，下一节再结合数据归属解释，避免同时塞入太多语法。

我们已经能读懂这行真实代码：

```rust
impl ToolExecutor<ToolInvocation> for ExecCommandHandler
```

意思是：

> **ExecCommandHandler 为 ToolInvocation 这种调用数据，实现 ToolExecutor 接口。**

在 [ExecCommandHandler](C:/Users/fyfjz/Desktop/fyf/codex/codex-rs/core/src/tools/handlers/unified_exec/exec_command.rs:112) 中，确实能看到 `tool_name()`、`spec()` 和 `handle()`；其中工具名称是 `exec_command`。

为什么把说明和执行入口关联起来？源码注释明确说明，这个契约让模型可见的 spec 与可执行 runtime 保持联系。进一步的工程收益是，减少“说明宣称能做 A，注册的执行器却负责 B”的错配机会；**这并不自动证明二者永远一致，仍需要检查和测试。**

**3. 从调用对象走到执行器，中间发生什么？**

先看精简链路：

```text
模型输出 FunctionCall
  ↓
ToolRouter：转换为内部 ToolCall
  ↓
ToolCallRuntime：安排执行与取消等控制
  ↓
ToolRouter：构造 ToolInvocation 并交给 Registry
  ↓
ToolRegistry：查找工具、检查调用、处理 hooks 等
  ↓
具体 Executor：解析参数、处理环境与执行逻辑
  ↓
Tool Result → 历史 → 下一次模型请求
```

注意 Router 在这里承担两处职责，不能简单把它理解为一个只出现一次的箭头。

我们只核对其中三个关口。

**关口 A：把模型协议转换成内部调用**

[ToolRouter::build_tool_call](C:/Users/fyfjz/Desktop/fyf/codex/codex-rs/core/src/tools/router.rs:244) 从 `FunctionCall` 取出名称、命名空间、参数和 `call_id`，构造内部 `ToolCall`。

这里的 `arguments` 仍然可以是字符串。**转换调用对象，不等于已经验证每个参数的业务含义。**

**关口 B：找得到这个工具吗？**

[ToolRegistry](C:/Users/fyfjz/Desktop/fyf/codex/codex-rs/core/src/tools/registry.rs:495) 中有真实的查找分支：

```rust
let tool = match self.tool(&tool_name) {
    Some(tool) => tool,
    None => {
        // 此处省略日志和遥测代码
        // 实际分支返回 RespondToModel 错误
    }
};
```

这是一段省略后的示意，不能直接编译。它表达的关键行为是：**模型不能仅凭输出一个工具名字，就创造出一个执行能力。**

比如模型输出 `teleport_file`，但 Registry 中没有这个工具，当前这条分支会返回可反馈给模型的错误。

该方法还检查 payload 类型，并处理执行前 hooks。这里只是初步定位，完整权限与审批逻辑留到后续课程。

**关口 C：参数能被执行器理解吗？**

[ExecCommandHandler::handle_call](C:/Users/fyfjz/Desktop/fyf/codex/codex-rs/core/src/tools/handlers/unified_exec/exec_command.rs:155) 提取参数，再解析为 Rust 类型。

使用的 [parse_arguments](C:/Users/fyfjz/Desktop/fyf/codex/codex-rs/core/src/tools/handlers/mod.rs:85) 核心是：

```rust
serde_json::from_str(arguments).map_err(|err| {
    FunctionCallError::RespondToModel(
        format!("failed to parse function arguments: {err}")
    )
})
```

解释：尝试把 JSON 字符串解析为目标类型；失败则生成能反馈给模型的错误。

但要区分：

> **能解析 ≠ 参数符合业务要求 ≠ 操作获得授权 ≠ 最终执行成功。**

例如，一条命令可以是合法字符串，但仍可能因为权限、路径不存在或进程启动失败而无法完成。

**4. 为什么不能只靠 Prompt？**

假设我们在工具说明中写：“只能操作工作区文件。”

这会向模型表达使用约束，但程序仍需要在实际执行路径落实相应限制。模型生成了符合 JSON 格式的调用，也不能替代权限判断。

从工程上，可以把问题分成四层：

| 层 | 要回答的问题 |
|---|---|
| 工具说明 | 模型是否知道怎样调用？ |
| 参数解析与检查 | 请求是否能被程序理解、是否满足要求？ |
| 授权与隔离 | 这次操作是否允许，执行环境怎样限制它？ |
| 实际执行 | OS 或远程服务是否完成了操作？ |

这些层是为了分析责任，不代表所有工具都必须经过完全相同的函数序列。

本课源码显示了说明、路由、查找和参数解析的具体位置。关于沙盒保证，我们今天不作超出已读代码的结论。

**5. 你自己写 Mini Agent 时怎么对应？**

概念上，你至少需要这两份信息：

```python
tool_definitions = [
    # 给模型看的名称、用途和参数说明
]

executors = {
    "read_file": read_file_handler,
    "exec_command": exec_command_handler,
}
```

模型输出调用后，程序按名称找到 executor，再解析、检查并执行参数。

这是未运行的教学示意，省略了错误处理、权限和并发等部分。它对应一个很实用的调试顺序：

> 模型有没有选对工具 → 工具有没有注册 → 参数能不能解析 → 操作是否允许 → 执行是否成功。

这个顺序对应 Agent 岗位中的工具集成、接口设计和故障定位能力。遇到“Agent 没完成任务”，你就能按层检查，而不只是笼统地说“模型不够聪明”。

**本节练习**

三题均为自拟题：

1. 给模型的工具说明里写了 `read_file`，但程序没有注册对应执行器。模型仍输出了这个调用：文件会被读取吗？为什么？
2. `exec_command` 已注册，但参数字符串是 `{"cmd":`。问题首先出在哪一层？
3. 参数是合法 JSON，命令也能解析，但它试图写入不允许访问的目录。能否因为“参数格式正确”就执行？

请先回答这三题。下一节我们会从这里进入 `ToolInvocation`：一次调用除了模型给出的参数，为什么还需要会话、取消信号和执行环境。
