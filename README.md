# agents_review（review-loop）

所有项目的集中评审仓库。Claude 作为评审方定时评审各项目编码方（如 Codex）的提交，评审意见和改善记录都存放在本仓库的 `projects/<项目名>/` 下。协作规则见 [AGENTS.md](AGENTS.md)。

## 工作方式

1. 编码方在项目仓库完成任务并提交，在本仓库写回应（`responses/`），并在 `projects/<项目名>/STATUS.md` 写上 `ready_for_review: true` 和项目仓库的 `head_commit`，然后推送。
2. launchd 定时运行 `review-run <项目名>`：先同步本仓库；只有 STATUS 标记就绪、且 `head_commit` 不同于 TRACKER 的 `last_reviewed_commit` 时才调用 Claude，否则只记一行日志，不产生调用。
3. `review-run` 在项目仓库 `head_commit` 的临时干净检出中运行 `TEST_CMD`，然后在该检出中调用 `claude -p`：核实已修复的问题、检查测试是否被弱化、写 `reviews/review-NNN.md`、更新 `TRACKER.md`。
4. Claude 只能写 `reviews/` 和 `TRACKER.md`，不能改项目代码。评审结束后由脚本只提交这两类文件并推送。

## 接入新项目

```bash
~/Documents/review-loop/bin/review-init <项目名> /path/to/repo
# 填写 projects/<项目名>/project.conf（TEST_CMD）和 PROJECT.md；本仓库和项目仓库分别提交
~/Documents/review-loop/bin/review-run <项目名> --force --dry-run   # 预览评审提示
~/Documents/review-loop/bin/review-install <项目名>                 # 安装每小时定时任务
```

## 命令

| 命令 | 作用 |
| --- | --- |
| `review-run <项目名> [--force] [--dry-run] [--no-push]` | 执行一次检查/评审。`--force` 忽略就绪判断，`--dry-run` 只打印提示，`--no-push` 只提交不推送 |
| `review-init <项目名> <项目仓库路径>` | 创建项目目录，并在项目 `AGENTS.md` 中追加协作指引 |
| `review-install <项目名> [--interval 秒] [--uninstall]` | 安装/卸载该项目的 launchd 任务（`com.review-loop.<项目名>`） |

## 项目配置 `projects/<项目名>/project.conf`

| 变量 | 默认值 | 说明 |
| --- | --- | --- |
| `REPO_PATH` | 无，必填 | 项目仓库本机路径 |
| `TEST_CMD` | 空 | 在干净检出中运行的测试命令 |
| `CLAUDE_BIN` | `~/.local/bin/claude` | Claude Code CLI 路径 |

## 日志

`~/Library/Logs/review-loop/<项目名>/`：`review.log` 记录每次检查与提交/推送，`review-*.log` 是每轮评审的 Claude 输出。

## 注意

- Mac 休眠时不会运行，醒来后继续按间隔检查。
- 需要 Claude Code CLI 已登录（`claude -p "OK"` 能返回），以及能推送本仓库的 git 凭据（当前用 SSH）。
- 规划中：调度脚本轮流唤醒编码方和 Claude，并通过 Telegram 推送摘要和人工关卡。
