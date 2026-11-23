# Operation / Worktree：状态与错误

- child result、error、timeout、cancelled 和 unreadable 分开记录。
- materialization claim 过期后可恢复，但同一 child result 不能重复应用。
- worktree 创建失败不启动该 child。
- deadline 到期后 late result 需要标记 late/inspection，不伪造按时完成。
