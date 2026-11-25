# Herdr Remote Endpoint / Plugin v4：把合法命令交给受限 runner

[上一版 v3](../v3/manifest-context.md) · [本版计划](plan.md) · [模块路线图](../plan.md)

manifest 通过校验后，仍不能直接调用 `subprocess`。插件命令可能输出无限内容、同时启动过多任务，或者把自己的路径写入核心状态。真实 Herdr 在 `start_plugin_command` 中为命令分配 log id，生成 `HERDR_PLUGIN_*` 环境变量，限制并发和输出，并通过 AppEvent 回传完成结果。本版将这些责任收敛到一个可注入 executor 的 runner。

## 1. 实现 bounded command runner

创建 `tutorials/01-herdr-runtime/modules/04-remote-plugins/v1/reference/src/plugins/runner.py`：

```python
from __future__ import annotations

import json
import time
from dataclasses import dataclass
from enum import Enum
from typing import Callable

from .context import InvocationContext, select_action
from .manifest import KNOWN_EVENTS, PluginManifest, platform_supported


OUTPUT_LIMIT = 64 * 1024
MAX_IN_FLIGHT = 32
LOG_LIMIT = 200


class CommandStatus(str, Enum):
    RUNNING = "running"
    SUCCEEDED = "succeeded"
    FAILED = "failed"


@dataclass(frozen=True)
class CommandResult:
    returncode: int
    stdout: str = ""
    stderr: str = ""


@dataclass(frozen=True)
class CommandLog:
    log_id: str
    plugin_id: str
    action_id: str | None
    event: str | None
    command: tuple[str, ...]
    status: CommandStatus
    started_at: float
    finished_at: float | None
    returncode: int | None
    stdout: str
    stderr: str
    error: str | None


Executor = Callable[[tuple[str, ...], dict[str, str], str], CommandResult]


def _capped(text: str, limit: int) -> str:
    raw = text.encode("utf-8")
    if len(raw) <= limit:
        return text
    return raw[:limit].decode("utf-8", errors="ignore") + f"\n[truncated after {limit} bytes]"


class PluginRunner:
    def __init__(self, *, max_in_flight: int = MAX_IN_FLIGHT, output_limit: int = OUTPUT_LIMIT):
        if max_in_flight < 1 or output_limit < 1:
            raise ValueError("runner limits must be positive")
        self.max_in_flight = max_in_flight
        self.output_limit = output_limit
        self._active = 0
        self._next_log_id = 1
        self._logs: list[CommandLog] = []

    @property
    def logs(self) -> tuple[CommandLog, ...]:
        return tuple(self._logs)

    def _append(self, log: CommandLog) -> None:
        self._logs.append(log)
        del self._logs[:-LOG_LIMIT]

    def _environment(self, manifest, context, action_id, event) -> dict[str, str]:
        environment = {
            "HERDR_ENV": "1",
            "HERDR_PLUGIN_ID": manifest.plugin_id,
            "HERDR_PLUGIN_ROOT": manifest.plugin_root,
            "HERDR_PLUGIN_SOURCE": manifest.source,
            "HERDR_PLUGIN_CONTEXT_JSON": json.dumps(
                context.__dict__, ensure_ascii=False, sort_keys=True
            ),
        }
        if action_id:
            environment["HERDR_PLUGIN_ACTION_ID"] = action_id
        if event:
            environment["HERDR_PLUGIN_EVENT"] = event
        if context.workspace_id:
            environment["HERDR_WORKSPACE_ID"] = context.workspace_id
        if context.pane_id:
            environment["HERDR_PANE_ID"] = context.pane_id
        return environment

    def _run(self, manifest, command, context, executor, *, action_id=None, event=None):
        log_id = f"plugin-log-{self._next_log_id}"
        self._next_log_id += 1
        started = time.time()
        if self._active >= self.max_in_flight:
            log = CommandLog(
                log_id, manifest.plugin_id, action_id, event, command,
                CommandStatus.FAILED, started, started, None, "", "",
                "maximum concurrent plugin commands reached",
            )
            self._append(log)
            return log
        self._active += 1
        try:
            result = executor(
                command,
                self._environment(manifest, context, action_id, event),
                manifest.plugin_root,
            )
            status = CommandStatus.SUCCEEDED if result.returncode == 0 else CommandStatus.FAILED
            log = CommandLog(
                log_id, manifest.plugin_id, action_id, event, command, status,
                started, time.time(), result.returncode,
                _capped(result.stdout, self.output_limit),
                _capped(result.stderr, self.output_limit),
                None if result.returncode == 0 else "plugin command failed",
            )
        except Exception as exc:
            log = CommandLog(
                log_id, manifest.plugin_id, action_id, event, command,
                CommandStatus.FAILED, started, time.time(), None, "", "", str(exc),
            )
        finally:
            self._active -= 1
        self._append(log)
        return log

    def run_action(self, manifest, action_id, context, executor, *, platform):
        action = select_action(manifest, action_id, context, platform=platform)
        return self._run(manifest, action.command, context, executor, action_id=action_id)

    def run_event(self, manifest, event, context, executor, *, platform):
        if not manifest.enabled or event not in KNOWN_EVENTS:
            return ()
        logs = []
        for hook in manifest.events:
            if hook.event != event:
                continue
            platforms = hook.platforms or manifest.platforms
            if not platform_supported(platforms, platform):
                continue
            logs.append(self._run(manifest, hook.command, context, executor, event=event))
        return tuple(logs)
```

