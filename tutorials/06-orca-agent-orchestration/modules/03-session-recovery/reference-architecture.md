# Orca Session / Recovery：参考架构

~~~text
HostObservation → LivenessStore → RecoveryCoordinator
                         ↓                 ↓
                   DispatchHistory    RetryGate
                         └────────→ CleanupEvidence
~~~

## 学习实现目录

~~~text
reference/recovery/liveness.ts       # live, exited and unknown observation
reference/recovery/stop.ts           # exact identity and stop receipt
reference/recovery/retry.ts          # same Task and predecessor gate
reference/recovery/preserve.ts       # worker evidence to cleanup blocker
reference/recovery.test.ts            # recovery and fencing checks
~~~

| Orca 源码 | 参考组件 | 验收 |
| --- | --- | --- |
| session-worker-liveness.ts | LivenessStore | timeout 不等于 exited |
| worker-stop-remote-receipt.ts | StopReceipt | unknown 保留未决状态 |
| recovery-and-cleanup.md | RetryGate | same Task、cause changed、breaker |
| workspace-cleanup.ts | CleanupEvidence | active/unknown worker 进入 keep |

它复用 02 的 Dispatch history 和 01 的 Workspace identity，不将 session recovery 变成一个
独立的 Task 状态机。
