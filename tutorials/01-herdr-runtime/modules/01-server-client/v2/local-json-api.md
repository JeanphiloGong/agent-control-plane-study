# Herdr Server / Client v2：通过本地 Socket 查询运行时

[上一版 v1](../v1/server-client-runtime.md) · [本版计划](plan.md) · [下一版 v3](../v3/multi-client-backpressure.md)

在 v1 里，只有拿到 Rust 对象的调用者能看到 pane。现在让另一个进程也能查询和创建 pane，并按顺序接收运行时事件。两个入口复用 v1 的 ServerRuntime，所以 API 创建的 pane 也会出现在后续客户端的握手结果里。

Herdr 固定提交中的 src/api/server.rs 通过 handle_connection_with_stop 读取 JSON 请求，再把命令交给 App。stream_subscriptions 先回订阅确认，再发事件。这里沿用这个顺序，采用一条连接一次请求的 JSON Lines 子集。每轮订阅有条数和等待时间上限，后续连接可以沿游标继续读。

在 Linux 或 macOS 上完成 v1 后，从仓库根目录进入已有的 Rust 项目。v2 到 v4 都在同一目录增量修改。

```bash
cd tutorials/01-herdr-runtime/modules/01-server-client/v1/reference
cargo test
```

## 1. 给已有运行时加上可读取的事件

在 Cargo.toml 末尾加入依赖段。

```toml
[dependencies]
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```

在 src/client_protocol.rs 中，仅给 PaneSnapshot 的 derive 列表追加 serde::Serialize。完整声明如下。

```rust
#[derive(Clone, Debug, PartialEq, Eq, serde::Serialize)]
pub struct PaneSnapshot {
    pub pane_id: u64,
    pub running: bool,
}
```

完整替换 src/runtime_events.rs。

```rust
#[derive(Clone, Debug, PartialEq, Eq, serde::Serialize)]
#[serde(tag = "type", rename_all = "snake_case")]
pub enum RuntimeEvent {
    ClientConnected { client_id: u64 },
    ClientDetached { client_id: u64 },
    ClientDisconnected { client_id: u64 },
    PaneCreated { pane_id: u64 },
}
```

在 src/server_runtime.rs 中替换 ServerRuntime::create_pane，再把 events_after 放在它后面。插入 pane 和追加事件持有同一把锁，因此订阅者读到事件时，查询已经可以看到这个 pane。

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
        pane
    }

    pub(crate) fn events_after(&self, cursor: usize, limit: usize) -> Option<Vec<RuntimeEvent>> {
        let state = self.inner.lock().unwrap();
        let events = state.events.get(cursor..)?;
        Some(events.iter().take(limit).cloned().collect())
    }
```

cursor 表示已收到的事件数。0 从第一条开始，收到编号为 1 的事件后用 after=1 继续。大于当前日志长度的游标会被拒绝。这个编号只在当前运行时的日志中有意义。

更新 tests/detach_preserves_runtime.rs 中 detach_preserves_runtime_for_a_second_client 的最后一个断言，加入最早发生的 pane 创建事件。

```rust
    assert_eq!(
        server.events(),
        vec![
            RuntimeEvent::PaneCreated { pane_id: 1 },
            RuntimeEvent::ClientConnected { client_id: 1 },
            RuntimeEvent::ClientDetached { client_id: 1 },
            RuntimeEvent::ClientConnected { client_id: 2 },
        ]
    );
```

运行 v1 回归，确认新事件和原有的连接事件一起按顺序出现。

```bash
cargo test --test detach_preserves_runtime
```

## 2. 让一个 Socket 请求进入运行时

pane.list 返回当前快照，pane.create 创建一条记录。events.subscribe 要求 params 提供 after 和 limit，收到确认后输出至多 limit 条事件。响应沿用请求的字符串 id，事件另外带 cursor。输入先经过长度限制和 JSON 解析，再进入任何会修改运行时的方法。

新建 src/api_socket.rs。

```rust
use std::io::{self, BufRead, BufReader, Read, Write};
use std::os::unix::net::UnixStream;
use std::thread;
use std::time::{Duration, Instant};