`_run` 在执行前分配 log id，检查 `_active`，再把 context、plugin root 和 action/event 放入受控环境。executor 返回后截断 stdout/stderr，非零 returncode 进入 Failed，异常也写入 Failed。`run_event` 只接受已知事件，并再次检查 hook/platform。真实 Herdr 用线程和子进程实现相同边界，参考项目通过 executor 注入保持回放确定性。

## 2. 添加客户端 projection

创建 `tutorials/01-herdr-runtime/modules/04-remote-plugins/v1/reference/src/endpoints/projection.py`：

```python
from typing import Any

from .model import EndpointState, EndpointStatus
from plugins.runner import CommandLog


def endpoint_projection(state: EndpointState) -> dict[str, Any]:
    return {
        "endpoint": state.endpoint_id.storage_key(),
        "status": state.status.value,
        "generation": state.generation,
        "online": state.status is EndpointStatus.ONLINE,
        "error": state.last_error,
    }


def plugin_log_projection(log: CommandLog) -> dict[str, Any]:
    return {
        "log_id": log.log_id,
        "plugin_id": log.plugin_id,
        "action_id": log.action_id,
        "event": log.event,
        "status": log.status.value,
        "returncode": log.returncode,
        "error": log.error,
    }
```

projection 只暴露客户端需要的状态，不把 transport 对象、executor 或完整环境变量返回给客户端。真实 Herdr 通过 endpoint shell/API 和 `PluginCommandLogInfo`/AppEvent 提供相同的读模型。

## 3. 冻结命令、并发和 projection

创建 `tutorials/01-herdr-runtime/modules/04-remote-plugins/v1/reference/test_runner.py`：

```python
from plugins.context import InvocationContext
from plugins.manifest import parse_manifest
from plugins.runner import CommandResult, CommandStatus, PluginRunner
from endpoints.projection import plugin_log_projection


MANIFEST = parse_manifest(
    {
        "id": "example.tools",
        "name": "Example Tools",
        "version": "0.1.0",
        "min_herdr_version": "1.0.0",
        "actions": [{
            "id": "inspect",
            "title": "Inspect",
            "contexts": ["pane"],
            "command": ["inspect-plugin"],
        }],
        "events": [{"on": "pane.focused", "command": ["focus-hook"]}],
    },
    plugin_root="/plugins/example",
    source="remote",
)


def test_successful_action_has_bounded_environment_and_log():
    runner = PluginRunner(output_limit=8)
    seen = {}

    def executor(command, environment, cwd):
        seen.update(command=command, environment=environment, cwd=cwd)
        return CommandResult(0, "0123456789", "")

    log = runner.run_action(
        MANIFEST, "inspect", InvocationContext(workspace_id="ws-1", pane_id="pane-1"),
        executor, platform="linux"
    )

    assert log.status is CommandStatus.SUCCEEDED
    assert log.stdout.endswith("[truncated after 8 bytes]")
    assert seen["cwd"] == "/plugins/example"
    assert seen["environment"]["HERDR_PLUGIN_SOURCE"] == "remote"
    assert seen["environment"]["HERDR_PANE_ID"] == "pane-1"
    assert plugin_log_projection(log)["status"] == "succeeded"


def test_nonzero_action_is_failed_and_nested_run_hits_limit():
    runner = PluginRunner(max_in_flight=1)
    nested = []

    def executor(command, environment, cwd):
        nested.append(runner.run_action(
            MANIFEST, "inspect", InvocationContext(pane_id="pane-2"),
            lambda *_: CommandResult(0), platform="linux"
        ))
        return CommandResult(3, "", "bad")

    log = runner.run_action(
        MANIFEST, "inspect", InvocationContext(pane_id="pane-1"),
        executor, platform="linux"
    )

    assert log.status is CommandStatus.FAILED
    assert log.returncode == 3
    assert nested[0].error == "maximum concurrent plugin commands reached"


def test_event_hook_runs_only_for_known_supported_event():
    runner = PluginRunner()
    calls = []

    def executor(command, environment, cwd):
        calls.append(command)
        return CommandResult(0)

    logs = runner.run_event(
        MANIFEST, "pane.focused", InvocationContext(pane_id="pane-1"),
        executor, platform="linux"
    )
    ignored = runner.run_event(
        MANIFEST, "future.event", InvocationContext(pane_id="pane-1"),
        executor, platform="linux"
    )

    assert len(logs) == 1
    assert ignored == ()
    assert calls == [("focus-hook",)]
```

从 v1/reference 执行最终回放：

```bash
cd tutorials/01-herdr-runtime/modules/04-remote-plugins/v1/reference
PYTHONPATH=src python3 -m pytest -q
```

预期是 v1–v4 的所有测试通过。再执行 projection 检查：

```bash
PYTHONPATH=src python3 - <<'PY'
from endpoints.model import EndpointId, EndpointRegistry
from endpoints.projection import endpoint_projection
from endpoints.transport import FakeTransport

endpoint = EndpointId.local()
state = EndpointRegistry().connect(endpoint, FakeTransport())
assert endpoint_projection(state)["online"] is True
print("endpoint projection is online")
PY
```

## 检查点与后续边界

专题 04 v4 已经把 endpoint status、metadata/retry、manifest/context、bounded runner、command log 和客户端 projection 串起来。它仍然没有实现真实 SSH 登录、插件安装市场或任意文件访问。下一个产品集成门是把这些边界接到 H1 的 server runtime，并用真实平台测试补齐 Python 参考项目无法证明的部分。
