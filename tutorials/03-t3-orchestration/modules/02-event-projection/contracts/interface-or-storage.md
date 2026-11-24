# Event / Projection：接口契约

Event 至少包含 event id、aggregate/thread id、sequence、type、payload 和 created time。Projection 查询返回服务端读模型与 cursor。订阅者可以从指定 cursor 恢复，不提交客户端生成的 event。
