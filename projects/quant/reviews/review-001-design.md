# Review 001：设计文档评审与实现方案

- 评审对象：`agent.md`（设计基线）、`agents.md`、`README.md`，以及 2026-10-19 05:40 UTC 时的 `quant_experiments/` 代码快照
- 评审视角：专业量化交易员 / 测试员
- 说明：评审时 Codex 正在修改 `quant_experiments/service.py`（验证结果、实验对比），测试有 9 个 `NameError`，属于改动未完成，不计为缺陷。本文件只对已稳定的部分提出代码问题，完整代码评审见 Review 002。

给 Codex 的执行规则：

1. 按编号逐条处理，每条一个独立提交，提交信息以编号开头（如 `R001-P0-1: ...`）。
2. 每条都有“验收标准”，必须补充对应自动化测试，测试名包含编号小写（如 `test_r001_p0_1_...`）。
3. 标记为【需用户决策】的条目不要自行实现，先在回复中列出选项等待确认。
4. 修改设计基线 `agent.md` 前必须得到用户确认；可以更新 `agents.md` / `README.md` 使其与代码一致。

---

## 一、总体判断

设计基线在**治理层面**写得很好：冻结版本、不可变清单、追加式事件、失败也保留、Agent 不得下单、公平比较规则，这些都是专业机构做法，方向正确，可行。

但从交易员角度看，它**重治理、轻“结果是否可信”**。一个实验结论是否可信，取决于三件事，而这三件事在设计里都只有原则、没有规格：

1. **点时数据（point-in-time data）**：用的数据在当时是否真的可得、是否有幸存者偏差、复权口径是否一致。
2. **确定性的回测引擎**：“历史验证”由谁计算？目前是人工填入指标，等于自我申报。
3. **回测与 Paper 用同一份策略代码**：否则 Paper 结果和历史验证无法比较，冻结也就失去意义。

治理做得再严格，如果底层数据有偏差、指标是手填的，系统只是把错误的结论“可审计地”保存下来。因此建议实施顺序调整为：先补齐数据快照、策略接口和回测引擎，再继续扩展实验管理的界面和流程。

---

## 二、需用户决策的问题

### D-1【需用户决策】实验系统的范围出现分歧

`agent.md` 的目标是完整闭环（信号 → 目标组合 → 风控 → 订单 → 账本 → 对账）。但当前代码已经把实验系统改成“登记簿”：

- `models.py` 中 `running` 的标签是“外部执行中”，`paper_ready` 是“可交接执行”；
- `DEFAULT_STRATEGY_CONFIG` 写着“本系统不解释或执行这些规则”；
- 验证结果由外部回传，系统只保存。

两条路都可行，但必须二选一并写进 `agent.md`：

- **A. 实验系统 = 登记与治理层**，执行与回测是独立系统，通过接口回传结果。需要定义回传接口的数据契约和签名/哈希校验。
- **B. 实验系统驱动执行**：同一 Python 应用内有回测引擎和 Paper Runner，实验系统直接调用。

我的建议：**当前阶段选 B 的简化版**。个人项目没有必要拆成多个系统，拆开后“结果是自我申报”的问题更严重。模块边界保留在代码层面（`quant_backtest/`、`quant_execution/`），但由系统自己计算验证结果。

### D-2【需用户决策】Paper 阶段要证明什么

设计默认最低观察期 20 天（且按自然日计算）。对日频策略，20 个交易日在统计上无法区分 alpha 和噪声（年化 Sharpe 1.0 的策略，20 天内亏损的概率约 40%）。

建议明确：**Paper 阶段的主要目的是运行验证，不是收益验证**。即证明“实盘运行结果 ≈ 同期回测结果”。核心指标改为：

- 跟踪误差：Paper 净值 vs. 用同期真实数据、同一冻结版本重跑的回测净值；
- 实施差额（implementation shortfall）：决策价 vs. 实际成交价；
- 运行完整性：调度成功率、对账差异、事故数。

收益层面的结论主要来自样本外历史验证 + 足够长的 Paper 累计期（通常至少 3–6 个月）。

### D-3【需用户决策】首个策略的股票池

当前股票池是 `SECTOR_ETFS` 中硬编码的代表股（今天的成分股），用于历史回测会产生幸存者偏差。可选：

