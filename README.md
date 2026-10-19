# review-loop

让 Claude 作为测试与评审方，定时评审编码方（如 Codex）的提交。双方只通过仓库里的 Markdown 文件沟通，不需要额外服务。

## 工作方式

1. 编码方按 `AGENTS.md` 中的协议完成任务并提交，然后在 `STATUS.md` 写上 `ready_for_review: true` 和 `head_commit`。
2. launchd 定时运行 `review-run`。只有 STATUS 标记就绪、且 `head_commit` 不同于 TRACKER 的 `last_reviewed_commit` 时才调用 Claude，否则只记一行日志，不产生调用。
3. `review-run` 在 `head_commit` 的临时干净检出中运行 `TEST_CMD`，然后调用 `claude -p`：核实已修复的问题、检查测试是否被弱化、写 `review-NNN.md`、更新 `TRACKER.md`。
4. Claude 只能读代码和 git 历史，只能写评审目录，不会改代码或提交；评审文件由编码方下次一并提交。

## 接入新仓库

```bash
~/Documents/review-loop/bin/review-init /path/to/repo
# 编辑 /path/to/repo/.review/config（TEST_CMD）和 .review/focus.md（项目特有评审重点），提交
~/Documents/review-loop/bin/review-run /path/to/repo --force --dry-run   # 预览评审提示
~/Documents/review-loop/bin/review-install /path/to/repo                  # 安装每小时定时任务
```

`review-init` 会创建 `.review/config`、`.review/focus.md`、`STATUS.md`、`TRACKER.md`，并在 `AGENTS.md` 末尾追加协议（已存在的不覆盖）。

## 命令

| 命令 | 作用 |
| --- | --- |
| `review-run <repo> [--force] [--dry-run]` | 执行一次检查/评审。`--force` 忽略就绪判断，`--dry-run` 只打印提示不调用 Claude |
| `review-init <repo>` | 初始化协议文件 |
| `review-install <repo> [--interval 秒] [--uninstall]` | 安装/卸载该仓库的 launchd 任务（`com.review-loop.<仓库名>`） |

## 仓库级配置 `.review/config`

| 变量 | 默认值 | 说明 |
| --- | --- | --- |
| `REVIEW_DIR` | `docs/reviews` | 评审文件目录，也是 Claude 唯一可写的目录 |
| `TEST_CMD` | 空 | 在干净检出中运行的测试命令 |
| `CLAUDE_BIN` | `~/.local/bin/claude` | Claude Code CLI 路径 |

## 日志

`~/Library/Logs/review-loop/<仓库名>/`：`review.log` 记录每次检查，`review-*.log` 是每轮评审的 Claude 输出。

## 注意

- Mac 休眠时不会运行，醒来后继续按间隔检查。
- 需要 Claude Code CLI 已登录（`claude -p "OK"` 能返回）。
- 规划中：调度脚本轮流唤醒编码方和 Claude，并通过 Telegram 推送摘要和人工关卡。
