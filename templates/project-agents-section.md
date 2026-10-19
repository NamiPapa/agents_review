## 评审协作（agents_review）

本项目的评审意见与改善记录不放在本仓库，而是集中存放在 agents_review 评审仓库：

- GitHub：https://github.com/NamiPapa/agents_review
- 本机路径：`~/code/review-loop`
- 本项目目录：`projects/{{PROJECT}}/`
- 完整规则：评审仓库的 `AGENTS.md`（「编码方工作流程」）

| 文件（相对 `projects/{{PROJECT}}/`） | 写入方 | 用途 |
| --- | --- | --- |
| `PROJECT.md` / `project.conf` | 用户 / 评审方 | 项目元信息、测试命令、评审重点 |
| `STATUS.md` | 编码方 | 告诉评审方“有新提交待评审” |
| `TRACKER.md` | 评审方 | 问题总表与状态、上次评审到的提交 |
| `reviews/review-NNN*.md` | 评审方 | 每轮评审意见 |
| `responses/review-NNN.response.md` | 编码方 | 对第 NNN 轮意见的改善记录 |

开始任何任务前：

1. 在评审仓库执行 `git pull --rebase`，阅读 `projects/{{PROJECT}}/TRACKER.md` 和与本次任务相关的最新评审。
2. `open` 的 P0 问题优先于新功能；用户明确指定的任务除外，但要在回复中提醒还有未处理的 P0。
3. `needs-user` 的条目不得自行实现，在回复中向用户列出选项等待决定。

完成任务后（每次都要做）：

1. 在本仓库运行测试（命令见 `project.conf` 的 `TEST_CMD`），记录通过/失败数量。
2. 每个逻辑改动在本仓库单独 `git commit`；处理评审问题时提交信息以问题编号开头，例如 `R001-P0-2: 简述修复内容`。
3. 若处理了评审问题，在评审仓库写或更新 `responses/review-NNN.response.md`，每个问题一节：
   - `状态：fixed`，附本仓库提交号和新增测试名；
   - `状态：disagree`，附理由（不得以“测试已通过”为理由）；
   - `状态：needs-user`，说明需要用户决定什么。
4. 更新评审仓库的 `STATUS.md`：`ready_for_review: true`、`head_commit`（本仓库 `git rev-parse --short HEAD`）、本次摘要、测试结果。
5. 在评审仓库只提交上述两类文件并推送，提交信息形如 `{{PROJECT}}: response-NNN`。

禁止事项：

- 不得修改或删除评审文件和 `TRACKER.md`；不同意的意见写在回应文件里。
- 不得为了通过评审而删除、跳过或弱化已有测试；确需调整时在回应中说明原因。
- 不得自行把问题标为关闭；只有评审方核实后才会关闭。
