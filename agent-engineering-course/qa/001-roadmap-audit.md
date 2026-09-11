# 问答 001：路线审核

日期：2026-09-11。保留用户问题与助手正式答复正文；不收录工具调用和执行过程消息。原答复中的路线图链接已更新到本文件夹，源码链接集中到文末索引；正文内容保留。

## 用户

附件：[完整课程要求](../sources/001-original-course-request.txt)

按上述路线开始，你要审核上述路线，防止其中的不实信息/幻觉

## 助手

**这条路线可以作为主线，但必须把“计划研究的问题”与“Codex 已实现的功能”分开。**我已完成首轮源码核验，并保存了可持续更新的[课程路线、架构地图与进度表](../CODEX_AGENT_LEARNING.md)。

本次依据本地提交 `02a8f038b87ad34d4a1dc5058eda26972ed7aa6c`，做了静态源码阅读和官方资料核验；没有运行 Agent、编译或测试。

**原路线需要修正的地方**

| 原路线容易引出的误解 | 核验结论／课程调整 |
|---|---|
| 可以沿用旧教程定位主循环 | 当前入口是 `session/turn.rs` 的 `run_turn`，旧版常见的 `core/src/codex.rs` 已不存在。 |
| Codex 同时提供 Responses 和 Chat Completions 两套协议 | 当前 `WireApi` 只有 Responses；解析 `chat` 会报错。Chat Completions 留作横向比较。 |
| provider abstraction 意味着模型随意替换 | 需要逐项核验协议、工具能力和参数。传输 fallback、压缩模型 fallback 也必须分开讲。 |
| Observe → Think → Act 就是源码结构 | 它是教学概括，不能据此虚构模块，也不能将其当成模型内部思维过程。 |
| 一个用户 Turn 就是一次模型调用 | 当前客户端明确支持一个 Turn 内发送多次 Responses 请求。 |
| 有文件系统模块，就一定有对应的模型工具 | 工具定义、曝光、处理器、执行后端是不同层。Git、搜索、网页访问也不能统一假设为本地独立工具。 |
| Codex 默认拥有符号索引、依赖图、向量检索 | 这些先列为待调查方案，不能从课程主题推导为已有实现。 |
| 完成课程即可胜任所有 Agent 岗位 | 必须按作品、测试、评测、部署及故障恢复验收；L0–L5 只是课程内部量表。 |

此外，**异步基础、错误处理、测试和安全边界需要提前**。不能写完带 shell 的 Agent，才开始讨论权限；也不能等全部模块学完，才开始做评测。

**当前源码的初步架构地图**

下面是抽象层级图，省略了中间派发、鉴权和扩展分支：

```text
用户输入
  ↓
CLI / TUI / 客户端
  ↓
App-server 接口与会话层
  ↓
Session → Turn 执行
  ↓
run_turn
  ├─ 历史与上下文 → 模型请求
  ├─ 模型流事件 → 工具调用解析
  ├─ ToolCallRuntime → ToolRouter → ToolRegistry
  │                                  ↓
  │                            具体工具处理器
  │                                  ↓
  │                       执行后端 / MCP 等服务
  ├─ 工具结果 → 对话历史 → 下一次模型请求
  └─ 后续工作与结束条件判断
  ↓
事件／通知 → 客户端
```

几个重要的阅读锚点：

- **入口**：`cli/main.rs`；非交互路径在 `exec::run_main`。
- **主循环**：`run_turn`。
- **工具调用进入执行**：`handle_output_item_done`。
- **结果回填**：`drain_in_flight` 收集工具结果，调用 `record_annotated_conversation_items`。
- **持久化**：`RolloutRecorder`；`state` 提供 SQLite 元数据层。

这里已经可以看出：**模型窗口、运行态和持久化记录需要分别理解。**

**调整后的学习顺序**

完整文件中，每阶段都写了模块、岗位能力和验收标准。主线如下：

