# Herdr Remote Endpoint / Plugin V0

## 版本身份与边界

- **模块**：`herdr-remote-plugins`
- **基线**：`sources/herdr` @ `e8d7d9705a7a2c89cef4dd42079379f05685af87`
- **V0 目标**：解释本地/SSH endpoint 和插件命令如何通过明确的 transport、manifest 和 invocation context 进入服务端。
- **包含**：endpoint identity/status、SSH bridge 启动与重连、metadata cache、plugin manifest、插件启动/事件 hook 和 command log。
- **不包含**：真实远程部署、凭据托管实现、插件市场、任意文件系统访问。

## 场景

用户从本地客户端选择一个保存的 SSH endpoint。客户端以 profile id 标识它，读取缓存或发现远程 API metadata，启动 `SshStdioBridge`，再连接一个本地 bridge socket。远程失败分为可重试的 transport 问题和需要用户处理的协议/权限/版本问题。

另一个场景是插件声明 startup、event、action 或 pane。Herdr 先解析并规范化 manifest，再把当前事件、plugin id、workspace/pane 等放入 `PluginInvocationContext`，通过 `start_plugin_command` 启动受限命令并记录状态。插件不是服务端核心状态的拥有者。

## 源码调用链

| 源码 | 符号 | 责任 |
| --- | --- | --- |
| `src/client/endpoint.rs` | `ProfileId`, `ClientEndpointId`, `ClientEndpointStatus` | 生成/解析稳定 profile id、生成 storage key、表达连接状态 |
| `src/remote/host.rs` | `run_remote_client_bridge`, `ensure_remote_server_running` | 确保远程 server 可用并把 stdio 转给本地 client socket |
| `src/remote/saved.rs` | `SavedSshApiBridge::start`, `connect_saved_ssh`, `recover_transport` | 校验保存配置、读取/刷新 metadata、启动 SSH bridge 和本地 socket |
| `src/remote/attach.rs` | `SshStdioBridge`, `discover_remote_api_metadata` | SSH 命令、stdio transport 和远程能力发现 |
| `src/app/api/plugins/manifest.rs` | `load_plugin_manifest`, `normalize_manifest_*` | 解析 manifest、验证 id/version/platform/action/pane/event |
| `src/app/api/plugins/runtime.rs` | `App::start_plugin_command`, `run_plugin_startup_hooks`, `run_plugin_event_hooks` | 产生 plugin command log、环境变量、并发限制和结果事件 |
| `src/app/api/plugins/context.rs` | `current_plugin_context`, `plugin_context_for_event` | 生成只读的插件调用上下文 |
| `src/persist/plugin_registry.rs` | `load`, `reload_manifests`, `update` | 保存已安装插件清单并重新读取 manifest |

## 状态流

```text
ClientEndpointId
  → Connecting
  → metadata cache / discover_remote_api_metadata
  → SshStdioBridge + local endpoint socket
  → Online
  → transport failure → Reconnecting or Attention

Plugin registry
  → manifest parse/normalize
  → enabled + platform/capability check
  → PluginInvocationContext
  → start_plugin_command
  → Running → Finished/Failed
```

## 前置、产出与交接

- **前置**：[`01-server-client`](../01-server-client/V0-design.md) 的 endpoint/socket 边界和 [`03-session-persistence`](../03-session-persistence/V0-design.md) 的稳定 identity 规则。
- **V1 产出**：本地 endpoint 与 fake transport 的连接状态、缓存命中和 transport failure 分类。
- **下游**：H1 使用 endpoint 状态连接已有 server runtime；插件版本再消费 manifest 和 invocation context。
- **冻结条件**：transport 失败不能伪造 `Online`，插件 manifest 在缺少必需字段或声明未知 action 时不会启动命令。

## 局部版本线

- **V1**：本地 endpoint 与一个 fake transport 的连接状态。
- **V2**：SSH metadata discovery、缓存和可重试/需关注错误分类。
- **V3**：manifest 解析、能力/平台检查和 startup/event hook。
- **V4**：插件 command log、并发上限、远程插件边界和客户端状态投影。

## V1 检查

用 fake transport 模拟首次发现、缓存命中、断开和权限失败。断言 profile storage key 不包含 target/session，transport 失败不会伪造 Online，插件 manifest 缺少必需 id/version 或声明未知 action 时不会启动命令。

契约见 [domain-flow](contracts/domain-flow.md)、[state-and-errors](contracts/state-and-errors.md)、[interface-or-storage](contracts/interface-or-storage.md)，参考实现见 [reference-architecture](reference-architecture.md)。
