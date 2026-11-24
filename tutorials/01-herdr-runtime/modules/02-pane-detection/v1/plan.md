# Herdr Pane Detection v1：把终端事实变成基础状态

[模块路线图](../plan.md) · [正文](pty-detection.md) · 前置：`herdr-server-client/v1`

## 版本定位

接收 PTY 输出和 process exit。文本检测只给出 `Working`、`Idle` 或 `Unknown`，状态保存在 `TerminalState`，检测函数不发送客户端消息。

## 明确不做

本版本不加入 manifest、blocker、idle 延迟确认、hook、多个 Agent 或真实终端解析器。

## 验收

固定输入覆盖工作标记、未知文本、shell prompt 和 process exit。process exit 直接得到 Idle，检查在临时 Rust 项目内运行。
