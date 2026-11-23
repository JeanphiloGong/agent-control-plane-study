# Sync / E2EE：接口契约

Document schema 输入版本化 Session data，输出合法 domain frame。Transport 输入 encrypted frame、device identity 和 cursor，输出 ack、remote frame 或明确 offline/conflict。E2EE 层只处理密钥和加密，不决定 Session 业务状态。
