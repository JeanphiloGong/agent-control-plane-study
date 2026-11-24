# Server / Client / Socket：接口契约

## API socket

- 输入是带字符串 `id`、方法和参数的 JSON request line。
- 输出是同一 `id` 的 success/error JSON line。
- 事件订阅先返回 `SubscriptionStarted`，之后按事件游标发送事件。
- API socket 文件权限由 `restrict_socket_permissions` 控制，socket identity 用于避免误删其他进程的 socket。

## Thin-client socket

- 客户端先发送 `TerminalHello` 或 endpoint control hello。
- 服务端通过 `ServerMessage::Welcome` 返回协议和能力。
- 输入、resize、attach、detach 使用 `ClientMessage`。
- 渲染、终端帧、通知、关闭和错误使用 `ServerMessage`。

## 兼容性

协议字段和生成号属于稳定边界。参考实现可以缩小消息集合，但必须保留 request id、握手结果、断开和错误的可观察语义，不能把 API JSON 和 TUI 二进制消息拼成一个 DTO。
