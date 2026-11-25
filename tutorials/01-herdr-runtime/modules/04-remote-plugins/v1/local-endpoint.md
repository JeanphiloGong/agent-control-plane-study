# Herdr Remote Endpoint / Plugin v1：先把连接状态放在 endpoint 边界

[V0 设计](../V0-design.md) · [模块路线图](../plan.md) · [本版计划](plan.md) · [下一版 v2](../v2/metadata-reconnect.md)

客户端选择 endpoint 时，需要知道连接是否真的建立。一个 profile id 只负责稳定标识，target、session 和 transport 细节不能进入持久化 key。连接建立后，发送失败还必须告诉上层这是一次连接丢失，而不是继续把 endpoint 显示为 Online。

真实 Herdr 的对应入口是 `src/client/endpoint.rs::ProfileId`、`ClientEndpointId`、`ClientEndpointStatus` 和 `src/client/endpoint/registry.rs::EndpointRegistry`。本篇用 Python 标准库模拟同一责任分配。

从仓库根目录创建参考项目：

```bash
mkdir -p tutorials/01-herdr-runtime/modules/04-remote-plugins/v1/reference/src/endpoints
touch tutorials/01-herdr-runtime/modules/04-remote-plugins/v1/reference/src/endpoints/__init__.py
```

## 1. 定义 endpoint identity 和状态

创建 `tutorials/01-herdr-runtime/modules/04-remote-plugins/v1/reference/src/endpoints/model.py`：

```python
from __future__ import annotations

import re
from dataclasses import dataclass
from enum import Enum
from typing import Protocol


PROFILE_RE = re.compile(r"^[0-9a-f]{32}$")


class EndpointError(Exception):
    def __init__(self, message: str, *, retryable: bool):
        super().__init__(message)
        self.retryable = retryable


class EndpointStatus(str, Enum):
    CONNECTING = "connecting"
    ONLINE = "online"
    RECONNECTING = "reconnecting"
    ATTENTION = "attention"
    DISABLED = "disabled"


@dataclass(frozen=True)
class EndpointId:
    kind: str
    profile_id: str | None = None

    @classmethod
    def local(cls) -> "EndpointId":
        return cls("local")

    @classmethod
    def ssh(cls, profile_id: str) -> "EndpointId":
        if not PROFILE_RE.fullmatch(profile_id):
            raise ValueError("profile id must be 32 lowercase hexadecimal characters")
        return cls("ssh", profile_id)

    def storage_key(self) -> str:
        if self.kind == "local":
            return "local"
        if self.kind == "ssh" and self.profile_id is not None:
            return f"ssh:{self.profile_id}"
        raise ValueError("endpoint identity is invalid")


class Transport(Protocol):
    def open(self) -> None: ...

    def send(self, message: str) -> None: ...

    def close(self) -> None: ...


@dataclass
class EndpointState:
    endpoint_id: EndpointId
    status: EndpointStatus
    generation: int
    transport: Transport | None = None
    last_error: str | None = None


class EndpointRegistry:
    def __init__(self) -> None:
        self._states: dict[EndpointId, EndpointState] = {}

    def connect(self, endpoint_id: EndpointId, transport: Transport, generation: int = 1) -> EndpointState:
        state = EndpointState(endpoint_id, EndpointStatus.CONNECTING, generation, transport)
        self._states[endpoint_id] = state
        try:
            transport.open()
        except EndpointError as exc:
            state.status = (
                EndpointStatus.RECONNECTING if exc.retryable else EndpointStatus.ATTENTION
            )
            state.last_error = str(exc)
        else:
            state.status = EndpointStatus.ONLINE
            state.last_error = None
        return state

    def send(self, endpoint_id: EndpointId, message: str) -> bool:
        state = self._states.get(endpoint_id)
        if state is None or state.status != EndpointStatus.ONLINE or state.transport is None:
            return False
        try:
            state.transport.send(message)
        except EndpointError as exc:
            state.status = (
                EndpointStatus.RECONNECTING if exc.retryable else EndpointStatus.ATTENTION
            )
            state.last_error = str(exc)
            state.transport.close()
            return False
        return True

    def state(self, endpoint_id: EndpointId) -> EndpointState | None:
        return self._states.get(endpoint_id)
```

