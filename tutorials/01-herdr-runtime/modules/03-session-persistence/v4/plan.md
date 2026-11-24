# Herdr Session Persistence v4：Agent resume 与 handoff 边界

[模块路线图](../plan.md) · [上一版 v3](../v3/plan.md) · [本版正文](agent-resume-handoff.md)

## 版本定位

v3 已经能保存结构、迁移格式、匹配 history 和保留恢复副本。v4 保存 pane 中可以重新计划 Agent 的元数据，以及 workspace 的 worktree 归属。启动恢复时只生成去重后的 resume metadata，真正启动进程仍由 runtime 的恢复流程决定。这样可以支持本地重启和 live handoff，又不会把“写入 JSON”误认为“已经接管了一个运行中的 Agent”。

真实源码对应 `src/persist/snapshot.rs::PaneAgentSessionSnapshot`、`PaneAgentResumeSnapshot`、`WorkspaceSnapshot::worktree_space`，`src/persist/restore.rs::restore`、`restore_handoff` 和 Agent resume plan 相关函数。

## 实现顺序

1. 为 pane snapshot 增加 agent、resume reference 和 worktree 元数据，并让 capture/restore 原样保留。
2. 为同一个 `resume_ref` 生成一份去重的 resume metadata，避免多个 pane 重复启动同一会话。
3. 用纯函数返回恢复计划输入，检查 handoff 只消费 snapshot metadata，不在持久化层启动 Agent 进程。

## 验收与边界

`python3 -m pytest -q` 运行前面各版和 v4 的 resume 测试，检查元数据恢复、重复引用去重和 worktree 信息保留。真实 Herdr 的 `restore_handoff` 还负责 imported runtime、失败计数和旧 pane ID 映射。
