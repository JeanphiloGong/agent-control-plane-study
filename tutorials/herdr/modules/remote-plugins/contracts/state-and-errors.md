# Remote Endpoint / Plugin：状态与错误

## Endpoint

- `Connecting` 只表示正在建立连接，不能被 UI 当作可执行。
- `Online` 必须有协议 generation/metadata 与本地期望匹配。
- transport 断开可以进入 `Reconnecting`，权限、host key、协议不兼容和远程未安装进入 `Attention`。
- stale metadata 可以失效缓存后重试一次；不能无限重试或把失败当作 Online。

## Plugin

- manifest parse、id/version/platform/action/pane/event 校验在启动命令前完成。
- 未启用、当前平台不支持、manifest 不存在或 action 未声明时不启动。
- command 启动失败写 `Failed` log；运行中和完成状态都要有 plugin id、log id 和错误上下文。
- 插件环境变量和 user/state 目录由 Herdr 生成，不能由插件把任意路径注入核心状态。