- **A.** 接受偏差，但只做 Paper 前向实验，不用历史回测得出收益结论（最简单）；
- **B.** 在冻结时生成股票池快照（例如当日按成交额排名前 N 的美股），之后只前向使用；
- **C.** 接入带历史成分的数据源（付费，例如 Norgate、Tiingo + 历史成分表）。

建议当前用 **A + B**：历史回测只用于检查逻辑和量级，前向实验的股票池在冻结时快照。

---

## 三、问题清单

严重程度：**P0** 阻断（会导致结论不可信或违反设计原则）、**P1** 重要、**P2** 建议。

### P0-1 仓库没有任何提交，`source_revision` 无法成立

- 现状：`git log` 显示 `main` 分支没有任何 commit，大部分文件未跟踪。`source_revision` 是可选自由文本。
- 风险：设计原则 1、2 依赖“代码版本可追溯”，目前无法实现；我也无法按提交评审 Codex 的改动。
- 要求：
  1. 立即做首次提交；之后每个任务至少一个提交。
  2. `.gitignore` 排除 `.venv/`、`.idea/`、`.DS_Store`、`.cache/`、`outputs/`、`__pycache__/`、`data/*.sqlite3*`。
  3. 冻结时由系统自动读取 `git rev-parse HEAD` 和 `git status --porcelain`；**工作区不干净时拒绝冻结**，或在清单中记录 `dirty: true` 并在启动时拒绝运行。脏检查排除 `docs/reviews/` 目录（评审文件由 Claude 写入，不影响交易行为）。
  4. 清单中记录 Python 版本和依赖锁定（`requirements.lock` 或 `pip freeze` 的哈希）。
- 验收标准：工作区有未提交修改时 `freeze` 返回明确错误；清单含 `code.commit`、`code.dirty`、`env.python`、`env.lock_hash`。

### P0-2 默认策略配置可以通过冻结校验

- 现状：`REQUIRED_STRATEGY_CONFIG_SECTIONS` 只有 `strategy_reference`、`research_scope`。默认配置中 `implementation_id` 为 `market-observation-system:pending`，也能通过 `_validate_strategy_config`，从而冻结并进入 `running`。错误提示文字却是“尚未定义完整交易规则”。
- 风险：可以冻结并“运行”一个没有任何交易规则的实验，违反 `agent.md` §7.2 和 §11。
- 要求：冻结前校验策略配置至少包含 `universe`、`signal`、`portfolio`（持仓数、仓位方法）、`rebalance`（频率、时点）、`exit`（常规退出、风险退出）、`risk` 六个部分；任何值包含 `pending` 占位符时拒绝冻结。用 JSON Schema 或 dataclass 定义，不要只检查键是否存在。
- 验收标准：使用 `DEFAULT_STRATEGY_CONFIG` 的实验调用 `freeze` 失败；补齐六个部分后成功。

### P0-3 冻结不要求历史验证证据

- 现状：`freeze_experiment` 只检查状态为 `historical_validation`，不检查是否存在验证结果。（Codex 正在添加验证结果功能，完成后需满足下述要求。）
- 风险：违反 `agent.md` §9 历史验证关卡。
- 要求：冻结前必须存在至少一条 `validation_type=historical`、`verdict=supports` 的结果，且其 `strategy_config_hash`、`conditions_hash` 与当前一致；否则拒绝。
- 验收标准：无验证结果、结论为 `does_not_support`、或配置哈希不一致时，冻结均失败。

### P0-4 数据占位符可以进入冻结清单

- 现状：`DEFAULT_CONDITIONS` 中 `data_reference`、`universe_reference` 为 `...:pending-...`，冻结时不校验。
- 风险：清单无法回答“用的是哪份数据”，违反 §5、§6。
- 要求：冻结时拒绝任何包含 `pending` 的条件值；数据和股票池引用必须指向真实存在的快照 ID（数据快照功能见第四节阶段 1，实现之前至少拒绝占位符）。
- 验收标准：条件中含 `pending` 时冻结失败。

### P0-5 历史价格序列可能混用复权价和未复权价

- 位置：`quant_data/providers/historical_us.py` 约第 159 行：`close = _indexed_number(adj_close, index) or _indexed_number(quote.get("close"), index)`。
- 风险：逐行回退会让同一序列中部分行是复权价、部分是原始价；同时 open/high/low 是未复权价而 close 是复权价，K 线不自洽。拆股前后会产生虚假的巨幅涨跌，直接污染动量、均线和回测收益。
- 要求：一个序列只用一种口径；同时保存 `close_raw` 和 `adj_factor`，按因子复权全部 OHLC；任一行缺失复权价时整段标记质量问题，不逐行回退。
- 验收标准：构造一行缺失 `adjclose` 的测试数据，结果应为质量错误而不是混合序列。

