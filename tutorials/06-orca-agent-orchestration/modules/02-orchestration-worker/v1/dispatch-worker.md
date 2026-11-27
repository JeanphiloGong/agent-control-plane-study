# Worker v1 — 将 Task 绑定到 Dispatch

[模块路线](../plan.md) · [Worktree v3](../../01-worktree-lifecycle/v3/cleanup-preview.md)

Coordinator 有一条具体任务和已创建的 Workspace。Task 保存任务说明和所属 Run，Dispatch 记录某个 worker 的这次尝试及其工作区。一个 Task 可以让几个 worker 分别处理实现、测试等职责。练习以 `worker` 字符串标识职责，每个职责先注册一次尝试。

Orca 的 CLI 负责解析 `worker-start` 参数，`startWorker` 在 runtime 内解析 run、task 和 placement 并组织启动。练习从成功解析后的已登记 Task 与 Workspace 开始。 [CLI](../../../../../sources/orca/src/cli/orchestration/workers/worker-launch-handler.ts) · [Runtime](../../../../../sources/orca/src/main/runtime/rpc/methods/orchestration/worker/worker-start-method.ts)

## 1. 记录一次派发

在 `ORCA_LAB` 创建完整文件 `dispatch.ts`。调用者持有 Task。成功派发只在 Task 追加一条记录，初始 `phase` 是 `workspace-created`。后续步骤会填充 session、liveness 和 report。

~~~ts file=dispatch.ts
import { randomUUID } from 'node:crypto'
import type { Workspace } from './workspace.ts'

export type Dispatch = {
  id: string
  taskId: string
  worker: string
  workspace: Workspace
  phase: 'workspace-created' | 'session-ready' | 'turn-started'
  sessionId?: string
  liveness: 'unknown' | 'live' | 'exited'
  outcome?: 'succeeded' | 'failed' | 'stopped'
  report?: { head: string; checksPassed: boolean }
}

export type Task = {
  id: string
  runId: string
  spec: string
  dispatches: Dispatch[]
}

export function startDispatch(task: Task, worker: string, workspace: Workspace): Dispatch {
  if (task.dispatches.some((dispatch) => dispatch.worker === worker)) {
    throw new Error('worker already assigned')
  }
  const dispatch: Dispatch = {
    id: randomUUID(),
    taskId: task.id,
    worker,
    workspace,
    phase: 'workspace-created',
    liveness: 'unknown'
  }
  task.dispatches.push(dispatch)
  return dispatch
}
~~~

## 2. 验证 task 与 worktree 的绑定

在 `ORCA_LAB` 执行，预期输出 `dispatch recorded`。

~~~bash check
node --input-type=module <<'NODE'
import assert from 'node:assert/strict'
import { createWorkspace } from './workspace.ts'
import { startDispatch } from './dispatch.ts'

const workspace = await createWorkspace({
  repo: 'repo', repoId: 'demo', path: 'dispatch-worker', branch: 'dispatch-worker'
})
const task = { id: 'task-1', runId: 'run-1', spec: 'Implement README', dispatches: [] }
const attempt = startDispatch(task, 'implementer', workspace)
assert.equal(attempt.taskId, task.id)
assert.equal(attempt.workspace.identity.key, workspace.identity.key)
assert.equal(attempt.phase, 'workspace-created')
assert.throws(() => startDispatch(task, 'implementer', workspace), /already assigned/)
assert.equal(task.dispatches.length, 1)
console.log('dispatch recorded')
NODE
~~~
