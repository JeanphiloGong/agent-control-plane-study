# Worktree v3 — 预览 cleanup

[模块路线](../plan.md) · [v2 Git 状态](../v2/inspect-worktree.md)

一个干净的 worktree 仍可能有尚未交付的 commit。使用新鲜 Git 读数，同时检查 HEAD 是否可从批准的集成 ref 到达。Orca 对应路径先计算策略和保护，再由 removal flow 重查目标身份、活动及 Git 证据。这里冻结只读预览，成功结果是 `eligible`。 [策略](../../../../../sources/orca/src/shared/workspace-cleanup/workspace-cleanup-evaluation.ts) · [执行检查](../../../../../sources/orca/src/main/workspaces/cleanup/workspace-cleanup-removal-step.ts)

## 1. 计算预览

创建完整文件 `cleanup.ts`。输入的 `workspace` 来自创建者保存的记录，`active` 来自调用方当前观测。本地练习用普通 Git ancestry 证明已集成，Orca 的生产检查还可使用 hosted review 与 squash/rebase 证据。

~~~ts file=cleanup.ts
import { git, inspectWorkspace } from './workspace.ts'
import type { Workspace } from './workspace.ts'

export function previewCleanup(workspace: Workspace, integrationRef: string, active: boolean):
  { action: 'eligible' } | { action: 'keep'; reason: string } {
  if (active) {
    return { action: 'keep', reason: 'active' }
  }
  const status = inspectWorkspace(workspace)
  if (status.kind === 'unknown') {
    return { action: 'keep', reason: 'git-unknown' }
  }
  if (status.dirty || status.ignored) {
    return { action: 'keep', reason: 'local-files' }
  }
  try {
    const integrated = git(workspace.path, 'rev-parse', '--verify', '--end-of-options',
      integrationRef + '^{commit}').trim()
    const common = git(workspace.path, 'merge-base', status.head, integrated).trim()
    return common === status.head
      ? { action: 'eligible' }
      : { action: 'keep', reason: 'unlanded' }
  } catch {
    return { action: 'keep', reason: 'git-unknown' }
  }
}
~~~

## 2. 验证 clean checkout 也能触发保留

在 `ORCA_LAB` 执行，预期输出 `cleanup preview verified`。

~~~bash check
node --input-type=module <<'NODE'
import assert from 'node:assert/strict'
import { existsSync } from 'node:fs'
import { createWorkspace, git } from './workspace.ts'
import { previewCleanup } from './cleanup.ts'

const w = await createWorkspace({ repo: 'repo', repoId: 'demo', path: 'preview-worker', branch: 'preview-worker' })
const base = 'refs/heads/main'
assert.equal(previewCleanup(w, base, true).reason, 'active')
assert.equal(previewCleanup(w, base, false).action, 'eligible')
git(w.path, '-c', 'user.name=Tutorial', '-c', 'user.email=tutorial@example.test',
  'commit', '--allow-empty', '-qm', 'unlanded')
assert.equal(previewCleanup(w, base, false).reason, 'unlanded')
assert.equal(previewCleanup(w, 'missing-ref', false).reason, 'git-unknown')
assert.equal(existsSync(w.path), true)
console.log('cleanup preview verified')
NODE
~~~

02 模块从 `Workspace` 接收已建立的 repo、host 和 occupant identity。
