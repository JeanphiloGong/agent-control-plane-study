# Orca 01 Worktree / Workspace：V0 计划

当前状态是 V0 计划，V1 正文待构建。

源码入口包括 `src/cli/workspaces/worktree.ts`、`src/cli/workspaces/workspace-cleanup.ts` 和 `src/shared/folder-workspaces/*`。本模块讲项目、worktree、执行主机和 cleanup candidate 的身份边界。

V1 创建并绑定一个 worktree。V2 增加状态检查和未提交变更保护。V3 增加安全 cleanup verdict。
