# Herdr V0：Agent 进程运行时总览

## 版本身份

- **模块**：`herdr-agent-runtime`
- **模式**：backend/control-plane
- **版本**：V0 design
- **源码基线**：`sources/herdr`，提交 `e8d7d9705a7a2c89cef4dd42079379f05685af87`
- **读者前置**：已经理解 Agent Runtime、PTY、工具调用、权限和持久化的基本概念。
- **本版本结果**：能够从 Herdr 的真实入口追踪“启动服务端 → 创建/保持 PTY → 客户端连接 → 状态事件 → 快照恢复”的责任分配。

V0 是设计和源码取证版本。它不修改 Herdr，不复制 Herdr 的终端渲染，不启动真实账号，也不声称学习实现已经具备远程或插件能力。

## 一个真实场景

用户启动 Herdr，在一个 pane 中运行 Codex。TUI 客户端随后退出，服务端和 PTY 继续存在。用户重新连接时，应看到原来的 workspace、tab、pane 和终端工作目录。Agent 是否 working、idle 或 blocked，则由服务端拥有的 terminal runtime 根据 PTY 和屏幕证据判断，再通过事件或查询交给客户端。

这条路径的事实来源是服务端状态。客户端只是输入、展示和订阅者，不能因为自己的窗口关闭就销毁 PTY，也不能用本地 UI 状态覆盖服务端的 Agent 状态。

## 端到端数据流

```text
src/main.rs::main
  ├─ server/headless/bootstrap.rs::run_server
  │    ├─ api/server.rs::start_server_with_stop_control
  │    ├─ app/mod.rs::App::try_new
  │    │    ├─ persist::load → persist::restore
  │    │    └─ TerminalRuntime / PaneRuntime / PTY
  │    └─ server/headless::HeadlessServer 处理 ServerEvent
  └─ client/mod.rs::run_client
       ├─ 本地 client socket 握手
       ├─ ClientMessage 输入/resize/attach
       └─ ServerMessage surface、terminal、事件和错误

PTY 输出 → TerminalRuntime / TerminalState → AgentDetection → AppEvent/EventHub
                                                          ↓
                                               API 查询、订阅或客户端帧
workspace 变化 → App::schedule_session_save → capture → SessionWriter → JSON
```

Herdr 同时有两条客户端边界。`src/api/server.rs` 提供本地 JSON API，供 CLI、自动化和远程控制使用。`src/server/client_transport.rs` 与 `src/client/mod.rs` 处理 TUI thin client 的二进制协议、握手、输入和渲染帧。两者都进入同一个服务端运行时，不应在教程中合并成一个“通用 socket”。

## 模块边界

| 模块 | 负责的问题 | V0 文档 |
| --- | --- | --- |
| 01 Server / Client / Socket | 服务端所有权、握手、命令、事件和断开 | [01-server-client](modules/01-server-client/V0-design.md) |
| 02 PTY / Pane / Agent Detection | PTY 输出如何变成可解释的 Agent 状态 | [02-pane-detection](modules/02-pane-detection/V0-design.md) |
| 03 Session Persistence | workspace 布局、pane 启动信息和恢复副本如何保存 | [03-session-persistence](modules/03-session-persistence/V0-design.md) |
| 04 Remote Endpoint / Plugin | SSH endpoint 和插件能力如何经过边界进入运行时 | [04-remote-plugins](modules/04-remote-plugins/V0-design.md) |

每个模块的契约和参考架构都在自己的目录中。模块版本不等于产品版本，四个模块达到交接条件后才进入 `H1`。

## 模块依赖

```text
01 Server / Client / Socket
  → 02 PTY / Pane / Agent Detection
  → 03 Session Persistence
  → 04 Remote Endpoint / Plugin
  → H1
```

`01` 先冻结服务端运行时所有权、客户端握手和断开边界。`02` 使用这个运行时接收
PTY 和进程证据，产生稳定的 Agent 状态。`03` 保存前两个模块拥有的 session 和
terminal 事实。`04` 最后把 endpoint transport 和插件命令接到已有的客户端边界，
避免远程或插件直接拥有核心运行时状态。

## V0 需要保留的不变量

1. **运行时所有权**：PTY、TerminalRuntime 和可恢复的 session 事实由服务端持有。客户端断开不能触发运行时销毁。
2. **状态分离**：`PaneState` 是视口相关的纯状态，`TerminalState` 保存 terminal identity、cwd、Agent 元数据和状态；不能把 TUI 视图状态当作 Agent 事实。
3. **证据优先**：检测器接收终端屏幕或 hook 证据，产生 `AgentDetection`，再由稳定化逻辑决定是否发布状态变化。
4. **恢复可审计**：快照带版本，未知未来版本拒绝加载；恢复失败不能覆盖仍可用的旧文件。
5. **边界不泄漏**：远程 endpoint 的持久键不含 SSH target；插件只能从已解析 manifest 和受限 invocation context 获得能力。

## 产品集成门 H1

### 输入

- `RuntimeServer` 或等价的服务端运行时
- `PaneState` / `TerminalState` 与一个可运行的 PTY
- `SessionSnapshot` 捕获和恢复路径
- API/client socket 的握手和断开语义

### 输出

在隔离学习实现中，客户端连接、创建一个 pane、发送退出/断开事件后，服务端仍保持 pane 运行；重新启动服务端时，快照恢复原 workspace、tab、pane 和 cwd，且不会重复创建同一 pane。

### 失败检查

- 客户端断开不能让服务端把 PTY 标记为已结束。
- 快照版本高于当前支持版本时必须拒绝恢复。
- 恢复副本写入失败时不能覆盖原 session 文件。

H1 不是生产 Herdr 的完整测试，也不包含真实 SSH、完整渲染帧、所有 Agent manifest 或插件市场。

## 模块后续版本

每个模块在自己的 V0 文档末尾定义局部版本线：先做最小业务闭环，再增加持久化/并发，最后加入远程、协作或插件等该模块特有能力。产品级 V1–V5 只在模块交接后描述端到端集成，不替代模块版本。

## V0 验收

- 四个模块都有唯一的 V0 文档目录。
- 每个模块都列出固定提交中的源码路径、符号、调用者、下游调用、状态所有权和 V1 检查。
- 每个模块都有 `domain-flow.md`、`state-and-errors.md`、`interface-or-storage.md` 和 `reference-architecture.md`。
- `H1` 的输入、输出和失败边界不依赖未定义的 UI 或真实远程环境。

源码阅读入口见 [Herdr 源码基线](../../plan.md#源码基线) 和 [教程规划](../../plan.md#herdragent-进程运行时)。
