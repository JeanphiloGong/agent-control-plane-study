# Herdr Session Persistence v1：把 workspace 布局变成可恢复快照

[V0 设计](../V0-design.md) · [模块路线图](../plan.md) · [本版计划](plan.md) · [下一版 v2](../v2/atomic-versioned-save.md)

用户在一个 workspace 中打开两个 tab，每个 tab 有 pane，并在 pane 中进入不同的 cwd。退出后，下一次启动必须得到相同的结构事实。当前缺少的是一个明确的持久化模型。若直接把运行时对象写进文件，恢复层会把不可序列化的 PTY 和 TUI 引用混入存储。V1 先把可重建字段分成两组：运行时 state 和纯数据 snapshot。

真实 Herdr 的对应入口是 `src/persist/snapshot.rs::SessionSnapshot`、`capture`、`parse_snapshot`，以及 `src/persist/restore.rs::restore`。下面的 Python 项目只保留同一条数据流。

参考项目需要 Python 3.10 或更新版本，以及当前 Python 环境中的 pytest。从仓库根目录执行以下命令创建参考项目。

```bash
mkdir -p tutorials/01-herdr-runtime/modules/03-session-persistence/v1/reference
```

## 1. 定义运行时 state 与 snapshot

创建 `tutorials/01-herdr-runtime/modules/03-session-persistence/v1/reference/session.py`。这些类型只保存教学所需字段。`*State` 表示进程内对象，`*Snapshot` 表示可以写入 JSON 的数据。两者字段相似，但恢复函数会创建新的 state 对象。

```python
from __future__ import annotations

import json
from dataclasses import asdict, dataclass
from pathlib import Path
from typing import Any


SNAPSHOT_VERSION = 1


class UnsupportedVersion(ValueError):
    pass


@dataclass(frozen=True)
class PaneState:
    pane_id: int
    cwd: str


@dataclass(frozen=True)
class TabState:
    tab_id: int
    panes: tuple[PaneState, ...]
    selected_pane: int


@dataclass(frozen=True)
class WorkspaceState:
    workspace_id: int
    cwd: str
    tabs: tuple[TabState, ...]
    active_tab: int


@dataclass(frozen=True)
class AppState:
    workspaces: tuple[WorkspaceState, ...]
    active_workspace: int


@dataclass(frozen=True)
class PaneSnapshot:
    pane_id: int
    cwd: str


@dataclass(frozen=True)
class TabSnapshot:
    tab_id: int
    panes: tuple[PaneSnapshot, ...]
    selected_pane: int


@dataclass(frozen=True)
class WorkspaceSnapshot:
    workspace_id: int
    cwd: str
    tabs: tuple[TabSnapshot, ...]
    active_tab: int


@dataclass(frozen=True)
class SessionSnapshot:
    version: int
    workspaces: tuple[WorkspaceSnapshot, ...]
    active_workspace: int


def _pane_snapshot(pane: PaneState) -> PaneSnapshot:
    return PaneSnapshot(pane.pane_id, pane.cwd)


def _tab_snapshot(tab: TabState) -> TabSnapshot:
    return TabSnapshot(
        tab.tab_id,
        tuple(_pane_snapshot(pane) for pane in tab.panes),
        tab.selected_pane,
    )


def capture(state: AppState) -> SessionSnapshot:
    return SessionSnapshot(
        version=SNAPSHOT_VERSION,
        workspaces=tuple(
            WorkspaceSnapshot(
                workspace.workspace_id,
                workspace.cwd,
                tuple(_tab_snapshot(tab) for tab in workspace.tabs),
                workspace.active_tab,
            )
            for workspace in state.workspaces
        ),
        active_workspace=state.active_workspace,
    )


def _pane_state(pane: PaneSnapshot) -> PaneState:
    return PaneState(pane.pane_id, pane.cwd)


def _tab_state(tab: TabSnapshot) -> TabState:
    return TabState(
        tab.tab_id,
        tuple(_pane_state(pane) for pane in tab.panes),
        tab.selected_pane,
    )


def restore(snapshot: SessionSnapshot) -> AppState:
    return AppState(
        workspaces=tuple(
            WorkspaceState(
                workspace.workspace_id,
                workspace.cwd,
                tuple(_tab_state(tab) for tab in workspace.tabs),
                workspace.active_tab,
            )
            for workspace in snapshot.workspaces
        ),
        active_workspace=snapshot.active_workspace,
    )


def save(path: Path, snapshot: SessionSnapshot) -> None:
    path.parent.mkdir(parents=True, exist_ok=True)
    path.write_text(
        json.dumps(asdict(snapshot), ensure_ascii=False, indent=2),
        encoding="utf-8",
    )


def _load_pane(raw: dict[str, Any]) -> PaneSnapshot:
    return PaneSnapshot(int(raw["pane_id"]), str(raw["cwd"]))


def _load_tab(raw: dict[str, Any]) -> TabSnapshot:
    return TabSnapshot(
        int(raw["tab_id"]),
        tuple(_load_pane(pane) for pane in raw["panes"]),
        int(raw["selected_pane"]),
    )


def _load_workspace(raw: dict[str, Any]) -> WorkspaceSnapshot:
    return WorkspaceSnapshot(
        int(raw["workspace_id"]),
        str(raw["cwd"]),
        tuple(_load_tab(tab) for tab in raw["tabs"]),
        int(raw["active_tab"]),
    )


def load(path: Path) -> SessionSnapshot:
    raw = json.loads(path.read_text(encoding="utf-8"))
    version = int(raw["version"])
    if version > SNAPSHOT_VERSION:
        raise UnsupportedVersion(version)
    return SessionSnapshot(
        version=version,
        workspaces=tuple(_load_workspace(workspace) for workspace in raw["workspaces"]),
        active_workspace=int(raw["active_workspace"]),
    )
```

