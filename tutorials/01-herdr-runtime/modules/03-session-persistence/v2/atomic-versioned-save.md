# Herdr Session Persistence v2：用原子替换保存快照

[上一版 v1](../v1/snapshot-restore.md) · [本版计划](plan.md) · [下一版 v3](../v3/migration-recovery.md)

v1 的 `save(path, snapshot)` 能生成合法 JSON，但它直接打开主文件。若进程在写入一半时退出，下一次启动读到的可能是截断 JSON，而且最后一份可用布局没有单独副本。真实 Herdr 将这条责任放在 `io::save_to_path` 和 `SessionWriter`，而不是让 `App` 自己拼接 JSON。

## 1. 给保存过程增加提交边界

继续使用 `tutorials/01-herdr-runtime/modules/03-session-persistence/v1/reference/session.py`。在 import 区域加入 `os` 和 `shutil`：

```python
import json
import os
import shutil
from dataclasses import asdict, dataclass
```

保留 v1 的 `save`、`load` 和 snapshot 类型，在文件末尾加入完整的 `SnapshotWriter`：

```python

class SnapshotWriter:
    """Commit a snapshot without exposing a partially written main file."""

    def __init__(self, path: Path):
        self.path = path
        self.backup = path.with_suffix(path.suffix + ".bak")

    def save(self, snapshot: SessionSnapshot) -> None:
        self.path.parent.mkdir(parents=True, exist_ok=True)
        temporary = self.path.with_suffix(self.path.suffix + ".tmp")
        try:
            save(temporary, snapshot)
            if self.path.exists():
                # Copying leaves the current main file available if the next
                # replacement fails.
                shutil.copyfile(self.path, self.backup)
            os.replace(temporary, self.path)
        finally:
            temporary.unlink(missing_ok=True)
```

这里的顺序是保存契约的一部分。先写完整临时文件，`save(temporary, snapshot)` 成功后才处理旧文件。`copyfile` 生成 `.bak`，不会先把唯一的主文件移走。最后 `os.replace` 只替换同一目录中的完整文件。`finally` 清掉失败留下的临时文件。真实 Herdr 还会在写入后同步文件和父目录，并处理 symlink、平台替换差异，这些是本参考项目明确省略的边界。

## 2. 验证主文件、备份和失败保护

创建 `tutorials/01-herdr-runtime/modules/03-session-persistence/v1/reference/test_writer.py`：

```python
from dataclasses import replace

import pytest

import session
from session import SnapshotWriter, capture, load
from test_session import state_fixture


def test_second_save_keeps_previous_snapshot_in_backup(tmp_path):
    path = tmp_path / "session.json"
    writer = SnapshotWriter(path)
    first_state = state_fixture()
    first = capture(first_state)
    first_workspace = first_state.workspaces[0]
    first_tab = first_workspace.tabs[1]
    changed_pane = replace(first_tab.panes[1], cwd="/work/project/ui-v2")
    changed_tab = replace(first_tab, panes=(first_tab.panes[0], changed_pane))
    second_state = replace(
        first_state,
        workspaces=(replace(first_workspace, tabs=(first_workspace.tabs[0], changed_tab)),),
    )
    writer.save(first)
    writer.save(capture(second_state))

    assert load(path) == capture(second_state)
    assert load(writer.backup) == first


def test_failed_replace_keeps_main_file_and_removes_temp(tmp_path, monkeypatch):
    path = tmp_path / "session.json"
    writer = SnapshotWriter(path)
    original = capture(state_fixture())
    writer.save(original)
    before = path.read_bytes()

    def fail_replace(source, target):
        raise OSError("simulated replace failure")

    monkeypatch.setattr(session.os, "replace", fail_replace)
    with pytest.raises(OSError, match="simulated replace failure"):
        writer.save(original)

    assert path.read_bytes() == before
    assert not path.with_suffix(".json.tmp").exists()
```

第二个测试使用 monkeypatch 模拟 `os.replace` 失败。它验证的是 writer 自己的失败边界。第一个测试先保存一个 cwd，再保存一个变化后的 cwd，因此可以直接断言主文件是新快照，`.bak` 是旧快照。

从仓库根目录执行以下验收命令。本版继续使用 `v1/reference`。

```bash
cd tutorials/01-herdr-runtime/modules/03-session-persistence/v1/reference
python3 -m pytest -q
```

预期是 v1 的 2 个测试加 v2 的 2 个测试全部通过。若替换失败，旧主文件仍可加载，临时文件被清理。v2 的能力冻结在文件提交边界，`App` 何时调用 writer 仍由 v3 之前的真实 `schedule_session_save` 调度。

## 检查点与下一步

v2 解决了单次保存的提交完整性，但 `.bak` 只保留上一份文件，也不能解释旧版本如何升级。下一版增加从已知旧版本到当前版本的显式迁移，并把 pane history 与布局指纹绑定。
