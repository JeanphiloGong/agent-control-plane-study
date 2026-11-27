# Orca 02 Orchestration / Worker：V0 计划

[V0 设计](V0-design.md) · [领域流](contracts/domain-flow.md) · [接口与存储](contracts/interface-or-storage.md) · [状态与错误](contracts/state-and-errors.md) · [参考架构](reference-architecture.md)

当前状态是 V0 设计和 V1–V3 正文已完成。

源码入口包括 `src/cli/orchestration/orchestration-handlers.ts`、`src/cli/specs/orchestration.ts` 和 `skill-guides/orchestration.md`。本模块讲 coordinator、Task、Dispatch、Worker 和 receipt 的生命周期。

V1 建立一次 worker dispatch。V2 区分 worktree created、session ready 和 turn started。V3 处理 Task settlement 与多 worker 结果汇总。

- [V1 Dispatch](v1/dispatch-worker.md)
- [V2 Receipt](v2/worker-readiness.md)
- [V3 Settlement](v3/settle-task.md)
