# Herdr Session Persistence v3：让格式演进和 history 有可信边界

[上一版 v2](../v2/atomic-versioned-save.md) · [本版计划](plan.md) · [下一版 v4](../v4/agent-resume-handoff.md)

一个用户可能用旧版本 Herdr 写入 session，再用新版本启动。直接把旧 JSON 当成当前版本会把缺失字段误认为空值，直接接受更高版本又可能丢字段。另一个问题是 pane ANSI history 与布局分离保存，如果用户新建或删除 pane 后仍加载旧 history，屏幕内容会归到错误的 pane。

真实 Herdr 在 `parse_snapshot` 中先判断版本，再走 `migrate_snapshot`。`restore_with_imports_and_failures` 将 history 限定在 `layout_fingerprint` 匹配的快照上。本版参考项目复现这两个判断，并增加有限恢复副本。

## 1. 把 v1 schema 显式迁移到 v2

继续修改 `tutorials/01-herdr-runtime/modules/03-session-persistence/v1/reference/session.py`。将版本常量改成 2，并在 import 区域加入 `hashlib`、`time`：

```python
import hashlib
import json
import os
import shutil
import time
```

将 v2 的 `load` 完整替换为下面的实现，并在它之前加入 `migrate_snapshot`：

```python
SNAPSHOT_VERSION = 2


def migrate_snapshot(raw: dict[str, Any]) -> dict[str, Any]:
    version = int(raw["version"])
    if version == SNAPSHOT_VERSION:
        return raw
    if version == 1:
        # v2 keeps the structural fields. Only the envelope version changes.
        return {**raw, "version": SNAPSHOT_VERSION}
    raise UnsupportedVersion(version)


def load(path: Path) -> SessionSnapshot:
    raw = json.loads(path.read_text(encoding="utf-8"))
    version = int(raw["version"])
    if version > SNAPSHOT_VERSION:
        raise UnsupportedVersion(version)
    raw = migrate_snapshot(raw)
    return SessionSnapshot(
        version=int(raw["version"]),
        workspaces=tuple(_load_workspace(workspace) for workspace in raw["workspaces"]),
        active_workspace=int(raw["active_workspace"]),
    )
```

这里的迁移不修改磁盘文件。`load` 把 v1 JSON 映射成 v2 内存对象，下一次成功保存才会写出 v2。版本 99 会在解析任何 workspace 前失败，避免未知结构被静默丢弃。真实源码的迁移还兼容更早的 legacy workspace 形态，本参考项目只承诺 v1 到 v2 这一条已知迁移。

## 2. 用布局指纹约束独立 history

在 `session.py` 中加入下面三个函数。指纹只依赖结构布局和 cwd，不依赖 history 文本，因此同一布局可以保存不同的屏幕内容。`save_history` 和 `load_history` 使用独立 JSON 文件。

```python
def _layout_payload(snapshot: SessionSnapshot) -> dict[str, Any]:
    return {
        "workspaces": [
            {
                "workspace_id": workspace.workspace_id,
                "cwd": workspace.cwd,
                "active_tab": workspace.active_tab,
                "tabs": [
                    {
                        "tab_id": tab.tab_id,
                        "selected_pane": tab.selected_pane,
                        "pane_ids": [pane.pane_id for pane in tab.panes],
                    }
                    for tab in workspace.tabs
                ],
            }
            for workspace in snapshot.workspaces
        ],
        "active_workspace": snapshot.active_workspace,
    }


def layout_fingerprint(snapshot: SessionSnapshot) -> str:
    encoded = json.dumps(
        _layout_payload(snapshot),
        ensure_ascii=False,
        sort_keys=True,
        separators=(",", ":"),
    ).encode("utf-8")
    return hashlib.sha256(encoded).hexdigest()


def save_history(path: Path, snapshot: SessionSnapshot, text: str) -> None:
    path.parent.mkdir(parents=True, exist_ok=True)
    path.write_text(
        json.dumps(
            {
                "version": snapshot.version,
                "layout_fingerprint": layout_fingerprint(snapshot),
                "text": text,
            },
            ensure_ascii=False,
            indent=2,
        ),
        encoding="utf-8",
    )


def load_history(path: Path, snapshot: SessionSnapshot) -> str | None:
    if not path.exists():
        return None
    raw = json.loads(path.read_text(encoding="utf-8"))
    if int(raw["version"]) > SNAPSHOT_VERSION:
        return None
    if raw.get("layout_fingerprint") != layout_fingerprint(snapshot):
        return None
    text = raw.get("text")
    return text if isinstance(text, str) else None
```

