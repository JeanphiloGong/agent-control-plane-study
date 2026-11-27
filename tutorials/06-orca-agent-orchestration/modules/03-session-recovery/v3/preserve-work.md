# Recovery v3 — 将存活证据交给 Cleanup

[模块路线](../plan.md) · [v2 重试](../v2/retry-worker.md) · [Worktree v3 预览](../../01-worktree-lifecycle/v3/cleanup-preview.md)

同一个 workspace 可能曾经被多个 Dispatch 使用。准备清理时按稳定 workspace identity 查找该目录相关的尝试。只要还有一次尝试在运行或退出不可确认，工作区就继续保留。Orca 还区分 worker release、进程活动和工作是否已合入。 [Worker release](../../../../../sources/orca/src/main/runtime/rpc/methods/orchestration/worker/worker-release.ts) · [清理候选证据](../../../../../sources/orca/src/shared/workspace-cleanup/workspace-cleanup.ts)

## 1. 使用相同的预览入口

将下方代码追加到 `recovery.ts` 末尾。这里的 `task` 是该练习工作区的所有 Dispatch 的记录来源。真实全局 cleanup 的范围由执行主机扫描提供。

~~~ts file=recovery.ts append
import { previewCleanup } from './cleanup.ts'

export function recoveryCleanup(task: Task, workspace: Workspace, integrationRef: string) {
  const attempts = task.dispatches.filter((dispatch) =>
    dispatch.workspace.identity.key === workspace.identity.key)
  if (attempts.some((dispatch) => dispatch.liveness !== 'exited')) {
    return { action: 'keep', reason: 'worker-exit-unverified' } as const
  }
  return previewCleanup(workspace, integrationRef, false)
}
~~~

## 2. 验证工作区保留原因可区分

在 `ORCA_LAB` 执行，预期输出 `recovery cleanup verified`。

~~~bash check
node --input-type=module <<'NODE'
import assert from 'node:assert/strict'
import { writeFileSync } from 'node:fs'
import { join } from 'node:path'
import { createWorkspace } from './workspace.ts'
import { startDispatch } from './dispatch.ts'
import { recordStop, recoveryCleanup } from './recovery.ts'

const workspace = await createWorkspace({
  repo: 'repo', repoId: 'demo', path: 'preserve-worker', branch: 'preserve-worker'
})
const task = { id: 'task-6', runId: 'run-1', spec: 'Preserve draft', dispatches: [] }
const attempt = startDispatch(task, 'worker', workspace)
assert.equal(recoveryCleanup(task, workspace, 'main').reason, 'worker-exit-unverified')
recordStop(attempt, 'local', true)
writeFileSync(join(workspace.path, 'draft.txt'), 'work in progress')
assert.equal(recoveryCleanup(task, workspace, 'main').reason, 'local-files')
console.log('recovery cleanup verified')
NODE
~~~
