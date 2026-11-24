# Channel / Thread / Task：状态与错误

- 无 visibility 的 Agent 不能读取消息或任务。
- Task claim 需要独立 capability 和当前可领取状态。
- 重复 claim 必须幂等或返回已有 owner。
- Thread message 发送成功不等于 Task 完成，evidence 缺失时保持待交付。
