# 01 Herdr Agent Runtime：从服务端所有权到可恢复控制面

[产品 V0 总览](V0-overview.md) · [总计划](../../plan.md) · [依赖图](../dependency-map.md) · 产品集成门 **H1**

## 教程阅读顺序

| 模块 | V0 设计 | 实现教程 | 产出 |
| --- | --- | --- | --- |
| 01 Server / Client / Socket | [V0](modules/01-server-client/V0-design.md) | [v1–v4 路线图](modules/01-server-client/plan.md) | 握手、Unix API、客户端隔离和接管 |
| 02 PTY / Pane / Agent Detection | [V0](modules/02-pane-detection/V0-design.md) | [v1–v4 路线图](modules/02-pane-detection/plan.md) | PTY 证据和稳定 Agent 状态 |
| 03 Session Persistence | [V0](modules/03-session-persistence/V0-design.md) | [v1–v4 路线图](modules/03-session-persistence/plan.md) | 快照、恢复、history 和 Agent resume |
| 04 Remote Endpoint / Plugin | [V0](modules/04-remote-plugins/V0-design.md) | v1 待构建 | endpoint transport 和插件边界 |

模块按照 `01 → 02 → 03 → 04` 交接。每个模块有自己的 v1-vn，不把产品级 H1 当成某个模块的 v1。

## 场景与学习边界

用户启动 Herdr，在 pane 中运行 Agent。TUI 客户端退出后，服务端和 pane 继续存在；用户再次连接时仍能看到同一份 server-owned 状态。后续模块再把 PTY 检测、Session 恢复、远程 endpoint 和插件能力接到这条链路。

本产品线研究真实 Herdr 的 server、client、pane、session 和 endpoint 边界。教程参考实现使用小范围可运行代码，解释真实源码中的责任，不复制 Herdr 的终端渲染、平台适配、SSH 或插件市场。

## 从 Server / Client 开始

先用 v1 建立 pane 与客户端的生命周期边界，然后在同一个项目完成 v2 API、v3 背压和 v4 接管。02 模块继续研究真实 PTY、进程退出和 Agent 状态。 [Server / Client v1 正文](modules/01-server-client/v1/server-client-runtime.md)
