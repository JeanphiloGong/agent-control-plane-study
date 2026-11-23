# Remote Endpoint / Plugin：参考架构

## 学习实现目录

```text
src/endpoints/model.py       # ProfileId、EndpointStatus
src/endpoints/transport.py   # Transport、FakeTransport、ReconnectPolicy
src/endpoints/bridge.py      # LocalSocketBridge / SshCommandBridge
src/plugins/manifest.py      # Manifest parser and capability checks
src/plugins/context.py       # InvocationContext
src/plugins/runner.py        # bounded command runner and command log
```

## 调用方向

```text
EndpointRegistry → MetadataProvider → Transport → LocalEndpointSocket
       ↓                                      ↓
 EndpointStatus                         ClientProjection

PluginRegistry → ManifestPolicy → InvocationContext → PluginRunner → CommandLog/Event
```

参考实现用 fake transport 和 fake process，不接触真实 SSH、凭据或用户插件目录。`EndpointRegistry` 只管理连接事实，`PluginRunner` 只执行通过 manifest 的命令。

## 源码到参考架构

| Herdr | 参考实现 | V1 检查 |
| --- | --- | --- |
| `ProfileId` / `ClientEndpointId` | `EndpointId` | storage key 不泄漏 target |
| `SavedSshApiBridge::start` | `EndpointConnector.connect` | metadata 缓存和发现分支可观察 |
| `recover_transport` | `ReconnectPolicy` | 仅允许声明的重试 |
| `load_plugin_manifest` | `ManifestParser.parse` | 非法 manifest 在 runner 前拒绝 |
| `start_plugin_command` | `PluginRunner.run` | log 状态和错误可查询 |

## V1 冻结检查

```text
endpoint = registry.connect(profile, fake_transport)
assert endpoint.status == Online
fake_transport.disconnect()
assert endpoint.status in {Reconnecting, Attention}

plugin = manifest_parser.parse(valid_manifest)
assert runner.run(plugin, context).status == Finished
assert runner.run(invalid_manifest, context).status == Rejected
```

这只证明 endpoint/plugin 的状态与边界模型，不证明 Herdr 的 SSH、平台进程或插件命令已经可用。
