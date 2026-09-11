# Codex 源码与 Agent 工程课程

核验日期：2026-09-11。源码基线：`02a8f038b87ad34d4a1dc5058eda26972ed7aa6c`。
首次检查工作区干净。本轮只做静态源码核验与外部资料阅读，未编译、运行 Agent 或测试。此文件是课程记录，不是 Codex 产品文档。

## 证据规则

- 源码事实必须绑定本地 commit、文件和符号；更新仓库后重新定位。
- 区分：已阅读实现、已执行验证、根据结构推断、通用设计经验、待调查。
- “为什么这样设计”若没有注释、设计文档或 PR 证据，标为工程分析，不冒充作者动机。
- 搜索不到不等于不存在；目录存在不等于功能已启用或能成功运行。
- 官方文档确认产品行为；源码确认当前 checkout 实现；不能据此推出本机桌面客户端与 checkout 完全一致。
- 第三方比较先核验官方资料，正式逐函数比较时再固定各自 commit/version。
- 自拟练习明确标为自拟题。只有找到具体面经来源才能称真实面试题。
- 当前不对学习者定级；通过讲解、实现、故障处理与测试验收后更新。

## 原路线审核

1. 原文主要是研究题目，不能把所有题目读作 Codex 已实现的功能清单。
2. 当前 Agent Loop 在 `codex-rs/core/src/session/turn.rs`，不是旧版常见的 `core/src/codex.rs`。
3. `model-provider-info/src/lib.rs` 的 `WireApi` 只有 `Responses`；反序列化 `chat` 明确报错。Chat Completions 放在跨项目比较，不作为当前 Codex 的并列实现。
4. 模型抽象不代表任意模型或参数都可互换。temperature、能力判定和 fallback 分别查调用条件；`core/src/client.rs` 的传输 fallback 与 `compact_model_fallback.rs` 的压缩模型 fallback 不能混称。
5. Observe/Think/Act 是教学概括，不代表代码存在三个同名模块，也不意味着可读取模型内部思维过程。
6. Thread、Session、Turn、一次模型请求、一次工具调用分别追踪。一个 Turn 可以发出多个 Responses 请求。
7. ToolSpec、工具处理器、执行后端、MCP 服务及模型托管工具分层讨论。搜索和 Git 操作不预设都存在独立模型工具；是否曝光取决于配置与工具构建。
8. Repository Understanding 中的符号索引、依赖图、向量检索是待比较方案，尚未核验为本版本默认机制。
9. 模型窗口、持久化历史、长期记忆分开。压缩窗口不等于删掉全部磁盘历史；恢复也不自动保证副作用 exactly-once。
10. 权限策略、审批、OS 隔离分别分析。命令黑名单或路径前缀检查不能作为“安全沙盒已完成”的验收依据。
11. 不将模型训练、商业桌面 UI 全部实现、云端调度平台视为此仓库已覆盖内容。
12. Async 基础、错误模型、测试和评测提前；框架比较与面试贯穿课程；Mini Codex 从早期逐步增长。
13. L0–L5 是课程内部量表，不是行业认证；读完源码不能保证胜任所有岗位。

## Architecture Map：首轮已定位入口

下列均相对仓库根目录，是导航入口而非完整覆盖声明。

