# Herdr Server / Client / Socket v1：让断开的客户端不带走运行时

[V0 设计](../V0-design.md) · [模块路线图](../plan.md) · [本版计划](plan.md) · [下一版 v2](../v2/local-json-api.md)

你在 Herdr 的一个 pane 里启动 Agent，退出 TUI，稍后再打开 Herdr。新客户端应看到刚才的 pane。这个行为取决于 pane 存在哪里，以及退出客户端时会清理哪些状态。

在 Herdr 的固定源码提交 `e8d7d970` 中，`HeadlessServer` 持有 App，`client_read_loop` 收到 `Detach` 后把 `ClientDetach` 送回服务端事件循环。可以沿 `src/server/headless.rs` 和 `src/server/client_transport.rs` 对照这条路径。上游测试 `tests/detach_reattach.rs::reattach_after_detach_shows_current_state` 给出了再次连接的场景。

本篇从 V0 的所有权设计和空的 `v1/reference/` 开始。我们用 Rust 标准库和进程内方法调用重现这条链路。pane 是一条内存记录，进程和终端行为由后续 PTY 模块承担。所有命令从仓库根目录执行，需要 Rust stable 和 Cargo。

## 1. 定义客户端能看到的消息

客户端先用 `TerminalHello` 提交终端尺寸。握手成功后，`Welcome` 返回连接 ID 和现有 pane 的 ID。客户端稍后可以读取 pane 快照，或者发送 `Detach` 来结束这次连接。用独立的连接 ID，服务端在清理会话时就能准确找到客户端。

```bash
mkdir -p tutorials/01-herdr-runtime/modules/01-server-client/v1/reference/src
mkdir -p tutorials/01-herdr-runtime/modules/01-server-client/v1/reference/tests
```

创建 `v1/reference/Cargo.toml`：

```toml
[package]
name = "herdr_server_client_v1"
version = "0.1.0"
edition = "2021"
```

创建 `v1/reference/src/lib.rs`，先注册本节的两个模块。

```rust
pub mod client_protocol;
pub mod runtime_events;
```

创建 `v1/reference/src/client_protocol.rs`：

```rust
#[derive(Clone, Debug, PartialEq, Eq)]
pub struct TerminalHello {
    pub cols: u16,
    pub rows: u16,
}

#[derive(Clone, Debug, PartialEq, Eq)]
pub enum ClientMessage {
    TerminalHello(TerminalHello),
    Detach,
}

#[derive(Clone, Debug, PartialEq, Eq)]
pub struct PaneSnapshot {
    pub pane_id: u64,
    pub running: bool,
}

#[derive(Clone, Debug, PartialEq, Eq)]
pub enum ServerMessage {
    Welcome { client_id: u64, pane_ids: Vec<u64> },
    Snapshot { panes: Vec<PaneSnapshot> },
    Closed,
}

#[derive(Clone, Debug, PartialEq, Eq)]
pub enum ProtocolError {
    AlreadyHandshaken,
    MissingHandshake,
    InvalidGeometry,
    Closed,
}
```

创建 `v1/reference/src/runtime_events.rs`：

```rust
#[derive(Clone, Debug, PartialEq, Eq)]
pub enum RuntimeEvent {
    ClientConnected { client_id: u64 },
    ClientDetached { client_id: u64 },
    ClientDisconnected { client_id: u64 },
}
```

`PaneSnapshot` 是服务端提供的状态副本。`RuntimeEvent` 记录已发生的连接变化，后面的验收会读取它来检查客户端 A 离开和客户端 B 加入的顺序。消息和事件将在第 3 节串起来运行。

## 2. 让服务端持有 pane

真实 `HeadlessServer` 同时持有 App、客户端集合、运行时和关闭状态。学习实现用一个
`Arc<Mutex<RuntimeState>>` 共享同一组 pane 和客户端事实，让多个 `ClientSession` 看到
同一个 server state。

创建 `v1/reference/src/server_runtime.rs`：

