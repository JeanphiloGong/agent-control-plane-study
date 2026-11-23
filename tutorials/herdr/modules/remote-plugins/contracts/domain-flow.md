# Remote Endpoint / Plugin：领域流

## Endpoint 流

```text
endpoint profile
  → validate profile/session
  → load or discover remote metadata
  → prepare shared SSH transport
  → start stdio bridge
  → connect local endpoint socket
  → Online / Reconnecting / Attention
```

## Plugin 流

```text
installed registry
  → load_plugin_manifest
  → normalize and validate
  → enabled/platform/event match
  → PluginInvocationContext
  → start_plugin_command
  → command log + completion event
```

Endpoint 负责连接外部运行时，Plugin 负责受声明约束的扩展命令。两者都通过边界适配器调用 App，不能直接修改核心 workspace 或绕过授权。
