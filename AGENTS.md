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
    PROJECT.md                # 项目元信息、必读文档、评审重点（用户 / 评审方维护）
    project.conf              # 脚本读取的配置：REPO_PATH、TEST_CMD（用户 / 评审方维护）
    STATUS.md                 # 编码方：有新提交待评审
    TRACKER.md                # 评审方：问题总表、状态、上次评审到的提交
    reviews/review-NNN.md     # 评审方：第 NNN 轮评审意见
    responses/review-NNN.response.md  # 编码方：对第 NNN 轮的改善记录
```

项目名使用小写短横线形式，与项目仓库名对应，例如 `Quant` → `projects/quant/`。

### PROJECT.md 内容

- 项目仓库地址（GitHub URL）与本机路径
- 主分支
- 测试命令（与 `project.conf` 的 `TEST_CMD` 一致）
- 必读的设计文档（如 `AGENTS.md`、`agent.md`）
- 本项目的评审重点（领域风险、易错点）
- 需要用户决策的长期原则（如有）

## 角色与写权限

| 文件 | 写入方 | 其他方 |
| --- | --- | --- |
| `PROJECT.md` / `project.conf` | 用户 / 评审方 | 只读 |
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
11. 提交并推送，提交信息形如 `quant: review-002`。定时任务中由 `bin/review-run` 负责同步、提交和推送，且只提交 `reviews/` 与 `TRACKER.md`。

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

```bash
~/code/review-loop/bin/review-init <项目名> <项目仓库路径>
```

它会创建 `projects/<项目名>/`（`project.conf`、`PROJECT.md`、`STATUS.md`、`TRACKER.md`、`reviews/`、`responses/`），并在项目仓库的 `AGENTS.md` 末尾追加「评审协作（agents_review）」一节，告诉该项目的编码方如何使用本仓库。已存在的文件不会覆盖。之后：

1. 填写 `project.conf` 的 `TEST_CMD` 和 `PROJECT.md` 的必读文档、评审重点。
2. 本仓库和项目仓库分别提交。
3. 运行 `bin/review-install <项目名>` 安装定时评审（见 README）。

## 迁移记录

- 2026-10-18：Quant 的评审记录从其仓库 `docs/reviews/` 迁入 `projects/quant/`，内容原样保留；Quant 的 `AGENTS.md` 已改为指向本仓库。
