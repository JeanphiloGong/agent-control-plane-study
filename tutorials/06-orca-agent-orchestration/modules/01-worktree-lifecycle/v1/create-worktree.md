# Worktree v1 — 创建并绑定 Git checkout

[模块路线](../plan.md) · [V0 设计](../V0-design.md)

给本机一个已经注册的 Git 项目和一个新的分支名，在独立目录建立 checkout。Project 由 `repoId` 定位，`hostId` 指明目录所在机器。路径用于查找，稳定身份由 Orca 的 `createWorktreeIdentity` 从 host 和新分配的 instance ID 得到。一次成功的 Git 创建才产生返回记录。

## 1. 准备固定源码

需要 Bash、Git 和 Node.js 22.18 或更高版本。从学习仓库根目录执行。后续各版沿用 `ORCA_LAB` 中的文件和已提交了一次的演示仓库。

~~~bash
export ORCA_LAB="$(mktemp -d)"
git -C sources/orca archive 27300a14e0170258814b67b2f4159446c637f551 \
  src/shared/worktree/identity.ts \
  src/main/workspaces/creation/worktree-create-base.ts |
  tar -x -C "$ORCA_LAB"
cd "$ORCA_LAB"
git init -q -b main repo
git -C repo -c user.name=Tutorial -c user.email=tutorial@example.test \
  commit --allow-empty -qm initial
~~~

真实 CLI 的 `WORKTREE_HANDLERS['worktree create']` 解析目标并调用 runtime。Runtime 本地创建路径调用 `resolveWorktreeCreateBase`，再进入 Git 创建。下方练习以已选定的本地 Git 项目为输入，直接复用固定提交中的 base resolver 和 identity 函数。使用项目目录之外的新目录作为 worktree 目标。 [CLI](../../../../../sources/orca/src/cli/workspaces/worktree.ts) · [本地创建](../../../../../sources/orca/src/main/runtime/workspaces/creation/runtime-local-worktree-create.ts)

## 2. 创建 checkout 并返回标识

新建 `workspace.ts`，以下是完整文件。`preferredBase` 是项目保存的 base，`base` 是本次显式选择。练习项目已知默认分支为 `main`，用辖内 Git 查验它和保存的 ref。Git 通过参数数组接收仓库路径、分支和目标。练习中的记录由调用者持有，进程内用同一记录交给后续步骤。

~~~ts file=workspace.ts
import { execFileSync } from 'node:child_process'
import { randomUUID } from 'node:crypto'
import { resolve } from 'node:path'
import { resolveWorktreeCreateBase } from './src/main/workspaces/creation/worktree-create-base.ts'
import { createWorktreeIdentity } from './src/shared/worktree/identity.ts'

export type Workspace = {
  id: string
  repoId: string
  hostId: string
  path: string
  branch: string
  base: string
  identity: ReturnType<typeof createWorktreeIdentity>
}

export function git(cwd: string, ...args: string[]): string {
  return execFileSync('git', ['-C', cwd, ...args], {
    encoding: 'utf8',
    stdio: ['ignore', 'pipe', 'pipe']
  })
}

export async function createWorkspace(input: {
  repo: string
  repoId: string
  path: string
  branch: string
  base?: string
  preferredBase?: string
}): Promise<Workspace> {
  const path = resolve(input.path)
  git(input.repo, 'check-ref-format', '--branch', input.branch)

  const canResolve = (ref: string) => {
    try {
      git(input.repo, 'rev-parse', '--verify', '--quiet', '--end-of-options', ref + '^{commit}')
      return true
    } catch (error) {
      if ((error as { status?: number }).status === 1) {
        return false
      }
      throw error
    }
  }
  const base = await resolveWorktreeCreateBase({
    requestedBaseBranch: input.base,
    repoWorktreeBaseRef: input.preferredBase,
    resolveDefaultBaseRef: async () => canResolve('refs/heads/main') ? 'refs/heads/main' : null,
    isBaseUsable: async (ref) => canResolve(ref)
  })
  if (!base) {
    throw new Error('no usable base')
  }

  const baseCommit = git(input.repo, 'rev-parse', '--verify', '--end-of-options',
    base + '^{commit}').trim()
  git(input.repo, 'worktree', 'add', '-b', input.branch, path, baseCommit)
  const id = input.repoId + '::' + path
  const hostId = 'local'
  return {
    id,
    repoId: input.repoId,
    hostId,
    path,
    branch: input.branch,
    base: baseCommit,
    identity: createWorktreeIdentity({
      worktreeId: id, executionHostId: hostId, instanceId: randomUUID()
    })
  }
}
~~~

`base` 在返回记录里固定为创建时的 commit。后面即使默认分支前进，仍能复查本次创建从哪里开始。

## 3. 验证 Git 与稳定身份

从 `ORCA_LAB` 执行。命令应输出 `workspace identity verified`。第二次执行可换一组分支名和路径。

~~~bash check
node --input-type=module <<'NODE'
import assert from 'node:assert/strict'
import { createWorkspace, git } from './workspace.ts'
import { canonicalWorktreeIdentity } from './src/shared/worktree/identity.ts'
import { resolveWorktreeCreateBase } from './src/main/workspaces/creation/worktree-create-base.ts'

const w = await createWorkspace({ repo: 'repo', repoId: 'demo', path: 'worker-1', branch: 'worker-1' })
assert.equal(git(w.path, 'branch', '--show-current').trim(), w.branch)
assert.equal(git(w.path, 'rev-parse', 'HEAD').trim(), w.base)
const ref = { ...w.identity, worktreeId: w.id }
assert.equal(canonicalWorktreeIdentity({ ...ref, worktreeId: 'moved' }), w.identity.key)
assert.notEqual(canonicalWorktreeIdentity({ ...ref, instanceId: 'replacement' }), w.identity.key)

const lookup = {
  resolveDefaultBaseRef: async () => { throw new Error('lookup failed') },
  isBaseUsable: async (ref) => ref === 'saved'
}
assert.equal(await resolveWorktreeCreateBase({ ...lookup, repoWorktreeBaseRef: 'saved' }), 'saved')
await assert.rejects(resolveWorktreeCreateBase(lookup), /default base branch/)
console.log('workspace identity verified')
NODE
~~~

v2 在同一个 checkout 读取工作区证据。
