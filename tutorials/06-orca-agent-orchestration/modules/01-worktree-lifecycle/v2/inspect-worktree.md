# Worktree v2 — 读取 Git 状态

[模块路线](../plan.md) · [v1 创建 checkout](../v1/create-worktree.md)

worker-1 已经有自己的 checkout。Coordinator 在处理工作区前需要知道当前 HEAD、未提交内容和 ignored 文件。Orca 的 `WorkspaceCleanupCandidate.git` 同时记录读到的值和 `checkedAt`，Git 读失败时可以表达证据缺失。练习将失败表示成 `unknown`，并对每次调用重新读取状态。 [源码字段](../../../../../sources/orca/src/shared/workspace-cleanup/workspace-cleanup.ts)

## 1. 追加状态读取

在 `workspace.ts` 末尾追加以下完整类型和函数。输入来自 v1 的 `Workspace`，输出由后续 cleanup 使用。Git 的 ignored 条目在 porcelain 中以 `!!` 开头。禁用 rename 汇总并使用 NUL 分隔，使文件名中的空格或换行保持在单个条目内。

~~~ts file=workspace.ts append
export type WorkspaceStatus =
  | { kind: 'checked'; head: string; dirty: boolean; ignored: boolean }
  | { kind: 'unknown'; reason: string }

export function inspectWorkspace(workspace: Workspace): WorkspaceStatus {
  try {
    const head = git(workspace.path, 'rev-parse', 'HEAD').trim()
    const entries = git(workspace.path, 'status', '--porcelain=v1', '-z',
      '--untracked-files=all', '--ignored=matching', '--no-renames').split('\0').filter(Boolean)
    return {
      kind: 'checked',
      head,
      dirty: entries.some((entry) => !entry.startsWith('!! ')),
      ignored: entries.some((entry) => entry.startsWith('!! '))
    }
  } catch (error) {
    return { kind: 'unknown', reason: error instanceof Error ? error.message : String(error) }
  }
}
~~~

## 2. 验证干净、脏和未知

在 `ORCA_LAB` 执行。用单独的 checkout 做测试，回放完毕后只移除测试自己写出的文件。输出应为 `workspace status verified`。

~~~bash check
node --input-type=module <<'NODE'
import assert from 'node:assert/strict'
import { rmSync, writeFileSync } from 'node:fs'
import { join } from 'node:path'
import { createWorkspace, inspectWorkspace } from './workspace.ts'

const workspace = await createWorkspace({
  repo: 'repo', repoId: 'demo', path: 'status-worker', branch: 'status-worker'
})
const clean = inspectWorkspace(workspace)
assert.deepEqual(clean, { kind: 'checked', head: workspace.base, dirty: false, ignored: false })
const draft = join(workspace.path, 'draft.txt')
try {
  writeFileSync(draft, 'unfinished')
  assert.equal(inspectWorkspace(workspace).dirty, true)
  assert.equal(inspectWorkspace({ ...workspace, path: join(workspace.path, 'missing') }).kind, 'unknown')
} finally {
  rmSync(draft)
}
console.log('workspace status verified')
NODE
~~~
