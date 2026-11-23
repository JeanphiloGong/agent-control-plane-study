# Failure / Review：接口契约

分类器输入 Run 错误和上下文，输出稳定 failure code、retryable、用户提示和 review requirement。Retry service 输入原 Issue/workflow identity 和延迟策略。Issue projection 只展示服务端事实，不重新推断 retryability。
