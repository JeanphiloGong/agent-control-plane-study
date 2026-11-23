# ACP Runtime：接口契约

Runtime adapter 输入 Machine、runtime name、session config 和 AbortSignal，输出统一 ACP session/client。公共事件至少区分 text、tool、status、error 和 completion。凭据留在机器/runtime 边界。
