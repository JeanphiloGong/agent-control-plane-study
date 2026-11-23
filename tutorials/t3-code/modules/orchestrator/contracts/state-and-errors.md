# Command / Orchestrator：状态与错误

- receipt 的 idempotency key 必须稳定。
- unsupported capability、busy source、invalid target 和 authorization failure 在副作用前返回。
- command accepted 只表示服务端接受事实，不表示 Provider 已完成。
- executor 串行化必须覆盖会改变同一 Thread 状态的命令。
