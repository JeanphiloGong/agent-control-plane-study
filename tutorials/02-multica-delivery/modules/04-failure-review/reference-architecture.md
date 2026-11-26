# Failure / Review：参考架构

~~~text
Run terminal result
       │
       ▼
taskfailure.Classify → Decision
       │                  ├─ Retryable → RetryPlan → queued/deferred child
       │                  └─ ReviewRequired → blocked/in_review
       ▼
IssueWakeupService.Tick → condition + receipt → next Run
       │
       ▼
Issue projection / client run steering
~~~

TaskService.FailTask 拥有 task 终态和 retry child 的事务。IssueWakeupService 拥有 wakeup 行锁、条件判断和 receipt 去重。客户端读取服务端状态。教程 reference 用纯 Go 值对象演示规则，真实 PostgreSQL 事务与 HTTP 入口仍由源码拥有。

V1 冻结分类与 Review/Retry 分离。V2 冻结次数、延迟和 workflow handoff。V3 冻结 Wakeup 条件、一次性去重和 in_review 投影。
