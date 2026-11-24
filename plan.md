# Agent 控制面系列教程规划

## 定位

这是 Agent Runtime 基础之后的补充手册。读者已经理解工具循环、Context、状态、
权限、沙箱、重试、持久化和多 Agent 基础；本系列继续研究真实产品如何把这些能力
组织成一个可使用的控制面。

本系列不重新实现五个产品，也不把它们接入 `social-media`。五个 `sources/` 仓库是
只读的源码依据；`tutorials/` 只写架构设计、版本教程和小范围可运行参考实现。
参考实现学习行为和边界，不复制上游源代码、品牌或受许可限制的实现。

## 源码基线

规划基线固定为 2026-11-22 克隆到本目录的浅克隆提交：

| 模块 | 源码 | 分支 / 提交 | 主要真实入口 |
| --- | --- | --- | --- |
| `t3-code-control-plane` | [`sources/t3code`](sources/t3code) | `main` / `11224f63d2a3d444ca1fd1f67cd3ff21c1f5f8a1` | `apps/server/src/orchestration-v2/Orchestrator.ts`、`EventSink.ts`、`EffectWorker.ts`、`ProjectionStore.ts`、`packages/contracts/src/orchestrationV2.ts` |
| `raft-collaboration-space` | [`sources/raft-source`](sources/raft-source) | `main` / `5214e34e726ed715f455e78c96fa521791ef2de3` | `packages/server`、`packages/daemon`、`packages/computer`、`packages/raft-sdk`、`packages/shared` |
| `lody-acp-workspace` | [`sources/lody`](sources/lody) | `main` / `762dee0431ecae0649b9fab13a1f357b096d0ac` | `apps/cli/src/agent`、`apps/cli/src/session`、`apps/cli/src/orchestration`、`packages/shared/src/session-*`、`packages/e2ee-core` |
| `multica-issue-orchestration` | [`sources/multica`](sources/multica) | `main` / `4ceffd536603c67620c7460f6c412dcbf092299e` | `server/internal/service`、`server/internal/scheduler`、`server/internal/runtimeapps`、`packages/core` |
| `herdr-agent-runtime` | [`sources/herdr`](sources/herdr) | `master` / `e8d7d9705a7a2c89cef4dd42079379f05685af87` | `src/main.rs`、`src/server`、`src/client`、`src/pane`、`src/persist`、`src/remote` |

源码事实已经足以确认模块边界，但每个模块进入 V0 时仍要重新核对该提交中的
具体调用链和测试；不能把 README 的宣传语当成实现证据。

## 规划层次

五个产品各自是一条独立教程线。每条线内部再拆成多个有明确业务责任的模块，模块
拥有自己的 `V0 → Vn`。产品级版本只表示一组模块已经可以连成一条可观察的核心链路，
不是把整个产品压缩成几层功能。

```text
产品手册
  ├── 产品整体 V0：场景、模块、数据流和集成门
  ├── 模块 A：V0 → Vn
  ├── 模块 B：V0 → Vn
  ├── 模块 C：V0 → Vn
  └── 产品集成门：验证模块之间的真实交接
```

每个模块的 V0 必须同时交付四类内容：

1. 真实业务流程。谁触发操作，谁拥有状态，状态如何变化，失败后谁继续处理。
2. 源码对应。列出真实目录、文件、类/函数和测试入口，说明它们在流程中的责任。
3. 参考架构。给出学习实现中对应的 `model`、`state store`、`runtime`、`transport`
   或 `effect` 组件，以及它们之间的调用方向。
4. 契约和验证。说明输入输出、错误、并发、外部副作用和 V1 可以直接执行的检查。

V0 只规划和解释模块，不声称产品已经运行。V1–Vn 才逐步实现一个模块能力。每个
实现版本必须指出模块起点、真实源码符号、参考实现文件、可观察变化和失败检查。

## 模块地图和 V0 源码对应

下面的目录是当前规划的模块边界。它们来自已克隆提交中的实际路径。V0 文档要把
这些路径展开成完整调用链，后续版本再选择其中一条链实现。

### Herdr 模块

