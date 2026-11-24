# Herdr Session Persistence V0

## 版本身份与边界

- **模块**：`herdr-session-persistence`
- **基线**：`sources/herdr` @ `e8d7d9705a7a2c89cef4dd42079379f05685af87`
- **V0 目标**：解释 workspace、tab、pane、cwd 和 Agent 启动信息如何捕获、保存、读取、迁移和恢复。
- **包含**：`SessionSnapshot`、history fingerprint、版本解析、App 启动恢复、异步保存、备份和清理。
- **不包含**：跨机器同步、数据库事务、PTY 内部状态复制。

## 场景

用户创建两个 pane 后退出 Herdr。下一次启动，App 先读取 session JSON，再把布局恢复成 workspace/tab/pane，并按快照中的 cwd、launch argv 和可选 Agent session 信息重建 runtime。保存过程不能因为一个旧文件损坏或恢复副本目录不可写而静默覆盖用户最后可用的布局。

## 源码调用链

| 源码 | 符号 | 责任 |
| --- | --- | --- |
| `src/app/mod.rs` | `App::try_new` | 读取快照，调用 `restore`，创建 `SessionWriter` |
| `src/app/session.rs` | `schedule_session_save`, `capture_session_save_job`, `start_background_session_save`, `save_session_now` | 在 App 变脏后 debounce，捕获纯快照并交给后台 writer |
| `src/persist/snapshot.rs` | `SessionSnapshot`, `WorkspaceSnapshot`, `TabSnapshot`, `PaneSnapshot` | 定义 JSON 结构和当前 `SNAPSHOT_VERSION` |
| `src/persist/snapshot.rs` | `capture`, `capture_history`, `parse_snapshot`, `layout_fingerprint` | 从运行时捕获、解析、迁移和计算布局指纹 |
| `src/persist/io.rs` | `load`, `save_to_path`, `save_history_to_path`, `clear_path` | 读写 session JSON 和 history JSON |
| `src/persist/restore.rs` | `restore`, `restore_workspace`, `restore_tab` | 把快照映射回 Workspace、TerminalState 和 TerminalRuntime |
| `src/persist/writer.rs` | `SessionWriter::save`, `SessionWriter::clear`, `preserve_existing_in` | 保存前保护旧文件、保留历史副本并限制副本数量 |

## 状态流

```text
App state dirty
  → schedule_session_save (debounce)
  → capture(SessionSnapshot + optional history)
  → SessionWriter::save
       ├─ preserve_unloaded / recovery copy
       ├─ io::save_to_path
       └─ optional history save

process start
  → persist::load
  → parse_snapshot / migrate_snapshot
  → persist::restore
  → Workspace + TerminalState + TerminalRuntime
```

快照是可重建 session 的结构事实，不是 PTY 当前屏幕的完整复制。PTY 进程和运行时在恢复阶段重新创建，历史 ANSI 另存并且必须与布局指纹匹配。

## 前置、产出与交接

- **前置**：[`01-server-client`](../01-server-client/V0-design.md) 的运行时所有权和 [`02-pane-detection`](../02-pane-detection/V0-design.md) 的 terminal/Agent 状态字段。
- **V1 产出**：一个可版本化的 workspace/tab/pane 快照，以及加载、恢复和未知版本拒绝路径。
- **下游**：[`04-remote-plugins`](../04-remote-plugins/V0-design.md) 使用稳定的 endpoint/session identity；产品集成门 H1 使用恢复后的 pane 继续服务客户端。
- **冻结条件**：临时目录中的快照恢复布局事实一致，高版本快照拒绝且原文件保持不变。

## 局部版本线

- **V1**：捕获和恢复一个 workspace/tab/pane/cwd。
- **V2**：异步 debounce 保存、版本化 JSON 和原子替换。
- **V3**：恢复副本、旧版本迁移、history fingerprint 和失败保护。
- **V4**：Agent resume、worktree membership 和 live handoff 的恢复边界。

## V1 检查

在临时目录写入一个 `SessionSnapshot`，加载并恢复后断言 workspace、tab、pane id、cwd 和 active/selected 索引一致。另写一个更高 `version` 的 JSON，断言加载失败且原文件未被修改。

契约见 [domain-flow](contracts/domain-flow.md)、[state-and-errors](contracts/state-and-errors.md)、[interface-or-storage](contracts/interface-or-storage.md)，参考实现见 [reference-architecture](reference-architecture.md)。
