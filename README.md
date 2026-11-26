# ValueTrace

跟踪国证成长 100 指数的成分股，采用[统一方法论](FinAnalysis.md)。

成分股、行业分类及优先研究方向见[ConstituentStock.md](ConstituentStock.md)。

标的统一存放在 `stocks/<code>.<exchange>/`，目录名采用六位证券代码和交易所后缀，例如 `stocks/603268.SH/`。

- `SH`: 上交所
- `SZ`: 深交所
- `BJ`: 北交所

公司更名或调出指数时保留目录及历史分析。

财报原文存放在标的目录下的 `report/`，按报告期存放。

> 只能在官方平台，如「巨潮资讯网」获得财报、投资者活动记录等资料。
>
> 若分析时需要用副本、补查资料及公告、调研记录等辅助资料，仅临时查阅，不入库。

分析按 `stocks/<code>.<exchange>/analyses/26Q3/` 归档，包含 `report.md`、`input.json` 和 `dcf.json`。

`examples/` 存放虚构标的的示例报告，不进入跟踪范围。

估值与价格口径：

- 价值基准日取最近一期资产负债表日，三阶段 DCF 与内在价值均以此为准。
- 行情基准日取财报发布日，即首个反映该财报信息的交易日不复权收盘价。该价格只用于 WACC 市值权重和结论页的一行价格对照。
- 报告不跟踪实时股价。价格对照标注行情日期，只说明当日价格与三情景内在价值的相对位置。

估值计算见 [scripts](scripts/README.md)。

## 依赖

- Python 3.11+，仅使用标准库。

在季度目录中运行：

```bash
python3 -B ../../../../scripts/dcf.py input.json --output dcf.json
```
