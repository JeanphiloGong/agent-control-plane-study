# Daemon / Credentials：接口契约

Daemon API 输入 Agent identity、machine、launch spec 和 capability scope，输出 process handle/activity 或明确 failure。Credential proxy 输入 secret reference 和 scope，输出短期注入结果，不返回原始 secret 给 Server。
