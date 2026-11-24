# Herdr Server / Client / Socket V0

## 版本身份与边界

- **模块**：`herdr-server-client`
- **基线**：`sources/herdr` @ `e8d7d9705a7a2c89cef4dd42079379f05685af87`
- **V0 目标**：解释服务端如何拥有运行时，如何接受 API 与 TUI 客户端，以及断开如何被限制在客户端边界内。
- **包含**：进程启动、两个 socket 协议、握手、命令/事件、客户端断开和服务端清理。
- **不包含**：PTY 内部实现、快照字段设计、真实 SSH、完整 surface 渲染优化。

## 场景

用户运行 `herdr` 后，服务端启动一个 `HeadlessServer`，同时监听本地 API socket 和 thin-client socket。TUI 完成 `TerminalHello` 后接收 `Welcome`，发送 `Input`、`Resize`、`AttachTerminal` 和 `Detach`。API 客户端则发送 JSON request，必要时通过 `events.subscribe` 接收事件。任一客户端断开，服务端仍拥有 `App` 和 PTY。

## 源码调用链

| 源码 | 符号 | 责任 |
| --- | --- | --- |
| `src/main.rs` | `main` | 根据启动参数选择 server、client、remote 或普通命令入口 |
| `src/server/headless/bootstrap.rs` | `run_server` | 创建 `App`、启动 API server、创建 `HeadlessServer` 并进入服务端循环 |
| `src/server/headless.rs` | `HeadlessServer` | 持有服务端事件循环、客户端集合、运行时和关闭状态 |
| `src/api/server.rs` | `start_server_with_stop_control`, `handle_connection_with_stop` | 绑定受权限保护的本地 JSON API socket，解析 request 并转发到 App |
| `src/api/server.rs` | `stream_subscriptions` | 从 `EventHub` 按订阅游标向 API 客户端发送事件 |
| `src/server/client_accept.rs` | `accept_pending_client_connections` | 接受 thin-client 连接，为每个连接启动握手线程 |
| `src/server/client_transport.rs` | `handle_client_handshake`, `client_read_loop` | 校验 hello、建立 `ClientWriter`、把 ClientMessage 转成 ServerEvent |
| `src/client/mod.rs` | `run_client_with_mode`, `run_client_loop` | 连接服务端、发送输入和尺寸变化、消费 ServerMessage |
| `src/protocol/*` | `ClientMessage`, `ServerMessage`, endpoint hello/welcome | 定义 wire contract，不由 UI 自行解释 |

## 状态流

```text
Stopped
  → socket prepared / permissions restricted
  → Listening
  → ConnectionPending
  → HandshakeAccepted
  → ClientConnected
  → command/event exchange
  → ClientDisconnected
  → server continues
  → ServerStopping → owned sockets removed
```

API JSON socket 和 TUI thin-client socket 都以服务端 App 为下游。API 请求失败只返回错误响应；客户端连接错误只结束该连接。`ServerHandle::Drop` 通过 `running` 和 socket identity 清理自己拥有的 API socket，不能删除新进程接管的同名 socket。

## 局部版本线

- **V1**：单机服务端、一个 TUI 客户端、一个 pane 的握手和断开。
- **V2**：API JSON request 与事件订阅，补充 request id、连接超时和错误响应。
- **V3**：多个 thin client、输出队列背压和客户端隔离。
- **V4**：live handoff、远程 endpoint 和重连恢复。

## V1 检查

在参考实现中创建一个服务端和 pane，客户端完成握手后发送 `Detach`。断言服务端仍有同一个 pane 和运行时句柄，再连接第二个客户端读取同一服务端状态。测试不能只断言 socket 返回 `Welcome`，还要断言客户端生命周期没有销毁运行时。

契约见 [domain-flow](contracts/domain-flow.md)、[state-and-errors](contracts/state-and-errors.md)、[interface-or-storage](contracts/interface-or-storage.md)，参考实现见 [reference-architecture](reference-architecture.md)。
