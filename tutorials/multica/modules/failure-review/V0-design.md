# Multica Failure / Review V0

## 版本与范围

- **源码**：`server/pkg/taskfailure/*`、`server/internal/service/retry_*`、`issue_wakeup*.go`、`packages/core/issues/*`
- **目标**：把 Run 结果分类为失败、可重试、需等待或需审核，并回到 Issue projection。
- **排除**：完整 Squad 编排、所有外部渠道。

## 真实流程

```text
Run result/error → taskfailure classifier
  → retry policy / issue wakeup
  → requeue or blocked/review state
  → packages/core issue projection
```

`IssueWakeupService` 还会检查条件、截止时间和 claim authority。重试不是简单复制一条 Task，必须保留原 Issue、workflow step 和交接 brief。

## 源码对应

| 路径 | 符号/责任 |
| --- | --- |
| `server/pkg/taskfailure/*` | 错误类别、失败原因和 retryability |
| `server/internal/service/retry_*` | 重新创建/延迟 Run 的服务边界 |
| `server/internal/service/issue_wakeup.go` | `IssueWakeupService`、`Validate`、`Tick`、`dispatch` |
| `server/internal/service/issue_wakeup_manage.go` | wakeup trigger、check-in 和关闭/删除 |
| `packages/core/issues/*` | Issue 查询和实时 projection |

## 局部版本线

- V1：Run failure 分类和 Review Gate。
- V2：有限重试、延迟和 Issue wakeup。
- V3：客户端 projection、workflow handoff 和自动唤醒门禁。

## V1 检查

对 provider network、lease lost、用户拒绝和真实业务失败分别分类。只有允许重试的类别进入 retry policy，Run done 不直接成为 Review approved。

契约见 `contracts/`，参考架构见 `reference-architecture.md`。
