# Server / Client / Socket：领域流

## 领域对象

| 对象 | 所有者 | 说明 |
| --- | --- | --- |
| `ServerRuntime` | 服务端 | 维持 App、PTY、TerminalRuntime 和客户端集合 |
| `ClientConnection` | 服务端连接层 | 记录握手状态、client id、尺寸和写入队列 |
| `ApiRequest` | API socket | 带 request id 的 JSON 命令，转给 App 处理 |
| `ServerEvent` | 服务端事件循环 | 连接建立、消息、断开和渲染/状态通知 |
| `ClientMessage` / `ServerMessage` | protocol | TUI socket 的双向 wire 消息 |

## 一次操作

```text
client connect
  → handle_client_handshake
  → ClientConnected(ServerEvent)
  → HeadlessServer 绑定 writer
  → ClientMessage(Input/Resize/Detach)
  → App 或 TerminalRuntime
  → ServerMessage / AppEvent
```

API 请求走 `handle_connection_with_stop`，订阅请求进入 `stream_subscriptions`，它们与 TUI transport 共用服务端事实但不共用 wire 格式。

## 所有权规则

1. 服务端创建、持有和关闭 PTY。
2. 客户端只拥有自己的连接、输入来源和展示状态。
3. 客户端断开产生连接事件，不直接调用 `TerminalRuntime::shutdown`。
4. 只有服务端关闭或明确的 pane/应用命令才能销毁运行时。