use serde::Deserialize;
use serde_json::{json, Value};

use crate::server_runtime::ServerRuntime;

const MAX_LINE: u64 = 4096;
const IO_TIMEOUT: Duration = Duration::from_secs(1);
const EVENT_WAIT: Duration = Duration::from_secs(2);

#[derive(Deserialize)]
struct Request {
    id: String,
    #[serde(flatten)]
    command: Command,
}

#[derive(Deserialize)]
#[serde(tag = "method")]
enum Command {
    #[serde(rename = "pane.list")]
    ListPanes,
    #[serde(rename = "pane.create")]
    CreatePane,
    #[serde(rename = "events.subscribe")]
    Subscribe { params: Subscription },
}

#[derive(Deserialize)]
struct Subscription {
    after: usize,
    limit: usize,
}

pub fn serve_connection(mut stream: UnixStream, runtime: &ServerRuntime) -> io::Result<()> {
    stream.set_read_timeout(Some(IO_TIMEOUT))?;
    stream.set_write_timeout(Some(IO_TIMEOUT))?;
    let mut line = Vec::new();
    let mut reader = BufReader::new(&mut stream).take(MAX_LINE + 1);
    match reader.read_until(b'\n', &mut line) {
        Ok(0) => return Ok(()),
        Ok(_) if line.len() as u64 > MAX_LINE || line.last() != Some(&b'\n') => {
            return write_error(&mut stream, None, "invalid_request");
        }
        Err(err) if matches!(err.kind(), io::ErrorKind::TimedOut | io::ErrorKind::WouldBlock) => {
            return write_error(&mut stream, None, "request_timeout");
        }
        Err(err) => return Err(err),
        Ok(_) => {}
    }

    let value: Value = match serde_json::from_slice(&line) {
        Ok(value) => value,
        Err(_) => return write_error(&mut stream, None, "invalid_request"),
    };
    let request_id = value.get("id").and_then(Value::as_str).map(str::to_owned);
    let request: Request = match serde_json::from_value::<Request>(value) {
        Ok(request) if !request.id.is_empty() => request,
        _ => return write_error(&mut stream, request_id.as_deref(), "invalid_request"),
    };
    let id = request.id.as_str();
    match request.command {
        Command::ListPanes => write_json(
            &mut stream,
            json!({"id": id, "result": {"panes": runtime.pane_snapshot()}}),
        ),
        Command::CreatePane => {
            let pane = runtime.create_pane();
            write_json(&mut stream, json!({"id": id, "result": {"pane": pane}}))
        }
        Command::Subscribe { params } => subscribe(&mut stream, runtime, id, params),
    }
}

