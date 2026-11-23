# Identity / Workspace：参考架构

```text
WorkspaceService → MembershipStore → AgentIdentity
                                      ↓
                              CapabilityScope / Presence
                                      ↓
                                  Raft SDK projection
```

学习实现为 `workspace.ts`、`agent_identity.ts`、`membership.ts`、`capability_scope.ts`。V1 检查 workspace 隔离、revoked identity 和 scope 拒绝。
