# Herdr Server / Client v3：让慢客户端只占自己的队列

[上一版 v2](../v2/local-json-api.md) · [本版计划](plan.md) · [下一版 v4](../v4/runtime-handoff.md)

两个客户端同时看着同一个服务端。A 暂停读取时，如果发布操作必须等 A 收完，B 也看不到新 pane。给每个连接单独分配发送队列，再用非阻塞发送发布更新。队列满时只结束这条连接，重连后可以从快照取得最新状态。

Herdr 的 src/server/client_transport.rs 中 ClientWriter 把控制消息与渲染消息分开处理，渲染层允许丢弃旧帧。本篇关注隔离规则，采用容量为 2 的进程内队列来发送 pane 创建通知。v2 的 Unix JSON API 仍调用同一个运行时。完成 v2 后，从仓库根目录进入同一个项目。

```bash
cd tutorials/01-herdr-runtime/modules/01-server-client/v1/reference
```

## 1. 在创建 pane 时通知所有已连接客户端

在 src/runtime_events.rs 的 RuntimeEvent 枚举末尾加入以下变体。这条事件记录因过载而移除的连接，API 订阅者可以据此区分主动断开与队列过载。

```rust
    ClientOverloaded { client_id: u64 },
```

在 src/server_runtime.rs 中，把集合的 import 替换为以下两行。

```rust
use std::collections::BTreeMap;
use std::sync::mpsc::{self, Receiver, SyncSender};
```

将 RuntimeState 中的 connected_clients 字段替换为以下类型，并将 start 对应的初始化改为 connected_clients: BTreeMap::new()。

```rust
    connected_clients: BTreeMap<u64, SyncSender<PaneSnapshot>>,
```

把 ServerRuntime::create_pane 替换为完整方法。

```rust
    pub fn create_pane(&self) -> PaneSnapshot {
        let mut state = self.inner.lock().unwrap();
        assert!(state.running, "cannot create a pane after shutdown");
        let pane = PaneSnapshot {
            pane_id: state.next_pane_id,
            running: true,
        };
        state.next_pane_id += 1;
        state.panes.insert(pane.pane_id, pane.clone());
        state.events.push(RuntimeEvent::PaneCreated { pane_id: pane.pane_id });

        let mut overloaded = Vec::new();
        state.connected_clients.retain(|&client_id, writer| {
            if writer.try_send(pane.clone()).is_ok() {
                return true;
            }
            overloaded.push(RuntimeEvent::ClientOverloaded { client_id });
            false
        });
        state.events.extend(overloaded);
        pane
    }
```

try_send 立即返回，服务端继续处理下一个连接。这里 receiver 由会话持有，主动离开会先从注册表删除 sender，因而正常发布时，保留在注册表中的队列发送失败意味着没有队列容量。运行时仍掌握完整 pane，队列只装发给某个连接的通知。

仍在该文件中，将 accept_client、detach_client、disconnect_client、remove_client 替换为下面四个方法，并在这组方法之前加入 is_connected 和 snapshot_for。

```rust
    pub(crate) fn is_connected(&self, client_id: u64) -> bool {
        self.inner.lock().unwrap().connected_clients.contains_key(&client_id)
    }

    pub(crate) fn snapshot_for(&self, client_id: u64) -> Option<Vec<PaneSnapshot>> {
        let state = self.inner.lock().unwrap();
        state.connected_clients.contains_key(&client_id)
            .then(|| state.panes.values().cloned().collect())
    }

    pub(crate) fn accept_client(
        &self,
        client_id: u64,
    ) -> Option<(Receiver<PaneSnapshot>, Vec<u64>)> {
        let mut state = self.inner.lock().unwrap();
        if !state.running {
            return None;
        }
        let (writer, reader) = mpsc::sync_channel(2);
        state.connected_clients.insert(client_id, writer);
        state.events.push(RuntimeEvent::ClientConnected { client_id });
        Some((reader, state.panes.keys().copied().collect()))
    }

    pub(crate) fn detach_client(&self, client_id: u64) -> bool {
        self.remove_client(client_id, RuntimeEvent::ClientDetached { client_id })
    }

    pub(crate) fn disconnect_client(&self, client_id: u64) {
        self.remove_client(client_id, RuntimeEvent::ClientDisconnected { client_id });
    }

    fn remove_client(&self, client_id: u64, event: RuntimeEvent) -> bool {
        let mut state = self.inner.lock().unwrap();
        if state.connected_clients.remove(&client_id).is_none() {
            return false;
        }
        state.events.push(event);
        true
    }
```

注册队列和读取迎宾快照在同一个锁内完成。否则客户端可能先读到快照，随后错过恰好在注册之前发生的更新。

## 2. 让会话读取自己的队列

完整替换 src/client_session.rs。客户端仍用原有的握手和 Detach 入口，增加 next_update 读取待发送的更新。

