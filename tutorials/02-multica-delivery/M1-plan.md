# Multica M1：Issue 驱动的带 Lease Run 集成门

[产品 V0 总览](V0-overview.md) · [依赖图](../dependency-map.md) · [集成正文](M1-integration.md)

## 集成门身份

M1 是产品级集成检查点。它把已完成的四个模块交接成一次可观察的 Run：

~~~text
Issue
  → invocation authorization
  → Runtime readiness
  → Scheduler claim / heartbeat
  → Run terminal result
  → Failure / Review decision
  → Issue projection
~~~

M1 没有独立的 v1–v3 版本线。模块版本已经冻结，M1 负责验证跨模块数据是否沿真实边界流动。

## 输入

- Issue、Agent 和调用者身份
- Runtime access/readiness verdict
- Scheduler 当前 execution 和 lease token
- Agent Run 的 success 或 failure result

## 输出

- 成功 Run 只能把 Issue 投影为 in_review
- provider network failure 产生可重试决策
- 权限或业务失败产生 Review/blocked 决策
- lease 丢失后旧 runner 不能写入终态
- 每个拒绝或终态都保留对应的 Run 和 Issue 事实

## 排除

- 完整 PostgreSQL service wiring
- HTTP handler 和前端看板
- 真实 Provider 进程启动
- Git diff、PR merge 和外部渠道
- Review 审批操作本身

这些边界由已完成模块和后续产品能力分别拥有。M1 只冻结一次端到端交接。