| 层 | 源码入口与符号 | 本轮确认范围 |
|---|---|---|
| CLI | `codex-rs/cli/src/main.rs`：`main` | 分发到 exec、TUI、app-server 等入口 |
| 非交互入口 | `codex-rs/exec/src/lib.rs`：`run_main`、`InProcessAppServerClient::start` | exec 使用进程内 app-server client |
| 客户端协议 | `codex-rs/app-server-protocol/src/protocol/v2.rs` | v2 API 阅读入口；具体请求下课追踪 |
| Session | `codex-rs/core/src/session/session.rs`：`Session` | 会话运行态入口 |
| 输入循环 | `codex-rs/core/src/session/handlers.rs`：`submission_loop` | 提交消息处理入口 |
| Turn | `codex-rs/core/src/session/turn_context.rs`：`TurnContext` | Turn 上下文入口 |
| Agent Loop | `codex-rs/core/src/session/turn.rs`：`run_turn` | 构造输入、采样、判断后续工作 |
| 采样与流 | 同上：`run_sampling_request`、`try_run_sampling_request` | 调用 client_session.stream，处理流事件 |
| 模型客户端 | `codex-rs/core/src/client.rs`：`ModelClient`、`ModelClientSession` | 会话级客户端与 Turn 级请求状态 |
| Provider 协议 | `codex-rs/model-provider-info/src/lib.rs`：`WireApi` | 当前仅 Responses |
| 上下文 | `codex-rs/core/src/context_manager/history.rs`；`core/src/compact.rs` | 历史与窗口压缩入口 |
| 仓库指令 | `codex-rs/core/src/agents_md.rs`、`agents_md_manager.rs` | AGENTS.md 处理入口 |
| 工具契约 | `codex-rs/tools/src/tool_executor.rs`：`ToolExecutor`、`ToolExposure` | 执行契约和不同曝光方式 |
| 工具调用解析 | `codex-rs/core/src/stream_events_utils.rs`：`handle_output_item_done` | 解析工具调用并排入执行，标记 follow-up |
| 工具路由 | `codex-rs/core/src/tools/router.rs`：`ToolRouter`；`registry.rs`：`ToolRegistry` | 路由构造 invocation，再交给 registry |
| 工具并发 | `codex-rs/core/src/tools/parallel.rs`：`ToolCallRuntime` | 工具任务、取消及错误结果入口 |
| Shell | `codex-rs/core/src/tools/handlers/unified_exec/exec_command.rs`：`ExecCommandHandler` | exec_command 处理器；后续追踪 runtimes/unified_exec.rs |
| 执行后端 | `codex-rs/exec-server/src/local_process.rs`、`remote_process.rs` | 本地与远程进程实现阅读入口 |
| 文件系统 | `codex-rs/file-system/src/lib.rs`：`ExecutorFileSystem` | 文件系统后端契约；不等同模型工具列表 |
| 修改文件 | `codex-rs/core/src/apply_patch.rs`、`codex-rs/apply-patch` | Patch 集成与实现阅读入口 |
| Sandbox | `codex-rs/sandboxing/src/manager.rs`：`SandboxManager` | 平台策略转换入口；审批另查 tools/orchestrator.rs |
| 历史格式 | `codex-rs/history/src/lib.rs`：`RolloutItem`、`InitialHistory` | 持久历史数据类型 |
| 持久化 | `codex-rs/rollout/src/recorder.rs`：`RolloutRecorder`；`state/src/lib.rs` | Rollout 记录器与 SQLite 元数据层 |
| 内部协议 | `codex-rs/protocol/src/protocol.rs`：`Submission` 等 | 与 app-server 对外协议分开阅读 |

执行链的初步地图（省略配置、鉴权、扩展与多数错误分支）：

```text
CLI main
  -> exec::run_main -> InProcessAppServerClient
  -> app-server / core 会话层（中间派发细节待第一课逐跳核验）
  -> submission_loop / Turn 执行
  -> run_turn
     -> clone_history().for_prompt(...)
     -> run_sampling_request -> try_run_sampling_request
     -> ModelClientSession::stream -> 模型服务
     -> 流事件 -> handle_output_item_done
        -> ToolRouter::build_tool_call
        -> ToolCallRuntime -> ToolRouter -> ToolRegistry -> 具体处理器
        -> 本地/远程执行后端或扩展服务（按工具分支）
     -> 工具结果返回模型输入链路
     -> follow-up 判断 -> 下一次请求或 Turn 结束
  -> 事件/通知 -> 客户端
```

`run_turn` 已读到 history 构造、采样和 follow-up 分支；工具结果的持久化及回填细节作为第一课重点继续展开。结束不能简化为“出现文本就停止”：已看到 pending input、end_turn 与 Stop hooks 相关分支。

