# Review v1 — 交给 Reviewer 的是哪个 Commit

[模块路线](../plan.md) · [Recovery v3](../../03-session-recovery/v3/preserve-work.md)

在 worker 上检查通过时，工作区的 HEAD 是 A。若后来切到 B，检查结论仍属于 A。Review 必须携带 Dispatch ID、workspace identity、base、head 和本次 diff。Orca 读取 branch diff 时也使用明确的 `headOid` 和 `mergeBase`。练习把创建时固定的 base commit 用作 diff 起点。 [Orca branch diff](../../../../../sources/orca/src/main/git/source-control/comparisons/branch-diff.ts)

## 1. 在所属工作区生成提交证据

创建完整文件 `review.ts`。`ReviewEvidence` 由执行 host 返回给 Coordinator，说明它读取的是哪一个 checkout、哪两个提交。Coordinator 再将它与 worker 报告绑定。`readLocalReview` 接收本机的 workspace，用真实 Git 读取状态和 diff。

~~~ts file=review.ts
import { git, inspectWorkspace } from './workspace.ts'
import type { Workspace } from './workspace.ts'
import type { Dispatch } from './dispatch.ts'

export type ReviewEvidence = {
  workspaceKey: string
  base: string
  head: string
  diff: string
}

export function readLocalReview(workspace: Workspace): ReviewEvidence {
  if (workspace.hostId !== 'local') {
    throw new Error('checkout belongs to another host')
  }
  const status = inspectWorkspace(workspace)
  if (status.kind !== 'checked' || status.dirty) {
    throw new Error('review requires a checked, clean worktree')
  }
  return {
    workspaceKey: workspace.identity.key,
    base: workspace.base,
    head: status.head,
    diff: git(workspace.path, 'diff', '--no-ext-diff', '--no-textconv',
      workspace.base, status.head, '--')
  }
}

export function reviewInput(dispatch: Dispatch, evidence: ReviewEvidence) {
  if (dispatch.outcome !== 'succeeded' || dispatch.report?.checksPassed !== true) {
    throw new Error('review requires a passing worker report')
  }
  if (evidence.workspaceKey !== dispatch.workspace.identity.key ||
      evidence.base !== dispatch.workspace.base || evidence.head !== dispatch.report.head) {
    throw new Error('review evidence does not match checked commits and workspace')
  }
  return { ...evidence, dispatchId: dispatch.id, taskId: dispatch.taskId }
}
~~~

## 2. 验证 diff 与检查提交的绑定

在 `ORCA_LAB` 执行，预期输出 `review input verified`。

~~~bash check
node --input-type=module <<'NODE'
import assert from 'node:assert/strict'
import { writeFileSync } from 'node:fs'
import { join } from 'node:path'
import { createWorkspace, git } from './workspace.ts'
import { startDispatch, observeStartup, settleDispatch } from './dispatch.ts'
import { readLocalReview, reviewInput } from './review.ts'

const w = await createWorkspace({ repo: 'repo', repoId: 'demo', path: 'review-worker', branch: 'review-worker' })
writeFileSync(join(w.path, 'README.md'), 'Review me\n')
git(w.path, 'add', '--', 'README.md')
git(w.path, '-c', 'user.name=Tutorial', '-c', 'user.email=tutorial@example.test', 'commit', '-qm', 'document')
const task = { id: 'task-7', runId: 'run-1', spec: 'Add README', dispatches: [] }
const attempt = startDispatch(task, 'worker', w)
observeStartup(attempt, { phase: 'session-ready', sessionId: 's1' })
observeStartup(attempt, { phase: 'turn-started', sessionId: 's1' })
const evidence = readLocalReview(w)
settleDispatch(task, {
  worker: 'worker', dispatchId: attempt.id, sessionId: 's1',
  outcome: 'succeeded', head: evidence.head, checksPassed: true
})
assert.match(reviewInput(attempt, evidence).diff, /\+Review me/)
git(w.path, '-c', 'user.name=Tutorial', '-c', 'user.email=tutorial@example.test',
  'commit', '--allow-empty', '-qm', 'head changed')
assert.throws(() => reviewInput(attempt, readLocalReview(w)), /does not match/)
console.log('review input verified')
NODE
~~~