| 模块 | 真实源码入口 | V0 参考架构 | V0 要解释的状态流 |
| --- | --- | --- | --- |
| Server/Client/Socket | `src/main.rs`、`src/server/*`、`src/client/*`、`src/api/server.rs` | `Server` 持有 Listener、Client 持有连接和订阅、API 命令进入 App | 启动 → Socket 握手 → 命令/事件 → 客户端断开 |
| PTY/Pane/Agent Detection | `src/pane/state.rs`、`src/pane/terminal.rs`、`src/pane/agent_detection.rs`、`src/detect/*` | `PaneRuntime` 持有 PTY，`PaneState` 保存纯状态，检测器读取屏幕证据 | PTY 输出 → 终端快照 → Agent 状态 → 事件发布 |
| Workspace/Session Persistence | `src/app/session.rs`、`src/persist/snapshot.rs`、`src/persist/restore.rs`、`src/persist/writer.rs` | `AppState` → `SessionSnapshot` → 原子写入 → 恢复计划 | 工作区变化 → 延迟保存 → 快照 → 重启恢复 |
| Remote Endpoint | `src/remote/host.rs`、`src/remote/saved.rs`、`src/client/endpoint.rs` | Local/SSH Endpoint、Transport 和重连状态 | 远程连接 → Endpoint → API/PTY → 断线/重连 |
| Agent Skill/Plugin | `src/app/api/plugins/*`、`src/integration/*`、`src/cli/agent/*` | 能力声明、命令入口和授权边界 | Agent 请求 → API 命令 → 运行时资源 → 结果 |

Herdr V0 的参考实现骨架应包含 `RuntimeServer`、`PaneState`、`SessionSnapshot`、
`EndpointTransport` 和 `AgentObservation`。它们只表达上述状态流，不复制 Herdr
的终端渲染和平台适配代码。

### Multica 模块

| 模块 | 真实源码入口 | V0 参考架构 | V0 要解释的状态流 |
| --- | --- | --- | --- |
| Issue/Agent Invocation | `server/internal/service/issue.go`、`agent_invocation.go`、`agent_ready.go` | Issue 服务调用 Agent Policy 和 Runtime Access | Issue 分配 → 权限判断 → Agent Invocation |
| Runtime Access/Agent Host | `server/internal/service/runtime_access.go`、`agent_runtime_lookup.go`、`server/internal/runtimeapps/*` | Runtime Registry、Access Policy、Runner Adapter | Runtime 可用性 → 启动请求 → Agent Run |
| Scheduler/Lease | `server/internal/scheduler/db_ops.go`、`manager.go`、`stale_steal_test.go` | `Scheduler`、PostgreSQL execution row、lease token、heartbeat | claim → running → heartbeat → done/failed/stale |
| Failure/Retry/Review | `server/pkg/taskfailure/*`、`server/internal/service/retry_*`、`issue_workflow*` | Failure Classifier、Retry Policy、Review Gate | Run 结果 → 错误分类 → 重试或审核 |
| Client Projection/Realtime | `packages/core/api/*`、`packages/core/issues/*`、`packages/core/runtimes/*` | API Query、Realtime Patch、Issue View Projection | 服务端事实 → API/WebSocket → 看板状态 |

Multica V0 的参考实现骨架应包含 `Issue`、`Run`、`Runtime`、`Scheduler`、
`LeaseClaim`、`FailurePolicy` 和 `ReviewGate`。Lease 的数据库时钟、旧 Runner 写入
拒绝和 Review Gate 的状态边界必须在 V0 契约里写清楚。

### T3 Code 模块

| 模块 | 真实源码入口 | V0 参考架构 | V0 要解释的状态流 |
| --- | --- | --- | --- |
| Command/Orchestrator | `apps/server/src/orchestration-v2/Orchestrator.ts`、`CommandPolicy.ts`、`ThreadCommandExecutor.ts` | `Command` → `Orchestrator` → Domain Event | 用户命令 → 幂等判断 → 线程/运行决策 |
| Event/Projection | `EventStore.ts`、`EventSink.ts`、`ProjectionStore.ts`、`packages/contracts/src/orchestrationV2.ts` | Event Store、Projection Store、订阅游标 | 事件事务 → 投影 → 客户端读取 |
| Provider Runtime | `ProviderAdapterRegistry.ts`、`ProviderSessionManager.ts`、`ProviderTurnStartService.ts`、`Adapters/*` | Provider Adapter、Session、Turn、Event Ingestor | Run → Provider Turn → Provider Event → Run 状态 |
| Effect Outbox | `EffectOutbox.ts`、`EffectWorker.ts`、`RunFinalizationService.ts` | Outbox、Effect Worker、Result Ingress | 事务提交 → 外部副作用 → 结果回写 |
| Workspace/Checkpoint | `ProjectStore.ts`、`CheckpointService.ts`、`CheckpointRollbackService.ts`、`TaskFolderStore.ts` | Project、Worktree、Checkpoint、Rewind | Thread 选择工作区 → 修改 → 快照 → 回退 |
| RPC/Remote/Client | `orchestration-v2/http.ts`、`ThreadStream.ts`、`packages/contracts/src/relay.ts`、`packages/client-runtime/*` | Authenticated RPC、Stream、Client Projection | 客户端命令 → 服务端事实 → 游标订阅/恢复 |

