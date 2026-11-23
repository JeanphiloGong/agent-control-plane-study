# Remote Endpoint / Plugin：接口与存储

## Endpoint contract

- `ProfileId` 是 32 位小写十六进制不透明标识。
- `ClientEndpointId::storage_key()` 只产生 `local` 或 `ssh:<profile_id>`，不写入 target、session 或 secret。
- bridge 暴露本地 socket；SSH target、session 和 options 只存在连接过程。
- metadata cache 可失效、重读和重新保存，不能成为永久真相来源。

## Plugin contract

- manifest 至少声明 id、name、version、min Herdr version 和合法命令/事件结构。
- `PluginInvocationContext` 是本次调用的只读输入；命令结果通过 `PluginCommandLogInfo` 和 AppEvent 回到服务端。
- registry JSON 是安装记录，不是 plugin runtime 的公共 API。