### P0-6 数据层静默吞掉异常

- 位置：`historical_us.py` 中 `list_sectors`、`list_stocks` 的 `except Exception: continue`；Yahoo 失败后静默回退 Stooq。
- 风险：违反 §7.1“不能静默回退”。某只股票拉取失败会悄悄从股票池中消失，结果看起来正常但样本已变。
- 要求：失败必须记录到数据质量报告（标的、原因、数据源）；回退数据源需在结果中标明来源；实验运行中若质量报告有缺失，按冻结规则决定跳过或暂停。
- 验收标准：模拟某个标的请求失败，返回结果中包含该标的的质量记录。

### P1-1 时间语义未定义，存在前视风险

- 现状：条件中有 `signal_cutoff`、`execution_timing` 字段但只是文本。数据层 `as_of` 包含当日收盘 K 线。
- 风险：如果用 T 日收盘数据计算信号又按 T 日收盘价成交，就是前视偏差。
- 要求：在设计中明确默认口径——**T 日收盘后计算信号，T+1 日开盘（或 T+1 收盘）成交**，回测和 Paper 一致。`execution_timing` 改为枚举值（`next_open` / `next_close`），回测引擎据此撮合。
- 验收标准：回测引擎测试中，T 日信号不能以 T 日价格成交。

### P1-2 观察期按自然日计算，没有交易日历

- 现状：`duration_days = (planned_end - planned_start).days + 1`；`trading_calendar` 只是字符串。
- 要求：引入 `exchange_calendars` 或 `pandas_market_calendars`，观察期、调仓日、对账日按 NYSE 交易日计算。
- 验收标准：跨越节假日的区间，交易日数计算正确。

### P1-3 系统时钟不可注入

- 现状：`_experiment_today` 直接调用 `datetime.now()`。
- 风险：测试依赖真实日期，会随时间失效；回测/重放无法复用同一逻辑。
- 要求：`ExperimentService` 接受 `clock` 参数（默认真实时钟），测试使用固定时钟。
- 验收标准：所有涉及日期的测试使用注入时钟。

### P1-4 默认模板与必填校验不一致

- 现状：`_validate_conditions` 要求 `account_mode`、`paper_account_id`、`trading_calendar`、`data_provider`、`signal_cutoff`、`rebalance_frequency`、`execution_timing`、`initial_capital`、`max_positions` 等，但 `DEFAULT_CONDITIONS` 中没有这些键。README 却称“使用默认模板补齐”。
- 要求：二选一——补齐默认值，或修改 README 说明这些字段必须由调用方提供。API 测试覆盖“只提供最少字段创建草稿”的场景。
- 验收标准：README 描述与实际行为一致，并有测试证明。

### P1-5 状态变更原因可以为空

- 现状：未填写原因时写入“未填写原因”。
- 要求：`paused`、恢复 `running`、`invalidated`、`rejected` 必须填写原因（§3 要求记录恢复原因、无效需保留证据）；`invalidated` 还应可关联事故记录 ID。
- 验收标准：上述状态迁移不带原因时返回校验错误。

### P1-6 清单哈希只能防误改，不能防篡改

- 现状：清单和哈希存于同一行，同一进程写入，自校验只能发现 JSON 损坏。
- 要求（轻量做法）：冻结时把清单另存为 `data/manifests/{experiment_id}.json`（内容寻址文件名，或记录到 git 标签），启动时与数据库比对。不必上复杂方案，但要明确这一校验的意义。
- 验收标准：手动修改数据库中的清单后，启动被拒绝。

### P1-7 实验对比结论由人工选择（基于正在编写的代码，Review 002 复核）

- 现状：`create_comparison` 中 `conclusion` 直接取自请求参数，可以在指标变差时选择 `improved`；`compatibility` 计算后不阻止不可比的对比。
- 要求：系统根据预先冻结的评价规则计算结论，人工只能补充说明；股票池、成本、基准、数据口径、时间区间不一致时，结论最多为 `inconclusive`，并在结果中列出不一致项。
- 验收标准：候选指标全面劣于基线时，无法记录 `improved`。

### P2-1 记录试验次数，用于防过拟合

