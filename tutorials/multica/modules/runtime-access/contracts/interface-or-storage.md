# Runtime Access：接口契约

`RuntimeLookup` 输入 Agent、workspace/member 和 Runtime 标识，输出 Runtime record、access verdict 和启动所需的 adapter input。adapter 不接收未通过 policy 的调用，也不持久化 Issue 状态。
