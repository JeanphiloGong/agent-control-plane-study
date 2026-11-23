# Channel / Thread / Task：参考架构

```text
ChannelService → VisibilityPolicy → ThreadStore
                                      ↓
                                 TaskService
                                      ↓
                                Claim/Handoff
```

学习实现为 `channel.ts`、`thread.ts`、`task.ts` 和 `visibility_policy.ts`。V1 检查跨 scope 不可见、重复 claim 和 evidence reference handoff。
