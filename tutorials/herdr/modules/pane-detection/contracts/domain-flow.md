# PTY / Pane / Agent Detection：领域流

## 对象与所有权

| 对象 | 所有者 | 作用 |
| --- | --- | --- |
| `PaneState` | App/view state | pane 的 attached terminal、seen 和交互偏好 |
| `TerminalState` | 服务端 App | terminal identity、cwd、Agent、hook/screen 状态和 revision |
| `PaneRuntime` / `TerminalRuntime` | 服务端 runtime | PTY actor、终端解析、输出和 resize |
| `AgentDetection` | 检测器 | 一次屏幕/OSC 观察的证据和可见信号 |
| `AgentState` | TerminalState | 对外发布的稳定状态 |

## 领域操作

```text
observe PTY
  → update terminal screen
  → read detection evidence
  → classify
  → stabilize
  → publish state change or retain prior state
```

检测器读取证据，不拥有 PTY，也不直接向客户端写消息。发布由 AppEvent/EventHub 完成。
