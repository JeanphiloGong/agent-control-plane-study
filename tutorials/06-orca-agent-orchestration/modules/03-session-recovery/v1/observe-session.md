# Recovery v1 — 记录 Worker 的存活证据

[模块路线](../plan.md) · [Worker v3](../../02-orchestration-worker/v3/settle-task.md)

一个已开始的 worker 长时间没有响应。Orca 的 worker view 分别给出 Dispatch、阶段和 liveness，恢复流程用这些证据查明 worker 是否退出。练习接受来自实际执行 host 的确切观测。连接不可用时，调用方以 `unknown` 表达。 [Worker view](../../../../../sources/orca/src/cli/orchestration/workers/worker-view-results.ts) · [Session liveness](../../../../../sources/orca/src/main/runtime/orchestration/workers/lifecycle/session-worker-liveness.ts)

## 1. 接收所属 host 的观测

创建完整文件 `recovery.ts`。这里使用同一执行 host 按序提供的观测，`exited` 对本次 Dispatch 是终态。调度调用者只传入已定位的 Dispatch。

~~~ts file=recovery.ts
import type { Dispatch } from './dispatch.ts'

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
~~~

## 2. 验证未知与退出各自独立

在 `ORCA_LAB` 执行，预期输出 `liveness verified`。

~~~bash check
node --input-type=module <<'NODE'
import assert from 'node:assert/strict'
import { createWorkspace } from './workspace.ts'
import { startDispatch, observeStartup } from './dispatch.ts'
import { observeLiveness } from './recovery.ts'

const workspace = await createWorkspace({
  repo: 'repo', repoId: 'demo', path: 'liveness-worker', branch: 'liveness-worker'
})
const task = { id: 'task-4', runId: 'run-1', spec: 'Inspect progress', dispatches: [] }
const attempt = startDispatch(task, 'worker', workspace)
observeStartup(attempt, { phase: 'session-ready', sessionId: 's1' })
assert.throws(() => observeLiveness(attempt, 'other-host', 'exited'), /another host/)
observeLiveness(attempt, 'local', 'unknown')
assert.equal(attempt.liveness, 'unknown')
assert.equal(attempt.outcome, undefined)
observeLiveness(attempt, 'local', 'exited')
observeLiveness(attempt, 'local', 'unknown')
assert.equal(attempt.liveness, 'exited')
console.log('liveness verified')
NODE
~~~
