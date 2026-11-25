# 02 Runtime Access：从 Runtime 事实到受限启动边界

[Multica 总览](../../V0-overview.md) · [V0 设计](V0-design.md) · [参考架构](reference-architecture.md)

`01 Issue / Invocation` 已经生成了带 Agent/Runtime identity 的 RunRequest。这个模块负责确认 Runtime 是否存在、调用者是否有权使用它、状态是否可以执行，以及最终把请求交给 runner adapter。它不负责 scheduler lease，也不把 adapter 的结果直接写成 Issue done。

| 版本 | 前置 | 独立计划 | 实现正文 | 验收 |
| --- | --- | --- | --- | --- |
| v1 | `01` RunRequest | [计划](v1/plan.md) | [Runtime registry 与 readiness](v1/runtime-registry.md) | online、offline、unknown 状态可区分 |
| v2 | v1 lookup | [计划](v2/plan.md) | [ownership 与 access policy](v2/access-policy.md) | owner、public、shared、越权边界 |
| v3 | v2 verdict | [计划](v3/plan.md) | [Runner adapter 启动](v3/runner-adapter.md) | ready 才启动、offline deferred、adapter failure 回传 |

参考项目从 `v1/reference/` 的 Go module 继续，使用内存 Registry 和 fake adapter。真实数据库、Runtime daemon、进程生命周期和 Scheduler lease 留在源码对照与后续模块。

## 真实调用链

```text
Agent/RunRequest
  → RuntimeLookup
  → RuntimeAllowsUser
  → AgentReadiness
  → RunnerAdapter.Start
  → start handle or classified error
```

`RuntimeAllowsUser` 只处理 owner/share 权限，`AgentReadiness` 只处理 runtime 绑定和状态。adapter 只接收已经通过 access/readiness 的 `StartRequest`，不能自己绕过服务层。

## 交接边界

- offline Runtime 产生可等待 verdict，不启动 adapter。
- unknown、missing、unusable 和 access denied 不能降级为 ready。
- adapter failure 只返回启动错误，不直接修改 Issue 或 Review 状态。

## 验证方式

从仓库根目录按 v1 创建 `v1/reference`，每版在同一 Go module 中追加文件。每个版本都提供 `go test ./...`，最终回放再运行真实 Multica service 的 Runtime access/readiness 测试。
