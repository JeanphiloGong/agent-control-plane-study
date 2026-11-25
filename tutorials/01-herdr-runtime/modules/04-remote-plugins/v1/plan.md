# Herdr Remote Endpoint / Plugin v1：本地 endpoint 与 fake transport

[模块路线图](../plan.md) · [V0 设计](../V0-design.md) · [本版正文](local-endpoint.md) · [下一版 v2](../v2/plan.md)

## 版本定位

先把 endpoint identity、transport 和状态注册表分开。`EndpointId` 只表达 local 或 SSH profile，`EndpointRegistry` 负责连接、发送和失败转移，`FakeTransport` 只用于确定性检查。连接失败进入明确状态，不能把“已创建对象”当成 Online。

## 实现顺序

1. 在 `v1/reference/src/endpoints/model.py` 定义 opaque profile、状态和 registry。
2. 在 `v1/reference/src/endpoints/transport.py` 定义可注入的 fake transport，并验证成功、retryable failure、attention failure 和 storage key。

## 验收与交接

从 `v1/reference` 执行 `python3 -m pytest -q`。预期覆盖 endpoint 状态和失败分支。v2 在这个 registry 上增加 remote metadata cache、发现和有限重试，不把 target/session 写进 profile storage key。