`EndpointId.storage_key()` 是唯一进入存储层的 endpoint 标识。`EndpointState` 保存连接事实和 generation，`EndpointRegistry.connect()` 先进入 `CONNECTING`，只有 `open()` 成功才发布 `ONLINE`。transport 错误的 retryable 属性决定进入 `RECONNECTING` 还是 `ATTENTION`，这对应真实客户端把可恢复断开与权限/协议问题分开。

## 2. 注入确定性的 fake transport

创建 `tutorials/01-herdr-runtime/modules/04-remote-plugins/v1/reference/src/endpoints/transport.py`：

```python
from dataclasses import dataclass, field

from .model import EndpointError


@dataclass
class FakeTransport:
    open_error: EndpointError | None = None
    send_error: EndpointError | None = None
    messages: list[str] = field(default_factory=list)
    opened: bool = False
    closed: bool = False

    def open(self) -> None:
        if self.open_error is not None:
            raise self.open_error
        self.opened = True

    def send(self, message: str) -> None:
        if self.send_error is not None:
            raise self.send_error
        if not self.opened or self.closed:
            raise EndpointError("transport is closed", retryable=True)
        self.messages.append(message)

    def close(self) -> None:
        self.closed = True
```

fake transport 不模拟 SSH 字节流，只提供可控的打开、发送和关闭结果。这样测试可以验证 registry 的状态转移，而不会依赖网络或凭据。

## 3. 冻结 V1

创建 `tutorials/01-herdr-runtime/modules/04-remote-plugins/v1/reference/test_endpoint.py`：

```python
from endpoints.model import EndpointError, EndpointId, EndpointRegistry, EndpointStatus
from endpoints.transport import FakeTransport


PROFILE = "0123456789abcdef0123456789abcdef"


def test_storage_key_is_opaque_and_successful_transport_becomes_online():
    endpoint = EndpointId.ssh(PROFILE)
    transport = FakeTransport()
    registry = EndpointRegistry()

    state = registry.connect(endpoint, transport, generation=7)

    assert endpoint.storage_key() == f"ssh:{PROFILE}"
    assert "example.com" not in endpoint.storage_key()
    assert state.status is EndpointStatus.ONLINE
    assert registry.send(endpoint, "ping")
    assert transport.messages == ["ping"]


def test_retryable_open_failure_never_reports_online():
    endpoint = EndpointId.local()
    transport = FakeTransport(open_error=EndpointError("socket closed", retryable=True))

    state = EndpointRegistry().connect(endpoint, transport)

    assert state.status is EndpointStatus.RECONNECTING
    assert state.last_error == "socket closed"


def test_permission_failure_requires_attention_and_send_failure_updates_state():
    endpoint = EndpointId.local()
    registry = EndpointRegistry()
    attention = registry.connect(
        endpoint,
        FakeTransport(open_error=EndpointError("permission denied", retryable=False)),
    )
    assert attention.status is EndpointStatus.ATTENTION
    assert not registry.send(endpoint, "ignored")

    transport = FakeTransport(send_error=EndpointError("broken pipe", retryable=True))
    state = registry.connect(endpoint, transport, generation=2)
    assert state.status is EndpointStatus.ONLINE
    assert not registry.send(endpoint, "request")
    assert registry.state(endpoint).status is EndpointStatus.RECONNECTING
```

从 `v1/reference` 执行：

```bash
cd tutorials/01-herdr-runtime/modules/04-remote-plugins/v1/reference
PYTHONPATH=src python3 -m pytest -q
```

预期是 `3 passed`。这证明 endpoint identity、连接状态和 transport failure 在 registry 内有明确结果。它不证明真实 socket 已绑定，也不证明 SSH target 可以连接。v2 会在这个连接边界前增加 metadata discovery 和有限重试。

## 检查点与下一步

V1 冻结了 endpoint 的最小状态机。没有 profile 解析和 transport 成功事实时，调用方不能看到 Online。下一版把远程可执行文件路径放进带 target/session 指纹的缓存，并把发现失败限制为一次声明的重试。
