# Issue / Invocation：接口契约

输入需要 Issue 内容、workspace/member、Agent 标识和可选 Runtime。输出包含 Issue 事实、readiness verdict 和可执行的 Run 请求。客户端只能消费 verdict 和 Issue projection，不能提交一个绕过 `CanMemberInvokeAgent` 的 Runner 参数。
