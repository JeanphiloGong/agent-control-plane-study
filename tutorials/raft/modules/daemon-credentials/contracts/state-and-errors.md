# Daemon / Credentials：状态与错误

- machine offline、capability denied、process start failure、stall 和 credential denied 分开记录。
- process restart 不代表 Task/Thread 自动完成。
- secret 只能经 scoped proxy 注入，日志、消息和 activity summary 必须脱敏。
- daemon 重连需要重新验证 identity/capability。
