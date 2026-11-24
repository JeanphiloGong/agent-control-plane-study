# Server / Client / Socket：状态与错误

## 状态

| 状态 | 进入 | 离开 |
| --- | --- | --- |
| `Listening` | socket 绑定成功且权限设置完成 | stop 或不可恢复的服务端错误 |
| `ConnectionPending` | accept 得到连接 | hello 成功、超时或协议错误 |
| `ClientConnected` | hello 校验成功并发出 `ClientConnected` | `Detach`、EOF、写入失败或服务端停止 |
| `ServerStopping` | shutdown/handoff 被请求 | 所有连接通知完成、socket 清理 |

## 必须保持的错误边界

- hello 超时只关闭该连接。
- 非法 JSON/API request 返回带 request id 的错误，不进入 App。
- thin-client 协议版本或尺寸不安全时发送拒绝或关闭连接。
- 写队列超过控制数据上限时断开超载客户端，不能阻塞其他客户端。
- accept 临时失败应退避重试，不能留下“socket 文件存在但没有 listener”的假在线状态。

## 不在本模块判断的错误

PTY 启动、Agent 检测、快照解析和 SSH 兼容性由对应模块处理。Server/Client 只负责把结果作为服务端事实或连接错误传递。
