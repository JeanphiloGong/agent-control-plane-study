# Orca Review / Remote：参考架构

~~~text
Dispatch + Worktree
        ↓
HostSelector → ReviewReader → ReviewInput
      ↓             ↓              ↓
Remote receipt  base/head/diff  HumanReview
                                      ↓
                         MergeEvidence → CleanupEvaluator
~~~

## 学习实现目录

~~~text
reference/review/evidence.ts       # base/head/diff and report binding
reference/review/placement.ts      # host-qualified workspace selection
reference/review/remote.ts         # reachable/unverified handoff
reference/review.test.ts            # review and remote boundaries
reference/review/cleanup.ts         # merge evidence to cleanup input
~~~

| Orca 源码 | 参考组件 | 验收 |
| --- | --- | --- |
| branch-diff.ts:getBranchDiff | ReviewReader | diff 使用明确 commit 范围 |
| worktree-review-link-flags.ts | ReviewLinkParser | provider link 不承担 approval |
| placement-and-remote.md | HostSelector | project 和 host 选择可追踪 |
| worker-view-results.ts | RemoteReceipt | liveness/reachability 不被隐藏 |
| workspace-cleanup-removal-step.ts | RemovalSettlement | verdict 与实际删除分离 |

V1 复用 01 的 Workspace identity 和 02 的 worker report。V2/V3 只增加 host 与 remote 事实，
不建立第二套 Task 或 Worktree 状态。
