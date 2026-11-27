# Worker v3 — 结算 Dispatch 并汇总 Task

[模块路线](../plan.md) · [v2 Receipt](../v2/worker-readiness.md)

收到 `worker_done` 时，Orca 依据 Dispatch、Task 与报告内容确认结算。练习只接收与当前职责尝试、已启动 session 对应的报告。单次 Task 汇总使用每个职责最新的一次尝试，保留旧 Dispatch 用于追踪。这个汇总是本练习对多个 worker 的规则。 [Orca 报告与结算](../../../../../sources/orca/src/main/runtime/orchestration/db/dispatch-context/worker-report-settlement.ts) · [CLI 确认](../../../../../sources/orca/src/cli/orchestration/workers/orchestration-worker-settlement.ts)

## 1. 让结算更新对应的尝试

在 `dispatch.ts` 末尾追加以下代码。`report.head` 是 worker 执行检查时的 commit。`succeeded` 依据 worker 报告，Review 还会独立检查证据。

~~~ts file=dispatch.ts append
export function currentDispatch(task: Task, worker: string): Dispatch | undefined {
  return task.dispatches.findLast((dispatch) => dispatch.worker === worker)
}

export function settleDispatch(task: Task, input: {
  worker: string
  dispatchId: string
  sessionId: string
  outcome: 'succeeded' | 'failed'
  head: string
  checksPassed: boolean
}): void {
  const dispatch = currentDispatch(task, input.worker)
  if (!dispatch || dispatch.id !== input.dispatchId || dispatch.sessionId !== input.sessionId) {
    throw new Error('report does not match current dispatch and session')
  }
  if (dispatch.phase !== 'turn-started') {
    throw new Error('turn has not started')
  }
  if (dispatch.outcome) {
    if (dispatch.outcome === input.outcome && dispatch.report?.head === input.head &&
        dispatch.report?.checksPassed === input.checksPassed) {
      return
    }
    throw new Error('dispatch already settled')
  }
  dispatch.report = { head: input.head, checksPassed: input.checksPassed }
  dispatch.outcome = input.outcome
}

export function taskOutcome(task: Task): 'running' | 'succeeded' | 'failed' {
  const current = [...new Set(task.dispatches.map((entry) => entry.worker))]
    .map((worker) => currentDispatch(task, worker)!)
  if (!current.length || current.some((entry) => !entry.outcome)) {
    return 'running'
  }
  return current.every((entry) => entry.outcome === 'succeeded') ? 'succeeded' : 'failed'
}
~~~

## 2. 验证重复报告和两个职责的结算

在 `ORCA_LAB` 执行。两个职责用同一个示例 checkout 假设串行执行，生产并行写入使用独立 checkout。预期输出 `task settlement verified`。

~~~bash check
node --input-type=module <<'NODE'
import assert from 'node:assert/strict'
import { createWorkspace } from './workspace.ts'
import { startDispatch, observeStartup, settleDispatch, taskOutcome } from './dispatch.ts'

const workspace = await createWorkspace({
  repo: 'repo', repoId: 'demo', path: 'settlement-worker', branch: 'settlement-worker'
})
const task = { id: 'task-3', runId: 'run-1', spec: 'Implement and verify', dispatches: [] }
const attempts = ['builder', 'tester'].map((worker) => startDispatch(task, worker, workspace))
for (const attempt of attempts) {
  const sessionId = attempt.worker + '-session'
  observeStartup(attempt, { phase: 'session-ready', sessionId })
  observeStartup(attempt, { phase: 'turn-started', sessionId })
  const result = {
    worker: attempt.worker, dispatchId: attempt.id, sessionId,
    outcome: 'succeeded', head: workspace.base, checksPassed: true
  }
  settleDispatch(task, result)
  settleDispatch(task, result)
  assert.throws(() => settleDispatch(task, { ...result, head: 'other' }), /already settled/)
  assert.equal(taskOutcome(task), attempt.worker === 'builder' ? 'running' : 'succeeded')
}
console.log('task settlement verified')
NODE
~~~
