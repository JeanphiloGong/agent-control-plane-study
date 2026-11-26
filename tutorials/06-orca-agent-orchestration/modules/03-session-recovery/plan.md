# Orca 03 Session / Recovery：V0 计划

当前状态是 V0 计划，V1 正文待构建。

源码入口包括 `src/cli/runtime/session-tool-caller.ts`、`src/cli/workspaces/worktree-setup-decision-recovery.ts` 和 `skill-guides/references/recovery-and-cleanup.md`。本模块讲 worker session 的可验证状态、lost、stop、retry 和 preserve 边界。

V1 定义 worker 状态。V2 建立失败和未知状态的恢复决策。V3 将恢复结果交给 review 和 cleanup 门禁。
