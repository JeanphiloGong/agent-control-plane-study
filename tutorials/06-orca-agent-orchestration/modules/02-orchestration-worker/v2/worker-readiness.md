# Worker v2 — 用 Receipt 表达实际启动阶段

[模块路线](../plan.md) · [v1 派发](../v1/dispatch-worker.md)

Git 已创建 checkout，Provider 可能还在准备 session。只有对应 session 的 turn 观测才能推进到 `turn-started`。Orca 的 worker start receipt 区分 stage 和 state，并在结果未知时使用独立状态。练习将这两个事实缩到顺序明确的 phase 和可读取的 receipt。 [真实 Receipt](../../../../../sources/orca/src/main/runtime/rpc/methods/orchestration/worker/start/worker-start-receipt.ts) · [CLI 投影](../../../../../sources/orca/src/cli/orchestration/workers/worker-start-agent-view.ts)

## 1. 写入观测并返回 Receipt

在 `dispatch.ts` 末尾追加下方完整代码。Runtime 传入一个已定位的 Dispatch 和 session 事件。相同事件可重复到达，较早的 session-ready 观测保留已经确认的 turn。

~~~ts file=dispatch.ts append
export function observeStartup(
  dispatch: Dispatch,
  event: { phase: 'session-ready' | 'turn-started'; sessionId: string }
): void {
  if (dispatch.outcome || dispatch.liveness === 'exited') {
    throw new Error('dispatch is settled or exited')
  }
  if (dispatch.sessionId && dispatch.sessionId !== event.sessionId) {
    throw new Error('session identity changed')
  }
  if (event.phase === 'session-ready') {
    if (dispatch.phase === 'workspace-created') {
      dispatch.sessionId = event.sessionId
      dispatch.phase = 'session-ready'
    }
    dispatch.liveness = 'live'
    return
  }
  if (dispatch.phase === 'workspace-created') {
    throw new Error('session is not ready')
  }
  dispatch.phase = 'turn-started'
  dispatch.liveness = 'live'
}

export function dispatchReceipt(dispatch: Dispatch) {
  return {
    dispatchId: dispatch.id,
    taskId: dispatch.taskId,
    workspaceKey: dispatch.workspace.identity.key,
    sessionId: dispatch.sessionId ?? null,
    phase: dispatch.phase,
    outcome: dispatch.outcome ?? null
  }
}
~~~

## 2. 验证阶段和晚到事件

在 `ORCA_LAB` 执行，预期输出 `receipt stages verified`。

~~~bash check
node --input-type=module <<'NODE'
import assert from 'node:assert/strict'
import { createWorkspace } from './workspace.ts'
import { startDispatch, observeStartup, dispatchReceipt } from './dispatch.ts'

const workspace = await createWorkspace({
  repo: 'repo', repoId: 'demo', path: 'receipt-worker', branch: 'receipt-worker'
})
const task = { id: 'task-2', runId: 'run-1', spec: 'Run checks', dispatches: [] }
const attempt = startDispatch(task, 'tester', workspace)
const initial = dispatchReceipt(attempt)
assert.equal(initial.phase, 'workspace-created')
assert.throws(() => observeStartup(attempt, { phase: 'turn-started', sessionId: 's1' }), /not ready/)
observeStartup(attempt, { phase: 'session-ready', sessionId: 's1' })
assert.equal(dispatchReceipt(attempt).phase, 'session-ready')
observeStartup(attempt, { phase: 'turn-started', sessionId: 's1' })
observeStartup(attempt, { phase: 'session-ready', sessionId: 's1' })
assert.equal(dispatchReceipt(attempt).phase, 'turn-started')
assert.throws(() => observeStartup(attempt, { phase: 'turn-started', sessionId: 's2' }), /identity/)
assert.equal(initial.sessionId, null)
console.log('receipt stages verified')
NODE
~~~
