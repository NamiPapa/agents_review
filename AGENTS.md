# agents_review 仓库 Agent 说明

本仓库是所有项目的**集中评审中心**。评审方 Agent（如 Claude）在这里为各项目写评审意见，编码方 Agent（如 Codex）在这里阅读意见并写改善记录。任何 Agent 进入本仓库开始工作前，先读完本文件。

## 核心约定

- 每个被评审的项目在 `projects/<项目名>/` 下有一个独立目录，所有评审与改善记录都放在这里，不放在项目代码仓库里。
- 被评审的代码仍在各自的项目仓库中；评审文件通过**项目仓库的提交号**引用代码。
- 评审方只写评审，不改项目代码；编码方只改项目代码和自己的改善记录，不改评审意见。双方通过本仓库的文件异步沟通。
- 所有记录只追加、不改写历史。结论变化通过新一轮评审或新回应表达。

## 目录结构

```text
AGENTS.md                     # 本文件
README.md                     # 工具用法
bin/                          # review-run / review-init / review-install
prompts/review_prompt.md      # 通用评审提示
templates/                    # 新项目模板
projects/
  <项目名>/
    PROJECT.md                # 项目元信息（评审方与用户维护）
    STATUS.md                 # 编码方：有新提交待评审
    TRACKER.md                # 评审方：问题总表、状态、上次评审到的提交
    reviews/review-NNN.md     # 评审方：第 NNN 轮评审意见
    responses/review-NNN.response.md  # 编码方：对第 NNN 轮的改善记录
```

项目名使用小写短横线形式，与项目仓库名对应，例如 `Quant` → `projects/quant/`。

### PROJECT.md 内容

- 项目仓库地址（GitHub URL）与本机路径
- 主分支
- 测试命令（在被评审提交的干净检出中运行）
- 必读的设计文档（如 `AGENTS.md`、`agent.md`）
- 本项目的评审重点（领域风险、易错点）
- 需要用户决策的长期原则（如有）

## 角色与写权限

| 文件 | 写入方 | 其他方 |
| --- | --- | --- |
| `PROJECT.md` | 用户 / 评审方 | 只读 |
| `STATUS.md` | 编码方 | 只读 |
| `TRACKER.md` | 评审方 | 只读 |
| `reviews/review-NNN*.md` | 评审方 | 只读，不得修改或删除 |
| `responses/review-NNN.response.md` | 编码方 | 只读 |

每个 Agent 提交时只能包含自己有权写入的文件。

## 评审方工作流程

1. `git pull --rebase` 同步本仓库。
2. 读 `projects/<项目名>/PROJECT.md`、`STATUS.md`、`TRACKER.md`，以及最新的评审和回应。
3. 仅当 `STATUS.md` 中 `ready_for_review: true` 且 `head_commit` 不同于 TRACKER 的 `last_reviewed_commit` 时开始评审。
4. 在项目仓库 `head_commit` 的干净检出中运行测试，并用 `git log` / `git diff` 查看 `last_reviewed_commit..head_commit`。
5. 逐条核实回应中声称 `fixed` 的问题：提交存在且以问题编号开头、满足验收标准、存在覆盖该问题的测试（测试名含编号小写，如 `test_r001_p0_1_...`）。通过则标 `closed`，否则保持 `open` 并写明原因。
6. 对 `disagree`：接受或反驳；无法一致的改为 `needs-user`。
7. 检查测试是否被删除、跳过或弱化；未在回应中说明理由的记为 P0。
8. 写 `reviews/review-NNN.md`，问题编号为 `RNNN-P0-x` / `P1` / `P2`（严重度递减）和 `RNNN-D-x`（需用户决策，列出选项）。每个问题写明：问题、影响、验收标准。
9. 更新 `TRACKER.md`：问题状态、修复提交、新问题、`last_reviewed_commit`、`next_review_number`、评审记录。
10. 某个 P0 连续两轮仍为 `open` 时，在评审开头加「⚠️ 需人工介入」。
11. 提交并推送，提交信息形如 `quant: review-002`。

## 编码方工作流程

在项目仓库开始任何任务前：

1. 同步本仓库，阅读 `projects/<项目名>/TRACKER.md` 和最新评审。
2. `open` 的 P0 优先于新功能；用户明确指定的任务除外，但要在回复中提醒用户还有未处理的 P0。
3. `needs-user` 的条目不得自行实现，向用户列出选项等待决定。

在项目仓库完成任务后：

1. 运行测试，记录通过/失败数量。
2. 每个逻辑改动在项目仓库单独提交；处理评审问题时提交信息以问题编号开头，例如 `R001-P0-2: 冻结前校验完整交易规则`。
3. 在本仓库写或更新 `responses/review-NNN.response.md`，每个问题一节：
   - `状态：fixed`，附项目仓库提交号和新增测试名；
   - `状态：disagree`，附理由（不得以“测试已通过”为理由）；
   - `状态：needs-user`，说明需要用户决定什么。
4. 更新 `STATUS.md`：`ready_for_review: true`、`head_commit`（项目仓库 `git rev-parse --short HEAD`）、本次摘要、测试结果、已回应的评审。
5. 在本仓库提交并推送，提交信息形如 `quant: response-001`。

禁止事项：

- 不得修改或删除评审文件和 `TRACKER.md`，不得自行关闭问题。
- 不得为了通过评审而删除、跳过或弱化测试；确需调整时在回应中说明原因。

## 接入新项目

1. 在 `projects/<项目名>/` 下从 `templates/` 创建 `PROJECT.md`、`STATUS.md`、`TRACKER.md`，以及空的 `reviews/`、`responses/`。
2. 在项目仓库的 `AGENTS.md`（或 `CLAUDE.md`）中加一段指引：

   ```markdown
   ## 评审协作
   本项目的评审意见与改善记录集中存放在 agents_review 仓库
   （https://github.com/NamiPapa/agents_review，本机 ~/Documents/review-loop）的
   `projects/<项目名>/` 下。开始任务前与完成任务后，按该仓库 AGENTS.md 中的「编码方工作流程」执行。
   ```

3. 安装定时评审任务（见 README）。

## 当前状态与迁移

- 现有脚本（`bin/review-*`）仍按旧方式工作：评审文件写在项目仓库的 `docs/reviews/`，配置在项目仓库的 `.review/`。
- 待迁移：脚本改为读写 `projects/<项目名>/`；Quant 已有的 `docs/reviews/` 历史（review-001、TRACKER、STATUS）迁到 `projects/quant/`；Quant 的 `AGENTS.md` 改为指向本仓库。
- 迁移完成前，项目仓库里的 `docs/reviews/` 仍是有效来源。
