# Orca 01 Worktree / Workspace：V0 计划

[V0 设计](V0-design.md) · [领域流](contracts/domain-flow.md) · [接口与存储](contracts/interface-or-storage.md) · [状态与错误](contracts/state-and-errors.md) · [参考架构](reference-architecture.md)

当前状态是 V0 设计和 V1–V3 正文已完成。

源码入口包括 `src/cli/workspaces/worktree.ts`、`src/cli/workspaces/workspace-cleanup.ts` 和 `src/shared/folder-workspaces/*`。本模块讲项目、worktree、执行主机和 cleanup candidate 的身份边界。

V1 创建并绑定一个 worktree。V2 增加状态检查和未提交变更保护。V3 增加安全 cleanup verdict。

- [V1 创建 worktree](v1/create-worktree.md)
- [V2 读取状态](v2/inspect-worktree.md)
- [V3 cleanup preview](v3/cleanup-preview.md)

V0 的设计交接固定了 `Worktree`、`WorktreeMeta`、Git catalog、cleanup candidate 和
removal authorization 的所有权。V1 必须从 [参考架构](reference-architecture.md) 的
identity 与 base resolver 开始，并在独立 replay 中验证 path 变化、base lookup error 和
未注册 worktree 拒绝路径。
