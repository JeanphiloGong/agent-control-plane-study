# Herdr Server / Client v4：交接运行时后继续读取同一个 pane

[上一版 v3](../v3/multi-client-backpressure.md) · [本版计划](plan.md) · [模块路线图](../plan.md)

v3 的客户端断开后，可以重新连接原来的运行时。如果服务端也要换掉这份运行时，单靠客户端重连会连进一个空实例。交接需要同时解决两件事：新运行时继承 pane 和事件游标，已经拿到旧运行时句柄的连接停止写入。

对照 Herdr 的 src/server/headless/bootstrap.rs 中的 run_handoff_import_server。接收方先恢复状态并报告 ready，收到 committed 才开始承担所有权。src/server/handoff.rs 定义了这组确认消息。本篇在一个服务端进程内交接两个 Rust 运行时实例，沿用同一个 Unix 监听器。真实 Herdr 的跨进程通信、PTY 文件描述符传递和接管失败回滚属于上游实现。

完成 v3 后，从仓库根目录进入同一个项目。

```bash
cd tutorials/01-herdr-runtime/modules/01-server-client/v1/reference
```

## 1. 一次交接只产生一个运行时所有者

在 src/runtime_events.rs 的 RuntimeEvent 枚举末尾加入以下变体。

```rust
    RuntimeHandedOff,
```

在 src/server_runtime.rs 的 impl ServerRuntime 中加入 handoff，放在 start 之后。

```rust
    pub fn handoff(&self) -> Option<Self> {
        let mut old = self.inner.lock().unwrap();
        if !old.running {
            return None;
        }

        let leaving = std::mem::take(&mut old.connected_clients);
        for (client_id, _) in leaving {
            old.events.push(RuntimeEvent::ClientDisconnected { client_id });
        }
        old.events.push(RuntimeEvent::RuntimeHandedOff);
        old.running = false;

        Some(Self {
            inner: Arc::new(Mutex::new(RuntimeState {
                running: true,
                next_pane_id: old.next_pane_id,
                next_client_id: old.next_client_id,
                panes: std::mem::take(&mut old.panes),
                connected_clients: BTreeMap::new(),
                events: std::mem::take(&mut old.events),
            })),
        })
    }
```

交接和创建 pane 争用同一把锁。先完成的创建会进入被移交的 pane 和事件日志。交接后才拿到锁的写入会看到旧实例的 running 已经清除。客户端的 sender 随旧注册表清理，重连者会获得新的连接 ID。pane ID、下一个分配用的 ID 和事件顺序一起移交。

把 create_pane 的返回类型改为 Option<PaneSnapshot>，将方法开头从签名到 assert 所在行替换为以下代码。

```rust
    pub fn create_pane(&self) -> Option<PaneSnapshot> {
        let mut state = self.inner.lock().unwrap();
        if !state.running {
            return None;
        }
```

将该方法结尾的 pane 改成 Some(pane)。发布更新的代码继续在持锁期间运行。

完整替换同一个文件的 events_after 和 pane_snapshot。对 API 来说，旧句柄要返回 runtime_stopped，所以检查和读取放在同一次加锁中。

```rust
    pub(crate) fn events_after(
        &self,
        cursor: usize,
        limit: usize,
    ) -> Result<Vec<RuntimeEvent>, &'static str> {
        let state = self.inner.lock().unwrap();
        if !state.running {
            return Err("runtime_stopped");
        }
        let events = state.events.get(cursor..).ok_or("invalid_subscription")?;
        Ok(events.iter().take(limit).cloned().collect())
    }
```

```rust
    pub(crate) fn pane_snapshot(&self) -> Option<Vec<PaneSnapshot>> {
        let state = self.inner.lock().unwrap();
        state.running.then(|| state.panes.values().cloned().collect())
    }
```

在 src/api_socket.rs 的 serve_connection 中，把 ListPanes 和 CreatePane 两个分支替换成以下内容。Subscribe 分支继续调用 subscribe。

