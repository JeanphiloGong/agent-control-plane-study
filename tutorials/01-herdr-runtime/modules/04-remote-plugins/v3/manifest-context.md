# Herdr Remote Endpoint / Plugin v3：在启动命令前冻结声明和上下文

[上一版 v2](../v2/metadata-reconnect.md) · [本版计划](plan.md) · [下一版 v4](../v4/command-runner.md)

插件 manifest 既是用户输入，也是命令启动的权限边界。若解析阶段只读取 `command`，启动时才发现空 id、未知平台或 action 不存在，错误已经越过了最容易审计的边界。真实 Herdr 先读取 manifest，规范化所有字段，再由 runtime 根据 enabled、平台和事件选择命令。

## 1. 定义 manifest 模型和规范化器

创建 `tutorials/01-herdr-runtime/modules/04-remote-plugins/v1/reference/src/plugins/__init__.py`，再创建 `manifest.py`：

```python
from __future__ import annotations

from dataclasses import dataclass
import re
from typing import Any


KNOWN_PLATFORMS = {"linux", "macos", "windows"}
KNOWN_EVENTS = {"workspace.created", "pane.focused", "pane.exited"}
PLUGIN_ID_RE = re.compile(r"^[A-Za-z0-9:._-]{1,120}$")
ACTION_ID_RE = re.compile(r"^[A-Za-z0-9:_-]{1,120}$")


class ManifestError(ValueError):
    pass


@dataclass(frozen=True)
class PluginAction:
    action_id: str
    title: str
    command: tuple[str, ...]
    contexts: tuple[str, ...]
    platforms: tuple[str, ...] | None


@dataclass(frozen=True)
class PluginEvent:
    event: str
    command: tuple[str, ...]
    platforms: tuple[str, ...] | None


@dataclass(frozen=True)
class PluginManifest:
    plugin_id: str
    name: str
    version: str
    min_herdr_version: str
    enabled: bool
    plugin_root: str
    platforms: tuple[str, ...] | None
    startup: tuple[tuple[str, ...], ...]
    actions: tuple[PluginAction, ...]
    events: tuple[PluginEvent, ...]
    warnings: tuple[str, ...]
    source: str = "local"

    def action(self, action_id: str) -> PluginAction | None:
        return next((item for item in self.actions if item.action_id == action_id), None)


def _required_string(raw: Any, field: str) -> str:
    if not isinstance(raw, str) or not raw.strip():
        raise ManifestError(f"{field} is required")
    return raw.strip()


def _version(value: str, field: str) -> tuple[int, int, int]:
    parts = value.split(".")
    if len(parts) != 3 or any(not part.isdigit() for part in parts):
        raise ManifestError(f"{field} must be a semantic version")
    return tuple(int(part) for part in parts)  # type: ignore[return-value]


def _platforms(raw: Any) -> tuple[str, ...] | None:
    if raw is None:
        return None
    if not isinstance(raw, list) or not raw or any(item not in KNOWN_PLATFORMS for item in raw):
        raise ManifestError("platforms must contain known platform names")
    return tuple(dict.fromkeys(raw))


def _command(raw: Any) -> tuple[str, ...]:
    if not isinstance(raw, list) or not raw or any(not isinstance(item, str) or not item for item in raw):
        raise ManifestError("command must contain non-empty argv strings")
    return tuple(raw)


def _action(raw: Any) -> PluginAction:
    if not isinstance(raw, dict):
        raise ManifestError("action must be an object")
    action_id = _required_string(raw.get("id"), "action id")
    if not ACTION_ID_RE.fullmatch(action_id):
        raise ManifestError("action id contains unsupported characters")
    title = _required_string(raw.get("title"), "action title")
    contexts = raw.get("contexts", [])
    if not isinstance(contexts, list) or any(item not in {"global", "workspace", "tab", "pane", "selection"} for item in contexts):
        raise ManifestError("action contexts are invalid")
    return PluginAction(
        action_id,
        title,
        _command(raw.get("command")),
        tuple(dict.fromkeys(contexts)),
        _platforms(raw.get("platforms")),
    )


def _event(raw: Any) -> PluginEvent:
    if not isinstance(raw, dict):
        raise ManifestError("event must be an object")
    event = _required_string(raw.get("on"), "event name")
    return PluginEvent(event, _command(raw.get("command")), _platforms(raw.get("platforms")))


def parse_manifest(
    raw: dict[str, Any],
    *,
    enabled: bool = True,
    plugin_root: str = ".",
    current_version: str = "1.0.0",
    source: str = "local",
) -> PluginManifest:
    plugin_id = _required_string(raw.get("id"), "plugin id")
    if not PLUGIN_ID_RE.fullmatch(plugin_id):
        raise ManifestError("plugin id contains unsupported characters")
    name = _required_string(raw.get("name"), "plugin name")
    version = _required_string(raw.get("version"), "plugin version")
    min_version = _required_string(raw.get("min_herdr_version"), "min_herdr_version")
    if _version(min_version, "min_herdr_version") > _version(current_version, "current version"):
        raise ManifestError("plugin requires a newer Herdr")
    raw_actions = raw.get("actions", [])
    if not isinstance(raw_actions, list):
        raise ManifestError("actions must be a list")
    actions = tuple(_action(item) for item in raw_actions)
    ids = [item.action_id for item in actions]
    if len(set(ids)) != len(ids):
        raise ManifestError("duplicate action id")
    raw_events = raw.get("events", [])
    if not isinstance(raw_events, list):
        raise ManifestError("events must be a list")
    events = tuple(_event(item) for item in raw_events)
    startup_raw = raw.get("startup", [])
    if not isinstance(startup_raw, list):
        raise ManifestError("startup must be a list")
    startup = tuple(_command(item) for item in startup_raw)
    warnings = tuple(
        f"unknown event '{item.event}'"
        for item in events
        if item.event not in KNOWN_EVENTS
    )
    return PluginManifest(
        plugin_id,
        name,
        version,
        min_version,
        enabled,
        plugin_root,
        _platforms(raw.get("platforms")),
        startup,
        tuple(sorted(actions, key=lambda item: item.action_id)),
        events,
        warnings,
        source,
    )


def effective_platforms(
    item: tuple[str, ...] | None,
    plugin: tuple[str, ...] | None,
) -> tuple[str, ...] | None:
    return item if item is not None else plugin


def platform_supported(platforms: tuple[str, ...] | None, platform: str) -> bool:
    return platforms is None or platform in platforms
```

