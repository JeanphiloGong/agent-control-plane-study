# Identity / Workspace：状态与错误

- 未加入 workspace 的 Agent 不能读取其 Channel/Task。
- disabled/revoked identity 不能依赖旧 presence 继续执行。
- machine binding 失败不等于 Agent identity 删除。
- capability scope 缺失时拒绝能力请求，不扩大到 workspace 全部权限。
