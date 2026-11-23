# PTY / Pane / Agent Detection：参考架构

## 学习实现目录

```text
src/runtime/pty.py              # PtyProcess、read bytes、process exit
src/runtime/terminal_state.py   # TerminalState 与 AgentState
src/runtime/detection.py        # AgentDetection、纯检测函数、稳定化
src/runtime/pane.py             # PaneState 与 runtime 关联
src/runtime/events.py            # 状态变化事件
```

## 调用方向

```text
PtyProcess → TerminalBuffer → DetectionEvidence → DetectionPolicy
                                      ↓                 ↓
                               AgentDetection → TerminalState → EventHub
PaneState ────────────────────────────────┘
```

`DetectionPolicy` 只接收值并返回决策。`TerminalState` 负责保存事实，`EventHub` 负责通知，`PaneState` 不保存 Agent 状态。

## 源码到参考架构

| Herdr | 参考实现 | V1 检查 |
| --- | --- | --- |
| `PtyIoActor` / `PaneRuntime` | `PtyProcess` | process exit 与 bytes 可分别注入 |
| `TerminalState` | `TerminalState` | cwd、Agent、state 各自可读取 |
| `AgentDetection` | `DetectionEvidence` | 未识别输入得到 `Unknown` |
| `PendingIdleConfirmation` | `IdleStabilizer` | 单次空屏不发布 idle |
| `decide_screen_detection_publish` | `DetectionPolicy.decide` | blocker/working/idle 发布条件可断言 |

## V1 冻结检查

```text
policy = DetectionPolicy()
assert policy.observe(working_screen).state == "working"
assert policy.observe(blank_screen).publish is False
assert policy.observe(blank_screen_again).publish is True
assert policy.observe(process_exit).state == "idle"
```

检查证明的是状态仲裁规则，不证明所有真实 Agent manifest 的识别准确率。
