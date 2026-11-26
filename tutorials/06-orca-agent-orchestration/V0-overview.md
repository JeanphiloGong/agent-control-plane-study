# Orca V0：Agent 工作编排与隔离工作区

## 版本身份

- **模块**：`orca-agent-orchestration`
- **基线**：`sources/orca` @ `27300a14e0170258814b67b2f4159446c637f551`
- **范围**：Worktree、Coordinator、Task、Dispatch、Worker Session、Review、Remote Execution 和 Cleanup。
- **当前状态**：产品源码已接入，模块 V0 设计和 V1–Vn 实现教程待继续构建。

## 真实场景

用户把一个编码目标交给 Orca。Coordinator 选择项目和执行主机，创建隔离 worktree，向
worker agent 发送完整任务说明。worker 在自己的 session 中修改代码并运行检查，结果通过
receipt 返回。Coordinator 或用户查看 diff 和检查结果，决定继续、review、merge 或保留
未完成的 worktree。

```text
Human goal
  → project / host / worktree selection
  → coordinator task
  → worker dispatch + receipt
  → worker session and terminal activity
  → checks + diff / review
  → merge decision
  → cleanup preview or preserve unfinished work
```

## 模块

| 模块 | 责任 | V0 状态 |
| --- | --- | --- |
| 01 Worktree / Workspace | 选择、创建、绑定和清理隔离工作树 | [计划](modules/01-worktree-lifecycle/plan.md) |
| 02 Orchestration / Worker | Coordinator、Task、Dispatch、Receipt 和 worker settlement | [计划](modules/02-orchestration-worker/plan.md) |
| 03 Session / Recovery | worker session 状态、lost、stop、retry、fence 和保留未完成工作 | [计划](modules/03-session-recovery/plan.md) |
| 04 Review / Remote | diff review、PR 交接、远程执行主机和 cleanup 门禁 | [计划](modules/04-review-remote/plan.md) |

## 模块依赖

```text
01 Worktree / Workspace
  → 02 Orchestration / Worker
  → 03 Session / Recovery
  → 04 Review / Remote
  → O1 product integration
```

Worktree 先提供隔离执行边界。Orchestration 再把一个目标分配给具体 worker。Recovery
负责判断 worker 是否仍然可验证，Review / Remote 最后处理人工反馈、远程主机和清理。
Orca 的事实不直接成为 Multica Issue 或 Scheduler 的事实。

## 产品集成门 O1

一个目标能够在独立 worktree 中启动 worker，返回可追踪的 dispatch receipt，完成检查后
生成 review/merge 输入。worker 失败、状态未知或工作树有未提交变更时，系统保留工作树，
不会自动强制删除。

## 源码对应

| 路径 | 责任 |
| --- | --- |
| `src/cli/workspaces/*` | worktree 选择、创建、状态、激活和 cleanup |
| `src/cli/orchestration/*` | orchestration 命令、receipt 和 worker 视图 |
| `src/cli/dispatch/*` | 执行主机和 dispatch 边界 |
| `src/shared/workspace-cleanup/*` | cleanup candidate、过滤器和 verdict |
| `skill-guides/orchestration.md` | coordinator、worker、Task 和 recovery 的操作契约 |
| `skill-guides/references/recovery-and-cleanup.md` | worker 失败、未知状态和清理恢复规则 |

## 明确排除

- 不复制 Orca 的 Electron UI。
- 不把 Orca worker 当成 Multica Scheduler 的 lease owner。
- 不自动执行真实 merge、强制删除 worktree 或强制 reset。
- 不在 V0 中实现完整远程主机部署。
