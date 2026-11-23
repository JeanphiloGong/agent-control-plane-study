# PTY / Pane / Agent Detection：接口契约

## 输入

- PTY bytes、process exit、terminal title/OSC progress。
- 当前 `AgentState`、已知 `Agent`、上一轮可见信号和 detection content sequence。
- 可选 hook authority。它是来源证据，不是任意调用方可写的最终状态。

## 输出

- `AgentDetection`：`state`、`skip_state_update`、`visible_idle`、`visible_blocker`、`visible_working`。
- `TerminalStateMutation`：有效状态变化、session ref 变化和 Agent release。
- 可选 `PaneAgentStatusChanged` 事件。

## 禁止的接口耦合

检测器不能依赖 ratatui 的视图渲染，也不能直接发送 socket 消息。客户端只消费最终事件或查询结果。测试可用纯字符串和时间输入验证稳定化，不需要真实终端窗口。