```rust
        Command::ListPanes => match runtime.pane_snapshot() {
            Some(panes) => write_json(&mut stream, json!({"id": id, "result": {"panes": panes}})),
            None => write_error(&mut stream, Some(id), "runtime_stopped"),
        },
        Command::CreatePane => match runtime.create_pane() {
            Some(pane) => write_json(&mut stream, json!({"id": id, "result": {"pane": pane}})),
            None => write_error(&mut stream, Some(id), "runtime_stopped"),
        },
```

完整替换 subscribe。开始订阅前和每次读取时都检查所属实例。一轮请求已经读出或提交的数据仍可能发送到客户端。

```rust
fn subscribe(
    stream: &mut UnixStream,
    runtime: &ServerRuntime,
    id: &str,
    params: Subscription,
) -> io::Result<()> {
    if !(1..=32).contains(&params.limit) {
        return write_error(stream, Some(id), "invalid_subscription");
    }
    if let Err(code) = runtime.events_after(params.after, 0) {
        return write_error(stream, Some(id), code);
    }
    let mut cursor = params.after;
    let mut remaining = params.limit;
    let deadline = Instant::now() + EVENT_WAIT;
    write_json(stream, json!({"id": id, "result": {"type": "subscription_started"}}))?;

    while remaining > 0 {
        if Instant::now() >= deadline {
            return write_error(stream, Some(id), "subscription_timeout");
        }
        let batch = match runtime.events_after(cursor, remaining) {
            Ok(batch) => batch,
            Err(code) => return write_error(stream, Some(id), code),
        };
        if batch.is_empty() {
            thread::sleep(Duration::from_millis(10));
        }
        for event in batch {
            cursor += 1;
            remaining -= 1;
            write_json(stream, json!({"id": id, "cursor": cursor, "event": event}))?;
        }
    }
    Ok(())
}
```

在 tests/detach_preserves_runtime.rs 和 tests/multi_client_backpressure.rs 中，将每个针对运行中服务端的 server.create_pane() 调用改为 server.create_pane().unwrap()。这些测试里的服务端始终处于运行状态，原有断言继续执行。

新建 tests/runtime_handoff.rs。用一对真实 Unix socket 调用旧实例，即使请求保留了交接前的句柄，也应该收到明确错误。

```rust
use std::io::{BufRead, BufReader, Write};
use std::os::unix::net::UnixStream;
use std::thread;

use herdr_server_client_v1::api_socket::serve_connection;
use herdr_server_client_v1::client_protocol::{ProtocolError, ServerMessage, TerminalHello};
use herdr_server_client_v1::runtime_events::RuntimeEvent;
use herdr_server_client_v1::server_runtime::ServerRuntime;
use serde_json::json;

#[test]
fn handoff_fences_old_connections_and_preserves_panes() {
    let old = ServerRuntime::start();
    let pane = old.create_pane().unwrap();
    let mut first = old.connect_client();
    let hello = TerminalHello { cols: 80, rows: 24 };
    first.handshake(hello.clone()).unwrap();
    let (mut client, stream) = UnixStream::pair().unwrap();
    let stale = old.clone();
    let worker = thread::spawn(move || serve_connection(stream, &stale).unwrap());

    let next = old.handoff().unwrap();
    assert!(!old.is_running());
    assert!(old.handoff().is_none());
    assert_eq!(first.snapshot(), Err(ProtocolError::Closed));
    writeln!(client, "{}", json!({"id": "late", "method": "pane.create"})).unwrap();
    let mut response = String::new();
    BufReader::new(client).read_line(&mut response).unwrap();
    assert_eq!(
        serde_json::from_str::<serde_json::Value>(&response).unwrap(),
        json!({"id": "late", "error": {"code": "runtime_stopped"}})
    );
    worker.join().unwrap();

    let mut second = next.connect_client();
    assert_eq!(second.handshake(hello), Ok(ServerMessage::Welcome {
        client_id: 2,
        pane_ids: vec![pane.pane_id],
    }));
    assert_eq!(second.snapshot(), Ok(ServerMessage::Snapshot { panes: vec![pane] }));
    assert_eq!(next.create_pane().unwrap().pane_id, 2);
    assert_eq!(next.events()[3], RuntimeEvent::RuntimeHandedOff);
}
```

```bash
cargo test
```

## 2. 让监听器把新连接交给接管后的运行时

