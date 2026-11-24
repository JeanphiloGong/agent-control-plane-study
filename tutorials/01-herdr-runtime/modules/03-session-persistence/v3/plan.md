# Herdr Session Persistence v3：迁移、history 指纹与恢复副本

[模块路线图](../plan.md) · [上一版 v2](../v2/plan.md) · [本版正文](migration-recovery.md) · [下一版 v4](../v4/plan.md)

## 版本定位

v2 能安全提交一个快照，但格式会随产品演进，pane history 也不能脱离布局使用。v3 把这两个边界显式化：已知旧版本通过迁移进入当前内存模型，未知未来版本拒绝；history 文件写入结构指纹，恢复时指纹不匹配就忽略；writer 在保存前保留有限的结构恢复副本。

真实源码对应 `src/persist/snapshot.rs::migrate_snapshot`、`parse_snapshot`、`layout_fingerprint`、`capture_history`，`src/persist/restore.rs::restore_with_imports_and_failures`，以及 `src/persist/writer.rs::preserve_snapshot_history`。

## 实现顺序

1. 把 snapshot schema 从 v1 迁移到 v2，并让 `load` 只接受已知版本。
2. 增加 `layout_fingerprint`、history 的保存/读取和不匹配忽略。
3. 扩展 `SnapshotWriter` 生成有界 `session-snapshots/` 恢复副本，并测试保留数量。

## 验收与交接

`python3 -m pytest -q` 运行 v1、v2、v3 的测试，覆盖旧版本迁移、未来版本拒绝、history 匹配与不匹配以及副本上限。参考实现使用简单 JSON 写入恢复副本，真实 Herdr 还包含时间间隔、布局重复检测、平台同步和日志。v4 在当前 pane snapshot 上加入 Agent resume、worktree membership 和 handoff 的恢复边界。
