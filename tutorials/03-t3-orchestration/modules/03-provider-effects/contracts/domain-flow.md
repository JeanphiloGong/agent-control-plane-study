# Provider / Effects：领域流

```text
committed domain event → pending effect → worker claim
  → provider session/turn → provider events
  → result ingress → domain event/projection
```

Effect 是待执行意图，Provider event 是外部执行观察，二者不能当作同一个事实。
