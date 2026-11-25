# Multica Runtime Access v2：ownership 与 access policy

[模块路线图](../plan.md) · [上一版 v1](../v1/plan.md) · [本版正文](access-policy.md) · [下一版 v3](../v3/plan.md)

## 版本定位

v1 只读取 Runtime 状态。v2 增加 `RuntimeAllowsUser` 和 workspace/owner 作用域检查，确保 shared Runtime 只允许明确的 owner、public 或 private shared 用户使用。访问拒绝发生在 adapter 之前。

## 实现顺序

1. 定义 Member 和 AccessVerdict。
2. 实现 owner/public/private shared policy，并检查 RunRequest workspace 与 Runtime workspace。
3. 用同一个 Runtime 记录测试所有权、共享、越权和 readiness 组合。

## 验收与交接

累计测试通过。v3 只接收 `Allowed + Ready` 的 StartRequest，offline 返回 deferred，拒绝和 unknown 不进入 adapter。
