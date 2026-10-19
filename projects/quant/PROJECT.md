# quant

量化交易系统原型：数据层、选股层、实验与版本管理、前端联调。

- 仓库：https://github.com/NamiPapa/Quant
- 本机路径：`/Users/binyao/Documents/Quant`
- 主分支：main
- 测试命令：`/usr/bin/python3 -m unittest discover -s tests`（见 `project.conf`）
- 编码方：Codex；评审方：Claude

## 必读文档

- `AGENTS.md`：当前代码结构、运行方式和评审协作指引
- `agent.md`：长期系统设计基线
- `README.md`

## 评审重点

以专业量化交易员 / 测试员视角评审，额外关注：

- 前视偏差：信号与成交的时间语义、`as_of` 之后的数据是否泄漏。
- 数据口径：复权价与原始价是否混用、数据源静默回退或吞异常、幸存者偏差。
- 与设计基线 `agent.md` 的一致性：固定版本、不可变清单、追加式事件、实验管理不得触发下单。
- 冻结与验证证据：冻结校验是否可被占位配置绕过，验证结果是否只能追加。

## 需用户确认的原则

- 修改设计基线 `agent.md` 前必须得到用户确认；`AGENTS.md` / `README.md` 可以随代码更新。

## 迁移记录

- 2026-10-18：评审记录从 Quant 仓库 `docs/reviews/` 迁入本目录（STATUS、TRACKER、review-001-design.md 原样保留）。TRACKER「来源」列中的文件名指 `reviews/` 下的同名文件。
