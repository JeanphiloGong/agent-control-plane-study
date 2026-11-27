# Orca Worktree / Workspace：参考架构

## 学习实现目录

```text
reference/worktree/model.ts       # Project, Host, Worktree, WorktreeMeta, Identity
reference/worktree/base.ts        # explicit/persisted/default base resolution
reference/worktree/catalog.ts     # registered Git checkout projection
reference/worktree/removal.ts     # path, registration and nested-worktree guards
reference/cleanup/candidate.ts    # activity, Git and runtime evidence
reference/cleanup/evaluator.ts    # blocker precedence and policy verdict
reference/worktree.test.ts        # V1 identity and base-resolution checks
reference/cleanup.test.ts         # V2/V3 safety and unknown-evidence checks
```

V1 不需要复制 Orca 的 Electron UI、remote transport 或完整 Git implementation。它需要
把 identity、base resolution 和 removal authorization 作为可注入的纯边界建模，随后再把
真实 CLI/runtime 调用映射回来。

## 调用方向

```text
CLI selector
   ↓
CreateCoordinator ──→ BaseResolver ──→ GitWorktreePort
   │                       │                  │
   │                       └─ persisted meta  └─ catalog facts
   ↓
WorktreeProjection ──→ CleanupCandidate ──→ CleanupEvaluator
                                             ↓
                                      RemovalAuthorizer
```

`CreateCoordinator` 组合边界并决定何时产生副作用。`BaseResolver` 只负责 ref 选择。
`GitWorktreePort` 是真实 Git host 的端口，测试使用 fake 记录调用。`CleanupEvaluator` 只
返回 verdict，不调用 `rm`。`RemovalAuthorizer` 再使用注册、路径、home 和 orphan proof
执行删除前检查。

## 真实源码到参考实现

| Orca 源码 | 参考组件 | V1–V3 检查 |
| --- | --- | --- |
| `src/shared/worktree/types.ts:Worktree`, `GitWorktreeInfo` | `Worktree`, `GitCheckoutFact` | path、branch、main-worktree 和 host projection 可读 |
| `src/shared/worktree/meta-types.ts:WorktreeMeta` | `WorktreeMeta` | instance、provenance、base 和 branch protection 不混入 Git fact |
| `src/shared/worktree/identity.ts:createWorktreeIdentity` | `Identity.create` | path 变化不改变 occupant key |
| `src/main/workspaces/creation/worktree-create-base.ts:resolveWorktreeCreateBase` | `BaseResolver.resolve` | explicit、persisted、default、lookup error 的优先级可断言 |
| `src/main/workspaces/removal/worktree-removal-safety.ts` | `RemovalAuthorizer` | unregistered、main、nested、dangerous 和 orphan proof 被拒绝 |
| `src/shared/workspace-cleanup/workspace-cleanup-evaluation.ts:evaluateWorkspaceCleanup` | `CleanupEvaluator.evaluate` | unknown Git、blocker precedence、grace window 和 policy action 可断言 |
| `src/shared/workspace-cleanup/workspace-cleanup-groups.ts:getWorkspaceCleanupGroup` | `CleanupGroup` | cannot-delete 优先于 local-files，未检查状态不进入 nothing-local |

## V1 冻结检查

设计完成后，V1 正文必须提供一个最小可运行检查，至少断言：

```text
identity(repo/path-A, host-1, instance-1) != identity(repo/path-A, host-1, instance-2)
identity(repo/path-A, host-1, instance-1) == identity(repo/path-B, host-1, instance-1)
resolve(explicit-base) == explicit-base
resolve(lookup-error, usable-persisted-base) == persisted-base
resolve(lookup-error, no-persisted-base) raises base lookup error
```

这只证明 Worktree 生命周期的第一条边界。它不证明真实 Git worktree 创建、远程 host
连接、cleanup filesystem 删除或 Orca UI 已经运行。
