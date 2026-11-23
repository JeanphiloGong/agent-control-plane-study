# Session Persistence：参考架构

## 学习实现目录

```text
src/session/model.py       # WorkspaceSnapshot、PaneSnapshot、SessionSnapshot
src/session/capture.py     # 从 AppState 生成快照
src/session/parse.py       # 版本检查与迁移
src/session/restore.py     # 快照到运行时对象
src/session/writer.py      # 原子保存、旧文件保护、history
src/session/app_hooks.py   # dirty/debounce/startup 调用点
```

## 调用方向

```text
AppState ──capture──> SessionSnapshot ──writer──> session.json
session.json ──parse/migrate──> RestorePlan ──restore──> AppState + Runtime
                                   │
                         history fingerprint check
```

参考实现必须把 `Snapshot` 与 `Runtime` 分开。写入快照不直接停止或序列化 PTY，恢复快照会创建新的 runtime，并把创建失败记录为恢复缺口。

## 源码到参考架构

| Herdr | 参考实现 | V1 检查 |
| --- | --- | --- |
| `SessionSnapshot` / `PaneSnapshot` | 同名不可变数据结构 | JSON round-trip 保留结构 |
| `capture` | `capture_state` | active/selected/cwd 保留 |
| `parse_snapshot` | `parse_snapshot` | future version 被拒绝 |
| `restore_workspace` / `restore_tab` | `restore_state` | 新对象不重复旧 runtime |
| `SessionWriter::save` | `SnapshotWriter.save` | 旧文件保护和主文件提交分离 |
| `layout_fingerprint` | `layout_fingerprint` | history layout 不匹配时忽略 |

## V1 冻结检查

```text
snapshot = capture_state(app_state)
write(snapshot)
loaded = load()
restored = restore(loaded)
assert restored.layout == app_state.layout
assert load(future_version_json) == UnsupportedVersion
```

检查证明的是结构恢复和版本边界，不证明真实 PTY、Agent resume 或文件系统崩溃恢复已经实现。
