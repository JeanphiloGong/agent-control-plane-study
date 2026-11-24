# Session Lifecycle：状态与错误

- running session 的 cancel 需要确认 provider stop outcome。
- steer applied、not-applied、unknown 不能折叠成 success。
- archive 不销毁可恢复的历史事实。
- resume 需要验证 Machine 和 ACP capability，离线时停在不可执行状态。
