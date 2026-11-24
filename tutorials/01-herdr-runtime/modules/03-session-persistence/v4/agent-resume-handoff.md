# Herdr Session Persistence v4：保存可重建的 Agent 元数据

[上一版 v3](../v3/migration-recovery.md) · [本版计划](plan.md) · [模块路线图](../plan.md)

用户退出 Herdr 时，pane 里可能有一个 Agent session。下一次启动时，session 文件可以告诉恢复层 Agent 名称、resume reference 和 worktree 归属，但它不应该直接在写 JSON 的函数里执行 shell 命令。否则一次普通保存就会产生进程副作用，也无法处理多个 pane 引用同一个 session 的情况。

真实 Herdr 在 `capture_tab` 中把 Agent session、reported resume 和 launch argv 放进 `PaneSnapshot`，在 `restore_workspace` 中根据配置生成 `AgentResumePlan`。live handoff 通过 `restore_handoff` 严格检查 imported runtime。下面的参考实现只保留数据边界和去重规则。

本版参考项目只冻结身份、resume reference 和 worktree。真实实现中的 `launch_argv` 与 Agent-specific session reference 仍由上游 `PaneAgentResumeSnapshot` 和 planner 负责，不能把本版的纯元数据检查当成 Agent 进程已经可以启动。

## 1. 扩展 pane 和 workspace 快照

继续修改 `tutorials/01-herdr-runtime/modules/03-session-persistence/v1/reference/session.py`。把 `PaneState`、`PaneSnapshot` 替换为下面的完整定义。默认值让 v1-v3 已有的调用仍然能创建没有 Agent 的 pane。

```python
@dataclass(frozen=True)
class PaneState:
    pane_id: int
    cwd: str
    agent: str | None = None
    resume_ref: str | None = None
    worktree: str | None = None


@dataclass(frozen=True)
class PaneSnapshot:
    pane_id: int
    cwd: str
    agent: str | None = None
    resume_ref: str | None = None
    worktree: str | None = None
```

将 `_pane_snapshot` 和 `_pane_state` 替换为完整实现：

```python
def _pane_snapshot(pane: PaneState) -> PaneSnapshot:
    return PaneSnapshot(
        pane_id=pane.pane_id,
        cwd=pane.cwd,
        agent=pane.agent,
        resume_ref=pane.resume_ref,
        worktree=pane.worktree,
    )


def _pane_state(pane: PaneSnapshot) -> PaneState:
    return PaneState(
        pane_id=pane.pane_id,
        cwd=pane.cwd,
        agent=pane.agent,
        resume_ref=pane.resume_ref,
        worktree=pane.worktree,
    )
```

`asdict` 会自动把这些字段写入 JSON，v1 的 `_load_pane` 仍需要显式读取新字段。将它替换为：

```python
def _load_pane(raw: dict[str, Any]) -> PaneSnapshot:
    return PaneSnapshot(
        pane_id=int(raw["pane_id"]),
        cwd=str(raw["cwd"]),
        agent=raw.get("agent"),
        resume_ref=raw.get("resume_ref"),
        worktree=raw.get("worktree"),
    )
```

为了让 history 指纹包含 worktree 归属，在 v3 的 `_layout_payload` 中给每个 tab 字典加入：

```python
"pane_worktrees": [pane.worktree for pane in tab.panes],
```

真实 Herdr 把 worktree 归属放在 `WorkspaceSnapshot::worktree_space`。本教程的最小模型把它放在 pane 上，重点是“归属随 snapshot 保存并参与恢复边界”，而不是复制 Rust 的 `WorktreeSpaceMembership` 类型。

## 2. 生成去重的 resume metadata

在 `session.py` 末尾加入：

```python
def resume_metadata(
    panes: tuple[PaneSnapshot, ...],
) -> tuple[dict[str, str], ...]:
    seen: set[str] = set()
    result: list[dict[str, str]] = []
    for pane in panes:
        if not pane.agent or not pane.resume_ref:
            continue
        if pane.resume_ref in seen:
            continue
        seen.add(pane.resume_ref)
        result.append({
            "agent": pane.agent,
            "resume_ref": pane.resume_ref,
            "worktree": pane.worktree or "",
        })
    return tuple(result)


def resume_metadata_for(snapshot: SessionSnapshot) -> tuple[dict[str, str], ...]:
    panes = tuple(
        pane
        for workspace in snapshot.workspaces
        for tab in workspace.tabs
        for pane in tab.panes
    )
    return resume_metadata(panes)
```

