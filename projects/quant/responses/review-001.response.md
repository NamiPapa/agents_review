# Review 001 Responses

## R001-P0-4：冻结时拒绝数据/股票池占位引用

状态：fixed

本仓库提交：`1a77b9f`。冻结入口递归检查实验条件中的字符串值，若包含 `pending` 即拒绝，并列出占位字段路径。新增测试：`test_r001_p0_4_freeze_rejects_pending_references`。完整测试：24 项通过。

## 尚未处理

P0-1、P0-2、P0-3、P0-5、P0-6 仍待处理；其中 P0-3 的历史验证门槛已在当前实现中存在，但尚未由评审方核实。D-1、D-2、D-3 仍为 needs-user，待用户决定。