T3 V0 的参考实现骨架应包含 `CommandReceipt`、`Orchestrator`、`DomainEvent`、
`Projection`、`PendingEffect`、`ProviderAdapter` 和 `Checkpoint`。V0 必须特别说明
Event Store 提交和 Effect Worker 执行之间的失败窗口。

### Lody 模块

| 模块 | 真实源码入口 | V0 参考架构 | V0 要解释的状态流 |
| --- | --- | --- | --- |
| ACP Agent Runtime | `apps/cli/src/agent/acp-runner.ts`、`managed-agent-runtime.ts`、`agent-client.ts` | ACP Adapter、Agent Process、Capability | Machine → ACP Session → Agent Event |
| Session Lifecycle | `apps/cli/src/session/session-manager.ts`、`session-execution-service.ts`、`commands/session.ts` | Session Store、Execution Service、Control API | create → running → steer/cancel → archive/resume |
| Cross-session Operation | `apps/cli/src/orchestration/coordinator-workflow.ts`、`operation-store.ts`、`session-dispatch-watcher.ts` | Operation、Child Item、Deadline、Coordinator | Operation → 多个 Session → 汇总结果 |
| Worktree/Review | `apps/cli/src/session/worktree/*`、`packages/shared/src/worktree-paths.ts` | Worktree Allocator、Diff、Review Feedback | Session → 隔离目录 → Diff → 反馈 |
| Sync/E2EE/Workspace | `packages/shared/src/session-data/*`、`session-doc-schemas.ts`、`packages/e2ee-core/*`、`packages/sync-core/*` | Document、Sync Domain、Device Key、Transport | 本地状态 → 同步帧/快照 → 设备恢复 |

Lody V0 的参考实现骨架应包含 `Machine`、`AgentRuntime`、`Session`、`Operation`、
`Worktree` 和 `SyncDocument`。ACP Provider 细节、Session 状态和 Operation 状态必须
分别建模，不能把它们压成一个 `Task.status`。

### Raft 模块

| 模块 | 真实源码入口 | V0 参考架构 | V0 要解释的状态流 |
| --- | --- | --- | --- |
| Workspace/Agent Identity | `packages/server` 的 Workspace/Agent 领域模块、`packages/shared` 的 Agent/Workspace 协议 | Workspace、Agent Identity、Presence、Capability Scope | 创建 Agent → 绑定 Workspace/机器 → Presence |
| Channel/Thread/Task | `packages/server` 的 Channel/Thread/Task 服务与 realtime 模块、`packages/shared` 的消息和会话协议 | Channel、Thread、Task、Visibility Policy | 消息/任务创建 → 可见性判断 → 交接 |
| Daemon/Computer Runtime | `packages/daemon/src/agentHost/*`、`agentProcessManager.ts`、`packages/computer/src/*` | Daemon、Agent Host、Machine Runtime、Lifecycle | Server 指令 → Machine Daemon → Provider Process |
| Credential/Capability Boundary | `packages/daemon/src/agentCredentialProxy.ts`、环境权限模块、`packages/server` 的 auth 模块 | Credential Broker、Access Policy、Scoped Secret | 请求能力 → 权限判断 → 代理凭据 → Agent |
| Memory/Attachment/Evidence | `packages/daemon/src/agentContextView/*`、`packages/shared/src/session-data/*`、Server Attachment 模块 | Context View、Attachment、Evidence Receipt | 证据产生 → 保存/引用 → Agent 可见投影 |
| Realtime/Sync | `packages/sync-core/src/core.ts`、`types.ts`、`domains/*`、`packages/shared` 的 streams | Sync Core、Domain Frame、Snapshot、Cursor | 本地事件 → 同步帧 → 快照/重连 |

Raft V0 的参考实现骨架应包含 `Workspace`、`AgentIdentity`、`CapabilityScope`、
`Channel`、`Thread`、`Task`、`CredentialBroker` 和 `SyncDomain`。V0 必须把消息可见性、
任务权限、机器权限和 Provider 凭据拆成四个检查点。

