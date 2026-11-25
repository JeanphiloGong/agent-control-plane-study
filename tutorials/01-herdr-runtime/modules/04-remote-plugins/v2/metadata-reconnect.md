# Herdr Remote Endpoint / Plugin v2：在 SSH bridge 前加入可失效 metadata

[上一版 v1](../v1/local-endpoint.md) · [本版计划](plan.md) · [下一版 v3](../v3/manifest-context.md)

保存的 SSH endpoint 不能每次都盲目执行完整 discovery，也不能把上一次的远程路径永久当成真相。远程主机或 session 改变后，旧 metadata 必须失效。真实 Herdr 将 metadata 存在按 profile 命名的私有文件中，同时把 target 和 session 写入记录，读取时再次比较。

## 1. 定义 metadata 和缓存

创建 `tutorials/01-herdr-runtime/modules/04-remote-plugins/v1/reference/src/endpoints/metadata.py`：

```python
from __future__ import annotations

import json
from dataclasses import asdict, dataclass
from pathlib import Path


@dataclass(frozen=True)
class RemoteMetadata:
    os: str
    executable: str

    def is_valid(self) -> bool:
        if not self.executable or len(self.executable) > 4096:
            return False
        if any(ord(char) < 32 for char in self.executable):
            return False
        if self.os in {"linux", "macos"}:
            return self.executable.startswith("/") and not self.executable.endswith("/mise/shims/herdr")
        if self.os == "windows":
            return self.executable.startswith("\\\\") or (
                len(self.executable) >= 3
                and self.executable[0].isalpha()
                and self.executable[1] == ":"
                and self.executable[2] in {"/", "\\"}
            )
        return False


class MetadataCache:
    def __init__(self, root: Path, profile_id: str, target: str, session: str):
        self.path = root / f"{profile_id}.json"
        self.target = target
        self.session = session

    def load(self) -> RemoteMetadata | None:
        try:
            raw = json.loads(self.path.read_text(encoding="utf-8"))
        except (FileNotFoundError, OSError, json.JSONDecodeError):
            return None
        if (
            raw.get("version") != 1
            or raw.get("target") != self.target
            or raw.get("session") != self.session
        ):
            return None
        metadata = RemoteMetadata(**raw.get("metadata", {}))
        return metadata if metadata.is_valid() else None

    def store(self, metadata: RemoteMetadata) -> None:
        if not metadata.is_valid():
            raise ValueError("remote metadata is invalid")
        self.path.parent.mkdir(parents=True, exist_ok=True)
        self.path.write_text(
            json.dumps(
                {
                    "version": 1,
                    "target": self.target,
                    "session": self.session,
                    "metadata": asdict(metadata),
                },
                ensure_ascii=False,
                indent=2,
            ),
            encoding="utf-8",
        )

    def invalidate(self) -> None:
        self.path.unlink(missing_ok=True)
```

cache 文件名只使用 profile id，记录内容还必须匹配 target 和 session。`RemoteMetadata.is_valid()` 要求 Unix 路径是绝对路径，Windows 路径是 UNC 或带盘符的绝对路径，并拒绝换行和已知 shim 路径。真实源码还限制缓存文件大小和权限，本参考项目把文件权限留给真实实现边界。

## 2. 把 discovery 和 retry 接到 endpoint registry

创建 `tutorials/01-herdr-runtime/modules/04-remote-plugins/v1/reference/src/endpoints/connector.py`：

```python
from __future__ import annotations

from dataclasses import dataclass
from typing import Protocol

from .metadata import MetadataCache, RemoteMetadata
from .model import EndpointError, EndpointId, EndpointState, EndpointStatus, EndpointRegistry, Transport


class Discovery(Protocol):
    def discover(self, target: str, session: str) -> RemoteMetadata: ...


@dataclass(frozen=True)
class ConnectResult:
    state: EndpointState | None
    metadata: RemoteMetadata | None
    used_cache: bool
    retries: int
    status: EndpointStatus
    error: str | None = None


class EndpointConnector:
    def __init__(self, registry: EndpointRegistry, cache: MetadataCache, discovery: Discovery):
        self.registry = registry
        self.cache = cache
        self.discovery = discovery

    def connect(
        self,
        profile_id: str,
        target: str,
        session: str,
        transport: Transport,
        *,
        allow_retry: bool = True,
    ) -> ConnectResult:
        endpoint_id = EndpointId.ssh(profile_id)
        metadata = self.cache.load()
        used_cache = metadata is not None
        retries = 0
        if metadata is None:
            while True:
                try:
                    metadata = self.discovery.discover(target, session)
                    self.cache.store(metadata)
                    break
                except EndpointError as exc:
                    if not exc.retryable or not allow_retry or retries == 1:
                        status = (
                            EndpointStatus.RECONNECTING
                            if exc.retryable
                            else EndpointStatus.ATTENTION
                        )
                        return ConnectResult(
                            None, None, False, retries, status, str(exc)
                        )
                    retries += 1
        state = self.registry.connect(endpoint_id, transport, generation=retries + 1)
        return ConnectResult(
            state,
            metadata,
            used_cache,
            retries,
            state.status,
            state.last_error,
        )
```

