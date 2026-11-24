# Sync / E2EE：领域流

```text
document mutation → schema validation → domain frame
  → encrypt/sign → transport → decrypt/validate → apply/cursor
```

文档状态、同步事实和设备密钥是三个边界，不能把密钥当作普通消息字段。
