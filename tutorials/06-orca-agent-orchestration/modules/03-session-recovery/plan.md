# Orca 03 Session / Recovery：V0 计划

[V0 设计](V0-design.md) · [领域流](contracts/domain-flow.md) · [接口与存储](contracts/interface-or-storage.md) · [状态与错误](contracts/state-and-errors.md) · [参考架构](reference-architecture.md)

当前状态是 V0 设计和 V1–V3 正文已完成。

源码入口包括 `src/main/runtime/orchestration/workers/lifecycle/session-worker-liveness.ts`、`src/cli/orchestration/recovery/orchestration-recovery-command.ts` 和 `skill-guides/orchestration/references/recovery-and-cleanup.md`。本模块讲 worker session 的可验证状态、lost、stop、retry 和 preserve 边界。

V1 定义 worker 状态。V2 建立失败和未知状态的恢复决策。V3 将恢复结果交给 review 和 cleanup 门禁。

- [V1 Liveness](v1/observe-session.md)
- [V2 Retry](v2/retry-worker.md)
- [V3 Preserve](v3/preserve-work.md)
