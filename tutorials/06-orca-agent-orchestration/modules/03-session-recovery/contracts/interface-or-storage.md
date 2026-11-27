# Orca Session / Recovery：接口与存储契约

## 输入

- Dispatch ID、Task ID、owning Run 和 session/terminal identity
- 所属 host、liveness observation 或 stop receipt
- retry predecessor、cause-resolved 证据和 placement
- cleanup scan 使用的 workspace identity

## 输出

- worker-show/read 的当前状态和 next action
- stop、abandon 或 release receipt
- 同一 Task 的新 Dispatch 或明确拒绝
- cleanup 使用的 agent-running、terminal-running、terminal-unknown 等 blocker

## 边界

- Recovery 通过 Dispatch ID 查询，不通过模糊名称操作。
- Retry 不删除 predecessor，不改变原始 failure count。
- Cleanup 只接收证据，不接受 recovery 的“应该可以删”结论。
- 远程读取失败返回 unverified/keep，不能转换成 succeeded 或 exited。