参考输入是已经解析成 dict 的 manifest，真实入口仍是 TOML `load_plugin_manifest`。`parse_manifest` 将 action 排序、拒绝重复 id、保留未知 event 为 warning，并要求所有启动命令是非空 argv。未知 event 不会自动变成可执行事件，v4 只选择 `KNOWN_EVENTS`。

## 2. 定义只读 invocation context 和匹配规则

创建 `tutorials/01-herdr-runtime/modules/04-remote-plugins/v1/reference/src/plugins/context.py`：

```python
from __future__ import annotations

from dataclasses import dataclass, replace

from .manifest import PluginAction, PluginManifest, effective_platforms, platform_supported


@dataclass(frozen=True)
class InvocationContext:
    workspace_id: str | None = None
    workspace_cwd: str | None = None
    tab_id: str | None = None
    pane_id: str | None = None
    pane_agent: str | None = None
    pane_status: str | None = None
    selected_text: str | None = None
    invocation_source: str | None = None
    correlation_id: str | None = None
    clicked_url: str | None = None
    link_handler_id: str | None = None


def merge_context(base: InvocationContext, provided: InvocationContext | None) -> InvocationContext:
    if provided is None:
        return base
    values = {
        field: getattr(provided, field) or getattr(base, field)
        for field in base.__dataclass_fields__
    }
    return replace(base, **values)


def context_kind(context: InvocationContext) -> str:
    if context.selected_text:
        return "selection"
    if context.pane_id:
        return "pane"
    if context.tab_id:
        return "tab"
    if context.workspace_id:
        return "workspace"
    return "global"


def select_action(
    manifest: PluginManifest,
    action_id: str,
    context: InvocationContext,
    *,
    platform: str,
) -> PluginAction:
    if not manifest.enabled:
        raise ValueError("plugin is disabled")
    action = manifest.action(action_id)
    if action is None:
        raise ValueError("action is not declared")
    platforms = effective_platforms(action.platforms, manifest.platforms)
    if not platform_supported(platforms, platform):
        raise ValueError("action is unsupported on this platform")
    if action.contexts and context_kind(context) not in action.contexts:
        raise ValueError("action context does not match invocation")
    return action
```

