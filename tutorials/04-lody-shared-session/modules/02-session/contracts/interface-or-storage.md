# Session Lifecycle：接口契约

Session API 输入 session id、turn input、control action 和 machine context，输出 session snapshot、turn event 或 control outcome。Session store 持久化状态，ExecutionService 才能产生 ACP 副作用。
