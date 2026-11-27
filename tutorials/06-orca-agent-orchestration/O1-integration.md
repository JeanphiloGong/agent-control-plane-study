# Orca O1 — 从 Worker 派发到 Review

[Orca 路线](plan.md) · [Review v3](modules/04-review-remote/v3/remote-review-handoff.md)

O1 使用各模块最后一个版本形成的 `workspace.ts`、`dispatch.ts`、`recovery.ts` 和 `review.ts`。一次 README 任务进入一个独立 worktree，首次尝试的停止结果未知。确认退出和修复原因后，同一 Task 使用新的 Dispatch ID 继续。新尝试完成检查，Reviewer 收到绑定提交的 diff。Worktree 在等待审批期间保留。

在 `ORCA_LAB` 执行。预期输出 `O1 verified: retry, review, preserve`。

~~~bash check
node --input-type=module <<'NODE'
import assert from 'node:assert/strict'
import { existsSync, readFileSync, writeFileSync } from 'node:fs'
import { join } from 'node:path'
import { createWorkspace, git } from './workspace.ts'
import { startDispatch, dispatchReceipt, observeStartup, settleDispatch, taskOutcome } from './dispatch.ts'
import { recordStop, retryDispatch, observeLiveness, recoveryCleanup } from './recovery.ts'
import { readLocalReview, handOffReview } from './review.ts'

const workspace = await createWorkspace({
  repo: 'repo', repoId: 'demo', path: 'o1-worker', branch: 'o1-worker'
})
const task = { id: 'o1-task', runId: 'o1-run', spec: 'Write README and check its title', dispatches: [] }
const first = startDispatch(task, 'builder', workspace)
const firstReceipt = dispatchReceipt(first)
assert.equal(firstReceipt.phase, 'workspace-created')
assert.equal(firstReceipt.sessionId, null)
recordStop(first, 'local', false)
assert.equal(recoveryCleanup(task, workspace, 'main').reason, 'worker-exit-unverified')
assert.throws(() => retryDispatch(task, {
  worker: 'builder', predecessorId: first.id, workspace, causeResolved: true
}), /settled predecessor/)
recordStop(first, 'local', true)
const retry = retryDispatch(task, {
  worker: 'builder', predecessorId: first.id, workspace, causeResolved: true
})
assert.notEqual(first.id, retry.id)
observeStartup(retry, { phase: 'session-ready', sessionId: 'o1-session' })
assert.equal(dispatchReceipt(retry).phase, 'session-ready')
observeStartup(retry, { phase: 'turn-started', sessionId: 'o1-session' })
assert.equal(dispatchReceipt(retry).phase, 'turn-started')
const path = join(workspace.path, 'README.md')
writeFileSync(path, '# Orca exercise\n')
git(workspace.path, 'add', '--', 'README.md')
git(workspace.path, '-c', 'user.name=Tutorial', '-c', 'user.email=tutorial@example.test',
  'commit', '-qm', 'add README')
const head = git(workspace.path, 'rev-parse', 'HEAD').trim()
const checksPassed = readFileSync(path, 'utf8').startsWith('# Orca exercise\n')
assert.equal(checksPassed, true)
settleDispatch(task, {
  worker: retry.worker, dispatchId: retry.id, sessionId: 'o1-session',
  outcome: 'succeeded', head, checksPassed
})
observeLiveness(retry, 'local', 'exited')
assert.equal(taskOutcome(task), 'succeeded')

const handoff = await handOffReview(retry, async (hostId) => ({
  state: 'reachable', hostId, evidence: readLocalReview(workspace)
}))
assert.equal(handoff.state, 'review-required')
assert.equal(handoff.review.head, head)
assert.match(handoff.review.diff, /\+# Orca exercise/)
assert.equal(recoveryCleanup(task, workspace, 'main').reason, 'unlanded')
assert.equal(existsSync(path), true)
console.log('O1 verified: retry, review, preserve')
NODE
~~~

验收使用真实 Git checkout、commit、diff 和一次内容检查。Worker 的 session、停止和退出取自测试观测，生产 CLI、Provider 进程和远程传输属于 Orca 源码对照范围。远程断线示例可在 Review v3 单独回放。 [真实 Coordinator 操作流程](../../sources/orca/skill-guides/orchestration/references/coordinator-loop.md)
