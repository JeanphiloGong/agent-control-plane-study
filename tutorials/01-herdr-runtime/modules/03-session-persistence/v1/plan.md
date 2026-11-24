# Herdr Session Persistence v1：快照、加载与恢复

[模块路线图](../plan.md) · [V0 设计](../V0-design.md) · [本版正文](snapshot-restore.md) · [下一版 v2](../v2/plan.md)

## 版本定位

从 V0 的 `SessionSnapshot` 契约开始，先把一个 workspace、多个 tab、pane、cwd 和 active/selected 索引变成可读 JSON，再从 JSON 创建一份新的 runtime state。版本号是输入边界，未来版本不能被当成当前结构解析。

## 实现顺序

1. 在 `v1/reference/session.py` 建立不可变的 runtime state 与 snapshot 类型，并实现 `capture`、`save`、`load`、`restore`。
2. 在 `v1/reference/test_session.py` 验证结构 round-trip 和未来版本拒绝，同时确认拒绝不会修改原文件。

## 验收与交接

`python3 -m pytest -q` 检查结构事实的保存与恢复。v2 使用这个 `SessionSnapshot` 作为 writer 输入，加入临时文件和旧文件保护。
