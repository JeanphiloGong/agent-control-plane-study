# Orca 01 Worktree / Workspace：V0 设计

[Orca 总览](../../V0-overview.md) · [模块计划](plan.md) · [参考架构](reference-architecture.md)

## 版本身份

- **模块**：`orca-worktree-lifecycle`
- **基线**：`sources/orca` @ `27300a14e0170258814b67b2f4159446c637f551`
- **模式**：backend / design
- **目标**：把一个项目目标解析成有主机归属、稳定 occupant identity 和可检查 cleanup 证据的 worktree。
- **V1 交接**：创建并绑定一个 worktree，验证项目、路径、host 和 instance identity 一致。

本版是设计交接，不声称已经实现 Orca 的 worktree 创建或 cleanup。后续实现应使用真实
CLI、runtime 和 Git host 入口，参考架构只提取可以独立验证的状态流。

## 真实场景

用户从当前项目创建一个新的 worktree，或者让 worker 在 `new-child` / `new-top-level`
位置工作。Orca 必须先决定目标 repo 和执行 host，再确定 base branch，创建 checkout，
记录 worktree metadata。之后 cleanup 扫描会读取 Git 状态、活动时间、Agent/terminal 事实
和保护 blocker，决定保留、删除或等待人工确认。

```text
CLI/runtime request
  → project/repo and execution-host selection
  → base ref resolution
  → Git worktree creation
  → Worktree + WorktreeMeta identity persistence
  → catalog observation and activity evidence
  → cleanup evaluation
  → keep / delete / archive decision
```

## 领域对象和所有权

| 对象 | 真实类型或符号 | 所有者 | 责任 |
| --- | --- | --- | --- |
| `GitWorktreeInfo` | `src/shared/worktree/types.ts:GitWorktreeInfo` | Git/catalog 读取侧 | 保存 path、HEAD、branch、main-worktree、lock 和 removal 事实 |
| `Worktree` | `src/shared/worktree/types.ts:Worktree` | Orca workspace projection | 将 Git 事实与 repo、project、host、display、activity 和 review 元数据组合成可见 workspace |
| `WorktreeMeta` | `src/shared/worktree/meta-types.ts:WorktreeMeta` | 持久化 metadata store | 保存 instance、host、creation provenance、base、branch protection 和 cleanup 所需信息 |
| `WorktreeIdentity` | `src/shared/worktree/identity.ts:createWorktreeIdentity` | identity boundary | 用 execution host 和 instance ID 识别一个 checkout occupant，允许 path 变化 |
| `WorkspaceCleanupCandidate` | `src/shared/workspace-cleanup/workspace-cleanup.ts` | cleanup scan | 收集 reasons、blockers、Git freshness、activity 和 fingerprint |
| `WorkspaceCleanupEvaluation` | `src/shared/workspace-cleanup/workspace-cleanup-evaluation.ts:evaluateWorkspaceCleanup` | cleanup policy | 给出 keep、delete 或 archive 以及可解释原因 |

`Worktree.id` 是 `repoId::path` 形式的可变定位符。`Worktree.identity.key` 才是跨路径
变化保留 occupant 身份的稳定键。`WorktreeMeta` 和 Git catalog 不是同一个事实源，前者
记录 Orca 的创建和用户元数据，后者确认当前 checkout 的 Git 事实，二者需要在投影时合并。

## 业务不变量

1. **创建先选边界再产生副作用**。显式 repo、parent workspace、execution host 和 startup
   参数必须在 `worktree.create` 之前解析，非法参数不能留下半成品 checkout。
2. **身份不依赖路径**。`canonicalWorktreeIdentity` 使用 `executionHostId` 和 `instanceId`，
   因此目录移动或 path-derived `worktreeId` 更新不能把旧 checkout 当成新 occupant。
3. **base ref 的失败和缺失不同**。默认分支查询失败时，只有已保存且仍可用的 repo base
   才能作为 fallback；查询成功但没有默认分支才返回 `null`，不能把 Git 错误当成没有 base。
4. **删除必须有注册和路径证据**。主 worktree、repo 根目录、空路径、home 根目录、包含其他
   registered worktree 的路径和未注册路径都不能直接删除。
