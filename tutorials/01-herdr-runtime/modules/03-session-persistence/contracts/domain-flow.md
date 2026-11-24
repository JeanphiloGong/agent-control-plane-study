# Session Persistence：领域流

## 对象

| 对象 | 作用 |
| --- | --- |
| `AppState` | 运行中的 workspace、terminal 和 selection |
| `SessionSnapshot` | 可序列化的结构布局和恢复信息 |
| `SessionHistorySnapshot` | 与布局指纹绑定的 pane 历史 |
| `RestorePlan` | 从快照重建 workspace、terminal 和 runtime 的中间结果 |
| `SessionWriter` | 保护旧文件并提交新快照的持久化边界 |

## 领域流

```text
App mutation → session_dirty → capture → write
restart → load/parse/migrate → restore → new runtime
```

`capture` 读 App 事实，`restore` 创建新的运行时对象。保存层不拥有业务状态，恢复层不把 JSON 当成已经运行的 PTY。
