# Review v2 — 用所选 Host 定位 Checkout

[模块路线](../plan.md) · [v1 Review 证据](../v1/review-evidence.md)

两台机器可以报出同一个 `repoId` 和路径。Orca 的 CLI 先选 host，再使用该 host 返回的 project setup 与 repo selector。练习把这条规则应用在已注册 workspace 的选择上，调用方必须提供 `projectId`、`hostId` 和 workspace identity。 [Placement 源码说明](../../../../../sources/orca/skill-guides/orchestration/references/placement-and-remote.md) · [Project target](../../../../../sources/orca/src/cli/workspaces/worktree-project-target.ts)

## 1. 从已登记记录中选择

创建完整文件 `placement.ts`。候选项由调用方从各 host 的目录读取。返回带 host 的完整 `Workspace`，供 v3 读取证据时使用。

~~~ts file=placement.ts
import type { Workspace } from './workspace.ts'

export function selectWorkspace(
  catalog: readonly { projectId: string; workspace: Workspace }[],
  target: { projectId: string; hostId: string; workspaceKey: string }
): Workspace {
  const matches = catalog.filter(({ projectId, workspace }) =>
    projectId === target.projectId &&
    workspace.hostId === target.hostId &&
    workspace.identity.executionHostId === target.hostId &&
    workspace.identity.key === target.workspaceKey
  )
  if (matches.length !== 1) {
    throw new Error('workspace selector must resolve on the chosen host')
  }
  return matches[0].workspace
}
~~~

## 2. 验证相同路径在两个 Host 上的归属

在 `ORCA_LAB` 执行。远程目录行是合成测试输入。预期输出 `host selection verified`。

~~~bash check
node --input-type=module <<'NODE'
import assert from 'node:assert/strict'
import { createWorkspace } from './workspace.ts'
import { selectWorkspace } from './placement.ts'
import { createWorktreeIdentity } from './src/shared/worktree/identity.ts'

const local = await createWorkspace({
  repo: 'repo', repoId: 'demo', path: 'placement-worker', branch: 'placement-worker'
})
const remote = {
  ...local, hostId: 'paired-1',
  identity: createWorktreeIdentity({
    worktreeId: local.id, instanceId: local.identity.instanceId, executionHostId: 'paired-1'
  })
}
const catalog = [local, remote].map((workspace) => ({ projectId: 'demo-project', workspace }))
const target = { projectId: 'demo-project', hostId: 'paired-1', workspaceKey: remote.identity.key }
assert.equal(selectWorkspace(catalog, target), remote)
assert.throws(() => selectWorkspace(catalog, { ...target, hostId: 'local' }), /chosen host/)
assert.throws(() => selectWorkspace(catalog, { ...target, projectId: 'other' }), /chosen host/)
console.log('host selection verified')
NODE
~~~
