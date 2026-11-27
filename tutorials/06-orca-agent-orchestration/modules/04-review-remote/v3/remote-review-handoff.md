# Review v3 — 远程结果未知时保留工作

[模块路线](../plan.md) · [v2 Host 选择](../v2/select-execution-host.md)

Coordinator 查一个远程 worker 的 Review 输入时，断线只能证明本次查询未成功。Orca 在远程 worker view 中保留 Dispatch 和可达性线索。练习允许上层用一个读请求提供已选 host 的证据，并把人工审批与 cleanup 的所需证据分别交还给 Coordinator。 [远程 Worker view](../../../../../sources/orca/src/cli/orchestration/workers/worker-view-results.ts) · [Orca teardown](../../../../../sources/orca/skill-guides/orchestration/references/worker-teardown.md)

## 1. 从已绑定的 Host 读取

在 `review.ts` 末尾追加以下类型和函数。生产环境的读取边界是所选 host 的 RPC。此处 `read` 是测试提供的异步读取，约定用 `unreachable` 表示本次读失败。成功返回时仍校验 host、workspace 和 commit。即使具备评审材料，返回状态也只推进到 `review-required`。

~~~ts file=review.ts append
export type HostReviewRead =
  | { state: 'reachable'; hostId: string; evidence: ReviewEvidence }
  | { state: 'unreachable' }

export async function handOffReview(
  dispatch: Dispatch,
  read: (hostId: string, workspaceKey: string) => Promise<HostReviewRead>
) {
  const { hostId, identity } = dispatch.workspace
  const readResult = await read(hostId, identity.key)
  if (readResult.state === 'unreachable') {
    return { state: 'unverified', cleanup: 'keep', reason: 'host-unreachable' } as const
  }
  if (readResult.hostId !== hostId) {
    throw new Error('review came from another host')
  }
  return {
    state: 'review-required',
    review: reviewInput(dispatch, readResult.evidence)
  } as const
}
~~~

## 2. 验证断线和重新读取

在 `ORCA_LAB` 执行。测试模拟 remote RPC，返回的 diff 来自演示仓库上的真实 Git。预期输出 `host review handoff verified`。

~~~bash check
node --input-type=module <<'NODE'
import assert from 'node:assert/strict'
import { createWorkspace } from './workspace.ts'
import { startDispatch, observeStartup, settleDispatch } from './dispatch.ts'
import { handOffReview, readLocalReview } from './review.ts'
import { selectWorkspace } from './placement.ts'
import { createWorktreeIdentity } from './src/shared/worktree/identity.ts'

const local = await createWorkspace({
  repo: 'repo', repoId: 'demo', path: 'remote-worker', branch: 'remote-worker'
})
const hostId = 'paired-1'
const remote = {
  ...local, hostId,
  identity: createWorktreeIdentity({
    worktreeId: local.id, executionHostId: hostId, instanceId: local.identity.instanceId
  })
}
const projectId = 'demo-project'
const selected = selectWorkspace([{ projectId, workspace: remote }],
  { projectId, hostId, workspaceKey: remote.identity.key })
const task = { id: 'task-8', runId: 'run-1', spec: 'Review on paired host', dispatches: [] }
const attempt = startDispatch(task, 'worker', selected)
observeStartup(attempt, { phase: 'session-ready', sessionId: 's1' })
observeStartup(attempt, { phase: 'turn-started', sessionId: 's1' })
const evidence = { ...readLocalReview(local), workspaceKey: selected.identity.key }
settleDispatch(task, {
  worker: attempt.worker, dispatchId: attempt.id, sessionId: 's1',
  outcome: 'succeeded', head: evidence.head, checksPassed: true
})
assert.deepEqual(await handOffReview(attempt, async () => ({ state: 'unreachable' })), {
  state: 'unverified', cleanup: 'keep', reason: 'host-unreachable'
})
await assert.rejects(handOffReview(attempt, async () => ({
  state: 'reachable', hostId: 'local', evidence
})), /another host/)
const handoff = await handOffReview(attempt, async (requestedHost, workspaceKey) => {
  assert.equal(requestedHost, hostId)
  assert.equal(workspaceKey, remote.identity.key)
  return { state: 'reachable', hostId, evidence }
})
assert.equal(handoff.state, 'review-required')
assert.equal(handoff.review.dispatchId, attempt.id)
console.log('host review handoff verified')
NODE
~~~
