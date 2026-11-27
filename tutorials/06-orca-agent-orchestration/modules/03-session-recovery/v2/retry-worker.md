# Recovery v2 — 确认停止后重试同一 Task

[模块路线](../plan.md) · [v1 存活证据](../v1/observe-session.md)

停止请求可能返回未知。Orca 要求通过确切 Dispatch ID 检查，重试指向同一 Task 的最新失败尝试。练习用终态退出证据和调用方提供的修复确认约束重试。旧尝试继续留在 Task 的尝试记录中。 [恢复指南](../../../../../sources/orca/skill-guides/orchestration/references/recovery-and-cleanup.md) · [远程停止结果](../../../../../sources/orca/src/main/runtime/rpc/methods/orchestration/worker/stop/worker-stop-remote-receipt.ts)

## 1. 记录停止结果并分配新尝试

在 `recovery.ts` 的首个 import 前加入 `import { randomUUID } from 'node:crypto'`。将现有 type import 移到新建文件时的位置即可。为了顺序应用，下方代码作为完整文件替换 v1 的 `recovery.ts`。

~~~ts file=recovery.ts
import { randomUUID } from 'node:crypto'
import { currentDispatch } from './dispatch.ts'
import type { Dispatch, Task } from './dispatch.ts'
import type { Workspace } from './workspace.ts'

export function observeLiveness(
  dispatch: Dispatch,
  hostId: string,
  liveness: Dispatch['liveness']
): void {
  if (hostId !== dispatch.workspace.hostId) {
    throw new Error('observation came from another host')
  }
  if (dispatch.liveness !== 'exited') {
    dispatch.liveness = liveness
  }
}

export function recordStop(dispatch: Dispatch, hostId: string, confirmed: boolean): void {
  observeLiveness(dispatch, hostId, confirmed ? 'exited' : 'unknown')
  if (confirmed && !dispatch.outcome) {
    dispatch.outcome = 'stopped'
  }
}

export function retryDispatch(task: Task, input: {
  worker: string
  predecessorId: string
  workspace: Workspace
  causeResolved: boolean
}): Dispatch {
  const previous = currentDispatch(task, input.worker)
  if (!previous || previous.id !== input.predecessorId) {
    throw new Error('retry must name the current attempt')
  }
  if (!input.causeResolved || previous.liveness !== 'exited' ||
      (previous.outcome !== 'failed' && previous.outcome !== 'stopped')) {
    throw new Error('retry requires settled predecessor and resolved cause')
  }
  const next: Dispatch = {
    id: randomUUID(), taskId: task.id, worker: previous.worker, workspace: input.workspace,
    phase: 'workspace-created', liveness: 'unknown'
  }
  task.dispatches.push(next)
  return next
}
~~~

`recordStop` 消费一次已收到的停止结果。它的调用者拥有实际 stop 任务和响应来源。生产 Orca 还保留 mutation receipt 与重试链熔断记录。练习只演示一次重新派发的前置条件。

## 2. 验证迟到报告与新尝试隔离

在 `ORCA_LAB` 执行，预期输出 `retry fencing verified`。

~~~bash check
node --input-type=module <<'NODE'
import assert from 'node:assert/strict'
import { createWorkspace } from './workspace.ts'
import { startDispatch, observeStartup, settleDispatch } from './dispatch.ts'
import { recordStop, retryDispatch } from './recovery.ts'

const workspace = await createWorkspace({
  repo: 'repo', repoId: 'demo', path: 'retry-worker', branch: 'retry-worker'
})
const task = { id: 'task-5', runId: 'run-1', spec: 'Recover test run', dispatches: [] }
const old = startDispatch(task, 'worker', workspace)
observeStartup(old, { phase: 'session-ready', sessionId: 's-old' })
observeStartup(old, { phase: 'turn-started', sessionId: 's-old' })
const retry = { worker: old.worker, predecessorId: old.id, workspace, causeResolved: true }
recordStop(old, 'local', false)
assert.throws(() => retryDispatch(task, retry), /settled predecessor/)
recordStop(old, 'local', true)
assert.throws(() => retryDispatch(task, { ...retry, causeResolved: false }), /resolved cause/)
const next = retryDispatch(task, retry)
assert.notEqual(next.id, old.id)
assert.equal(next.taskId, old.taskId)
assert.throws(() => settleDispatch(task, {
  worker: old.worker, dispatchId: old.id, sessionId: 's-old',
  outcome: 'succeeded', head: workspace.base, checksPassed: true
}), /current dispatch/)
assert.equal(next.outcome, undefined)
console.log('retry fencing verified')
NODE
~~~
