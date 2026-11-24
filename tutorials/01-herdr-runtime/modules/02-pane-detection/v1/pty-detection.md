# Herdr Pane Detection v1：把 PTY 事实保存为状态

[V0 设计](../V0-design.md) · [模块路线图](../plan.md) · [下一版 v2](../v2/manifest-evidence.md)

用户在 pane 中运行 Agent。服务端收到 PTY 输出或进程退出事实，需要先保存一个可读取的 Agent 状态。

## 1. 创建检测核心

创建 `v1/reference/Cargo.toml`、`src/lib.rs`、`src/detection.rs` 和测试目录。

```toml
[package]
name = "herdr_pane_detection"
version = "0.1.0"
edition = "2021"

[lib]
path = "src/lib.rs"
```

```rust
// src/lib.rs
pub mod detection;

// src/detection.rs
#[derive(Clone, Copy, Debug, Eq, PartialEq)]
pub enum AgentState { Unknown, Working, Idle }

#[derive(Clone, Debug, Eq, PartialEq)]
pub enum Evidence { Output(String), ProcessExited }

#[derive(Clone, Debug, Eq, PartialEq)]
pub struct TerminalState { pub state: AgentState, pub revision: u64 }

impl TerminalState { pub fn new() -> Self { Self { state: AgentState::Unknown, revision: 0 } } }

pub fn observe(terminal: &mut TerminalState, evidence: Evidence) -> AgentState {
    let next = match evidence {
        Evidence::ProcessExited => AgentState::Idle,
        Evidence::Output(text) if text.contains("working") => AgentState::Working,
        Evidence::Output(text) if text.trim_end().ends_with("$ ") => AgentState::Idle,
        Evidence::Output(_) => AgentState::Unknown,
    };
    if terminal.state != next { terminal.revision += 1; terminal.state = next; }
    next
}
```

## 2. 冻结输入边界

创建 `v1/reference/tests/detection.rs`：

```rust
use herdr_pane_detection::detection::{observe, AgentState, Evidence, TerminalState};

#[test]
fn output_and_exit_update_terminal_state() {
    let mut terminal = TerminalState::new();
    assert_eq!(observe(&mut terminal, Evidence::Output("agent working".into())), AgentState::Working);
    assert_eq!(observe(&mut terminal, Evidence::Output("plain text".into())), AgentState::Unknown);
    assert_eq!(observe(&mut terminal, Evidence::ProcessExited), AgentState::Idle);
    assert_eq!(terminal.revision, 3);
}
```

```bash
cargo test --manifest-path tutorials/01-herdr-runtime/modules/02-pane-detection/v1/reference/Cargo.toml
```
