# Orca Session / Recovery：状态与错误契约

## 状态

~~~text
turn-started
  → live
  → stopped / failed / exited / unknown
  → inspect
  → retry same Task or preserve
~~~

## 错误边界

- start_unknown、operation_unknown、stop_unknown 都要求读取，不是成功或退出。
- 身份不匹配拒绝 stop、read、retry 和 release。
- retry predecessor 不是当前 attempt 时拒绝。
- failure cause 未变化时拒绝自动 retry。
- breaker 达到上限后保持 Task failed，不能新建 Task 绕过计数。
- 保留路径必须能说明是 worker 未退出、terminal unknown、dirty workspace 或未合入变更。

## 必测状态

1. unknown liveness 不能推进到 retry。
2. confirmed exited 且原因已解决才允许新 Dispatch。
3. 新 Dispatch 产生后，旧 Dispatch 的 report 不能影响当前 attempt。
4. cleanup 在 worker liveness 未确认时返回 keep。
