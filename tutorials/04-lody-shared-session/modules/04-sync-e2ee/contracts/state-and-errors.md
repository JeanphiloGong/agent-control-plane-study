# Sync / E2EE：状态与错误

- schema invalid、decrypt failed、cursor expired、conflict 和 transport offline 分开分类。
- 已确认 frame 重放必须幂等。
- 密钥不存在时进入 key-required，而不是清空本地文档。
- 冲突解决不能自动扩大 Session/Operation 权限。