```rust
use std::collections::{BTreeMap, BTreeSet};
use std::sync::{Arc, Mutex};

use crate::client_protocol::PaneSnapshot;
use crate::client_session::ClientSession;
use crate::runtime_events::RuntimeEvent;

#[derive(Clone)]
pub struct ServerRuntime {
    inner: Arc<Mutex<RuntimeState>>,
}

struct RuntimeState {
    running: bool,
    next_pane_id: u64,
    next_client_id: u64,
    panes: BTreeMap<u64, PaneSnapshot>,
    connected_clients: BTreeSet<u64>,
    events: Vec<RuntimeEvent>,
}

impl ServerRuntime {
    pub fn start() -> Self {
        Self {
            inner: Arc::new(Mutex::new(RuntimeState {
                running: true,
                next_pane_id: 1,
                next_client_id: 1,
                panes: BTreeMap::new(),
                connected_clients: BTreeSet::new(),
                events: Vec::new(),
            })),
        }
    }

    pub fn create_pane(&self) -> PaneSnapshot {
        let mut state = self.inner.lock().unwrap();
        assert!(state.running, "cannot create a pane after shutdown");
        let pane = PaneSnapshot {
            pane_id: state.next_pane_id,
            running: true,
        };
        state.next_pane_id += 1;
        state.panes.insert(pane.pane_id, pane.clone());
        pane
    }

    pub fn connect_client(&self) -> ClientSession {
        let mut state = self.inner.lock().unwrap();
        let client_id = state.next_client_id;
        state.next_client_id += 1;
        ClientSession::new(self.clone(), client_id)
    }

    pub fn pane(&self, pane_id: u64) -> Option<PaneSnapshot> {
        self.inner.lock().unwrap().panes.get(&pane_id).cloned()
    }

    pub fn is_running(&self) -> bool {
        self.inner.lock().unwrap().running
    }

    pub fn client_count(&self) -> usize {
        self.inner.lock().unwrap().connected_clients.len()
    }

    pub fn events(&self) -> Vec<RuntimeEvent> {
        self.inner.lock().unwrap().events.clone()
    }

    pub(crate) fn accept_client(&self, client_id: u64) -> bool {
        let mut state = self.inner.lock().unwrap();
        if !state.running || !state.connected_clients.insert(client_id) {
            return false;
        }
        state.events.push(RuntimeEvent::ClientConnected { client_id });
        true
    }

    pub(crate) fn detach_client(&self, client_id: u64) {
        self.remove_client(client_id, RuntimeEvent::ClientDetached { client_id });
    }

    pub(crate) fn disconnect_client(&self, client_id: u64) {
        self.remove_client(client_id, RuntimeEvent::ClientDisconnected { client_id });
    }

    fn remove_client(&self, client_id: u64, event: RuntimeEvent) {
        let mut state = self.inner.lock().unwrap();
        if state.connected_clients.remove(&client_id) {
            state.events.push(event);
        }
    }

    pub(crate) fn pane_snapshot(&self) -> Vec<PaneSnapshot> {
        self.inner.lock().unwrap().panes.values().cloned().collect()
    }
}
```

`connect_client` 分配 ID。握手通过后，`accept_client` 才将 ID 写入 `connected_clients`。`detach_client` 从该集合移除 ID 并记录事件，`panes` 由服务端继续持有。两个 `Arc` 句柄指向同一份 `RuntimeState`，因此后来的客户端可以读取此前创建的 pane。下一节接上 `ClientSession` 后，一并注册这两个模块。

## 3. 把握手和断开接到同一份状态

真实 `handle_client_handshake` 在连接进入正常读取循环前校验 hello，并发送
`ServerMessage::Welcome`。学习实现把这个过程收敛到 `ClientSession::handshake`，再让
`send(Detach)` 生成服务端事件。

创建 `v1/reference/src/client_session.rs`：

```rust
use crate::client_protocol::{
    ClientMessage, ProtocolError, ServerMessage, TerminalHello,
};
use crate::server_runtime::ServerRuntime;

#[derive(Clone, Copy, Debug, PartialEq, Eq)]
enum SessionState {
    Pending,
    Connected,
    Closed,
}

pub struct ClientSession {
    server: ServerRuntime,
    client_id: u64,
    state: SessionState,
}

impl ClientSession {
    pub(crate) fn new(server: ServerRuntime, client_id: u64) -> Self {
        Self {
            server,
            client_id,
            state: SessionState::Pending,
        }
    }

    pub fn handshake(
        &mut self,
        hello: TerminalHello,
    ) -> Result<ServerMessage, ProtocolError> {
        if self.state != SessionState::Pending {
            return Err(ProtocolError::AlreadyHandshaken);
        }
        if hello.cols == 0 || hello.rows == 0 {
            return Err(ProtocolError::InvalidGeometry);
        }
        if !self.server.accept_client(self.client_id) {
            self.state = SessionState::Closed;
            return Err(ProtocolError::Closed);
        }
        self.state = SessionState::Connected;
        Ok(ServerMessage::Welcome {
            client_id: self.client_id,
            pane_ids: self
                .server
                .pane_snapshot()
                .into_iter()
                .map(|pane| pane.pane_id)
                .collect(),
        })
    }

    pub fn send(
        &mut self,
        message: ClientMessage,
    ) -> Result<ServerMessage, ProtocolError> {
        if self.state == SessionState::Closed {
            return Err(ProtocolError::Closed);
        }
        if self.state == SessionState::Pending {
            return Err(ProtocolError::MissingHandshake);
        }
        match message {
            ClientMessage::TerminalHello(_) => Err(ProtocolError::AlreadyHandshaken),
            ClientMessage::Detach => {
                self.server.detach_client(self.client_id);
                self.state = SessionState::Closed;
                Ok(ServerMessage::Closed)
            }
        }
    }

    pub fn snapshot(&self) -> Result<ServerMessage, ProtocolError> {
        if self.state != SessionState::Connected {
            return Err(ProtocolError::Closed);
        }
        Ok(ServerMessage::Snapshot {
            panes: self.server.pane_snapshot(),
        })
    }

    pub fn is_closed(&self) -> bool {
        self.state == SessionState::Closed
    }
}

impl Drop for ClientSession {
    fn drop(&mut self) {
        if self.state == SessionState::Connected {
            self.server.disconnect_client(self.client_id);
        }
    }
}
```

