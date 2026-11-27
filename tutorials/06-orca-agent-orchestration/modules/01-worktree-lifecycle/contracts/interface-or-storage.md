# Orca Worktree / Workspace：接口与存储契约

## 外部入口

CLI 命令由 `src/cli/workspaces/worktree.ts:WORKTREE_HANDLERS` 注册：

- `worktree create` 接收 name、repo/parent selector、base branch、host、review link 和
  可选 startup agent。
- `worktree list` 返回 catalog projection，支持 repo 和 limit 过滤。
- `worktree show/current` 解析 selector 后返回一个 `RuntimeWorktreeRecord`。
- `worktree set` 更新用户 metadata 和 parent/workspace fields。
- `worktree rm` 先拒绝 current worktree，再进入 removal request 和 host-specific checks。

CLI 在请求前完成 flags 和 selector 校验。runtime RPC 的响应不是持久化 record 的直接替代，
`RuntimeWorktreeRecord` 是给调用方使用的查询 projection。

## 核心输入

```text
CreateWorktreeRequest
  repo target or inferred current repo
  parent worktree/workspace or no-parent
  execution host route
  name, base branch and optional review/task links

CleanupCandidate
  worktree identity and path
  reasons and blockers
  activity timestamp
  checked Git state and fingerprint
```

## 核心输出

- 创建成功后返回 `RuntimeWorktreeCreateResult`，其中的 worktree 需要能定位 repo、path、
  host 和 identity。
- cleanup 扫描返回 `WorkspaceCleanupCandidate`，不是删除结果。
- `evaluateWorkspaceCleanup` 返回 `WorkspaceCleanupEvaluation`。
- removal settlement 另行返回成功、失败、leftover 或 still-registered 事实。

## 持久化规则

- `WorktreeMeta.instanceId` 和 `hostId` 共同参与稳定 identity。
- `WorktreeMeta.orcaCreatedAt`、`orcaCreationSource` 等 provenance 只能由 Orca-owned
  flow 写入，用户 metadata update 不得覆盖它们。
- `Worktree.id` 是 path-derived locator，可以变化。跨路径恢复使用 `identity.key` 和
  `priorWorktreeIds` 等迁移信息。
- cleanup dismissal 必须绑定 candidate fingerprint 和 classifier version，旧的扫描证据
  失效后不能继续隐藏新的状态。

## 禁止的耦合

- renderer 不直接调用 filesystem removal。
- cleanup evaluator 不读取 CLI flags，也不产生 Git 命令。
- base resolver 不把 lookup exception 转成 no-default-base。
- `WorktreeMeta` 不能单独证明 Git checkout 仍然存在。
- `WorktreeCleanupEvaluation.action = 'delete'` 不能被解释为删除已经成功。
