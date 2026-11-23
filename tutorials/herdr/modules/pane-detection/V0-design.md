# Herdr PTY / Pane / Agent Detection V0

## 版本身份与边界

- **模块**：`herdr-pane-detection`
- **基线**：`sources/herdr` @ `e8d7d9705a7a2c89cef4dd42079379f05685af87`
- **V0 目标**：解释 PTY 输出、终端纯状态、Agent 检测证据和最终状态事件之间的关系。
- **包含**：PaneState、TerminalState、PaneRuntime/TerminalRuntime、屏幕尾部读取、AgentDetection、稳定化和发布抑制。
- **不包含**：为所有 Agent 编写新 manifest、UI 绘制、业务任务调度。

## 场景

Codex 在 pane 中工作时，PTY 持续产生输出。用户看到的“working”不是客户端自己猜出来的，而是服务端先记录终端事实，再根据检测 manifest、hook 和进程退出信息判断。短暂的空屏不能立即把 working 变成 idle，明确的 blocker 则需要发布给客户端。

## 源码调用链

| 源码 | 符号 | 责任 |
| --- | --- | --- |
| `src/pane.rs` | `PaneRuntime`, `PaneRuntime::spawn*`, detection loop | 持有 PTY-backed runtime，并把输出交给终端和检测逻辑 |
| `src/pane/state.rs` | `PaneState` | 只保存 attached terminal、seen 和右键透传等 pane/view 状态 |
| `src/pane/terminal.rs` | `PaneTerminal`, `detection_text_for_cache`, snapshot methods | 把 PTY bytes 应用到终端屏幕并提供检测/读取视图 |
| `src/terminal/runtime.rs` | `TerminalRuntime` | 对外暴露 server-owned runtime，内部复用 PaneRuntime |
| `src/terminal/state.rs` | `TerminalState`, `set_detected_agent_process_at`, state mutation APIs | 保存 cwd、detected agent、hook authority、AgentState、revision 和恢复信息 |
| `src/detect/mod.rs` | `AgentState`, `AgentDetection`, `Agent`, `detect_agent_with_osc` | 定义可序列化的检测结果和 Agent 识别入口 |
| `src/pane/agent_detection.rs` | `DetectionTextCache`, `detection_update_for_publish_with_osc`, `decide_screen_detection_publish` | 缓存屏幕证据、处理进程退出、稳定化并决定是否发布 |
| `src/pty/actor.rs` | `PtyIoActor` | 隔离 PTY 读写和 resize，向 runtime 提供字节/退出事实 |

## 状态流

```text
PTY bytes / process exit / hook report
  → TerminalRuntime / PaneTerminal
  → detection content sequence + bottom-buffer text
  → detect_agent_with_osc → AgentDetection
  → stabilize_agent_detection
  → PendingIdleConfirmation / visible signal arbitration
  → TerminalState mutation
  → AppEvent::PaneAgentStatusChanged / client projection
```

`AgentDetection` 是证据结果，不等于最终发布状态。`AgentState` 有 `Idle`、`Working`、`Blocked`、`Unknown` 四个值。`process_exited` 会产生明确的 idle 证据，但检测器不会因此伪造一个仍存活的 Agent。

## 局部版本线

- **V1**：PTY 输出和进程退出进入 `TerminalState`，支持 `Idle/Working/Unknown`。
- **V2**：屏幕 manifest、`AgentDetection` 和 blocker 证据。
- **V3**：pending-idle 确认、内容序列缓存、hook 与屏幕来源仲裁。
- **V4**：多 Agent manifest 更新和 API/客户端状态投影。

## V1 检查

给检测器输入工作中输出、稳定 prompt、明确 blocker 和 process exit 四种证据。断言单次空屏不立即发布 idle，连续确认后才发布；process exit 直接产生 idle；检测器本身不修改 `PaneTerminal` 的屏幕内容。

契约见 [domain-flow](contracts/domain-flow.md)、[state-and-errors](contracts/state-and-errors.md)、[interface-or-storage](contracts/interface-or-storage.md)，参考实现见 [reference-architecture](reference-architecture.md)。
