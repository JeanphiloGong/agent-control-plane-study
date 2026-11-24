# Multica V0：Issue 驱动的 Agent 交付

## 版本身份

- **模块**：`multica-issue-orchestration`
- **基线**：`sources/multica` @ `4ceffd536603c67620c7460f6c412dcbf092299e`
- **范围**：Issue、Agent、Runtime、Run、Scheduler、Lease、失败分类和 Review Gate。
- **结果**：读者能追踪一个 Issue 从授权、Runtime readiness、领取执行、心跳、终态更新到审核投影的完整控制流。

## 真实场景

用户创建一个 Issue 并指定 Agent。服务端检查成员是否可以调用该 Agent，再检查 Agent 绑定的 Runtime 是否可用。调度器用数据库时间领取 Run，持有者定期 heartbeat，失去 lease 的旧 runner 不能写入终态。Run 完成后仍要经过 Review Gate，不能把执行成功直接当成已经交付。

```text
IssueService.Create
  → CanMemberInvokeAgent / AgentReadiness
  → Scheduler.tryClaim
  → runClaimed + heartbeat
  → finishSuccess / finishFailure
  → FailureClassifier / RetryPolicy
  → ReviewGate / Issue projection
```

## 模块

| 模块 | 责任 | V0 |
| --- | --- | --- |
| 01 Issue / Invocation | Issue 创建、Agent 调用授权和 Run 请求 | [01-issue-invocation](modules/01-issue-invocation/V0-design.md) |
| 02 Runtime Access | Agent Runtime 查找、可用性和 Runner 适配 | [02-runtime-access](modules/02-runtime-access/V0-design.md) |
| 03 Scheduler / Lease | claim、数据库时钟、heartbeat、stale steal | [03-scheduler](modules/03-scheduler/V0-design.md) |
| 04 Failure / Review | 错误分类、重试、唤醒和审核投影 | [04-failure-review](modules/04-failure-review/V0-design.md) |

## 不变量

1. Issue 权限、Agent 权限和 Runtime 可用性在执行前分别判断。
2. Lease token 是终态写入的围栏，旧 runner 即使进程仍活着也不能覆盖新 owner。
3. 数据库时间决定 stale_after，不使用 worker 本地时钟作最终判断。
4. `Run done`、`review approved` 和 `shipped` 是不同事实。

## 集成门 M1

学习实现需要完成一次带 lease 的 Run，并把结果回写 Issue 后进入 Review。必须检查竞争领取、失去 lease 的旧写入和 Review Gate 拒绝路径。真实 PostgreSQL、完整看板和外部 Git 渠道留到后续版本。
