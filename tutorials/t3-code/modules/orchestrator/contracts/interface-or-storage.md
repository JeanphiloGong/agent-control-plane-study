# Command / Orchestrator：接口契约

Command 输入包含 thread/environment、command id、消息或控制意图。输出是 receipt、事件引用或明确的 policy error。客户端传输 schema 由 `packages/contracts` 所有，UI 不自行扩展命令枚举。
