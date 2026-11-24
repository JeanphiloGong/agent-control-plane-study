# Herdr Server / Client / Socket v1：让断开的客户端不带走运行时

[模块路线图](../plan.md) · [模块 V0 设计](../V0-design.md) · [本版正文](server-client-runtime.md) · [下一版 v2](../v2/plan.md)

## 版本定位

这是 `herdr-server-client` 模块内部的 v1 教程/实现计划。V0 已经从 Herdr 的真实源码中确认 `HeadlessServer`、客户端握手、消息循环和 detach 的所有权边界。现在用一个最小 Rust 参考实现冻结这条规则：客户端断开后，服务端仍持有 pane，新的客户端可以看到同一个 pane。

源码基线是 `sources/herdr` 的 `e8d7d9705a7a2c89cef4dd42079379f05685af87`。按正文在本模块的 `v1/reference/` 从空目录搭建 Rust 项目。

## 从 V0 开始

V0 已经写清楚了以下事实：

- `HeadlessServer` 拥有 App、运行时和客户端集合。
- `handle_client_handshake` 在正常消息循环前校验 hello 并返回 welcome。
- `ClientMessage::Detach` 结束客户端连接，不应调用 pane 或 PTY 的 shutdown。
- `tests/detach_reattach.rs` 与 `tests/cross_area.rs` 提供 detach、reattach 和多客户端共享状态的上游用例。

本版把这些事实缩小为可运行的 `ServerRuntime`、`ClientSession`、协议消息和事件记录。

## 明确不做

本版本不处理真实 Unix socket、JSON API、PTY、终端渲染、多个客户端的队列背压、SSH、插件或 live handoff。真实 socket 在 v2，队列隔离在 v3，重连和 handoff 在 v4；SSH 与插件仍由 `herdr-remote-plugins` 模块拥有。

## 拟改文件与边界

教程会创建以下参考项目文件：

```text
v1/reference/Cargo.toml
v1/reference/src/lib.rs
v1/reference/src/client_protocol.rs
v1/reference/src/runtime_events.rs
v1/reference/src/server_runtime.rs
v1/reference/src/client_session.rs
v1/reference/tests/detach_preserves_runtime.rs
```

`ServerRuntime` 负责 pane、客户端集合和事件事实。`ClientSession` 负责自己的握手状态和连接结束。协议只保留 `TerminalHello`、`Welcome`、`Detach`、snapshot 和错误边界，不复制 Herdr 的完整 wire protocol。

## 验收

固定检查需要证明两个结果：

1. 一个客户端完成握手并发送 `Detach` 后，客户端会话关闭，服务端仍处于运行状态，原 pane 仍是 running。
2. 第二个客户端完成握手后，welcome 和 snapshot 都包含原 pane ID。

边界检查使用零尺寸 hello，断言它返回 `InvalidGeometry`，且服务端连接数保持为零。所有检查在无网络、无真实账号和无 PTY 的临时 Rust 项目中执行。

从仓库根目录运行验收命令。

```bash
cargo test --manifest-path tutorials/01-herdr-runtime/modules/01-server-client/v1/reference/Cargo.toml
```

教程正文的代码块必须能按顺序创建这些文件，不允许依赖读者自行补齐导入、调用方或测试夹具。通过测试才算 v1 冻结；正文中的预期输出不替代实际执行。

## 教程交接与冻结

v1 在两项验收测试通过后冻结。v2 在同一个 Rust 项目沿用 `ServerRuntime` 和事件记录，新增独立进程可调用的本地 API。 [v2 计划](../v2/plan.md)
