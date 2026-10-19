## Review 协作协议（编码方 ⇄ Claude）

Claude 担任测试与评审，由 review-loop 定时检查仓库。双方只通过 `{{REVIEW_DIR}}/` 下的文件沟通。

| 文件 | 写入方 | 用途 |
| --- | --- | --- |
| `{{REVIEW_DIR}}/STATUS.md` | 编码方 | 告诉评审方“有新提交待评审” |
| `{{REVIEW_DIR}}/TRACKER.md` | Claude | 所有评审问题的总表与状态、上次评审到的提交 |
| `{{REVIEW_DIR}}/review-NNN*.md` | Claude | 每轮评审意见 |
| `{{REVIEW_DIR}}/review-NNN.response.md` | 编码方 | 对第 NNN 轮意见的逐条回应 |

开始任何任务前：

1. 阅读 `{{REVIEW_DIR}}/TRACKER.md`。状态为 `open` 的 P0 问题优先于新功能；用户明确指定的任务除外，但要在回复中提醒还有未处理的 P0。
2. 状态为 `needs-user` 的条目不得自行实现，在回复中向用户列出选项等待决定。
3. 阅读与本次任务相关的最新 `review-NNN*.md`。

完成任务后（每次都要做）：

1. 运行 `.review/config` 中的 `TEST_CMD`，记录通过/失败数量。
2. 每个逻辑改动单独 `git commit`；处理评审问题时，提交信息以问题编号开头，例如 `R001-P0-2: 简述修复内容`。
3. `{{REVIEW_DIR}}/` 下 Claude 新写入的文件，在本次提交中一并提交（不要修改其内容）。
4. 若处理了评审问题，写或更新 `{{REVIEW_DIR}}/review-NNN.response.md`，每个问题一节：
   - `状态：fixed`，附提交号和新增测试名；或
   - `状态：disagree`，附理由（不得以“测试已通过”为理由）；或
   - `状态：needs-user`，说明需要用户决定什么。
5. 最后更新 `{{REVIEW_DIR}}/STATUS.md`：`ready_for_review: true`、`head_commit`（`git rev-parse --short HEAD`）、本次摘要、测试结果，然后提交。

禁止事项：

- 不得修改或删除 `review-NNN*.md` 和 `TRACKER.md`；不同意的意见写在 response 文件里。
- 不得为了让评审或测试通过而删除、跳过或弱化已有测试；确需调整测试时在 response 中说明原因。
- 不得自行把 TRACKER 中的问题标为关闭；只有 Claude 核实后才会关闭。