上一节已经能在测试中交接。实际入口需要替换监听器之后的运行时句柄。这里用服务端进程自己的标准输入接收 handoff 命令，调用 socket 的客户端仍只走原来的 JSON API。完整替换 src/main.rs。

```rust
use std::fs;
use std::io::{self, BufRead};
use std::os::unix::fs::PermissionsExt;
use std::os::unix::net::UnixListener;
use std::path::PathBuf;
use std::sync::{Arc, Mutex};
use std::thread;

use herdr_server_client_v1::api_socket::serve_connection;
use herdr_server_client_v1::server_runtime::ServerRuntime;

fn main() -> io::Result<()> {
    let path = std::env::args_os().nth(1).map(PathBuf::from).ok_or_else(|| {
        io::Error::new(io::ErrorKind::InvalidInput, "usage: herdr_server_client_v1 SOCKET")
    })?;
    let listener = UnixListener::bind(&path)?;
    fs::set_permissions(path, fs::Permissions::from_mode(0o600))?;

    let current = Arc::new(Mutex::new(ServerRuntime::start()));
    let control = Arc::clone(&current);
    thread::spawn(move || {
        for command in io::stdin().lock().lines().map_while(Result::ok) {
            if command != "handoff" {
                continue;
            }
            let mut runtime = control.lock().unwrap();
            *runtime = runtime.handoff().expect("current runtime owns its state");
            println!("handoff complete");
        }
    });

    for stream in listener.incoming() {
        let stream = stream?;
        let runtime = current.lock().unwrap().clone();
        thread::spawn(move || {
            if let Err(error) = serve_connection(stream, &runtime) {
                eprintln!("connection ended: {error}");
            }
        });
    }
    Ok(())
}
```

监听器锁住 current 的时间仅够复制句柄。套接字读取和事件等待都发生在各自的线程内。服务端打印 handoff complete 后，新连接会取得接管后的句柄。

通过同一个 socket 连接两次，检查接管前的 pane 和游标在接管之后仍可用。

```bash
cargo build
python3 - <<'PY'
import json
import pathlib
import select
import socket
import subprocess
import tempfile
import time

with tempfile.TemporaryDirectory(prefix="herdr-handoff-") as directory:
    endpoint = pathlib.Path(directory) / "api.sock"
    service = subprocess.Popen(
        ["target/debug/herdr_server_client_v1", str(endpoint)],
        stdin=subprocess.PIPE, stdout=subprocess.PIPE,
    )
    try:
        for _ in range(100):
            if endpoint.exists():
                break
            time.sleep(0.02)

        def request(method, **fields):
            with socket.socket(socket.AF_UNIX, socket.SOCK_STREAM) as stream:
                stream.settimeout(4)
                stream.connect(str(endpoint))
                stream.sendall(json.dumps({"id": method, "method": method, **fields}).encode() + b"\n")
                with stream.makefile("rb") as source:
                    return [json.loads(line) for line in source]

        first = request("pane.create")[0]["result"]["pane"]
        assert request("events.subscribe", params={"after": 0, "limit": 1})[1]["cursor"] == 1
        service.stdin.write(b"handoff\n")
        service.stdin.flush()
        assert select.select([service.stdout], [], [], 4)[0], "handoff did not finish"
        assert service.stdout.readline() == b"handoff complete\n"
        assert request("pane.list")[0]["result"]["panes"] == [first]
        event = request("events.subscribe", params={"after": 1, "limit": 1})[1]
        assert event["cursor"] == 2
        assert event["event"]["type"] == "runtime_handed_off"
        assert request("pane.create")[0]["result"]["pane"]["pane_id"] == first["pane_id"] + 1
        print("Socket reconnect retained the pane and event cursor after handoff")
    finally:
        service.terminate()
        service.wait(timeout=5)
PY
```

## 检查点与下一步

四个 Rust 用例应通过，独立进程应打印 Socket reconnect retained the pane and event cursor after handoff。这个检查覆盖同进程运行时状态交接和真实 Unix 客户端重连。pane 仍是内存记录，服务端进程终止会丢失这一份状态。PTY 和 Session 恢复由 Herdr 的后续模块分别扩展。 [Herdr 路线图](../../../plan.md)