函数只返回数据。它不读取 resume 文件、不调用 subprocess、不改变 snapshot，也不把两个相同 `resume_ref` 的 pane 变成两个启动请求。真实恢复层可将返回值交给 Agent-specific planner，再由 `restore` 创建 terminal runtime。`resume_ref` 是去重键，不应使用 pane id，因为同一个 Agent 会话可能被不同的 pane 记录引用。

## 3. 保留 handoff 的纯恢复边界

在 `session.py` 末尾再加入一个明确的边界函数：

```python
def handoff_resume_plan(snapshot: SessionSnapshot) -> tuple[dict[str, str], ...]:
    """Return inputs for a runtime handoff; never start an Agent here."""
    return resume_metadata_for(snapshot)
```

调用者可以把 `handoff_resume_plan(snapshot)` 交给 live handoff 的接收方。接收方需要先确认 imported runtime 已经 ready，再提交所有权。这个参考函数没有实现跨进程 FD、旧 runtime fence 或失败回滚，因此不能被描述成 Herdr 的完整 handoff。

## 4. 冻结元数据恢复和去重

创建 `tutorials/01-herdr-runtime/modules/03-session-persistence/v1/reference/test_resume.py`：

```python
from session import (
    PaneSnapshot,
    SessionSnapshot,
    TabSnapshot,
    WorkspaceSnapshot,
    restore,
    resume_metadata,
)


def test_duplicate_resume_reference_is_emitted_once():
    panes = (
        PaneSnapshot(1, "/work/a", "codex", "session-7", "/work/a"),
        PaneSnapshot(2, "/work/a", "codex", "session-7", "/work/a"),
        PaneSnapshot(3, "/work/b", "codex", "session-8", "/work/b"),
        PaneSnapshot(4, "/work/c"),
    )

    assert resume_metadata(panes) == (
        {"agent": "codex", "resume_ref": "session-7", "worktree": "/work/a"},
        {"agent": "codex", "resume_ref": "session-8", "worktree": "/work/b"},
    )


def test_resume_metadata_is_data_and_preserves_worktree():
    pane = PaneSnapshot(1, "/work/project", "codex", "session-1", "/work/project")

    result = resume_metadata((pane,))

    assert result[0]["worktree"] == "/work/project"
    assert isinstance(result, tuple)


def test_restore_preserves_agent_metadata_without_starting_processes():
    snapshot = SessionSnapshot(
        2,
        (
            WorkspaceSnapshot(
                10,
                "/work/project",
                (
                    TabSnapshot(
                        20,
                        (PaneSnapshot(1, "/work/project", "codex", "session-1", "/work/project"),),
                        1,
                    ),
                ),
                0,
            ),
        ),
        0,
    )

    restored = restore(snapshot)

    pane = restored.workspaces[0].tabs[0].panes[0]
    assert pane.agent == "codex"
    assert pane.resume_ref == "session-1"
    assert pane.worktree == "/work/project"
```

从仓库根目录执行最终回放。本版继续使用 `v1/reference`。

```bash
cd tutorials/01-herdr-runtime/modules/03-session-persistence/v1/reference
python3 -m pytest -q
```

预期是 v1-v4 的所有测试通过。读者可以在交互式检查中确认 `handoff_resume_plan` 只返回 tuple，不产生子进程：

```bash
python3 - <<'PY'
from session import AppState, capture, handoff_resume_plan

snapshot = capture(AppState(workspaces=(), active_workspace=0))
assert handoff_resume_plan(snapshot) == ()
print("handoff plan is metadata only")
PY
```

## 检查点与后续模块

Session Persistence 到 v4 已经覆盖结构快照、原子提交、格式迁移、history 绑定、恢复副本和 Agent resume 元数据。它仍然没有复制 PTY bytes，也没有保证 Agent 原生 session 可用。下游 [04 Remote Endpoint / Plugin](../../04-remote-plugins/V0-design.md) 使用这里稳定的 session/worktree identity 接入 endpoint 和插件边界。