5. **未知 Git 状态默认保留**。cleanup 没有 fresh `git.clean` 或 `checkedAt` 时只能产生
   `git-unknown` 保护，不能因为没有读到脏文件就推断 workspace 可删。
6. **cleanup 决策和删除执行分离**。`evaluateWorkspaceCleanup` 只产生 verdict，真正的
   removal 仍由 host-specific removal flow 执行并保留失败或 leftover 事实。

## 状态流和失败边界

### 创建

```text
requested selector
  → resolve repo / parent / host
  → resolveWorktreeCreateBase
  → create Git checkout
  → persist WorktreeMeta and identity
  → publish Worktree projection
```

`resolveWorktreeCreateBase` 的输入包括用户显式 base、repo 保存的 base、默认 base lookup
和 `isBaseUsable`。显式 base 优先，保存 base 只有在仍可用时才能覆盖新检测到的默认值。
lookup 抛错时不能静默返回 `null`。

### 删除和 cleanup

```text
catalog row + metadata + activity + Git scan
  → cleanup reasons
  → blocker precedence
  → policy and grace window
  → keep / delete / archive
  → removal authorization and settlement
```

`findRegisteredDeletableWorktree` 只接受当前 catalog 中的非主 worktree，并检查路径安全和
nested worktree。对于已经从 Git 注册表消失但目录仍在的 orphan，
`canSafelyRemoveOrphanedWorktreeDirectory` 要求 `.git` 与 owner admin entry 双向证明，且
execution host 的 home authority 已解析。任何未知或拒绝读取都返回不可安全清理。

cleanup evaluation 的保护优先于规则命中。`main-worktree` 和 `folder-repo` 永远不可入队，
活动 Agent、运行中的 terminal、未提交变更、unpushed commits、ignored files 和 unknown
Git status 都会把结果留在 `keep`。最近活动还有 grace window，避免刚使用过的 workspace
被时间规则立即清掉。

## 接口和存储边界

CLI `worktree create/list/show/set/rm` 通过 `src/cli/workspaces/worktree.ts` 进入 runtime
RPC。CLI 负责 flags、selector 和用户可见格式，host/runtime 负责 Git 操作、metadata 写入
和实际删除。cleanup 的扫描、分组和策略属于共享纯函数，不能从 renderer 直接删除目录。

V0 不决定数据库表或新的持久化格式。学习实现可以用内存 store 表达相同所有权，但必须
保留 `Worktree`、`WorktreeMeta`、`WorkspaceCleanupCandidate` 和 `WorkspaceCleanupEvaluation`
之间的字段边界。

## V1–V3 交接

- **V1 Worktree identity**：给定 repo、host、instance 和 base，创建一个可查询的 worktree，
  并证明同一路径复用不会覆盖不同 instance。
- **V2 Status evidence**：加入 Git clean、unpushed、ignored、terminal 和 active-agent 证据，
  检查 unknown status 进入保护路径。
- **V3 Cleanup verdict**：实现规则、grace window、blocker precedence、registered removal
  和 orphan proof，禁止把 verdict 直接当成删除成功。

## V0 验收

设计检查必须能回答以下问题：

- `Worktree.id` 改变时，哪个字段继续识别同一个 checkout occupant？
- 默认 base lookup 抛错和返回 `null` 时，创建流程分别如何处理？
- 为什么 `git.clean = null` 不能进入 `nothing-local`？
- 一个目录包含另一个 registered worktree 时，哪个边界拒绝删除？
- cleanup 产生 `delete` 后，哪个后续步骤才真正执行 filesystem/Git removal？

真实源码中的静态验收入口是：

- `src/main/workspaces/creation/worktree-create-base.test.ts`
- `src/main/workspaces/removal/worktree-removal-safety.test.ts`
- `src/shared/workspace-cleanup/workspace-cleanup-groups.test.ts`

这些测试验证具体纯函数和安全边界。它们不代表已经完成 V1 的新参考实现。V1 正文应从
本设计交接继续，并在独立 replay 中提供可运行的 identity 与 base-resolution 检查。
