# Review 002：首次代码评审（全部历史至 1a77b9f）

- 评审范围：项目仓库全部历史 `9b5949d..1a77b9f`（首次按提交评审；review-001 为设计评审，`last_reviewed_commit: none`）
- head_commit：`1a77b9f`（工作区干净）
- 评审日期：2026-10-19
- 评审视角：专业量化交易员 / 测试员；重点见 PROJECT.md（前视偏差、数据口径、设计基线一致性、冻结与验证证据）

## ⚠️ 需人工介入

以下 P0 在 review-001 提出，本轮（review-002）仍为 `open`，已连续两轮未关闭：

| ID | 问题 | 当前情况 |
| --- | --- | --- |
| R001-P0-1 | 冻结需记录 commit、dirty、环境锁定 | 仓库已有提交、`.gitignore` 已补齐（要求 1、2 已满足）；冻结时仍未读取 `git rev-parse HEAD` / `git status`，清单无 `code.commit`、`code.dirty`、`env.python`、`env.lock_hash`。 |
| R001-P0-2 | 默认占位策略配置可通过冻结 | 未处理。`REQUIRED_STRATEGY_CONFIG_SECTIONS` 仍只有两项，`implementation_id: market-observation-system:pending` 仍可冻结。 |
| R001-P0-3 | 冻结要求历史验证证据 | 代码已有门槛，但回应未声明 `fixed`，也没有 `test_r001_p0_3_*` 测试（见下文）。 |
| R001-P0-5 | 复权价/原始价混用 | `main` 未改；PLAN.md 称修复在未合并分支 `tg/codex-20261019-001940`，不在本轮评审范围。 |
| R001-P0-6 | 数据层静默吞异常/静默回退 | 未处理。`historical_us.py:42`、`:74` 仍为 `except Exception: continue`。 |

说明：这与 PLAN.md 中 PD-13“每次只处理一项问题”的节奏有关，不代表编码方忽视问题。但 P0-2 和 P0-3 是冻结门槛本身，P0-5/P0-6 是数据可信度。请用户决定是否催促合并 P0-5 分支，并确认后续处理顺序（建议：P0-3 回应 → P0-2 → P0-5 合并 → P0-6 → P0-1）。

## 测试结果

`/usr/bin/python3 -m unittest discover -s tests`：**24 项，全部通过**（仅有 urllib3/LibreSSL 警告，与本项目无关）。

测试删除/跳过/弱化检查：

- 全部历史中只有 `1a77b9f` 改动了测试。`_draft_payload` 的条件从 `{}` 改为给出 `data_reference` / `universe_reference` 的非占位值。这是新冻结规则的必要配合，没有放宽断言，不算弱化。建议以后在回应中顺带说明这类测试辅助函数的改动。
- 提醒：`test_freeze_requires_validation_then_produces_immutable_manifest` 第 147 行断言冻结清单的配置等于 `DEFAULT_STRATEGY_CONFIG`，相当于把 R001-P0-2 的缺陷写成了预期行为。修复 P0-2 时需要改这条断言，属于合理调整，请在回应中注明，不会记为弱化。

## 已核实问题

### R001-P0-4：冻结时拒绝 pending 占位引用 → **closed**

- 提交 `1a77b9f`，提交信息以 `R001-P0-4:` 开头。✔
- 实现：`freeze_experiment` 先把条件与默认值合并（`_normalize_experiment_fields`），再用 `_pending_value_paths` 递归检查所有字符串值，出现 `pending`（不区分大小写）就拒绝冻结，并列出字段路径。调用方省略 `data_reference` 时，默认的 `...:pending-...` 也会被合并进来并被拦下。✔
- 测试：`test_r001_p0_4_freeze_rejects_pending_references`。测试先进入历史验证，再记录一条 `historical/supports` 结果，然后断言冻结失败且错误信息包含 `conditions.data_reference`。这确实覆盖了“其他门槛都满足，只因占位值被拒”的场景。✔
- 验收标准“条件中含 `pending` 时冻结失败”已满足。原要求中“引用必须指向真实快照 ID”依赖快照功能，review-001 已允许在快照功能实现前先只拒绝占位符。
- 遗留的改进（不阻止关闭）另记为 R002-P1-1（检查时机太晚）和 R002-P2-1（字符串匹配太粗）。

## 未核实 / 仍为 open 的问题

### R001-P0-3：冻结要求历史验证证据 → 保持 **open**

代码审阅：`freeze_experiment`（`service.py:563-577`）要求同一实验下存在 `historical` + `supports` 且 `validation_input_hash` 一致的结果。输入哈希包括版本 ID、假设 ID、计划窗口、配置哈希、条件哈希和四项评价规则，满足 review-001 的要求。`_assert_frozen_integrity` 在进入 `running` 前会再次核对证据。

不能关闭的原因：

