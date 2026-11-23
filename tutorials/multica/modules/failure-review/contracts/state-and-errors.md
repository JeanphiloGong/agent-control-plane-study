# Failure / Review：状态与错误

- provider/network、runtime offline、lease lost、权限拒绝和业务失败分别分类。
- Retry policy 必须有次数、时间或原因边界，不能无限重试。
- wakeup 条件未满足时保持等待，条件失效时不能重复 dispatch。
- Review 未通过时保留 Run 结果和缺口，不能回写为 shipped。
