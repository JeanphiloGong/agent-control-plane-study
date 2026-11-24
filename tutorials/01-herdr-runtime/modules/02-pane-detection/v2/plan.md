# Herdr Pane Detection v2：Manifest 与 blocker 证据

[模块路线图](../plan.md) · [上一版 v1](../v1/plan.md) · [正文](manifest-evidence.md)

接续 v1 的 `TerminalState`，把检测规则移到 `AgentManifest`。匹配 working、idle 和 blocker 后返回 `AgentDetection`，并保留 source 与 visible 信号。

验收覆盖 blocker、working、未知文本和 process exit。检测函数保持纯函数，不修改 TerminalState。