```rust
use std::sync::mpsc::{Receiver, TryRecvError};

use crate::client_protocol::{
    ClientMessage, PaneSnapshot, ProtocolError, ServerMessage, TerminalHello,
};
use crate::server_runtime::ServerRuntime;

enum SessionState {
    Pending,
    Connected(Receiver<PaneSnapshot>),
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

    pub fn handshake(&mut self, hello: TerminalHello) -> Result<ServerMessage, ProtocolError> {
        if self.is_closed() {
            return Err(ProtocolError::Closed);
        }
        if !matches!(self.state, SessionState::Pending) {
            return Err(ProtocolError::AlreadyHandshaken);
        }
        if hello.cols == 0 || hello.rows == 0 {
            return Err(ProtocolError::InvalidGeometry);
        }
        let Some((reader, pane_ids)) = self.server.accept_client(self.client_id) else {
            self.state = SessionState::Closed;
            return Err(ProtocolError::Closed);
        };
        self.state = SessionState::Connected(reader);
        Ok(ServerMessage::Welcome {
            client_id: self.client_id,
            pane_ids,
        })
    }

    pub fn send(&mut self, message: ClientMessage) -> Result<ServerMessage, ProtocolError> {
        if self.is_closed() {
            return Err(ProtocolError::Closed);
        }
        if matches!(self.state, SessionState::Pending) {
            return Err(ProtocolError::MissingHandshake);
        }
        match message {
            ClientMessage::TerminalHello(_) => Err(ProtocolError::AlreadyHandshaken),
            ClientMessage::Detach => {
                let detached = self.server.detach_client(self.client_id);
                self.state = SessionState::Closed;
                if detached {
                    Ok(ServerMessage::Closed)
                } else {
                    Err(ProtocolError::Closed)
                }
            }
        }
    }

    pub fn snapshot(&self) -> Result<ServerMessage, ProtocolError> {
        if !matches!(self.state, SessionState::Connected(_)) {
            return Err(ProtocolError::Closed);
        }
        let panes = self.server.snapshot_for(self.client_id).ok_or(ProtocolError::Closed)?;
        Ok(ServerMessage::Snapshot { panes })
    }

    pub fn next_update(&self) -> Result<Option<PaneSnapshot>, ProtocolError> {
        let SessionState::Connected(reader) = &self.state else {
            return Err(ProtocolError::Closed);
        };
        if self.is_closed() {
            return Err(ProtocolError::Closed);
        }
        match reader.try_recv() {
            Ok(pane) => Ok(Some(pane)),
            Err(TryRecvError::Empty) => Ok(None),
            Err(TryRecvError::Disconnected) => Err(ProtocolError::Closed),
        }
    }

    pub fn is_closed(&self) -> bool {
        match self.state {
            SessionState::Pending => false,
            SessionState::Connected(_) => !self.server.is_connected(self.client_id),
            SessionState::Closed => true,
        }
    }
}

impl Drop for ClientSession {
    fn drop(&mut self) {
        if matches!(self.state, SessionState::Connected(_)) {
            self.server.disconnect_client(self.client_id);
        }
    }
}
```

Ok(None) 表示连接仍在，只是当前队列为空。过载后服务端已移除连接，next_update 返回 Closed，客户端从新的握手和快照继续。显式 Detach 或过载清理完成后，Drop 找不到注册项，因此只会留下原来的离开事件。

## 3. 用一个停读的客户端验证隔离

新建 tests/multi_client_backpressure.rs。测试以固定顺序填满队列，无需 sleep 或依赖操作系统写缓冲的大小。

```rust
use herdr_server_client_v1::client_protocol::{ClientMessage, ProtocolError, TerminalHello};
use herdr_server_client_v1::runtime_events::RuntimeEvent;
use herdr_server_client_v1::server_runtime::ServerRuntime;

#[test]
fn slow_client_is_removed_while_other_clients_keep_receiving() {
    let server = ServerRuntime::start();
    let mut slow = server.connect_client();
    let mut healthy = server.connect_client();
    let hello = TerminalHello { cols: 80, rows: 24 };
    slow.handshake(hello.clone()).unwrap();
    healthy.handshake(hello.clone()).unwrap();
    assert_eq!(healthy.next_update(), Ok(None));

    for _ in 0..3 {
        let pane = server.create_pane();
        assert_eq!(healthy.next_update(), Ok(Some(pane)));
    }

    assert_eq!(slow.next_update(), Err(ProtocolError::Closed));
    assert_eq!(server.client_count(), 1);
    drop(slow);
    assert!(server.events().ends_with(&[RuntimeEvent::ClientOverloaded { client_id: 1 }]));

    let mut returned = server.connect_client();
    let welcome = returned.handshake(hello).unwrap();
    use herdr_server_client_v1::client_protocol::ServerMessage::{Closed, Welcome};
    assert_eq!(welcome, Welcome { client_id: 3, pane_ids: vec![1, 2, 3] });
    assert_eq!(returned.send(ClientMessage::Detach), Ok(Closed));

    let pane = server.create_pane();
    assert_eq!(healthy.next_update(), Ok(Some(pane)));
    assert_eq!(server.client_count(), 1);
}
```

```bash
cargo test
```

## 检查点与下一步

两个 v1 用例和 v3 的队列隔离用例都应通过。新用例同时观察过载连接、仍在读取的客户端和重连者，确认三者看到的状态来自同一个服务端。v4 接着处理拥有这个状态的服务端本身发生交接的情况。 [继续 v4](../v4/runtime-handoff.md)
