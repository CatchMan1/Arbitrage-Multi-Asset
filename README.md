# ETF PCA 统计套利

一个按步骤编写的日频实验 Notebook：从 ETF 对数收益中提取 PCA 因子，对累计残差拟合 AR(1)，生成均值回归信号和对冲权重，分别查看训练与测试回测。所有计算直接写在 Notebook 中，不做网格搜索。

## 运行

```bash
conda activate py311_env
python -m pip install -r requirements.txt
jupyter lab etf_pca_arbitrage.ipynb
```

使用 Python 3.11，从项目根目录打开 Notebook，选择 `py311_env` 对应的内核，从上到下运行。参数直接在开头修改。

## 文件

```text
etf_pca_arbitrage.ipynb       实验入口
requirements.txt            计算和 Notebook 运行依赖
data/etf.csv                唯一输入数据
outputs/pca_etfs.csv        本次入选 ETF 名单
outputs/pca_train_*.csv     训练信号、权重、回测明细
outputs/pca_test_*.csv      测试信号、权重、回测明细
references/                方法来源和论文
```

输出由 Notebook 重新生成，修改参数后需要从头运行，避免混用不同参数的结果。

## 数据与划分

`data/etf.csv` 为项目已有数据，共 430,353 行、483 只 ETF、891 个交易日，范围为 2023-02-02 至 2026-09-30。保留六列：`date`、`code`、`name`、`fund_type`、`close_hfq`、`log_ret_1d`。直接使用提供的价格和收益，不重新下载行情。

对数收益的口径为 `log_ret_1d[t] = log(close_hfq[t] / close_hfq[t-1])`。

按 `split_date` 划分，当前为 `2025-08-22`：

- 训练：不晚于分界日，共 623 个交易日。
- 测试：分界日之后，2025-08-25 至 2026-09-30，共 268 个交易日。

只按训练期数据筛选 ETF：训练首日已有的股票型 ETF，训练收益及价格完整且有限、价格为正、收益波动非零。当前入选 440 只；不随机抽样，也不按测试表现筛选。本版本没有额外的停牌、成交量或历史可借券检查。

## 参数与流程

| 参数 | 当前值 | 含义 |
|---|---:|---|
| `window` | 100 | 滚动窗口，单位为交易日 |
| `k` | 3 | 去除的主成分数量 |
| `s_open` | 1.25 | 主动开仓阈值 |
| `s_close` | 0.5 | 对称平仓带阈值 |
| `fee` | 0.0005 | 单边换手费率，即 5bp |

1. 对每个历史窗口的对数收益做标准化，用 SVD 提取前 k 个主成分，并计算重构残差。
2. 按日期累计残差得到 X，拟合 `X_t = a + b * X_(t-1) + e_t`。仅保留 `0 < b < 1` 的均值回归估计。
3. 计算 `m = a/(1-b)`、`sigma_eq = std(e)/sqrt(1-b²)`、`s = (X末值-m)/sigma_eq`，以及半衰期 `-log(2)/log(b)`。
4. 半衰期在 5～60 日时允许主动开仓。`s < -s_open` 做多残差，`s > s_open` 做空残差；`abs(s) < s_close` 或信号失效时退出，否则维持方向。
5. 主动方向减去其在 PCA 因子上的投影，再除以 ETF 波动率，得到对冲权重；非空仓时总绝对权重归一为 1。最终非零权重可能来自对冲，并不表示每只 ETF 都触发了主动信号。
6. 保存 s 和权重 CSV，再读取权重进行回测。

前 `window-1` 日为预热期，不保存到信号和权重 CSV。测试数据从测试首日向前取 `window` 个交易日，保留测试首日前一天的有效信号供 `shift(1)` 使用；只统计测试期收益。训练与测试的信号状态和账户分别从空仓开始。

## 回测口径

```python
weights = signal_weights.shift(1).loc[start:]
asset_returns = np.expm1(returns.loc[weights.index])
turnover = weights.diff().fillna(weights).abs().sum(axis=1)
strategy_ret = (weights * asset_returns).sum(axis=1) - turnover * fee
net_value = 1 + strategy_ret.cumsum()
```

- 昨日权重对应今日收盘到收盘收益。模型输入为对数收益，记账时还原成简单收益。
- 以固定初始资金为基数，加法累计净值，不使用复利。权重的总绝对值为 1 是相对初始资金而言，不保证相对每日净值的敞口始终为 1。
- 换手按目标权重变化估算，包含首次建仓；不额外计算价格变化引起的再平衡换手。交易费用为换手乘单边费率，不计借券费、现金利息或期末强制平仓费。
- 基准为相同 ETF 的每日等权、固定初始资金加法累计曲线，不扣费用，不是外部市场指数；其市场暴露与 PCA 多空组合不同。
- 图中训练和测试各自从净值 1 开始。初始点放在首个收益区间的前一交易日，测试图不包含预热期收益。
- 指标为累计收益、日频年化夏普（252 日、无风险利率为 0）、百分比最大回撤、累计换手和交易费用。

当前沿用理想化的信号日收盘成交假设：模型用了当天收盘数据，却假设能以该收盘价成交，`shift(1)` 不保证这一成交价格在实际交易中可得。卖空为理论模拟，没有模拟券源、保证金、盘口、滑点或冲击成本。ETF 历史分类和存续范围沿用输入数据。本项目用于方法尝试，不是原论文的完整复现。

## 参考资料

方法来源、阈值依据和当前实现差异见 [references/README.md](references/README.md)。
