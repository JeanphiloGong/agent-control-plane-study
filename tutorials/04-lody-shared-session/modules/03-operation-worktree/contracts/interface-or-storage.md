# Operation / Worktree：接口契约

Operation store 保存 parent、child item、fingerprint、deadline、claim 和 completion。Worktree allocator 输入 session/operation id 和 workspace，输出唯一目录及 cleanup/review 信息。Watcher 只提交观察到的状态，不能替 child 生成结果。
