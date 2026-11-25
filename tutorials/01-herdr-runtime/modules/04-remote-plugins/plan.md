# 04 Remote Endpoint / Plugin：从连接状态到受限扩展命令

[Herdr 路线图](../../plan.md) · [V0 设计与契约](V0-design.md) · [参考架构](reference-architecture.md)

Herdr 的客户端既可以连接本地 server，也可以连接一个保存的 SSH endpoint。插件则从 manifest 声明自己的命令和 hook，再由 Herdr 生成只读上下文并负责启动、限流和记录。两条路径共享状态投影，但 endpoint transport 不拥有插件状态，插件命令也不能绕过 endpoint 或 workspace 边界。

| 版本 | 前置 | 独立计划 | 实现正文 | 验收 |
| --- | --- | --- | --- | --- |
| v1 | V0 设计 | [计划](v1/plan.md) | [本地 endpoint 与 fake transport](v1/local-endpoint.md) | storage key 不泄漏 target，连接失败不伪造 Online |
| v2 | v1 endpoint | [计划](v2/plan.md) | [metadata 缓存与重连](v2/metadata-reconnect.md) | 缓存命中、发现、一次重试和 Attention 分类 |
| v3 | v2 connection boundary | [计划](v3/plan.md) | [manifest 与 invocation context](v3/manifest-context.md) | 非法 manifest、平台和 action/context 校验在 runner 前拒绝 |
| v4 | v3 manifest/context | [计划](v4/plan.md) | [命令 runner 与状态投影](v4/command-runner.md) | 输出上限、并发上限、command log 和安全环境 |

每个版本都从 `v1/reference/` 的同一份 Python 参考项目继续。参考项目不启动真实 SSH、不读取用户凭据、不扫描真实插件目录。真实源码仍以固定提交为准，正文会标出对应的 Rust 文件和符号。

## 真实调用链

```text
ClientEndpointId
  → EndpointRegistry
  → SavedSshApiBridge / SshStdioBridge
  → metadata cache / remote discovery
  → local endpoint socket
  → Connecting / Online / Reconnecting / Attention

plugin registry
  → load_plugin_manifest / normalize_manifest_*
  → PluginInvocationContext
  → start_plugin_command
  → PluginCommandLogInfo / AppEvent
```

endpoint 的 `storage_key` 只使用 `local` 或 opaque profile id。target、session 和 SSH options 只存在连接过程。plugin manifest 是声明输入，`PluginInvocationContext` 是本次调用的数据快照，命令日志是服务端事实。插件可以通过生成的环境变量读取上下文和受限目录，但不能把任意路径写进核心 runtime。

## 阅读和验证方式

从仓库根目录进入本模块，先按 v1 创建 `v1/reference`，随后每版在同一项目中追加文件或替换指定方法。每篇正文提供完整代码、读者检查和测试边界。参考项目的通过结果只证明状态模型和边界规则，不能替代 Herdr 的真实 SSH、Unix socket、平台命令和插件进程测试。
