# ValueTrace

跟踪国证成长 100 指数的成分股，采用[统一方法论](FinAnalysis.md)。

季度样本和已分析标的见[成分股档案](constituents/index.md)。

标的统一存放在 `stocks/<code>.<exchange>/`，目录名采用六位证券代码和交易所后缀，例如 `stocks/002396.SZ/`。

- `SH`：上交所
- `SZ`：深交所
- `BJ`：北交所

公司更名或调出指数时保留目录及历史分析。

财报原文直接存放在标的目录下的 `report/`，以报告期命名 PDF，如 `report/26Q3.pdf`、`report/26H1.pdf` 和 `report/25FY.pdf`。

> 财报、公告与投资者活动记录只在官方平台获取，如「巨潮资讯网」。
>
> 以 PDF 入库的仅限财报原文。公告、调研记录等其余材料按需临时查阅，不入库，引用时在附注给出官方链接。

分析按 `stocks/<code>.<exchange>/analyses/26Q3/` 归档，包含 `report.md`、`input.json` 和 `dcf.json`。

## 依赖

- Python 3.11+，仅使用标准库。
- 估值脚本的参数含义与运行方式见 [scripts](scripts/README.md) 文件夹。
