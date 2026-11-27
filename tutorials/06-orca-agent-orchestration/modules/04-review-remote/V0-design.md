# Orca 04 Review / Remote：V0 设计

[Orca 总览](../../V0-overview.md) · [模块计划](plan.md) · [参考架构](reference-architecture.md)

## 版本身份

- **模块**：orca-review-remote
- **基线**：sources/orca @ 27300a14e0170258814b67b2f4159446c637f551
- **模式**：backend / design
- **目标**：把 worker 的检查结果、Git diff、执行 host 和远程可达性交给独立 Review/merge 流程。
- **前置**：03 Session / Recovery V3 已将 worker 终态和 preserve 事实交给上层。
- **V1 交接**：生成绑定 workspace identity、base 和 head 的 diff review 输入。

本版不执行真实 merge、PR 创建或 worktree 删除。Review 是决策输入，merge 是另一个授权动作。

## 真实场景

Worker 在本机或 paired host 的 worktree 中完成代码修改。Coordinator 需要让 Reviewer 知道
它查看的具体 checkout、base、head、检查结果和 Dispatch。远程 host 暂时不可达时只能说
“未验证”，不能说“没有变更”或“已完成”。Cleanup 还必须等待 review/merge、活动和本地文件证据。

~~~text
worker report + host identity
  → read exact worktree/head/base
  → build diff/review input
  → remote handoff or unverified
  → human/external review
  → merge decision
  → cleanup preview
~~~

## 领域对象和真实入口

| 对象 | 真实源码 | 责任 |
| --- | --- | --- |
| Review link | src/cli/workspaces/worktree-review-link-flags.ts | 解析 PR/MR 目标并保持 provider 字段边界 |
| Branch diff | src/main/git/source-control/comparisons/branch-diff.ts | 用明确 headOid、mergeBase 和 file path 读取差异 |
| Execution host | src/cli/hosts/*、skill-guides/orchestration/references/placement-and-remote.md | 选择 project、repo、workspace 所在机器 |
| Worker projection | src/cli/orchestration/workers/worker-view-results.ts | 给出 worker、workspace、liveness、report 和 remote warning |
| Cleanup evidence | src/shared/workspace-cleanup/workspace-cleanup.ts | 判断 merged、unchanged、dirty、active 和 unknown |
| Removal flow | src/main/workspaces/cleanup/workspace-cleanup-removal-step.ts | 在 verdict 后再次执行 host-local removal 检查 |

## 业务不变量

1. Review input 必须绑定 Dispatch ID、Task ID、workspace identity、base 和 head。
2. worker report 的检查结果不能代替 Reviewer 对 diff 和 head 的读取。
3. host 是 workspace identity 的一部分，同路径不同 host 不是同一个 checkout。
4. remote unreachable 产生 unverified/keep，不产生 succeeded、merged 或 deleted。
5. clean worktree 不等于已经合入，HEAD 是否落在 default/coordinator branch 需要独立 merge evidence。
6. Review、merge 和 cleanup 是三个事实，任何一个完成都不能伪造另外两个。

## V1–V3 交接

- **V1 Review evidence**：生成 base/head/diff 和 worker report 的绑定输入。
- **V2 Host selection**：以 projectId、hostId 和 workspace identity 选择本机或 paired host。
- **V3 Remote handoff**：远程读取失败保留工作，重新可达后再次验证，不自动 merge。

## V0 验收

- 一个 worker report 的 head 与当前 workspace HEAD 不同时，为什么 review 必须拒绝？
- 为什么同一路径在两个 host 上不能只用 path 选择？
- remote RPC 断线时，Reviewer 和 cleanup 各自得到什么状态？
- 为什么 clean checkout 仍然可能得到 unlanded？

真实检查入口包括 src/main/git/source-control/comparisons/branch-diff.ts、src/main/workspaces/cleanup/workspace-cleanup-removal-step.ts、src/cli/orchestration/workers/worker-view-results.ts 和 skill-guides/orchestration/references/placement-and-remote.md。

[领域流](contracts/domain-flow.md)、[接口与存储](contracts/interface-or-storage.md)、[状态与错误](contracts/state-and-errors.md) 和[参考架构](reference-architecture.md)定义后续实现边界。