history 不匹配时返回 `None`，主布局仍然可以被 `load` 和 `restore` 使用。这正是“辅助数据失效不拖垮结构恢复”的边界。真实 `capture_history` 会按 workspace、tab、pane 保存 ANSI 和行数，本参考项目把内容压缩为一段文本以便突出指纹关系。

## 3. 给 writer 增加有限恢复副本

用下面的完整 `SnapshotWriter` 替换 v2 文件末尾的类。`backup` 仍表示上一份主文件，`recovery_dir` 保存更早的结构副本。副本数量由 `keep` 限制，旧副本先于新副本排序清理。

```python
class SnapshotWriter:
    def __init__(self, path: Path, keep: int = 3):
        if keep < 1:
            raise ValueError("keep must be positive")
        self.path = path
        self.backup = path.with_suffix(path.suffix + ".bak")
        self.recovery_dir = path.parent / "session-snapshots"
        self.keep = keep

    def _preserve_recovery(self) -> None:
        if not self.path.exists():
            return
        self.recovery_dir.mkdir(parents=True, exist_ok=True)
        timestamp = time.time_ns()
        target = self.recovery_dir / f"session-{timestamp}.json"
        while target.exists():
            timestamp += 1
            target = self.recovery_dir / f"session-{timestamp}.json"
        shutil.copyfile(self.path, target)
        files = sorted(self.recovery_dir.glob("session-*.json"))
        for old in files[:-self.keep]:
            old.unlink()

    def save(self, snapshot: SessionSnapshot) -> None:
        self.path.parent.mkdir(parents=True, exist_ok=True)
        temporary = self.path.with_suffix(self.path.suffix + ".tmp")
        try:
            save(temporary, snapshot)
            self._preserve_recovery()
            if self.path.exists():
                shutil.copyfile(self.path, self.backup)
            os.replace(temporary, self.path)
        finally:
            temporary.unlink(missing_ok=True)
```

本参考实现把恢复副本失败当成这次保存失败，因而不会在副本不可写时覆盖主文件。真实 `SessionWriter` 对 `session-snapshots` 的失败会记录 warning，并区分主布局保存和可选历史副本。这个差异是教学边界，不能把参考项目的异常策略当成 Rust 代码的逐行复制。

## 4. 冻结迁移、history 和副本边界

创建 `tutorials/01-herdr-runtime/modules/03-session-persistence/v1/reference/test_recovery.py`：

```python
from dataclasses import replace
import json

from session import (
    SnapshotWriter,
    capture,
    layout_fingerprint,
    load,
    load_history,
    save_history,
)
from test_session import state_fixture


def test_known_version_migrates_without_rewriting_input(tmp_path):
    path = tmp_path / "old.json"
    snapshot = capture(state_fixture())
    raw = json.loads(json.dumps(snapshot, default=lambda value: value.__dict__))
    raw["version"] = 1
    path.write_text(json.dumps(raw), encoding="utf-8")
    before = path.read_bytes()

    migrated = load(path)

    assert migrated.version == 2
    assert path.read_bytes() == before


def test_history_requires_matching_layout(tmp_path):
    path = tmp_path / "history.json"
    snapshot = capture(state_fixture())
    save_history(path, snapshot, "pane output")

    assert load_history(path, snapshot) == "pane output"
    changed_workspace = replace(snapshot.workspaces[0], cwd="/work/other")
    changed = replace(snapshot, workspaces=(changed_workspace,))
    assert layout_fingerprint(changed) != layout_fingerprint(snapshot)
    assert load_history(path, changed) is None


def test_recovery_copies_are_bounded(tmp_path):
    path = tmp_path / "session.json"
    writer = SnapshotWriter(path, keep=2)
    snapshot = capture(state_fixture())

    writer.save(snapshot)
    writer.save(snapshot)
    writer.save(snapshot)
    writer.save(snapshot)

    assert len(list(writer.recovery_dir.glob("session-*.json"))) == 2
```

`test_history_requires_matching_layout` 检查相同布局可以读到 history，workspace 的 `cwd` 变化后则返回 `None`。从仓库根目录运行累计测试。

```bash
cd tutorials/01-herdr-runtime/modules/03-session-persistence/v1/reference
python3 -m pytest -q
```

预期是 v1、v2、v3 的测试全部通过。真实源码还测试 legacy fixture、history 版本和失败 import，读者可以从 `sources/herdr/src/persist/snapshot.rs` 与 `src/persist/restore.rs` 继续对照。

## 检查点与下一步

V3 现在有了三种边界：未知 schema 不进入内存、history 不越过布局指纹、旧布局有有限恢复副本。下一版把 pane 中可重建的 Agent session、resume 参数和 worktree membership 放入 snapshot，但恢复只产生计划和元数据，不直接启动 Agent。
