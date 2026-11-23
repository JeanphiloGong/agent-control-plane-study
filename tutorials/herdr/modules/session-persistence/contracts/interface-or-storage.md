# Session Persistence：存储契约

## 主文件

主 JSON 至少包含 `version`、`workspaces`、`active` 和 `selected`。workspace 包含 identity cwd、tab 列表、public number 和 active tab。tab 包含 layout、pane snapshot、focus、root pane 和 zoom。pane 只保存 cwd、label、Agent/launch/resume 信息等可重建字段。

## 辅助文件

- `session-history.json` 保存可选 pane ANSI 历史和匹配布局 fingerprint。
- `session-backups/` 保存遇到 unload/失败保存前的旧文件副本。
- `session-snapshots/` 按时间保留有限的结构快照。

## 存储边界

`io::save_to_path` 负责安全写入和替换，`SessionWriter` 负责何时保护旧文件以及何时清理副本。应用层通过 `App::session_writer` 使用它们，不在 App 里重新拼 JSON。