## 每个模块的 V0 交付格式

每个产品先有一个产品总览，之后每个模块使用自己的目录。这样四个模块不会共用
同一个 `V0-design.md`，源码说明和参考实现也能保持一一对应：

```text
tutorials/<numbered-product>/
  V0-overview.md
  modules/<numbered-module>/
    V0-design.md
    contracts/
      domain-flow.md
      state-and-errors.md
      interface-or-storage.md
    reference-architecture.md
```

产品目录编号表示推荐学习顺序，模块目录编号表示产品内部的推荐学习顺序。
`V0-overview.md` 只讲产品场景、模块关系、端到端数据流和第一个产品集成门。
模块自己的 `V0-design.md` 讲该模块的完整业务流程、边界和后续版本。`contracts/`
记录真实消费者需要遵守的状态、输入、输出、错误和副作用契约。
`reference-architecture.md` 将源码文件/符号映射到学习实现的文件和职责，给出调用
方向和最小可运行链路。它不提供完整产品实现，也不把源码片段拼成复制品。

V0 的源码对应表至少包含以下字段：

| 字段 | 说明 |
| --- | --- |
| `source_path` | 当前固定提交中的真实路径 |
| `symbol` | 类、函数、接口、测试或数据库操作 |
| `owns` | 该符号拥有的状态或副作用 |
| `called_by` | 入口和上游调用者 |
| `calls` | 下游调用和外部效果 |
| `reference_file` | 学习实现对应文件 |
| `v1_check` | V1 可直接执行的检查 |

## 产品集成门

模块版本完成后才进入产品集成门。集成门验证模块之间的真实交接，不要求复制产品
全部功能。

| 产品 | 第一个集成门 | 集成输入 | 集成输出 |
| --- | --- | --- | --- |
| Herdr | `H1` | Server、Pane、Session | 客户端断开后仍能恢复 Agent Pane |
| Multica | `M1` | Issue、Runtime、Scheduler、Run | 一次带 Lease 的 Run 回写 Issue 并进入 Review |
| T3 Code | `T1` | Command、Orchestrator、Event、Provider、Effect | 一个 Provider Turn 的状态和副作用可追踪 |
| Lody | `L1` | Machine、ACP、Session、Operation | 一个父 Session 派发并汇总一个子 Operation |
| Raft | `R1` | Agent Identity、Channel、Task、Daemon | 有权限的 Agent 从频道领取任务并回传证据 |

集成门另有自己的检查和版本号，不能把某个模块的 V2 直接称为产品 V2。

## 五条教程线各自研究什么

### Herdr：Agent 进程运行时

Herdr 是 Rust 终端运行时，不拥有业务任务。服务端保持 PTY、Workspace、Tab、Pane
和 Agent 检测状态；客户端通过本地 Socket/API 连接，`persist` 保存会话快照，`remote`
负责 SSH/远程桥接。它最适合作为第一条线，把已经掌握的 Agent Loop 接到真实进程、
终端和断线重连上。

版本路线：

- **V0 设计**：Server/Client、PTY、Pane、Workspace、Session Snapshot、Agent Detection
  和 Socket API 的所有权及状态转移。
- **V1 单机持久会话**：启动一个 Server，创建 Workspace/Pane，保存并恢复布局；检查
  客户端退出不会杀掉 Server 或 PTY。
- **V2 Agent 状态检测**：从终端输出和进程事实得到 working/idle/blocked；检查空闲
  误判不会覆盖工作状态。
- **V3 多客户端控制**：Socket 握手、订阅、命令和断开；检查一个客户端崩溃不影响
  其他客户端及服务端。
- **V4 远程连接**：SSH/remote bridge、端点重连和机器状态；检查远程断线不丢失本地
  会话事实。
- **V5 Agent Skill/插件边界**：Agent 通过 CLI/Socket 请求创建 Pane、互相提示和等待；
  检查运行时控制权限不会变成任意文件系统权限。

### Multica：Issue 驱动的 Agent 交付

Multica 的中心不是聊天，而是 Issue。Issue 绑定 Agent 和 Runtime，执行形成 Run，
运行输出、失败原因、重试、阻塞和 Review 回到同一个 Issue。Go 服务端中的 scheduler
使用数据库时间、claim、lease、heartbeat 和 stale-steal 处理后台执行；`packages/core`
负责客户端 API、查询和实时状态。

版本路线：

