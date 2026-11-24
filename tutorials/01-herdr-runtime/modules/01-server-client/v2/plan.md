# Herdr Server / Client v2：本地 JSON API 与事件订阅

[模块路线图](../plan.md) · [上一版 v1](../v1/plan.md) · [本版正文](local-json-api.md) · [下一版 v3](../v3/plan.md)

## 版本定位

从已经通过握手与 Detach 检查的 v1 Rust 项目继续。用真实 Unix socket 接收 JSON Lines 请求。CLI 可以按 request ID 查找响应，通过 pane.list 与 pane.create 访问共享的 pane，也可以按事件游标继续订阅。

## 实现顺序

1. 给运行时增加 pane 创建事件和按游标读取的入口，用 v1 回归检查事件顺序。
2. 让 src/api_socket.rs 校验请求、派发命令，订阅先确认再送事件。src/main.rs 绑定本地监听器，为每条连接执行同一个请求入口。

## 验收与交接

从独立进程走一次订阅、创建、查询，核对事件游标、pane ID 和 request ID。畸形输入、未来游标和空闲连接进入各自错误分支，后续查询仍能读出同一 pane。验收使用私有临时目录。事件日志和订阅游标在进程内维护。v3 在同一运行时增加 thin client 的有界发送队列。
