# Failure / Review：领域流

```text
run terminal result → FailureClassifier → RetryPolicy / ReviewGate
                         ↓                    ↓
                    retry/requeue        review/blocked/shipped
                         ↓
                  IssueWakeupService → Issue projection
```

失败分类拥有“是否能自动继续”的判断，Review Gate 拥有“是否允许交付”的判断，两者不能合并。
