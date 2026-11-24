# Herdr Server / Client v4：运行时接管与客户端重连

[模块路线图](../plan.md) · [上一版 v3](../v3/plan.md) · [本版正文](runtime-handoff.md)

## 版本定位

接续 v3 的连接注册表和 v2 的 Unix 监听器。在一个进程中把 pane、编号和事件日志从旧运行时移交给新运行时。服务端的本地控制入口触发接管，JSON 客户端继续访问原来的 socket。

## 实现顺序

1. 用运行时已有的锁完成所有权转移。旧连接失效，延迟到达的创建请求返回 runtime_stopped。新客户端从当前快照和原游标继续读取。
2. 服务端从标准输入接收 handoff，更新监听器使用的运行时句柄，再输出完成确认。

## 验收与交接

cargo test 检查重复接管、旧请求拒绝、客户端重新握手、pane ID 和连接 ID 延续，并保留 v1、v3 的回归。独立进程检查接管后仍能在相同 Unix socket 上查询 pane，并从旧游标读取接管事件。练习覆盖同进程交接。真实 Herdr 跨进程 PTY 接管见上游源码，持久化重启由 Session Persistence 模块处理。
