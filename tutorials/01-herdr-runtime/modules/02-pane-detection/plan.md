# 02 Pane Detection：从终端证据到 Agent 状态

[Herdr 路线图](../../plan.md) · [V0 设计](V0-design.md) · [参考架构](reference-architecture.md)

| 版本 | 前置 | 正文 | 验收 |
| --- | --- | --- | --- |
| v1 | `01-server-client/v1` | [PTY 与基础状态](v1/pty-detection.md) | output、unknown、process exit |
| v2 | v1 状态核心 | [Manifest 与 blocker](v2/manifest-evidence.md) | working/blocker/unknown 证据 |
| v3 | v2 detection | [Idle 稳定化](v3/idle-stabilization.md) | 单次空屏不发布 idle |
| v4 | v3 稳定化 | [多 Agent 投影](v4/multi-agent-projection.md) | registry、事件和客户端快照 |

参考项目从 `v1/reference/` 开始逐版追加。它模拟 PTY 字节和进程退出，不创建真实 PTY。真实入口是 `src/pty/actor.rs`、`src/pane/terminal.rs`、`src/detect/mod.rs` 和 `src/pane/agent_detection.rs`。