cache hit跳过 discovery。cache miss 下只允许对 retryable 错误再试一次，`allow_retry=False` 则直接返回 `RECONNECTING`。非 retryable 错误返回 `ATTENTION`，不会创建一个伪造的 Online state。连接成功后才把 transport 交给 v1 的 `EndpointRegistry`。

## 3. 冻结缓存和重连边界

创建 `tutorials/01-herdr-runtime/modules/04-remote-plugins/v1/reference/test_metadata.py`：

```python
from dataclasses import dataclass

from endpoints.connector import EndpointConnector
from endpoints.metadata import MetadataCache, RemoteMetadata
from endpoints.model import EndpointError, EndpointRegistry, EndpointStatus
from endpoints.transport import FakeTransport


PROFILE = "0123456789abcdef0123456789abcdef"


@dataclass
class FakeDiscovery:
    metadata: RemoteMetadata
    calls: int = 0
    failures: list[EndpointError] | None = None

    def discover(self, target: str, session: str) -> RemoteMetadata:
        self.calls += 1
        if self.failures:
            failure = self.failures.pop(0)
            raise failure
        return self.metadata


def test_cache_hit_skips_discovery_and_target_change_invalidates(tmp_path):
    metadata = RemoteMetadata("linux", "/opt/herdr")
    discovery = FakeDiscovery(metadata)
    cache = MetadataCache(tmp_path, PROFILE, "build.example", "agent")
    connector = EndpointConnector(EndpointRegistry(), cache, discovery)

    first = connector.connect(PROFILE, "build.example", "agent", FakeTransport())
    second = connector.connect(PROFILE, "build.example", "agent", FakeTransport())
    changed_target = MetadataCache(tmp_path, PROFILE, "other.example", "agent")
    changed = EndpointConnector(EndpointRegistry(), changed_target, discovery)
    third = changed.connect(PROFILE, "other.example", "agent", FakeTransport())

    assert first.status is EndpointStatus.ONLINE
    assert not first.used_cache
    assert second.used_cache
    assert third.status is EndpointStatus.ONLINE
    assert discovery.calls == 2


def test_retryable_discovery_is_retried_once(tmp_path):
    discovery = FakeDiscovery(
        RemoteMetadata("macos", "/Applications/herdr"),
        failures=[EndpointError("temporary ssh failure", retryable=True)],
    )
    cache = MetadataCache(tmp_path, PROFILE, "mac", "default")

    result = EndpointConnector(EndpointRegistry(), cache, discovery).connect(
        PROFILE, "mac", "default", FakeTransport()
    )

    assert result.status is EndpointStatus.ONLINE
    assert result.retries == 1
    assert discovery.calls == 2


def test_non_retryable_discovery_requires_attention(tmp_path):
    discovery = FakeDiscovery(
        RemoteMetadata("linux", "/opt/herdr"),
        failures=[EndpointError("host key verification failed", retryable=False)],
    )
    cache = MetadataCache(tmp_path, PROFILE, "prod", "default")

    result = EndpointConnector(EndpointRegistry(), cache, discovery).connect(
        PROFILE, "prod", "default", FakeTransport()
    )

    assert result.status is EndpointStatus.ATTENTION
    assert result.retries == 0
    assert discovery.calls == 1
```

从 `v1/reference` 执行累计验收：

```bash
cd tutorials/01-herdr-runtime/modules/04-remote-plugins/v1/reference
PYTHONPATH=src python3 -m pytest -q
```

预期是 v1 的 3 个测试加 v2 的 3 个测试共 `6 passed`。这证明 cache hit、target/session 隔离、一次 retry 和 Attention 分类。真实 `SavedSshApiBridge` 还负责 SSH command、local bridge socket、metadata invalidation 和 transport recovery，本版没有启动这些外部资源。

## 检查点与下一步

V2 冻结了“缓存是可失效提示，连接状态才是运行事实”。下一版开始处理 plugin manifest，把声明校验和调用上下文放在进程启动前。
