# Multica Runtime Access v1：Runtime registry 与 readiness

[模块路线图](../plan.md) · [V0 设计](../V0-design.md) · [本版正文](runtime-registry.md) · [下一版 v2](../v2/plan.md)

## 版本定位

建立 Runtime record、lookup 和 availability verdict。online 是 ready，offline 是 waitable，unknown 或 unusable 是 blocked。此版本不检查调用者权限，也不启动 adapter。

## 实现顺序

1. 定义 Runtime、Agent、RunRequest 和 Registry。
2. 实现 lookup 与 readiness。
3. 测试状态分类和查询失败不降级。

## 验收与交接

`go test ./...` 通过。v2 在相同 Runtime record 上增加 owner/share policy，v3 再接 adapter。