`InvocationContext` 是本次调用的只读输入，`merge_context` 只允许调用方提供字段覆盖当前 App 快照中的非空字段。`select_action` 先检查 enabled、action 是否存在和平台，再检查 action 声明的 context。runner 只能接收它返回的 `PluginAction`。

## 3. 冻结 V3

创建 `tutorials/01-herdr-runtime/modules/04-remote-plugins/v1/reference/test_plugins.py`：

```python
import pytest

from plugins.context import InvocationContext, merge_context, select_action
from plugins.manifest import ManifestError, parse_manifest


VALID = {
    "id": "example.tools",
    "name": "Example Tools",
    "version": "0.1.0",
    "min_herdr_version": "1.0.0",
    "platforms": ["linux"],
    "actions": [
        {
            "id": "inspect",
            "title": "Inspect",
            "contexts": ["pane", "selection"],
            "command": ["inspect-plugin", "--json"],
        }
    ],
    "events": [{"on": "pane.focused", "command": ["on-focus"]}],
}


def test_manifest_normalizes_actions_and_keeps_unknown_event_as_warning():
    raw = {**VALID, "events": [*VALID["events"], {"on": "future.event", "command": ["future"]}]}

    manifest = parse_manifest(raw, plugin_root="/plugins/example")

    assert manifest.action("inspect").command == ("inspect-plugin", "--json")
    assert manifest.plugin_root == "/plugins/example"
    assert manifest.warnings == ("unknown event 'future.event'",)


@pytest.mark.parametrize(
    "change",
    [
        {"id": "bad id"},
        {"min_herdr_version": "2.0.0"},
        {"actions": [{"id": "inspect", "title": "", "command": ["run"]}]},
        {"actions": [
            {"id": "inspect", "title": "One", "command": ["run"]},
            {"id": "inspect", "title": "Two", "command": ["run"]},
        ]},
    ],
)
def test_invalid_manifest_is_rejected_before_runner(change):
    raw = {**VALID, **change}

    with pytest.raises(ManifestError):
        parse_manifest(raw)


def test_context_and_platform_gate_action_selection():
    manifest = parse_manifest(VALID)
    base = InvocationContext(workspace_id="ws-1", pane_id="pane-1")
    provided = InvocationContext(selected_text="hello", correlation_id="corr-1")
    context = merge_context(base, provided)

    assert context.selected_text == "hello"
    with pytest.raises(ValueError, match="context"):
        select_action(manifest, "inspect", InvocationContext(workspace_id="ws-1"), platform="linux")
    assert select_action(manifest, "inspect", context, platform="linux").action_id == "inspect"
    with pytest.raises(ValueError, match="unsupported"):
        select_action(manifest, "inspect", context, platform="windows")
```

从 `v1/reference` 执行累计验收：

```bash
cd tutorials/01-herdr-runtime/modules/04-remote-plugins/v1/reference
PYTHONPATH=src python3 -m pytest -q
```

预期是 v1 的 3 个测试、v2 的 3 个测试和本版的 6 个测试，共 `12 passed`。测试证明 manifest 错误在 runner 前被拒绝，context/platform gate 能区分可调用和不可调用 action。真实 Herdr 还校验 pane、link handler、source checkout、当前版本和平台 enum，本参考项目只保留 runner 需要的核心字段。

## 检查点与下一步

V3 冻结了“声明先于副作用”的边界。下一版将 action、startup 和 event hook 交给一个有输出上限、并发上限和 command log 的 runner，并把 endpoint/plugin 结果投影给客户端查询。
