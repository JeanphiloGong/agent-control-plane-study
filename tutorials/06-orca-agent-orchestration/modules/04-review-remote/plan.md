# Orca 04 Review / Remote：V0 计划

当前状态是 V0 计划，V1 正文待构建。

源码入口包括 `src/cli/hosts/*`、`src/cli/workspaces/worktree-review-link-flags.ts` 和 `src/renderer/src/store/hosted-review/*`。本模块讲 review 输入、远程执行主机、PR 交接和 cleanup 的最终边界。

V1 形成 diff review 输入。V2 增加远程 host 与 worktree 绑定。V3 验证远程断线、未完成工作和人工 merge 决策不会被误判为完成。
