# Event / Projection：状态与错误

- event 与 projection 的提交边界必须原子，提交失败不能让客户端看到半个事实。
- projection lag 要可观察，不能静默制造旧状态。
- 不支持的 schema/version 需要显式拒绝或迁移。
- replay 只能产生相同 projection，不得触发 Provider 外部副作用。
