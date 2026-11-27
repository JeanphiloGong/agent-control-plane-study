# Orca 04 Review / Remote：V0 计划

[V0 设计](V0-design.md) · [领域流](contracts/domain-flow.md) · [接口与存储](contracts/interface-or-storage.md) · [状态与错误](contracts/state-and-errors.md) · [参考架构](reference-architecture.md)

当前状态是 V0 设计和 V1–V3 正文已完成。

源码入口包括 `src/cli/hosts/*`、`src/cli/workspaces/worktree-review-link-flags.ts` 和 `src/renderer/src/store/slices/hosted-review/*`。本模块讲 review 输入、远程执行主机、PR 交接和 cleanup 的最终边界。

V1 形成 diff review 输入。V2 增加远程 host 与 worktree 绑定。V3 验证远程断线、未完成工作和人工 merge 决策不会被误判为完成。

- [V1 Review evidence](v1/review-evidence.md)
- [V2 Host selection](v2/select-execution-host.md)
- [V3 Remote handoff](v3/remote-review-handoff.md)
