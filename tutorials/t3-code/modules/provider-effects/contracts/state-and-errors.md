# Provider / Effects：状态与错误

- effect 只有在事务提交后可见。
- adapter 不支持、认证失败、进程退出和网络失败分别分类。
- worker 重试必须使用 effect idempotency key。
- Provider 已执行但 result ingress 丢失时进入 needs inspection，不能直接重启 turn。