- **V0 设计**：Issue、Agent、Runtime、Run、Review、Skill、Scheduler 的生命周期，
  以及 Runtime 不可用、权限拒绝、Lease 丢失和重试边界。
- **V1 Issue 领取和一次执行**：从 Issue 分配到 Runtime 启动一个 Agent Run；检查权限
  和 Runtime 所有权在执行前生效。
- **V2 数据库调度**：实现一个固定 Job 的 claim、lease、heartbeat 和 terminal update；
  检查旧 Runner 失去 lease 后不能写终态。
- **V3 Run 结果与审核**：输出、错误分类、Diff/PR 和 Review Gate 回到 Issue；检查
  “运行结束”不等于“允许合并”。
- **V4 Squad、Skill 和自动唤醒**：把角色和解决方案转成可复用协作路径；检查自动
  唤醒仍遵守 Runtime、Workspace 和权限门禁。
- **V5 外部渠道与自托管**：只研究消息渠道、Git Host 和 self-host 边界，不实现完整
  平台复制。

### T3 Code：完整 Agent 控制面

T3 Code 的环境服务器拥有机器、项目、Provider 进程和文件系统；Web、Desktop、Mobile
通过认证 RPC/WebSocket 访问。`orchestration-v2` 将命令决策、事件日志、Projection 和
Effect Outbox 分开：`EventSink` 在事务中提交事实和投影，`EffectWorker` 在事务外执行
Provider、Checkpoint、通知等副作用。Provider Adapter 把 Codex、Claude、Cursor 等统一
到同一运行模型。

版本路线：

- **V0 设计**：Environment、Project、Thread、Run、Attempt、Provider Session/Turn、
  Subagent、Command Receipt、Event、Projection、Effect 和 Client Subscription。
- **V1 单 Environment/Provider Turn**：一个线程启动一个 Provider Turn，回传文本和工具
  事件；检查客户端只消费服务端状态。
- **V2 Command Receipt 与 Event/Projection**：命令幂等、事件写入和线程投影；检查
  重复命令不产生重复运行。
- **V3 Provider Adapter 与 Effect Outbox**：Provider 进程和外部副作用在事务外执行，
  结果再次进入编排；检查数据库提交失败不会假装 Provider 已运行。
- **V4 Worktree、Checkpoint 与 Rewind**：把文件系统隔离和线程历史绑定；检查回退失败
  不会留下部分恢复的仓库。
- **V5 RPC、Remote Environment 与多端订阅**：一个服务支持本地、远程、Relay/Tunnel
  客户端；检查断线重连从服务端游标恢复。
- **V6 Subagent、Handoff 与 Provider Switch**：父子线程、上下文交接和切换 Provider；
  检查子 Agent 结果回传不绕过父线程状态机。

### Lody：ACP 多机器共享会话

Lody 的 CLI/Daemon 连接多台机器，把 Claude Code、Codex、OpenCode 等 ACP Agent 变成
可共享的 Session。`apps/cli/src/agent` 负责 ACP Runtime 和进程，`session` 负责会话
生命周期，`orchestration` 负责跨会话 Operation；共享包保存 Session/Workspace/Message
等协议，后续还包含 Loro/E2EE/local-first 相关边界。

版本路线：

- **V0 设计**：Machine、Agent Runtime、Session、Worktree、Operation、Message、ACP
  Capability、Cloud/E2EE 的所有权和连接状态。
- **V1 Machine Daemon 与 ACP Session**：一台机器启动一个 ACP Agent Session；检查
  Agent Provider 细节被适配器隔离。
- **V2 Session Control**：创建、读取、Steer、Cancel、Archive 和 Resume；检查离线
  机器不会被 UI 当成在线。
- **V3 跨会话协调**：父会话派发 Operation 到其他 Session，跟踪子项状态和截止时间；
  检查 Operation 超时不会伪造 Agent 完成。
- **V4 Worktree 与 Review**：会话拥有隔离工作树、Diff 和评审反馈；检查会话之间不
  混写同一个工作树。
- **V5 Local-first/E2EE/Remote Sync**：研究 Loro、机器共享和端到端加密的边界；检查
  同步数据和 Agent 凭据不混在一起。

### Raft：人和 Agent 的共享工作空间

Raft 的中心对象是 Workspace、Agent、Channel、Thread、Task 和能力范围。Server 负责
API、Realtime、持久化和集成；Daemon 在机器上运行 Agent 并代理凭据；Computer Runtime
负责机器安装和连接；CLI/SDK 同时服务人和 Agent。它更强调“Agent 是长期参与者”，而
不是单次 Provider Turn。

