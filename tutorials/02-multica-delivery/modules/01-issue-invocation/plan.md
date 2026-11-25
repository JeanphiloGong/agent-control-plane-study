# 01 Issue / Invocation：从 Issue 授权到 Run 请求

[Multica 总览](../../V0-overview.md) · [V0 设计](V0-design.md) · [参考架构](reference-architecture.md)

Issue 是用户要求交付的事实，Agent 是被授权的执行主体，Run 是一次具体执行请求。这个模块先把权限入口冻结，再把 Runtime readiness 接到请求上，最后把执行结果写回 Issue。调度器的 claim、lease 和 heartbeat 由下一个模块拥有。

| 版本 | 前置 | 独立计划 | 实现正文 | 验收 |
| --- | --- | --- | --- | --- |
| v1 | V0 设计 | [计划](v1/plan.md) | [Issue 授权与 invocation](v1/issue-invocation.md) | owner、workspace target、guest 和无权限拒绝 |
| v2 | v1 RunRequest | [计划](v2/plan.md) | [Runtime readiness](v2/runtime-readiness.md) | online 可执行、offline 可等待、缺少 runtime 拒绝 |
| v3 | v2 readiness | [计划](v3/plan.md) | [Run 结果回写](v3/run-result.md) | runner 成功/失败回写 Issue，review 仍独立 |

参考项目从 `v1/reference/` 开始使用 Go 标准库逐版追加。它保存内存中的 Issue、Agent、Runtime 和 Run，不连接真实 PostgreSQL，不启动 Agent 进程。真实源码以 `sources/multica` 的固定提交为对照。

## 真实调用链

```text
IssueService.Create
  → CanMemberInvokeAgent
  → AgentReadiness / RuntimeLookup
  → RunRequest
  → later Scheduler claim
```

V1 只决定调用者是否有权请求 Agent。V2 读取 Agent 绑定的 Runtime，区分可立即执行、可等待和必须人工处理的状态。V3 让注入的 runner 返回结果并更新 Issue，Review Gate 仍由 `04-failure-review` 负责。

## 交接边界

- `01` 不实现 lease、数据库时间或 stale steal。
- `01` 不把 Issue `done` 当成 review approved 或 shipped。
- 普通 offline Runtime 可以产生 deferred RunRequest，缺少 Runtime 和不可修复 Runtime 状态会拒绝请求。

## 验证方式

从仓库根目录按 v1 创建 `v1/reference`，后续版本在同一 Go module 中追加或替换指定文件。每个版本都提供 `go test ./...`，并在实际回放中单独运行前一版本测试和新增测试。
