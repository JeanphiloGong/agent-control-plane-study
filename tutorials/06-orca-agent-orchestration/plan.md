# Orca Agent Orchestration：模块路线图

[产品总览](V0-overview.md) · [全局源码规划](../../plan.md)

Orca 是独立的 Agent 工作编排产品。它通过 worktree、worker session 和 dispatch receipt
把一个编码目标交给另一个 Agent 执行，再把结果带回 review 和 cleanup 流程。它与 Multica
的 Issue、Run 和 Scheduler 是两个事实系统，教程只在交接处说明边界。

| 版本 | 前置 | 目标 | 实现正文 | 验收 |
| --- | --- | --- | --- | --- |
| v1 | V0 Worktree contract | 创建并绑定一个隔离 worktree | 待构建 | worktree 路径、项目和 owner 一致 |
| v2 | v1 worktree | 创建 coordinator Task、启动 worker 并读取 receipt | 待构建 | 创建、session ready、turn started 可区分 |
| v3 | v2 dispatch | 处理 worker lost、停止、重试和未完成工作保留 | 待构建 | 未知状态不触发破坏性 cleanup |
| v4 | v3 recovery | 连接 review、远程 host 和 cleanup 门禁 | 待构建 | review 输入完整，远程断线不会伪造完成 |

## 真实调用链

```text
orca worktree / project selection
  → orchestration task
  → dispatch worker
  → session / terminal events
  → check and diff evidence
  → review or preserve
  → cleanup preview
```

## 主要源码

- `src/cli/workspaces/worktree.ts` 和 `src/cli/workspaces/workspace-cleanup.ts`。
- `src/cli/orchestration/orchestration-handlers.ts` 和 `src/cli/specs/orchestration.ts`。
- `src/cli/dispatch/*` 与 `src/cli/hosts/*`。
- `src/shared/workspace-cleanup/*`。
- `skill-guides/orchestration.md` 及其 `references/coordinator-loop.md`、`references/recovery-and-cleanup.md`。

## 交接边界

- Worktree 创建成功不等于 worker session 已经启动。
- Session ready 不等于 worker 已经开始执行 turn。
- Worker 报告失败不等于可以删除工作树。
- Review 或 merge 需要独立的人工或外部流程决定。
- Orca 的 worker 结果可以被上层 Agent 控制面消费，但不能绕过上层状态机写成功。

## V1–V4 构建顺序

每一版都从同一 Orca 基线提取一个最小状态流，使用独立参考实现验证，再对照真实
TypeScript/CLI 符号。V1 先冻结工作区身份，V2 冻结 dispatch receipt，V3 冻结恢复和
cleanup 保护，V4 冻结 review 与 remote handoff。
