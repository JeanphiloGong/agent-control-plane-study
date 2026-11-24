# Channel / Thread / Task：接口契约

Message API 输入 channel/thread、sender identity 和 content。Task API 输入 task spec、visibility scope 和 claim identity，输出 task state/owner。Handoff 使用 message/task reference，不复制整套 Agent credential 或上下文。
