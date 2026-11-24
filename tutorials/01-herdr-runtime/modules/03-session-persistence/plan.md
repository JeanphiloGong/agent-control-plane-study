# 03 Session Persistence：从运行时事实到可恢复 session

[Herdr 路线图](../../plan.md) · [V0 设计与契约](V0-design.md) · [参考架构](reference-architecture.md)

用户退出 Herdr 后，下一次启动需要重新得到 workspace、tab、pane、cwd 和可选的 Agent 启动信息。这个模块沿着真实源码的保存边界学习这条链路。主布局、pane history、恢复副本和 Agent resume 元数据各有自己的可信度和失败处理。

| 版本 | 前置 | 独立计划 | 实现正文 | 验收 |
| --- | --- | --- | --- | --- |
| v1 | V0 设计 | [计划](v1/plan.md) | [快照与恢复](v1/snapshot-restore.md) | JSON round-trip、active/selected/cwd 保留、未来版本拒绝 |
| v2 | v1 快照 | [计划](v2/plan.md) | [原子版本化保存](v2/atomic-versioned-save.md) | 临时文件提交、旧快照保护、失败不破坏主文件 |
| v3 | v2 writer | [计划](v3/plan.md) | [迁移与恢复副本](v3/migration-recovery.md) | 已知旧版本迁移、history fingerprint、恢复副本和清理 |
| v4 | v3 history | [计划](v4/plan.md) | [Agent resume 与 handoff](v4/agent-resume-handoff.md) | resume 元数据去重、worktree 归属保留、恢复不重复启动 |

每个版本都从 `v1/reference/` 的同一份 Python 参考项目继续。参考项目只缩小数据结构和文件边界，不替代 Herdr 的 Rust runtime。真实源码位置在 `sources/herdr`，代码步骤会标出对应的模块和符号。

## 真实调用链

```text
App::schedule_session_save
  → capture_session_save_job
  → persist::capture / capture_history
  → SessionWriter::save
  → io::save_to_path / save_history_to_path

App::try_new
  → io::load / load_history
  → parse_snapshot / layout_fingerprint
  → restore
  → 新的 Workspace、TerminalState 和 TerminalRuntime
```

`SessionSnapshot` 是可以重建布局的结构事实。它不复制 PTY 的内部字节。pane ANSI history 是单独的 `SessionHistorySnapshot`，只有布局指纹匹配时才恢复。Agent session 和 launch 信息只保存为可重建元数据，真正的 runtime 创建仍属于 `restore`。

## 阅读和验证方式

从仓库根目录进入本模块，先按 v1 创建 `v1/reference`，随后每版在同一个目录中追加修改。每个正文都给出完整的新文件或完整替换的方法、读者检查和测试边界。本文档中的验收命令是参考项目的检查，不代表真实 Herdr 的 PTY、平台文件替换或跨进程 handoff 已被 Python 项目验证。
