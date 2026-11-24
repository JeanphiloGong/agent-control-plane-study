# Runtime Access：状态与错误

- `unknown`、`offline`、`blocked` 和 `ready` 必须可区分。
- ownership/access 拒绝先于 Runner 启动。
- Runtime 查询失败不能降级成 ready。
- adapter 启动失败要回传可分类错误，不直接修改 Issue 为 done。
