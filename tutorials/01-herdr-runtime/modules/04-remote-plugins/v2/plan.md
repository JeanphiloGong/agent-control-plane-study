# Herdr Remote Endpoint / Plugin v2：metadata 缓存与有限重连

[模块路线图](../plan.md) · [上一版 v1](../v1/plan.md) · [本版正文](metadata-reconnect.md) · [下一版 v3](../v3/plan.md)

## 版本定位

v1 的 registry 只知道 transport 成功或失败。保存的 SSH endpoint 还需要先知道远程 Herdr 的平台和可执行文件路径。v2 增加带 profile、target、session 校验的 metadata cache，cache miss 才进行 discovery，discovery 遇到 retryable transport 错误最多重试一次，权限、协议和不支持的平台进入 Attention。

真实源码对应 `src/client/endpoint/ssh_metadata.rs::SshMachineMetadata`、`SshMetadataCache`、`src/remote/saved.rs::SavedSshApiBridge::start` 和 `recover_transport`。参考项目继续使用 fake discovery 和 v1 registry。

## 实现顺序

1. 在 `v1/reference/src/endpoints/metadata.py` 实现 metadata 校验、缓存读写、target/session 指纹和失效。
2. 在 `v1/reference/src/endpoints/connector.py` 实现 cache hit、discovery、一次 retry 和 Attention/Reconnecting 分类。
3. 用 fake discovery 验证缓存命中不会重新发现，stale cache 会重新发现，重试次数不会无限增长。

## 验收与交接

从 v1/reference 执行累计测试。预期覆盖 metadata 路径、target/session 隔离、retryable discovery 和 attention failure。v3 消费 v2 的连接边界，开始处理 plugin manifest 和 invocation context。
