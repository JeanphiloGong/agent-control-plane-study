# ACP Runtime：状态与错误

- Runtime 安装、启动、ready、offline、auth required 和 incompatible 要分别表示。
- ACP request timeout 不等于 Agent 已停止。
- transport retry 必须受次数/截止时间限制。
- provider raw event 不能直接成为跨 Session 公共状态。
