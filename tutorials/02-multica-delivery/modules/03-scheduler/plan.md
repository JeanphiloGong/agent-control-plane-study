# 03 Scheduler / Lease：系统任务的领取、续期与接管

[Multica 总览](../../V0-overview.md) · [V0 设计](V0-design.md) · [参考架构](reference-architecture.md)

两个 Multica 服务实例可能同时发现同一个系统任务到期。这个模块从已注册 job 的一个到期计划开始，跟踪 execution row 如何记录当前尝试、续期和终态。服务端注册的 `issue_wakeup_dispatch` 会在 handler 内调用 `IssueWakeupService.Tick`，处理到期的 Issue 唤醒。

| 版本 | 前置 | 独立计划 | 实现正文 | 验收 |
| --- | --- | --- | --- | --- |
| v1 | V0 设计与已计算的 `Plan` | [计划](v1/plan.md) | [claim 与 terminal update](v1/claim-terminal.md) | 竞争 claim 只有一个 owner，当前 token 才能 finish |
| v2 | v1 execution row | [计划](v2/plan.md) | [心跳续期](v2/heartbeat.md) | heartbeat 延长 stale_after，错误 token 返回 ErrLeaseLost |
| v3 | v2 heartbeat | [计划](v3/plan.md) | [stale steal 与重试](v3/stale-steal.md) | stale 接管旋转 token，旧 finish 被拒绝，失败可按时间重试 |

代码基线为 `sources/multica` 的 `4ceffd536603c67620c7460f6c412dcbf092299e`。V1 创建 `v1/reference/` Go module，V2 和 V3 在同一个 `Store` 中演进。参考代码用内存、mutex 和可控 `Clock` 演示状态转换。真实 SQL 的唯一约束、条件更新和数据库时间由 PostgreSQL 测试覆盖。

## 真实调用链

```text
Manager.RunOnce
  → dbNow
  → runJob / plansForTick
  → processPlan / tryClaim
  → runClaimed
       ├─ heartbeat
       └─ finishSuccess / finishFailure
```

`(job_name, scope_kind, scope_id, plan_time)` 唯一标识一行 execution。接管更新这一行，递增 attempt，替换 runner 和 token。用旧 token 提交时会得到 `ErrLeaseLost`。

## 交接

系统 job 的终态由 Scheduler 写入 `sys_cron_executions`。下游模块继续沿 Issue Wakeup 和 Agent Run 自己的状态追踪失败分类、重试与交付审核。 [Failure / Review](../04-failure-review/V0-design.md)

## 验证方式

每版在 `v1/reference/` 运行 `go test ./...`。V3 给出使用一次性 PostgreSQL 测试库运行真实 Scheduler 测试的命令。 [V3 验收](v3/stale-steal.md#5-对照真实-scheduler-的-postgresql-验收)
