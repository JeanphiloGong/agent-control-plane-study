# Herdr Session Persistence v2：debounce 之后的安全提交

[模块路线图](../plan.md) · [上一版 v1](../v1/plan.md) · [本版正文](atomic-versioned-save.md) · [下一版 v3](../v3/plan.md)

## 版本定位

v1 已经能生成结构快照，但 `Path.write_text` 直接覆盖主文件。v2 把保存边界收敛到 `SnapshotWriter`：先在同一目录写临时文件，再把旧主文件复制到 `.bak`，最后用 `os.replace` 提交新文件。临时文件和主文件在同一文件系统中，替换操作不会留下半个 JSON。

真实源码对应 `src/persist/io.rs::save_to_path` 和 `src/app/session.rs::start_background_session_save`。本版只模拟 writer 的文件提交，不把线程 debounce 假装成已经完成的并发调度。

## 实现顺序

1. 在 v1 的 `session.py` 中加入 `SnapshotWriter`，保留原来的 `save` 作为纯序列化函数。
2. 在 `test_writer.py` 中检查第二次保存的主文件、`.bak` 和替换失败时旧主文件的完整性。

## 验收与交接

`python3 -m pytest -q` 运行 v1 和 v2 的测试，检查替换失败时旧主文件仍可读取。v3 在这个 writer 上加入旧版本迁移、history 指纹和有界恢复副本。
