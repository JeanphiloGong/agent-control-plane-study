# Failure / Review：参考架构

```text
RunResult → FailureClassifier → RetryPolicy
       │           │                 ↓
       │           └────────────── ReviewGate
       ↓                              ↓
IssueWorkflowState ← WakeupService ← IssueProjection
```

学习实现文件为 `failure_classifier.go`、`retry_policy.go`、`review_gate.go`、`wakeup_service.go` 和 `issue_projection.ts`。重试创建新 attempt，但保留同一 Issue 和父 workflow 引用。

V1 冻结检查包含可重试网络错误、不可重试权限错误和 Review 拒绝。
