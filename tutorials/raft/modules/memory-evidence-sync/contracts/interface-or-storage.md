# Memory / Evidence / Sync：接口契约

ContextView 输入 Agent identity、scope 和 session activity，输出脱敏的消息/文件/任务摘要。EvidenceReceipt 输入 source、hash、attachment reference 和 producer，输出可核验引用。SyncDomain 输入授权 domain frame，输出 ack/cursor/snapshot。
