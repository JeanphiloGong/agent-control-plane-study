# Channel / Thread / Task：领域流

```text
Channel message → Thread context → Task command
  → visibility policy → claim/handoff → evidence reply
```

Thread 是上下文，Task 是可执行事实，Message 是交流事实，三者不共用一个 status。
