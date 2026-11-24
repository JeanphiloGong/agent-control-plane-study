# Issue / Invocation：状态与错误

- Issue 创建失败不产生 Run。
- 成员没有调用权限时在授权边界拒绝。
- Agent readiness 为 blocked/offline 时不进入 Runner。
- readiness 的修复建议可以展示给用户，但不能把建议当成执行成功。
- Issue 创建、Run 创建和通知是否同一事务，必须由应用服务显式决定并记录。