## Agent Concept Map

```text
Agent 工程
├── 决策：模型交互、工具选择、终止条件
├── 状态：Turn、运行态、模型窗口、持久历史、长期记忆
├── 行动：工具契约、路由、执行后端、外部服务
├── 控制：权限、审批、取消、并发、预算、重试
├── 证据：轨迹、日志、测试、评测、成本与延迟
└── 扩展：工作流、多 Agent、远程环境、部署
```

## 按依赖排列的课程路线

每阶段默认完成问题解释、源码链路、一个比较点、自拟三类面试题和验收。Python 用于早期 Mini 实现，Rust 用于源码阅读；TypeScript 在客户端和横向比较时补充。

| 阶段 | 学什么 / Codex 入口 | 学完能做什么 / 岗位能力 | Mini 项目与验收 |
|---|---|---|---|
| P0 全局地图 | CLI、exec、session、protocol；区分 UI/runtime/model | 解释各层责任、标出信任边界；架构沟通 | 手画一次请求链，辨别源码事实与假设 |
| P1 阅读基础 | Rust Result/enum/trait、Arc、Future、channel；协议与事件 | 沿跨 crate 调用读代码；类型与异步基础 | 用假模型驱动事件流，无需真实 API |
| P2 Agent Loop | session/turn.rs、stream_events_utils.rs | 区分 Turn/请求/工具；终止与错误语义 | Project 1 Minimal Agent：工具结果通过 call_id 对齐，覆盖循环上限与工具失败 |
| P3 Model Layer | client.rs、codex-api、model-provider-info | 请求构造、流式、结构化结果、传输重试；API 集成 | 接入模型适配器，模拟断流、超时；明确重试预算 |
| P4 Tools 与代码操作 | tools、router、registry、apply-patch、file-system、codex-mcp | schema、验证、执行、序列化；工具与 MCP 工程 | Project 2 Mini Coding Agent：read/write/search，错误参数、越界路径、patch 冲突有用例；shell 留到 P6 |
| P5 Context 与检索 | context_manager、compact、agents_md、history | 上下文预算、工具对配、摘要损失、按需检索；Context Engineering | Project 3 Context Manager：超预算后保留约束与调用对，比较任务效果；向量 RAG 为选修 |
| P6 Process 与基础隔离 | unified_exec、exec-server、sandboxing | PTY、输出、超时、进程树、取消、权限；系统编程 | Project 4 Sandbox Agent：在受限环境加入 shell，验证越界和网络策略；不用黑名单替代隔离 |
| P7 State 与恢复 | session、rollout、state、history；memories 后续定点阅读 | 区分窗口/持久历史/记忆；恢复、一致性、幂等 | Project 5 Persistent Agent：重启恢复，对已执行但未记账的副作用制定明确策略 |
| P8 Runtime 可靠性 | tools/parallel、session/turn、client、exec-server | 并发、取消竞态、背压、资源回收；Runtime 工程 | 故障注入：断流、工具崩溃、取消、重启；禁止无条件重试有副作用操作 |
| P9 安全深化 | sandboxing、execpolicy、network-proxy、审批与 MCP 边界 | 威胁建模、最小权限、注入与外泄；安全工程 | 只用合成秘密和隔离夹具验证拒绝/允许边界；列明不能保证的攻击面 |
| P10 可观测与评测 | otel、rollout-trace、analytics、core/suite | 轨迹、任务成功率、工具准确性、成本、延迟与回归 | Project 6 Runtime：固定任务集，报告成功率、延迟、token/成本、失败分类；不能仅凭 judge 分数 |
| P11 Workflow 与多 Agent | agent-roles、agent-graph-store、core/agent_communication.rs，具体调度待定位 | 分工、共享状态、监督与通信成本；编排能力 | 在同任务集比较单 Agent / workflow / 多 Agent，不预设多 Agent 更优 |
| P12 Production 与求职 | 综合已有 Mini Codex，云平台内容单独补充 | 应用：部署和用户反馈；Runtime：恢复和负载；Infra：隔离、调度与扩容 | 架构文档、可复现实验、评测报告、事故复盘、演示及自拟系统设计面试 |