`capture` 只读取 state，`save` 只负责 JSON，`load` 只负责版本边界和数据解析，`restore` 再创建新的 state。这个顺序对应 Herdr 的 `capture → io → parse → restore`，也说明了为什么快照不是 PTY 本身。

## 2. 用一个 round-trip 检查冻结 V1

创建 `tutorials/01-herdr-runtime/modules/03-session-persistence/v1/reference/test_session.py`：

```python
import json

import pytest

from session import (
    AppState,
    PaneState,
    TabState,
    UnsupportedVersion,
    WorkspaceState,
    capture,
    load,
    restore,
    save,
)


def state_fixture() -> AppState:
    return AppState(
        workspaces=(
            WorkspaceState(
                10,
                "/work/project",
                (
                    TabState(20, (PaneState(1, "/work/project"),), 1),
                    TabState(
                        21,
                        (PaneState(2, "/work/project/api"), PaneState(3, "/work/project/ui")),
                        3,
                    ),
                ),
                1,
            ),
        ),
        active_workspace=0,
    )


def test_capture_save_load_restore_preserves_structure(tmp_path):
    original = state_fixture()
    path = tmp_path / "session.json"

    save(path, capture(original))
    restored = restore(load(path))

    assert restored == original
    assert restored.workspaces[0].tabs[1].panes[1].cwd == "/work/project/ui"
    assert restored.workspaces[0].active_tab == 1


def test_future_version_is_rejected_without_rewriting_file(tmp_path):
    path = tmp_path / "session.json"
    save(path, capture(state_fixture()))
    before = path.read_bytes()
    raw = json.loads(before)
    raw["version"] = 99
    path.write_text(json.dumps(raw), encoding="utf-8")
    future = path.read_bytes()

    with pytest.raises(UnsupportedVersion):
        load(path)

    assert path.read_bytes() == future
    assert before != future
```

从仓库根目录执行以下验收命令。

```bash
cd tutorials/01-herdr-runtime/modules/03-session-persistence/v1/reference
python3 -m pytest -q
```

预期是 `2 passed`。第一个断言证明 workspace、tab、pane、cwd 和 active/selected 索引穿过 JSON 后仍一致。第二个断言证明未来版本在读取边界失败，读取函数不会把文件改写成当前格式。它不证明损坏文件可以恢复，损坏输入的日志和继续运行策略由真实 `persist::load` 负责。

## 检查点与下一步

V1 冻结的是结构快照，不是可靠保存。直接写主文件时进程可能在写入中途退出，旧文件也没有恢复副本。下一版保留这些类型和 `save` 的序列化结果，把提交动作交给 `SnapshotWriter`。
