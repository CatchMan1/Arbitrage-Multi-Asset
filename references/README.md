# PCA 统计套利参考资料

## 1. Avellaneda–Lee：原始方法

Marco Avellaneda、Jeong-Hyun Lee，*Statistical Arbitrage in the U.S. Equities Market*。发表于 *Quantitative Finance*，2010，10(7)，761–782。

- [作者托管的 2008 年工作论文](https://math.nyu.edu/~avellane/AvellanedaLeeStatArb071108.pdf)
- [本地 PDF：2008 年工作论文](AvellanedaLeeStatArb071108.pdf)

相关部分：第 2 节的 PCA 因子、第 3 节的残差均值回归、第 4.1 节的 s-score 与交易阈值，以及附录的参数估计。论文使用日频数据，给出开仓绝对阈值 1.25、多头退出线 -0.50、空头退出线 +0.75；这些阈值来自历史模拟经验。

当前项目采用“PCA 去因子 → 累计残差 → AR(1) → s-score”的研究思路。窗口 100 日、3 个主成分、半衰期 5～60 日以及中国 ETF 数据是本项目设置。平仓采用 `abs(s) < 0.5` 的对称带，区别于论文按持仓方向设置的退出规则；日频跳跃越过平仓带时，两者行为也可能不同。净值使用固定初始资金加法累计，费用按权重差估算，因此不是逐项复现论文。

## 2. Quant67：中文流程说明

[《统计套利：协整、配对交易、PCA 残差》](https://quant67.com/post/quant/10-statistical-arbitrage/10-statistical-arbitrage.html)

与项目直接相关的是 OU/AR(1) 均值回归、半衰期，以及第五节 PCA 残差套利。可用于理解从多资产共同因子到残差信号的流程；具体参数与论文结论应以原始文献为准。网页中的其他策略和扩展没有全部实现。
