# Workspace / Remote：接口契约

Project/Checkpoint API 输入 project、thread、worktree 和 expected revision，输出 checkpoint/diff/rollback result。ThreadStream 输入认证上下文和 cursor，输出有序事件。客户端 runtime 只保存 projection 和 cursor，不接受任意文件路径写入。
