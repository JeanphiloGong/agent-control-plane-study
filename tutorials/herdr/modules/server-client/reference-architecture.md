# Server / Client / Socket：参考架构

## 学习实现目录

```text
src/server_runtime.rs   # ServerRuntime、客户端集合和关闭
src/api_socket.rs       # JSON request/response、事件订阅
src/client_protocol.rs  # ClientMessage/ServerMessage、握手
src/client_session.rs   # 客户端连接和发送/接收循环
src/runtime_events.rs   # 服务端事实到客户端事件的投影
```

## 调用方向

```text
ClientSession ──messages──> ServerRuntime ──commands──> App/TerminalRuntime
       ^                          │
       └──── ServerMessage <──────┴──── RuntimeEvent

ApiSocket ──ApiRequest──────> ServerRuntime/App
ApiSocket <──Response/Event── EventHub
```

`ServerRuntime` 是唯一的 runtime owner。`ClientSession` 和 `ApiSocket` 都是边界适配器，不能直接修改 `TerminalState`。学习实现可以使用内存队列，不需要复制 Herdr 的渲染编码，但必须保留断开后 runtime 仍存在的关系。

## 源码到参考架构

| Herdr | 参考实现 | V1 检查 |
| --- | --- | --- |
| `HeadlessServer` | `ServerRuntime` | runtime 在 client detach 后仍存在 |
| `start_server_with_stop_control` | `ApiSocket.start` | socket 绑定和清理只作用于自有 identity |
| `handle_client_handshake` | `ClientSession.handshake` | 非法/超时 hello 不创建 session |
| `ClientWriter` 队列 | `OutboundQueue` | 控制消息有序，超载连接被隔离 |
| `ClientMessage` / `ServerMessage` | `ClientMessage` / `ServerMessage` | 输入和状态通过同一 runtime 交接 |

## V1 冻结检查

```text
server = ServerRuntime.start()
pane = server.create_pane()
client = server.connect_client()
client.handshake()
client.detach()
assert server.pane(pane.id).is_running()
second = server.connect_client()
assert second.snapshot().pane_ids == [pane.id]
```

这是学习实现的行为检查，不证明 Herdr 的完整 Rust socket、渲染和 live handoff 已被复制。
