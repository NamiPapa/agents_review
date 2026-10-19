# Review Tracker（由 Claude 维护，Codex 只读）

last_reviewed_commit: 1a77b9f
next_review_number: 003
last_review_at: 2026-10-19

状态说明：`open` 待处理；`claimed-fixed` Codex 称已修复、待 Claude 核实；`closed` 已核实；`needs-user` 等待用户决策；`wontfix` 用户决定不做。

## 待用户决策

| ID | 问题 | 状态 | 来源 |
| --- | --- | --- | --- |
| R001-D-1 | 实验系统范围：登记层 vs 系统自行驱动回测与 Paper（agent.md/PD-10 已倾向 A，待正式确认） | needs-user | review-001-design.md, review-002.md |
| R001-D-2 | Paper 阶段的目标：运行一致性 vs 收益证明 | needs-user | review-001-design.md |
| R001-D-3 | 首个策略的股票池方案（幸存者偏差） | needs-user | review-001-design.md |

## 问题清单

| ID | 严重度 | 问题 | 状态 | 修复提交 | 来源 |
| --- | --- | --- | --- | --- | --- |
| R001-P0-1 | P0 | 仓库无提交；冻结需记录 commit、dirty、环境锁定（已有提交和 .gitignore，冻结记录未做） | open ⚠️ | | review-001-design.md |
| R001-P0-2 | P0 | 默认占位策略配置可通过冻结校验 | open ⚠️ | | review-001-design.md |
| R001-P0-3 | P0 | 冻结不要求历史验证证据（门槛已存在，缺回应和 r001_p0_3 测试） | open ⚠️ | | review-001-design.md |
| R001-P0-4 | P0 | 数据/股票池占位符可进入冻结清单 | closed | 1a77b9f | review-001-design.md |
| R001-P0-5 | P0 | 历史价格逐行混用复权价与原始价（修复在未合并分支） | open ⚠️ | | review-001-design.md |
| R001-P0-6 | P0 | 数据层静默吞异常与静默回退数据源 | open ⚠️ | | review-001-design.md |
| R001-P1-1 | P1 | 信号与成交时间语义未定义（前视风险） | open | | review-001-design.md |
| R001-P1-2 | P1 | 观察期按自然日计算，缺交易日历 | open | | review-001-design.md |
| R001-P1-3 | P1 | 系统时钟不可注入 | open | | review-001-design.md |
| R001-P1-4 | P1 | 默认模板与必填校验、README 不一致 | open | | review-001-design.md |
| R001-P1-5 | P1 | 暂停/恢复/无效/拒绝可不填原因 | open | | review-001-design.md |
| R001-P1-6 | P1 | 清单哈希只能防误改不能防篡改 | open | | review-001-design.md |
| R001-P1-7 | P1 | 对比结论人工选择且不校验可比性 | open | | review-001-design.md |
| R001-P2-1 | P2 | 记录试验次数以防过拟合 | open | | review-001-design.md |
| R001-P2-2 | P2 | 验证报告自动给出 2 倍成本结果 | open | | review-001-design.md |
| R001-P2-3 | P2 | 筛选结果增加 eligible 标记 | open | | review-001-design.md |
| R002-P0-1 | P0 | get_price_history 不按 as_of 截断，回退路径泄漏未来数据 | open | | review-002.md |
| R002-P0-2 | P0 | 冻结只找任意一条 supports，忽略之后追加的否定/无效证据 | open | | review-002.md |
| R002-P1-1 | P1 | pending 检查只在冻结时做，带占位协议进入历史验证后无法修正 | open | | review-002.md |
| R002-P2-1 | P2 | "pending" 子串匹配误拦/漏拦，应改结构化占位标记 | open | | review-002.md |
| R002-P2-2 | P2 | 冻结时重新合并默认条件并写回，与验证时哈希口径不一致 | open | | review-002.md |
| R002-P2-3 | P2 | Yahoo 日线日期按本机时区转换 | open | | review-002.md |

⚠️ = P0 已连续两轮（001、002）为 open，需人工介入。

## 评审记录

| 轮次 | 文件 | 评审范围 | 日期 |
| --- | --- | --- | --- |
| 001 | review-001-design.md | 设计文档 + 2026-10-19 代码快照 | 2026-10-19 |
| 002 | review-002.md | 全部历史 9b5949d..1a77b9f（首次代码评审） | 2026-10-19 |