`agent.md` §10 要求保留全部尝试。建议每个策略组统计累计试验次数（包括参数搜索），在对比报告中显示，后续可用于 Deflated Sharpe Ratio 校正。

### P2-2 成本敏感性

回测默认成本 5bps + 滑点 10bps 对大盘股合理；验证报告中应自动给出 2 倍成本下的结果，2 倍成本下失效的策略应标为弱证据。

### P2-3 `StockScreeningService` 的补足逻辑

用未达阈值股票补足列表只能用于展示。建议在返回结果中给每只股票加 `eligible: bool`，执行和回测只消费 `eligible=true` 的集合，并有测试保证。

---

## 四、建议的实现方案（交易员视角）

核心思想：**一份策略代码、一份数据快照、一个撮合口径，同时服务回测和 Paper。**

### 阶段 0：工程基础（1–2 天）

- P0-1（git、依赖锁定）、P1-3（时钟注入）、P1-2（交易日历）。

### 阶段 1：数据快照（`quant_data/snapshots/`）

- 日线 OHLCV 以 Parquet 存储，字段包含 `symbol, date, open, high, low, close, volume, close_raw, adj_factor, source, fetched_at`。
- 快照 ID = 内容哈希。快照包含股票池成员列表和数据质量报告（缺失、重复、异常跳空、数据源回退）。
- 实验清单引用快照 ID；同一快照 ID 重复运行结果必须完全一致。

### 阶段 2：统一策略接口（`quant_strategy/`）

```python
class Strategy(Protocol):
    def generate(self, as_of: date, data: SnapshotView, state: PortfolioState,
                 config: StrategyConfig) -> Decision: ...
```

- `SnapshotView` 只暴露 `as_of` 当日及之前的数据（在接口层强制，防止前视）。
- `Decision` 包含：信号快照（全部候选、分数、是否合格、拒绝原因）、目标权重、理由。
- 风控是独立函数：`apply_risk(decision, state, risk_config) -> RiskDecision`。
- 首个策略：板块轮动 + 量价动量，例如：每周最后一个交易日收盘后，选热度前 3 的板块，每个板块取合格股票中分数最高的 2–3 只，等权，单股上限 15%，板块上限 35%，个股跌破 MA20 或回撤 10% 退出，下周一开盘调仓。参数在配置中，冻结后不可改。

### 阶段 3：回测引擎（`quant_backtest/`）

- 按交易日循环：T 日收盘后调用策略 → 风控 → 生成订单意图 → T+1 开盘撮合（含成本、滑点）→ 更新账本。
- 输出与 Paper 相同结构的记录：信号快照、目标组合、风控决定、订单意图、成交、每日持仓与净值。
- 历史验证运行器：按时间切分训练 / 验证 / 留出区间，加 2 倍成本和相邻参数的稳健性检查；结果**由系统自动写入** `validation_results`，带上数据快照 ID、代码提交、配置哈希。这是满足 P0-3 的正确方式。

### 阶段 4：Paper 执行（`quant_execution/`）

- 独立进程运行（例如 `python -m quant_execution.runner --experiment exp_xxx`，由 launchd 或 cron 调度），不依赖 HTTP 请求。
- 先做**干跑模式**：生成订单意图但不提交，连续运行一周确认稳定。
- `client_order_id = sha256(experiment_id, batch_id, symbol, side)` 前 32 位，保证幂等；重试前先查询券商订单。
- 每日收盘后对账：本地账本 vs Alpaca 持仓和现金，不一致时实验自动转 `paused` 并生成事故记录。
- **影子回测**：每天用当天真实数据和同一冻结版本跑一遍回测，计算 Paper 与回测的跟踪误差（D-2 的核心指标）。

### 阶段 5：复盘与归因

- 基于已记录的事实计算：收益、超额、回撤、波动、换手、成本、实施差额、信号后续表现（包括未成交和被风控拒绝的候选）。
- 归因按 §7.4 的四类来源。

### 建议的里程碑判定

完成阶段 0–3 后，应能做到：给定一个实验 ID，用一条命令在任意时间重跑历史验证，得到与数据库记录完全一致的结果哈希。这是整个系统可信的基础，建议作为首个里程碑的验收标准。

---

## 五、下一轮评审（Review 002）

Codex 完成当前验证结果 / 对比功能并提交后，我将：

1. 按提交逐个检查改动是否对应设计条目；
2. 运行全部测试并补充边界测试；
3. 复核 P0-3、P1-7 的实现；
4. 更新需求追踪表。