1. 回应没有声明 `状态：fixed`，也没有引用提交（门槛是在初始提交里引入的，没有以 `R001-P0-3` 开头的提交）。
2. 没有 `test_r001_p0_3_*` 测试。现有 `test_freeze_requires_validation_...` 覆盖了“没有结果”“结论为 inconclusive”“类型为 external”三种情况，但没有覆盖验收标准中的 **`does_not_support`** 和 **配置/条件哈希不一致** 两种情况。
3. 新发现的绕过问题 R002-P0-2（只取最新的 `supports`，忽略之后追加的否定或无效证据）属于同一门槛，应一并处理。

关闭条件：写回应（可以引用 `9b5949d` 并说明门槛已在初始提交中存在），补充 `test_r001_p0_3_...`，覆盖 `does_not_support` 和哈希不一致（例如直接改数据库中的 `conditions_json`，或在测试里打补丁替换 `DEFAULT_CONDITIONS`），并同时修复 R002-P0-2。

### R001-P0-1 / P0-2 / P0-5 / P0-6

现状见开头的「⚠️ 需人工介入」。R001-P1-1 ~ P1-7、P2-1 ~ P2-3 均未处理，保持 open。

## 待用户决策（沿用）

- **R001-D-1**：说明一点：`agent.md` §1 和 PLAN.md PD-10 已经把实验系统写成“研究控制层 / 登记层”，执行与回测由独立系统负责，相当于方案 A。但 PLAN.md 第 4 节仍把 D-1 列为待决定。请用户明确确认 A（届时需要补上外部回传结果的数据契约和来源校验，见 R002-P0-2 后面的说明），或改选 B。确认前保持 `needs-user`。
- **R001-D-2、R001-D-3**：没有新信息，保持 `needs-user`。

---

## 新问题

### R002-P0-1 `get_price_history` 不按 `as_of` 截断，回退路径会泄漏未来数据

- 位置：`quant_data/providers/historical_us.py:93-106`、`:174-186`；消费方 `quant_data/services/stock_profile.py:30-36`、`quant_data/services/technical.py:10-22`、`quant_app/api.py:63-64`。
- 问题：主路径 `_yahoo_history` 返回空时，会回退到 `_yahoo_recent_history`。这个函数固定请求 `range=1y`，返回**截至今天**的一年数据，完全忽略 `start` / `end`。`get_price_history` 拿到结果后不按 `[start, end]` 过滤就直接返回并缓存。`calculate_technicals` 用 `close.iloc[-1]`、`tail(5/20/60)`、`rolling` 计算指标时，也不按 `as_of` 截断。所以在 `as_of` 为过去日期、主请求失败的情况下，个股详情、量价分（`build_profile_for_snapshot`）和趋势图用到的“最新收盘价”、MA、RSI、MACD、动量都来自 `as_of` 之后的行情。`_stock_snapshot` 和 `list_sectors` 通过 `_latest_row` 做了截断，因此**同一只股票在列表页和详情页的数值会不一致**，问题更难发现。另外，`as_of` 早于一年前时，这条回退路径会返回完全不相关的区间。
- 核实方式：代码阅读。路径确定、没有分支条件，本轮环境不允许执行临时复现脚本。
- 影响：直接违反评审重点第一条（前视偏差）。目前虽然只用于页面展示，但 `quant_data` 会演进为数据与市场观察系统（`agent.md` §11），这些分数将来会成为验证输入。严重度与 R001-P0-5 相同，所以列为 P0。
- 验收标准：
  1. `get_price_history` 返回的每一行都满足 `start <= date <= end`，所有数据源和回退路径都一样；`_yahoo_recent_history` 不能返回请求区间以外的数据（不能覆盖请求区间时视为失败，并按 R001-P0-6 记录到质量报告）。
  2. `calculate_technicals(symbol, prices, as_of)` 自己先截断到 `date <= as_of`（防御式编程，不依赖调用方）。
  3. 测试 `test_r002_p0_1_...`：替换数据源函数，让回退路径返回超过 `as_of` 的行，断言 `get_price_history` 的最大日期 `<= as_of`，并断言 `build_profile_for_snapshot` 的 `ma_5` / 收盘价只使用 `as_of` 及之前的数据。

### R002-P0-2 冻结只找“任意一条 supports”，之后追加的否定/无效证据被忽略

- 位置：`quant_experiments/service.py:563-577`。
- 问题：冻结查询取的是输入哈希一致、`verdict='supports'` 的**最新一条**，完全不看同一输入下的其他结果。由此产生两个问题：
  1. **纠错无效**：`agent.md` §2 原则 3 规定“修正数据或方法时，新增一条结果”。但先记录 `supports`、发现数据泄漏后再追加 `invalid` 或 `does_not_support`，冻结仍然会用旧的 `supports` 通过。追加式纠错对冻结门槛没有任何作用。
  2. **挑选结果**：同一输入下记录 10 次 `does_not_support` 和 1 次 `supports` 也能冻结，违背 §9“保留全部尝试、不能挑选样本”的原则。