| 阶段 | 学习重点 | 工程产出 |
|---|---|---|
| P0 | 全局地图、生命周期与边界 | 能解释一次请求的完整流程 |
| P1 | Rust 阅读、协议、异步基础 | 假模型驱动的事件流 |
| P2 | Agent Loop、工具结果、终止条件 | Minimal Agent |
| P3 | 模型接口、流式、请求与重试 | 模型适配器及断流测试 |
| P4 | Tools、MCP、文件读取与 Patch | Mini Coding Agent |
| P5 | Context、压缩、仓库指令、按需检索 | Context Manager |
| P6 | Shell、PTY、取消、基础隔离 | Sandbox Agent |
| P7 | Session、持久化、恢复、幂等 | Persistent Agent |
| P8 | 并发、背压、资源回收、故障处理 | Runtime 故障注入 |
| P9 | 注入、权限、网络与秘密边界 | 威胁模型及边界测试 |
| P10 | 轨迹、指标、评测与回归 | 可复现的评测报告 |
| P11 | Workflow、多 Agent、协调成本 | 同任务集对照实验 |
| P12 | 部署、系统设计与求职表达 | Mini Codex 综合作品 |

横向比较和面试练习贯穿各阶段；不会最后再一次性补齐。

**岗位能力不能混成一个无限清单**

本轮读取了三份官方岗位样本：

- **应用与效果方向**：强调 Python、LLM 产品和模型评测等。[OpenAI Codex Core Agent](https://openai.com/careers/applied-ai-engineer-codex-core-agent-san-francisco/)
- **Runtime 方向**：强调类型、异步、并发、状态、恢复、幂等和验证能力。[Anthropic Environments Infrastructure](https://job-boards.greenhouse.io/anthropic/jobs/5367436008)
- **Infrastructure 方向**：进一步要求容器／虚拟化、分布式性能、API 与基础设施管理。[OpenAI Agent Infrastructure](https://openai.com/careers/software-engineer-agent-infrastructure-san-francisco/)

因此课程采用“共同基础＋方向选修”。这三份样本偏海外资深岗位，不能代表全部市场，也不能据此判断你当前的求职水平。

**横向比较选五个，按问题阅读**

| 项目 | 主要比较内容 |
|---|---|
| [OpenAI Agents SDK](https://github.com/openai/openai-agents-python) | Runner、工具契约、handoff、session、tracing |
| [LangGraph](https://docs.langchain.com/oss/python/langgraph/overview) | 图执行、状态、检查点、持久执行与人工介入 |
| [PydanticAI](https://pydantic.dev/docs/ai/overview/) | 类型、依赖注入、结构化输出和参数验证 |
| [OpenHands Software Agent SDK](https://github.com/OpenHands/software-agent-sdk) | Coding Agent SDK 与执行环境分层 |
| [Gemini CLI](https://github.com/google-gemini/gemini-cli) | 同类终端 Agent 的工具、上下文与权限设计 |

本轮确认了选择依据，还没有完成这些项目的逐函数审计。正式比较时再固定版本，未知项不会凭印象填满。

**我们的正式第一课**

> **一次用户输入如何变成多次模型请求——分清 Thread、Turn、Response 与 Tool Call。**

从这里开始，是因为后续的上下文、工具回填、取消、重试和恢复，都依赖生命周期边界。第一课会采用 `codex exec` 路径，补齐入口到 `run_turn` 的逐跳调用链，追踪一个 `call_id` 如何连起工具请求和结果。

先留一道自拟思考题：**如果工具已经修改了文件，但 Agent 在保存工具结果之前崩溃，恢复后应该重新执行工具吗？你需要哪些记录才能决定？**

我会依据你的分析和实际练习判断进度，不会因为讲完一课就自动宣布“已经掌握”。

### 原答复源码引用索引

归档保留正文；原答复中的源码链接集中于此，避免长路径干扰阅读：

- [run_turn](../../codex-rs/core/src/session/turn.rs)：163；结果回填：2356。
- [WireApi](../../codex-rs/model-provider-info/src/lib.rs)：68。
- [ModelClient 说明](../../codex-rs/core/src/client.rs)：1。
- [CLI main](../../codex-rs/cli/src/main.rs)：1126。
- [exec run_main](../../codex-rs/exec/src/lib.rs)：259。
- [handle_output_item_done](../../codex-rs/core/src/stream_events_utils.rs)：302。
- [RolloutRecorder](../../codex-rs/rollout/src/recorder.rs)：86。
- [state](../../codex-rs/state/src/lib.rs)：1。
