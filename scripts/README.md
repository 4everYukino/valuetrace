# 估值计算脚本

仅使用 Python 3 标准库，无第三方依赖。

## 调用

在季度目录（例如 `stocks/<code>.<exchange>/analyses/26Q3/`）运行：

```bash
python3 -B ../../../../scripts/dcf.py input.json --output dcf.json
```

输入、输出不能指向同一文件。输入错误时非零退出，不覆盖输出。三个情景均可计算，或均填 `not_applicable_reason` 时，返回 `0`。

- 退出码 `0`：输出已写入。顶层 `status` 为 `ok` 表示至少一个情景可计算，为 `not_applicable` 表示三个情景均不适用。
- 退出码 `2`：参数或输入有误，输出文件不被改写，原文件内容保持不变。
- 部分情景不适用时，输出在该情景的位置给出 `status: "not_applicable"` 与 `reason`。

## 输入

| 字段 | 约定 |
| --- | --- |
| `currency` | 固定为 `CNY` |
| `units` | 固定为 `{"money":"亿元","shares":"亿股","price":"元/股","eps":"元/股"}` |
| `valuation_date` | 价值基准日，取最近一期资产负债表日，格式为 `YYYY-MM-DD` |
| `balance_date` | 价值桥接起始日，通常与 `valuation_date` 相同，不能晚于价值基准日 |
| `market_price` | 行情基准日的不复权收盘价，元/股。仅用于 WACC 市值权重与报告中的价格对照 |
| `price_date` | 行情基准日，即首个反映该财报信息的交易日，与 `market_price` 对应 |

比例使用小数。不适用原因填入相应情景的 `not_applicable_reason`。

### 日期与价格口径

- `valuation_date` 与 `balance_date` 同为资产负债表日时，季报后的当年尾段按基准日至年末的剩余期间直接填列，使用 `allocation: "remaining"` 与 `bridge_roll: "explicit"`。`bridge_adjustments` 为空数组表示已确认无调整。
- `price_date` 为首个反映该财报信息的交易日，`market_price` 取该日收盘价。确值时点按公告时点判断：
  - 公告早于当日开盘，取当日收盘价。
  - 盘中或收盘后公告，取次一交易日收盘价。
  - 非交易日公告，取次一交易日收盘价。
  - 停牌时取停牌前最后收盘价，并标注停牌。
  - 不便取得该日收盘价时，可改用价值基准日收盘价，并在报告中注明。
- 脚本不读取 `market_price`，`price_date` 也不参与计算。内在价值由输入的经营假设与 `wacc` 决定，后者是外部给定的参数，脚本不据股价推导。
- WACC 的市值权重由使用者按 `price_date` 的市值算好后写入 `scenarios.*.wacc`，脚本不校验其来源。

## 三阶段 DCF

### 价值桥接

`bridge` 按 `balance_date` 填列：

| 字段 | 含义 |
| --- | --- |
| `cash_equivalents` | 经核实可用于估值的现金。受限资金另行判断 |
| `operating_cash` | 必要经营现金储备。超过现有现金时，差额体现为待补足的资金需求 |
| `nonoperating_financial_assets` | 未计入经营现金流的非经营资产价值 |
| `debt_components` | 融资性债务的名称与价值，不与营运资金重复 |
| `other_claims` | 与合并现金流对应的少数股东权益及其他索取权价值 |
| `shares` | 与权益价值口径一致的估值股数 |

`bridge_adjustments` 每项填 `amount` 和 `reason`。正值增加权益价值。

### 情景与逐期假设

`scenarios` 包含 `bear`、`base`、`bull`，各情景设置 `wacc`、`terminal_growth`、`terminal_roic`、`terminal_margin` 和 `terminal_tax_rate`。

- `base_annual_revenue`：估值所在年度的完整收入估计。
- `incremental_nwc_ratio`：收入增量对应的营运资金需求比例，可为负。逐年给定 `delta_nwc` 时可省略。
- `forecast`：从估值次年开始的连续年度预测，至少一年。