- 影响：冻结门槛可以被绕过（评审重点第四条）。在 D-1 倾向方案 A 的前提下，验证结果全部来自外部回传，这个门槛几乎是唯一的防线。
- 验收标准：
  1. 冻结使用的证据必须是同一 `validation_input_hash` 下**最新的** `historical` 结果，且结论为 `supports`。最新一条是 `does_not_support`、`inconclusive` 或 `invalid` 时，冻结失败。或者另一种做法：只要在该 `supports` 之后出现过 `invalid` / `does_not_support` 就拒绝。两种做法任选其一，在回应中说明。
  2. （建议）验证结果增加可选的 `supersedes_result_id`，让纠错关系可以追溯。冻结清单的 `validation_gate` 记录同一输入下的历史结果总数和各结论的计数，供后续 P2-1（试验次数）使用。
  3. 测试 `test_r002_p0_2_...`：先记录 `supports` 再追加 `invalid`，冻结失败；先记录 `does_not_support` 再记录 `supports`，冻结按所选规则得到明确结果。

### R002-P1-1 占位检查太晚：实验带着 pending 协议进入历史验证后无法修正

- 位置：`service.py:549-553`（只在冻结时检查）；`models.py` `ALLOWED_TRANSITIONS["historical_validation"] = {"rejected", "invalidated"}`；`update_experiment` 只允许修改草稿。
- 问题：`agent.md` §2 原则 2 要求“验证协议先于结果”。现在一个条件仍是 `pending-data-snapshot` 的草稿可以进入 `historical_validation`，并记录 `supports` 结果（这些结果绑定的是占位协议）。直到冻结时才被拒绝，而这时条件已经不能修改，也不能退回草稿，实验只能被拒绝或标记无效。前端草稿表单默认填入 `DEFAULT_CONDITIONS`（`app.js:1898`），按默认流程操作就会走进这条死路。
- 影响：协议不完整的“证据”被写进不可变记录；用户只能放弃这个实验和假设（一个假设只能属于一项实验），白白消耗一次试验次数。
- 验收标准：`draft → historical_validation` 迁移时执行同样的占位检查（`record_validation_result` 也建议检查）；冻结时保留这项检查作为最后一道防线。测试 `test_r002_p1_1_...`：条件含 pending 的草稿无法进入历史验证，错误信息列出字段路径。

### R002-P2-1 `"pending" in value` 子串匹配既会误拦也会漏拦

- 位置：`service.py:1687-1703`。
- 问题：任何正常文字中出现 pending 都会被拦，例如 `data_availability_rule: "pending corporate actions excluded"`。而 `DEFAULT_STRATEGY_CONFIG` 中的 `"待填写本轮相对基线的研究变量。"` 这类中文占位符却拦不住（策略配置部分将由 R001-P0-2 处理）。
- 验收标准：占位符改用结构化标记（例如统一前缀 `__placeholder__:`，或在 models 中维护占位值集合），默认模板只使用这类标记；数据和股票池引用改用格式校验（例如 `^[a-z0-9-]+:[A-Za-z0-9._-]+$`，且不能是占位符）。为误拦和漏拦各补一个测试。

### R002-P2-2 冻结时重新合并默认条件，验证时却用原始条件计算哈希

- 位置：`record_validation_result`（`service.py:222-224`）用数据库中原样的 `conditions_json` 计算 `conditions_hash`；`freeze_experiment`（`:548`、`:557`、`:603`）则先与**当前代码**的 `DEFAULT_CONDITIONS` 合并再计算哈希，并把合并后的结果**写回** `conditions_json`。
- 问题：只要 `DEFAULT_CONDITIONS` 在实验创建后新增了键（例如修复 R001-P1-4 补默认值时），已有实验的冻结哈希就会对不上。用户看到的错误是“必须记录一条支持假设的历史验证结果”，实际上结果存在，错误信息有误导性。如果哈希恰好一致，冻结又会悄悄把新的默认值写进已验证的协议。目前不会放错（哈希不一致时冻结失败），但修复 R001-P1-4 时一定会碰到。
- 验收标准：冻结只读取并计算已保存的条件，不再合并默认值、不写回 `conditions_json`（校验可以在副本上做）；哈希不一致时给出“协议已变化（缺少或多出哪些键）”的明确错误。

### R002-P2-3 Yahoo 日线日期按本机时区转换

- 位置：`historical_us.py:163` 的 `datetime.fromtimestamp(timestamp).date()`，以及 `:130-131` 用本机时区午夜计算 `period1` / `period2`。
- 问题：K 线日期取决于运行机器的时区，应按交易所时区（`America/New_York`，或响应 `meta.exchangeTimezoneName`）转换。目前美洲和亚洲时区下结果碰巧正确，但这是隐含假设；放到 UTC 的服务器或 CI 上，`period2` 的边界可能多取或少取一天。
- 验收标准：时间戳按交易所时区转换为交易日期，查询区间也按该时区计算；补充一个测试，在 `TZ` 不同时得到相同的日期序列。

---

## 本轮小结

- 核实关闭：1（R001-P0-4）。
- 新增：P0 × 2、P1 × 1、P2 × 3。
- 需人工介入：R001-P0-1/2/3/5/6 已连续两轮为 open。
- 待用户决策：R001-D-1/2/3（D-1 设计基线已倾向方案 A，请正式确认）。
