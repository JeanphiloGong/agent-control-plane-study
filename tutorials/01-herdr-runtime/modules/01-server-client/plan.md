# 01 Server / Client：从客户端断开到运行时接管

[Herdr 路线图](../../plan.md) · [V0 设计与契约](V0-design.md) · [参考架构](reference-architecture.md)

用户退出 TUI 后，pane 仍由服务端持有。先把这条规则跑通，再让独立进程看到同一份状态，接着验证慢连接和旧运行时的影响范围。

| 版本 | 前置 | 独立计划 | 实现正文 | 验收 |
| --- | --- | --- | --- | --- |
| v1 | V0 设计 | [计划](v1/plan.md) | [运行时所有权](v1/server-client-runtime.md) | Detach 后再连，仍能读取原 pane |
| v2 | v1 运行时 | [计划](v2/plan.md) | [本地 JSON API](v2/local-json-api.md) | 独立进程按 request ID 查询，从订阅读到 pane 事件 |
| v3 | v2 API 与事件 | [计划](v3/plan.md) | [多客户端与背压](v3/multi-client-backpressure.md) | 停读客户端过载，健康客户端继续收更新 |
| v4 | v3 连接注册表 | [计划](v4/plan.md) | [接管与重连](v4/runtime-handoff.md) | 新连接延续 pane 与事件游标，旧实例拒绝延迟写入 |

从 v1 开始创建一个 Rust 项目，后续各版直接在这个项目追加或替换正文指定的代码。v2 起需要 Unix socket，独立进程验收使用 Python 3 标准库。每篇正文提供完整的代码改动和冻结命令。

本模块用内存 pane 讲解所有权和连接生命周期。v4 的接管发生在同一进程内。后续 [02 PTY / Pane / Agent Detection](../02-pane-detection/V0-design.md) 使用 v1 建立的服务端状态与事件边界接入真实终端数据。
