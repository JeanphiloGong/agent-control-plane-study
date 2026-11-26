# Orca 02 Orchestration / Worker：V0 计划

当前状态是 V0 计划，V1 正文待构建。

源码入口包括 `src/cli/orchestration/orchestration-handlers.ts`、`src/cli/specs/orchestration.ts` 和 `skill-guides/orchestration.md`。本模块讲 coordinator、Task、Dispatch、Worker 和 receipt 的生命周期。

V1 建立一次 worker dispatch。V2 区分 worktree created、session ready 和 turn started。V3 处理 Task settlement 与多 worker 结果汇总。
