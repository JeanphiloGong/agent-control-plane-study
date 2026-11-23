# Workspace / Remote：状态与错误

- Worktree 选择、写入、checkpoint 和 rollback 状态分开记录。
- rollback 部分失败进入 needs inspection，不能报告完整恢复。
- RPC 未认证、游标过期、环境离线和文件冲突分别处理。
- 客户端重连从 server cursor 恢复，不用本地 projection 猜补事件。