每个预测年度包含：

- `year`：年度。
- `stage`：`growth` 或 `transition`，按此顺序排列，期限自行确定。
- `revenue`、`ebit_margin`、`tax_rate`：收入、EBIT 利润率及经营税率。
- `da`、`capex`：折旧摊销与估值口径资本开支，均为非负金额。
- `delta_nwc`：可直接输入经营营运资金增加额。省略时按收入增量乘 `incremental_nwc_ratio` 计算。

资本开支包含增长所需的收购净支出。租赁与资本化研发须保持口径一致。

### 年中估值与日期衔接

12 月 31 日估值可省略 `stub`。其他日期需提供截至当年 12 月 31 日的尾段，包含与年度预测相同的经营字段及 `delta_nwc`，另加：

| 字段 | 可选值与规则 |
| --- | --- |
| `end` | 估值当年 12 月 31 日 |
| `allocation` | `remaining`：输入金额均为基准日之后的剩余金额。`uniform_days`：输入 `balance_date` 至年末的整段金额，再按实际天数分摊到基准日之后 |
| `bridge_roll` | `explicit`：使用已列明的 `bridge_adjustments`。`uniform_fcff`：用已过去部分的 FCFF 减税后利息估计净金融头寸变化 |
| `after_tax_interest` | `uniform_fcff` 使用，整段税后利息金额 |

- `uniform_fcff` 只与 `uniform_days` 配合，且不能再叠加显式桥接调整。
- 使用 `explicit` 且 `balance_date` 早于价值基准日时，必须提供调整数组。空数组表示已确认无调整。
- 已过去期间只影响价值桥接，不再进入未来经营价值。基准日恰为尾段末日时，不重复计入尾段现金流。

### 计算规则

$$
\mathrm{NOPAT}_t = \mathrm{Revenue}_t\,m_t(1-\tau_t),\qquad
\mathrm{FCFF}_t = \mathrm{NOPAT}_t+\mathrm{DA}_t-\mathrm{CapEx}_t-\Delta\mathrm{NWC}_t
$$

$$
\mathrm{FCFF}_{n+1}=\mathrm{NOPAT}_{n+1}\left(1-\frac{g}{\mathrm{ROIC}_{\infty}}\right),\qquad
\mathrm{TV}_n=\frac{\mathrm{FCFF}_{n+1}}{w-g}
$$

$$
\mathrm{EV}_0=\sum_{t=1}^{n}\frac{\mathrm{FCFF}_t}{(1+w)^{u_t}}
+\frac{\mathrm{TV}_n}{(1+w)^{u_n}},\qquad
P_0=\frac{\mathrm{EV}_0+\mathrm{NetFinancial}_0}{N_0}
$$

- `m` 为 EBIT 利润率，`τ` 为经营税率。要求 `0 <= g < WACC` 且成熟期新增投入回报率 `ROIC > g`。
- 固定 WACC、期末折现，`u_t` 为实际天数除以 `365`。
- 终值使用最后预测年度收入乘 `(1 + g)`，再按成熟期利润率、税率和再投资规则计算。
- 净金融头寸为符合条件的现金减经营储备，加非经营资产，减债务及其他索取权，再加桥接调整。
- 当前模型不支持永续收缩、非正稳定期回报或分阶段折现率。

### 敏感性与输出

`sensitivity` 支持：

- `wacc`、`growth`：按两者的交叉组合计算。
- `capex_multipliers`：测算未来显式期资本开支成本变化，例如 `[0.9, 1, 1.1]`。产能或长期效率变化写入三情景的经营假设，不另立情景。

输出包含三情景现金流、终值、价值桥接、每股价格及敏感性。其中：

- `terminal_share`：终值占经营价值的比例，经营价值为零时为 `null`。
- `full_stub_fcff`：分摊前的尾段 FCFF，无尾段时为 `null`。
- 任一情景填了 `not_applicable_reason` 时，该情景只输出 `status` 与 `reason`。