版本路线：

- **V0 设计**：Workspace、Agent Identity、Capability Scope、Channel、Thread、Task、
  Memory、Machine Daemon 和 Credential Broker 的生命周期。
- **V1 Persistent Agent Identity**：创建 Agent、绑定能力和机器，读取 Presence；检查
  共享 Workspace 不等于共享机器凭据。
- **V2 Channel/Thread/Task 协作**：人和多个 Agent 在同一频道里发消息、创建任务和
  交接；检查消息可见性和任务权限独立判断。
- **V3 Daemon/Computer Runtime**：机器连接、Agent 启动和凭据代理；检查 Server 不把
  Provider Secret 直接交给模型。
- **V4 Memory、Attachment 与 Evidence**：把长期上下文、附件和工作证据关联到协作
  空间；检查记忆引用不自动扩大 Agent 权限。
- **V5 Realtime/Sync 与跨机器协作**：研究实时连接和状态同步；检查离线重连不重复发送
  已确认的操作。

## 模块版本和产品集成

每个模块使用自己的行为版本线。下面的顺序用于理解模块如何逐步变得可运行，实际
模块可以根据源码依赖增加或拆分版本：

```text
模块 V0 设计与契约
  → V1 最小业务闭环
  → V2 状态、持久化或并发边界
  → V3 外部执行和副作用
  → V4 协作、远程、恢复或工作树
  → Vn 该模块的产品特有能力
```

这不是五个产品共享的固定六步。每个模块选择实际需要的版本。`versions` 中的
产品版本只表示集成路线，具体模块版本必须写在对应模块的 `V0-design.md` 中。
产品级的 `H1`、`M1`、
`T1`、`L1`、`R1` 只在相关模块达到集成条件后出现。

每一版必须说明：

- 上一版本实际提供了什么；
- 本版本新增的可观察行为；
- 具体源码路径和符号；
- 输入、状态变化、输出和副作用；
- 至少一个正向和一个失败检查；
- 明确排除的产品能力。

模块之间的前置关系不是隐含约定。每个模块必须明确写出它读取的上游事实、产生的
下游契约和进入 V1 的冻结条件。完整的交接图见
[`tutorials/dependency-map.md`](tutorials/dependency-map.md)。产品之间没有代码依赖，
根目录编号只表示推荐阅读顺序；产品内部的模块编号才表示实现前置关系。

参考实现使用最小、独立、可运行的学习代码。教程解释真实源码的边界，但不复制源码、
不启动生产服务、不使用真实账号或凭据。每个模块的 V0 设计文档、契约和参考架构先
完成，再写该模块的 V1–Vn 正文。模块的参考实现完成不代表整个产品完成。

## 推荐阅读和构建顺序

1. **Herdr V0–V2**：补上 Agent Runtime 到真实进程、PTY、会话和状态检测的桥。
2. **Multica V0–V3**：理解 Issue、Runtime、Run、Scheduler、Lease 和 Review Gate。
3. **T3 Code V0–V3**：理解完整控制面中的 Command、Event、Projection、Outbox 和
   Provider Adapter。
4. **Lody V0–V3**：理解 ACP、多机器、Session 和跨会话 Operation。
5. **Raft V0–V3**：理解长期 Agent 身份、共享频道、能力范围和凭据代理。
6. 最后再分别补每条线的远程、同步、工作树、记忆或插件能力。

这个顺序不是产品优劣排序，而是从运行时基础到业务控制面，再到多机器和协作空间，
减少重复学习。

## 当前版本和下一步

五条教程线的 V0 初始化已完成。每条线都有一个产品总览，以及四个模块各自的
`V0-design.md`、三份契约和 `reference-architecture.md`。源码映射固定在各自的
基线提交，验收包含源码符号、数据流、错误边界和参考架构静态检查，不把设计文档
当成可运行产品。

第一篇实现正文从 Herdr 的连接生命周期开始。后续模块沿用各自的 V0 契约，在达到集成条件后验证跨模块交接。
[Herdr 路线图](tutorials/01-herdr-runtime/plan.md)

## 当前明确不包含

- 不建立 `framework-tutorials/`；
- 不建立单独的 `comparisons/` 目录；
- 不把五个产品接入 `social-media`；
- 不复制产品源码或绕过许可证；
- 不在没有确定边界前克隆完整产品功能；
- 不把外部控制面的状态当成 `social-task-queue` 的事实来源。