测试从 P2 开始，安全边界在首次文件写入前引入，指标从首次真实模型请求开始记录。P9/P10 是深化而非首次接触。框架比较不拖到最后。实际修改本仓库时遵守 AGENTS.md 的格式化、定向测试和审批规则。

## 岗位证据与技能矩阵

2026-09-11 打开阅读的样本，仅三份海外偏资深岗位，不能代表中国招聘市场或所有初级岗位；不推断薪资、录用概率或技能出现频率。

- [OpenAI Applied AI Engineer, Codex Core Agent](https://openai.com/careers/applied-ai-engineer-codex-core-agent-san-francisco/)：Python、LLM 产品、评测/微调/提示设计。
- [OpenAI Software Engineer, Agent Infrastructure](https://openai.com/careers/software-engineer-agent-infrastructure-san-francisco/)：训练与部署基础设施、FastAPI/gRPC、Terraform、虚拟化与容器、分布式性能。
- [Anthropic Staff Software Engineer, Environments Infrastructure](https://job-boards.greenhouse.io/anthropic/jobs/5367436008)：Python 类型、async/并发、API 设计、状态恢复、幂等、一致性、验证习惯。

以下优先级为课程建议，不是 JD 逐字汇总。

| 能力域 | 共同基础 | 按岗位深入 | 验收证据 |
|---|---|---|---|
| Programming | Python、Git、shell、Linux 基础、HTTP、测试；Rust 阅读 | Rust 系统编程或 TypeScript 产品接口 | 能跨模块定位问题并提交小范围修复 |
| LLM | token/context、采样、reasoning 配置、结构化输出、tool calling | 应用：embedding/RAG；模型方向：Transformer、训练、微调 | 能解释失败来自模型、接口还是运行时 |
| Agent | Loop、状态、工具、MCP、HITL、终止与恢复 | Planning/reflection/workflow/multi-agent 按任务验证 | 能实现与测量，不能仅罗列模式名 |
| Infrastructure | API、存储、队列概念、容器、可观测性 | Infra：Kubernetes/调度、虚拟化、Terraform、分布式系统；Redis 按需 | 部署、压测、恢复与资源隔离证据 |
| Security | 权限、秘密、网络边界、注入、审批 | OS 隔离、MCP 信任、远程执行 | 威胁模型与正反边界测试 |
| Evaluation | 固定任务集、工具/轨迹评测、回归、人工审查 | benchmark、judge 校准、在线指标 | 可复现报告，说明样本和统计局限 |

## 横向比较：五个项目

本轮依据官方介绍选择研究对象，尚未完成五个项目核心源码审计；正式比较时固定版本，并对未知项保留空白。

| 项目 | 为什么选 | 主要比较问题 |
|---|---|---|
| [OpenAI Agents SDK](https://github.com/openai/openai-agents-python) | 通用 Agent SDK；官方列出 tools、handoffs、sessions、tracing | Runner 与 Codex Turn、工具契约、交接与跟踪 |
| [LangGraph](https://docs.langchain.com/oss/python/langgraph/overview) | 状态化编排 runtime，强调持久执行与 HITL | 图执行/检查点与命令式循环；恢复意味着什么 |
| [PydanticAI](https://pydantic.dev/docs/ai/overview/) | 类型化依赖和结构化输出验证 | 类型边界、依赖注入、工具参数验证及测试 |
| [OpenHands Software Agent SDK](https://github.com/OpenHands/software-agent-sdk) | 明确使用 SDK 仓库作为 agent 源码对象 | Coding Agent 的运行环境与 SDK 分层；具体实现下课核验 |
| [Gemini CLI](https://github.com/google-gemini/gemini-cli) | 同属终端 Coding Agent，官方列出文件/shell/MCP 能力 | 工具权限、上下文文件、终端交互与执行 |

注意：本轮打开的 OpenHands/OpenHands 主仓 README 已是 Agent Canvas 入口；不能沿用旧文章路径定位 Python runtime。Agents SDK 当前 README 也含 Sandbox agents，不能套用“SDK 完全没有 sandbox”的旧比较。

## Agent Engineer Progress

2026-09-11 第二课更新：学习者对第一课简单场景回答“1个turn，请求了2次模型，tool result”，三项正确。验收范围仅为该场景中 Turn、模型请求次数与工具结果的区分；不据此认定恢复、幂等或源码阅读已经掌握。第二课进入模型与 Runtime 分工、ToolExecutor 契约及工具路由/参数解析，作为 P0 到 P1 的桥接，不提前展开完整 P4 工具系统。新课文与正式答复要求逐字相同，已有课文与问答不回改。

2026-09-11 第二课练习验收：三题通过。已能判断未注册执行器不会产生文件读取、格式错误首先属于参数解析层、合法参数仍必须经过授权与隔离检查。第 2 题的精度要求补充为：要区分非法 JSON 的语法解析失败、合法 JSON 缺少必填字段的结构/语义校验失败，以及执行阶段失败。下一轮提高难度，改用完整调用链、部分成功和副作用重试场景。

2026-09-11 更新：课程已迁入 `agent-engineering-course/`；问答保存在 `qa/`，课文保存在 `lessons/`。第一课已提供并通过简单轨迹题验收。学习者明确表示不理解“保存工具结果”，并提出恢复修改前状态后重做的思路；第一课据此补充工具结果、文件副作用、历史记录与恢复不确定性的区分。第一课补齐了 exec 的 TurnStart → turn_processor → start_or_steer_turn → Op::TurnInput → RegularTask 调用链，以及工具结果 → 内存 history → rollout 持久化入口。上方首轮地图中的“待核验”保留为初次审阅状态，更新证据见第一课。

- 状态：P0 路线和入口地图已建立；第一课基础概念与第二课工具契约练习已通过，下一步进入 `ToolInvocation`、取消信号和执行环境。
- 已掌握：在已给定的简单轨迹中区分 Turn、模型请求、Tool Call/Tool Result；能区分工具 Spec、Executor、Registry、参数解析与授权/隔离的责任。
- 正在学习：把上述边界应用到真实调用链、部分成功、取消、持久化失败和重试决策；第 2 题的 JSON 语法/结构/语义分类仍需强化。
- 尚待验收：P1–P12 全部能力，不据此断言学习者从未学过。
- 求职短板：需要作品与练习结果才能判断，不根据提问方式定级。
- L0–L5 保留为内部量表；L2 至少有可运行简单 Agent，L3 能独立追踪源码，L4 能设计并验证 Runtime，L5 需要部署、监控、恢复和真实运维证据。

## 正式第一课

主题：一次用户输入如何变成多次模型请求——分清 Thread、Turn、Response 与 Tool Call。

为什么从这里开始：模型调用、上下文、工具结果、恢复和取消都依赖这些生命周期边界。

第一课采取 exec 入口减少 UI 细节：cli main -> exec run_main -> app-server 请求 -> session -> run_turn。逐跳补齐派发链，重点追踪一个工具 call_id 如何进入下一次模型输入。暂不逐行讲解大型函数。

练习（自拟，不是真实公司面试原题）：

1. 基础：模型输出 Tool Call 后，谁实际运行命令？
2. 源码：在哪个位置标记还需继续请求模型？需要核对哪些结束分支？
3. 系统设计：命令已经修改文件，但写入工具结果前进程崩溃，恢复时应不应该重新执行？你需要保存哪些证据？

验收：学习者能画出一条含 call_id、工具结果与下一次请求的轨迹，准确指出三个生命周期边界；再进入 P1/P2。
