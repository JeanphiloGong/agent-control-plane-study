# Session Lifecycle：领域流

```text
create → ready/running → turn events → completed/failed
                         ↓
                 steer/cancel/archive/resume
```

Session 状态记录长期事实，Turn 状态记录一次 provider 执行，Machine 状态记录可达性。