fn subscribe(
    stream: &mut UnixStream,
    runtime: &ServerRuntime,
    id: &str,
    params: Subscription,
) -> io::Result<()> {
    if !(1..=32).contains(&params.limit) || runtime.events_after(params.after, 0).is_none() {
        return write_error(stream, Some(id), "invalid_subscription");
    }
    let mut cursor = params.after;
    let mut remaining = params.limit;
    let deadline = Instant::now() + EVENT_WAIT;
    write_json(stream, json!({"id": id, "result": {"type": "subscription_started"}}))?;

    while remaining > 0 {
        if Instant::now() >= deadline {
            return write_error(stream, Some(id), "subscription_timeout");
        }
        let batch = runtime.events_after(cursor, remaining).unwrap();
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

fn write_error(stream: &mut UnixStream, id: Option<&str>, code: &str) -> io::Result<()> {
    write_json(stream, json!({"id": id, "error": {"code": code}}))
}

fn write_json(stream: &mut UnixStream, value: Value) -> io::Result<()> {
    serde_json::to_writer(&mut *stream, &value)?;
    stream.write_all(b"\n")
}
```

调用方无法从残缺 JSON 中可靠取回 id，所以这类错误返回 null。能解析 JSON、但命令或参数无效时，错误带回已读取的 id。连接读超时按空闲时间计算，订阅则有独立的等待窗口。订阅确认发出后，循环从日志读取后续批次，所以刚好在确认后产生的事件仍能拿到。

在 src/lib.rs 追加模块声明。

```rust
pub mod api_socket;
```

## 3. 从独立进程调用 API

新建 src/main.rs。启动时传入一个由当前用户创建的私有目录内的 socket 路径。Unix 文件权限限制这个练习的本地访问范围。

```rust
use std::fs;
use std::io;
use std::os::unix::fs::PermissionsExt;
use std::os::unix::net::UnixListener;
use std::path::PathBuf;
use std::thread;

use herdr_server_client_v1::api_socket::serve_connection;
use herdr_server_client_v1::server_runtime::ServerRuntime;

fn main() -> io::Result<()> {
    let path = std::env::args_os().nth(1).map(PathBuf::from).ok_or_else(|| {
        io::Error::new(io::ErrorKind::InvalidInput, "usage: herdr_server_client_v1 SOCKET")
    })?;
    let listener = UnixListener::bind(&path)?;
    fs::set_permissions(path, fs::Permissions::from_mode(0o600))?;
    let runtime = ServerRuntime::start();

    for stream in listener.incoming() {
        let stream = stream?;
        let runtime = runtime.clone();
        thread::spawn(move || {
            if let Err(error) = serve_connection(stream, &runtime) {
                eprintln!("connection ended: {error}");
            }
        });
    }
    Ok(())
}
```

每条连接在自己的线程内处理，所以订阅读取期间，另一条连接仍能创建 pane。用下面的命令启动独立服务端和 Python 标准库客户端。临时目录随验证结束清理。

```bash
cargo build
python3 - <<'PY'
import json
import pathlib
import socket
import subprocess
import tempfile
import time

with tempfile.TemporaryDirectory(prefix="herdr-api-") as directory:
    endpoint = str(pathlib.Path(directory) / "api.sock")
    service = subprocess.Popen(["target/debug/herdr_server_client_v1", endpoint])
    try:
        for _ in range(100):
            if pathlib.Path(endpoint).exists():
                break
            time.sleep(0.02)

        def connect():
            stream = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
            stream.settimeout(4)
            stream.connect(endpoint)
            return stream

        def request(message):
            with connect() as stream, stream.makefile("rb") as reader:
                stream.sendall(message + b"\n")
                return json.loads(reader.readline())

        assert request(b'{"id":"before","method":"pane.list"}') == {
            "id": "before", "result": {"panes": []}
        }
        with connect() as subscriber, subscriber.makefile("rb") as events:
            subscriber.sendall(b'{"id":"watch","method":"events.subscribe","params":{"after":0,"limit":1}}\n')
            assert json.loads(events.readline())["result"]["type"] == "subscription_started"
            created = request(b'{"id":"create","method":"pane.create"}')
            pane = created["result"]["pane"]
            assert created["id"] == "create"
            assert json.loads(events.readline()) == {
                "id": "watch",
                "cursor": 1,
                "event": {"type": "pane_created", "pane_id": pane["pane_id"]},
            }
        assert request(b'{"id":"bad","method":"missing"}')["error"]["code"] == "invalid_request"
        assert request(b'{"method":')["id"] is None
        assert request(b'{"id":"future","method":"events.subscribe","params":{"after":9,"limit":1}}')[
            "error"
        ]["code"] == "invalid_subscription"
        with connect() as stream, stream.makefile("rb") as reader:
            assert json.loads(reader.readline())["error"]["code"] == "request_timeout"
        assert request(b'{"id":"after","method":"pane.list"}')["result"]["panes"] == [pane]
        print("API request IDs, event cursor, and connection errors verified")
    finally:
        service.terminate()
        service.wait(timeout=5)
PY
```

## 检查点与下一步

独立进程可以查询和修改同一个运行时。验收同时检查订阅确认后的事件、请求 ID、未来游标、畸形输入和空闲超时。当前日志随进程存放在内存，连接离开后可用最后收到的游标继续读取。

v2 让 API 连接并行等待。接下来要解决 thin client 自己的发送队列，避免一个慢客户端拖住共享状态的发布。 [继续 v3](../v3/multi-client-backpressure.md)
