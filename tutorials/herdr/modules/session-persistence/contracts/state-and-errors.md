# Session Persistence：状态与错误

## 必须保持的规则

- `SNAPSHOT_VERSION` 高于当前版本时拒绝解析。
- 缺失旧字段使用明确的 serde default 或迁移逻辑，不能把未知结构静默当成空 session。
- history 只有在 `layout_fingerprint` 与当前结构一致时才可恢复，否则忽略 history 而保留主布局。
- 保存前的旧文件保护失败时，若可能丢失唯一旧布局，writer 应保持保护状态并拒绝覆盖。
- history/恢复副本是辅助数据，辅助数据失败不能把已经提交的主布局错误标记为未保存。
- 清空空 workspace 是明确的 clear 操作，不等于保存失败。

## 错误归属

JSON 解析和版本错误由 `persist::load`/`parse_snapshot` 记录。布局映射和 runtime 重建错误由 `restore` 记录并返回失败项。文件替换、备份和权限错误由 `SessionWriter` 记录。App 负责决定是否继续运行，而不是让每层重复吞错。
