# Herdr Pane Detection v3：避免空屏误报 idle

[上一版 v2](../v2/manifest-evidence.md) · [本版计划](plan.md) · [下一版 v4](../v4/plan.md)

v2 已能识别 blocker，但一次终端刷新可能读到空屏。v3 引入最小 pending idle 稳定器。

## 1. 添加 IdleStabilizer

在 `v1/reference/src/detection.rs` 中加入：

```rust
#[derive(Clone, Copy, Debug, Eq, PartialEq)]
pub enum Publish { Retain, Publish(AgentState) }

#[derive(Default)]
pub struct IdleStabilizer { pending_sequence: Option<u64>, confirmations: u8 }

impl IdleStabilizer {
    pub fn decide(&mut self, previous: AgentState, next: &AgentDetection, sequence: u64, hook_idle: bool) -> Publish {
        if next.state == AgentState::Blocked || next.visible_blocker || next.source == EvidenceSource::Process || hook_idle {
            self.pending_sequence = None;
            self.confirmations = 0;
            return Publish::Publish(next.state);
        }
        if previous == AgentState::Working && next.state == AgentState::Idle {
            if self.pending_sequence != Some(sequence) { self.pending_sequence = Some(sequence); self.confirmations = 0; }
            self.confirmations += 1;
            if self.confirmations < 2 { return Publish::Retain; }
        }
        self.pending_sequence = None;
        self.confirmations = 0;
        Publish::Publish(next.state)
    }
}
```

sequence 是 terminal detection content 的单调序号。测试 Working 后第一次 idle 为 Retain，第二次为 Publish，blocker、process exit 和 hook idle 直接 Publish。

```bash
cargo test --manifest-path tutorials/01-herdr-runtime/modules/02-pane-detection/v1/reference/Cargo.toml
```
