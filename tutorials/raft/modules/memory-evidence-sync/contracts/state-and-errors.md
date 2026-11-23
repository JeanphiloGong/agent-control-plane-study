# Memory / Evidence / Sync：状态与错误

- evidence missing source、attachment unavailable、scope denied、sync offline 和 cursor stale 分开处理。
- context view 过期时需要标识版本，不能伪造最新。
- 重复 frame 幂等，冲突需要 domain policy。
- memory/evidence 引用不能自动授予机器或凭据 capability。