握手成功时会话从 `Pending` 进入 `Connected`，然后读取服务端已有的 pane ID。零尺寸使握手提前返回，所以 `connected_clients` 和事件记录都保持原样。显式 `Detach` 将会话置为 `Closed`，后续释放这个会话时 `Drop` 会跳过断开逻辑。客户端直接离开作用域时，`Drop` 走的是 `ClientDisconnected`，同样只移除连接。

将 `v1/reference/src/lib.rs` 替换为以下完整内容，让调用者能使用新模块。

```rust
pub mod client_protocol;
pub mod client_session;
pub mod runtime_events;
pub mod server_runtime;
```

先编译已接通的库。

```bash
cargo check --manifest-path tutorials/01-herdr-runtime/modules/01-server-client/v1/reference/Cargo.toml
```

命令以状态码 0 退出后，可以在下一节通过公开入口验证两次连接之间的状态。

## 4. 用两个客户端验证 pane 保留

创建 `v1/reference/tests/detach_preserves_runtime.rs`：

```rust
use herdr_server_client_v1::client_protocol::{
    ClientMessage, ProtocolError, ServerMessage, TerminalHello,
};
use herdr_server_client_v1::runtime_events::RuntimeEvent;
use herdr_server_client_v1::server_runtime::ServerRuntime;

#[test]
fn detach_preserves_runtime_for_a_second_client() {
    let server = ServerRuntime::start();
    let pane = server.create_pane();
    let mut first = server.connect_client();

    assert_eq!(
        first.handshake(TerminalHello { cols: 80, rows: 24 }),
        Ok(ServerMessage::Welcome {
            client_id: 1,
            pane_ids: vec![pane.pane_id],
        })
    );
    assert_eq!(first.send(ClientMessage::Detach), Ok(ServerMessage::Closed));
    assert!(first.is_closed());
    assert!(server.is_running());
    assert_eq!(server.client_count(), 0);
    assert!(server.pane(pane.pane_id).unwrap().running);
    drop(first);

    let mut second = server.connect_client();
    assert_eq!(
        second.handshake(TerminalHello { cols: 100, rows: 30 }),
        Ok(ServerMessage::Welcome {
            client_id: 2,
            pane_ids: vec![pane.pane_id],
        })
    );
    assert_eq!(second.snapshot().unwrap(), ServerMessage::Snapshot {
        panes: vec![pane],
    });
    assert_eq!(
        server.events(),
        vec![
            RuntimeEvent::ClientConnected { client_id: 1 },
            RuntimeEvent::ClientDetached { client_id: 1 },
            RuntimeEvent::ClientConnected { client_id: 2 },
        ]
    );
}

#[test]
fn invalid_handshake_does_not_register_client() {
    let server = ServerRuntime::start();
    let mut client = server.connect_client();

    assert_eq!(
        client.handshake(TerminalHello { cols: 0, rows: 24 }),
        Err(ProtocolError::InvalidGeometry)
    );
    assert!(!client.is_closed());
    assert_eq!(server.client_count(), 0);
    assert!(server.events().is_empty());
    assert!(server.is_running());
}
```

从仓库根目录运行。

```bash
cargo test --manifest-path tutorials/01-herdr-runtime/modules/01-server-client/v1/reference/Cargo.toml
```

预期结果是两个测试通过：

```text
test detach_preserves_runtime_for_a_second_client ... ok
test invalid_handshake_does_not_register_client ... ok
```

## 测试指导

第一个测试在删除客户端 A 后让 B 重新读取同一个 pane，并检查显式断开和 `Drop` 合起来只记一条离开事件。第二个测试以零列数触发 `InvalidGeometry`，用连接数和空事件列表验证请求在注册之前被拒绝。

## 检查点与下一步

以两个离线用例通过作为 v1 的检查点。断言中的 `running` 是 pane 记录上的布尔值，验证范围到进程内的状态和事件为止。

现在客户端可以在同一个进程内读取 pane。下一版会让另一个进程通过本地 API 带着 request ID 查询同一份状态，并通过事件订阅跟踪变化。 [继续 v2](../v2/local-json-api.md)
