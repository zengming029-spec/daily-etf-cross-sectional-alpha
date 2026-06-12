# 日频 ETF 横截面 Alpha 策略与交易成本分析

## 1. 项目简介

本项目使用 Python 构建一个基于日频数据的 ETF 横截面 Alpha 策略，核心目标是研究：通过动量、波动率、回撤和成交额变化等因子，是否能够在 ETF 横截面上识别出未来表现相对更好的资产，并在考虑交易成本和调仓频率后构建具有一定稳健性的量化交易策略。

与简单的 ETF 月度轮动不同，本项目采用 **每日生成 Alpha 信号** 的方式，并比较 **每日调仓、每周调仓、每月调仓** 三种交易频率下的策略表现。项目重点不只是回测收益，而是系统分析信号有效性、交易成本、换手率和策略稳健性之间的关系。

本项目覆盖以下完整量化研究流程：

```text
金融数据获取
数据清洗与对齐
Alpha 因子构造
横截面标准化
因子 IC / Rank IC 检验
分组收益检验
组合构建与动态调仓
交易成本建模
绩效评估
稳健性分析
研究报告输出
```

本项目仅用于学术研究和个人学习，不构成任何投资建议。

---

## 2. 研究问题

本项目主要研究以下问题：

1. 动量、波动率、回撤和成交额变化等日频因子是否对 ETF 未来收益具有解释力？
2. 基于横截面 Alpha 排名构建的 ETF 策略能否相对于基准获得更好的风险调整收益？
3. 每日、每周、每月调仓下，策略收益、风险和换手率有何差异？
4. 交易成本会在多大程度上侵蚀高频调仓策略的收益？
5. 因子 IC、Rank IC 和分组收益检验是否支持该策略的有效性？
6. 策略在不同参数设定下是否具有稳健性，还是只依赖特定参数组合？

---

## 3. 数据说明

本项目选取若干只流动性较好的中国 ETF 作为研究对象，ETF 池覆盖宽基指数、成长风格、行业主题和另类资产等类别。

初始 ETF 池可以包括：

| 资产类别  | ETF 示例     |
| ----- | ---------- |
| 宽基指数  | 沪深 300 ETF |
| 中小盘指数 | 中证 500 ETF |
| 成长风格  | 创业板 ETF    |
| 科技成长  | 科创 50 ETF  |
| 消费行业  | 消费 ETF     |
| 医药行业  | 医药 ETF     |
| 金融行业  | 证券 ETF     |
| 另类资产  | 黄金 ETF     |

原始数据包括日频开盘价、最高价、最低价、收盘价、成交量和成交额等字段。本项目主要使用日收盘价、成交量和成交额构造收益率、动量、波动率、回撤和流动性相关指标。

主要字段如下：

```text
date
symbol
open
high
low
close
volume
amount
daily_return
```

其中：

```text
date = 交易日期
symbol = ETF 代码
close = 收盘价
volume = 成交量
amount = 成交额
daily_return = 日收益率
```

---

## 4. 策略核心思想

本项目的基本逻辑是：

```text
每天根据历史价格、波动率、回撤和成交额变化计算每只 ETF 的 Alpha 得分；
在横截面上对 ETF 进行排序；
买入 Alpha 得分最高的一只或多只 ETF；
根据设定的调仓频率进行组合再平衡；
比较扣除交易成本前后的策略表现。
```

策略不是依赖主观判断，而是完全基于规则、数据和可复现代码。

---

## 5. 收益率计算

日收益率定义为：

```text
ret_t = close_t / close_{t-1} - 1
```

其中：

```text
close_t = 第 t 个交易日收盘价
close_{t-1} = 第 t-1 个交易日收盘价
```

未来收益率用于检验因子预测能力，例如未来 1 日、5 日或 20 日收益率：

```text
future_return_{t,h} = close_{t+h} / close_t - 1
```

其中：

```text
h = 预测窗口，例如 1、5、20 个交易日
```

---

## 6. Alpha 因子构造

本项目初步构造以下日频 Alpha 因子。

### 6.1 动量因子

动量因子衡量 ETF 过去一段时间的累计收益率：

```text
mom_5 = close_t / close_{t-5} - 1
mom_20 = close_t / close_{t-20} - 1
mom_60 = close_t / close_{t-60} - 1
```

直觉是：如果某只 ETF 在过去一段时间内表现较强，它可能在短期内继续保持相对强势。

---

### 6.2 波动率因子

波动率因子使用过去一段时间的日收益率标准差计算：

```text
vol_20 = std(ret_{t-19}, ..., ret_t) × sqrt(252)
```

直觉是：在相同收益水平下，波动率越低的资产风险调整后表现可能更好。

---

### 6.3 回撤因子

过去 20 日最大回撤用于衡量近期下行风险：

```text
drawdown_20 = min(close_t / rolling_max(close, 20) - 1)
```

直觉是：近期回撤较大的 ETF 可能处于趋势恶化或风险释放阶段。

---

### 6.4 成交额变化因子

成交额变化用于近似衡量市场关注度和流动性变化：

```text
amount_change_20 = amount_t / mean(amount_{t-20:t-1}) - 1
```

直觉是：成交额明显放大的 ETF 可能受到更多资金关注，但也可能意味着短期拥挤交易，需要结合其他因子判断。

---

## 7. 横截面标准化

由于不同因子的量纲不同，需要在每日横截面上对因子进行标准化。

对每个交易日，将 ETF 池内所有 ETF 的某个因子做 z-score 标准化：

```text
z_factor_{i,t} = (factor_{i,t} - mean(factor_t)) / std(factor_t)
```

其中：

```text
i = ETF
t = 交易日
factor_t = 当日所有 ETF 的该因子值
```

横截面标准化的目的是让不同因子可以被合成为统一的 Alpha 得分。

---

## 8. 综合 Alpha 得分

本项目使用一个简单的线性综合打分模型：

```text
alpha_score = z(mom_20) + z(mom_60) - z(vol_20) - z(drawdown_20)
```

其中：

```text
z(mom_20) = 过去 20 日动量的横截面标准化值
z(mom_60) = 过去 60 日动量的横截面标准化值
z(vol_20) = 过去 20 日波动率的横截面标准化值
z(drawdown_20) = 过去 20 日最大回撤的横截面标准化值
```

基本含义是：

```text
偏好中期趋势较强、波动率较低、近期回撤较小的 ETF。
```

后续可以进一步测试不同因子权重，例如：

```text
alpha_score = 0.4 × z(mom_20) + 0.4 × z(mom_60) - 0.1 × z(vol_20) - 0.1 × z(drawdown_20)
```

---

## 9. 组合构建规则

在每个调仓日，根据当日可获得的历史信息计算所有 ETF 的 Alpha 得分，并按照得分从高到低排序。

组合构建方式包括：

```text
Top 1：只持有 Alpha 得分最高的 ETF
Top 2：等权持有 Alpha 得分最高的 2 只 ETF
Top 3：等权持有 Alpha 得分最高的 3 只 ETF
```

基准策略设定为：

```text
信号频率：每日生成 Alpha 信号
调仓频率：每日、每周、每月三种频率对比
持仓数量：Top 1 / Top 2 / Top 3 对比
初始净值：1.0
交易成本：0%、0.05%、0.10%、0.20% 对比
```

---

## 10. 调仓频率设定

本项目重点比较三种调仓频率：

| 调仓频率 | 含义                     | 特点                 |
| ---- | ---------------------- | ------------------ |
| 每日调仓 | 每个交易日根据最新 Alpha 得分调整组合 | 信号利用充分，但换手率高，交易成本高 |
| 每周调仓 | 每周固定一个交易日调整组合          | 在信号及时性和交易成本之间折中    |
| 每月调仓 | 每月最后一个交易日调整组合          | 交易成本低，但信号反应较慢      |

该部分的核心目的不是简单追求最高收益，而是分析：

```text
更高的调仓频率是否真的带来更高收益？
更高收益是否被交易成本抵消？
每日调仓的信号是否足够强，能够覆盖更高换手率？
每周调仓是否是更合理的折中方案？
```

---

## 11. 交易成本建模

交易成本通过组合换手率建模。

组合换手率定义为：

```text
turnover_t = sum(abs(weight_t - weight_{t-1}))
```

交易成本定义为：

```text
cost_t = turnover_t × cost_rate
```

扣除交易成本后的策略收益率为：

```text
strategy_return_after_cost = strategy_return_before_cost - cost_t
```

其中：

```text
weight_t = 第 t 期组合权重
cost_rate = 单边交易成本率
```

本项目比较以下交易成本假设：

```text
0%
0.05%
0.10%
0.20%
```

通过比较扣费前和扣费后的净值曲线，可以观察交易成本对策略收益的侵蚀程度。

---

## 12. 因子有效性检验

除了直接回测策略，本项目还会对 Alpha 因子本身进行有效性检验。

### 12.1 IC 检验

IC，即 Information Coefficient，表示因子值与未来收益率之间的相关系数。

```text
IC_t = corr(alpha_score_t, future_return_{t+1})
```

如果 IC 长期为正，说明 Alpha 得分较高的 ETF 在未来更可能取得较高收益。

---

### 12.2 Rank IC 检验

Rank IC 使用排名相关系数，比较因子排名和未来收益排名之间的关系：

```text
Rank_IC_t = corr(rank(alpha_score_t), rank(future_return_{t+1}))
```

Rank IC 对极端值更加稳健，因此在横截面因子分析中常用。

---

### 12.3 分组收益检验

每天按照 Alpha 得分从低到高对 ETF 分组，例如分为三组：

```text
Low Alpha
Middle Alpha
High Alpha
```

然后比较不同组别的未来收益：

```text
High Alpha 组未来收益 - Low Alpha 组未来收益
```

如果 High Alpha 组长期表现优于 Low Alpha 组，说明因子具有一定横截面区分能力。

---

## 13. 策略评价指标

本项目使用以下指标评估策略表现：

| 指标           | 含义              |
| ------------ | --------------- |
| 年化收益率        | 策略平均年度复合收益      |
| 年化波动率        | 策略收益率的年度化波动程度   |
| Sharpe Ratio | 单位风险下的超额收益      |
| 最大回撤         | 净值从历史高点到低点的最大跌幅 |
| Calmar Ratio | 年化收益率与最大回撤的比值   |
| 胜率           | 正收益交易期占比        |
| 换手率          | 组合调仓和交易强度       |
| 交易成本前收益      | 不考虑交易成本时的策略收益   |
| 交易成本后收益      | 扣除交易成本后的策略收益    |
| 超额收益         | 策略相对于基准的收益差     |

其中，Sharpe Ratio 可以定义为：

```text
Sharpe Ratio = (annualized_return - risk_free_rate) / annualized_volatility
```

在本项目的初始版本中，可以将无风险利率近似设为 0。

---

## 14. 基准设定

本项目可以使用两类基准：

### 14.1 单一宽基 ETF 基准

例如使用沪深 300 ETF 作为市场基准。

### 14.2 ETF 池等权基准

将 ETF 池内所有 ETF 等权持有，作为被动配置基准：

```text
benchmark_return_t = mean(ret_{1,t}, ret_{2,t}, ..., ret_{n,t})
```

等权基准可以更直接地反映策略是否真的通过横截面排序创造了额外收益，而不是单纯受益于 ETF 池整体上涨。

---

## 15. 稳健性检验

为了避免策略结论依赖单一参数设定，本项目进行以下稳健性检验。

### 15.1 不同持仓数量

比较以下组合构建方式：

```text
Top 1
Top 2
Top 3
```

### 15.2 不同调仓频率

比较以下调仓频率：

```text
每日调仓
每周调仓
每月调仓
```

### 15.3 不同交易成本

比较以下交易成本假设：

```text
0%
0.05%
0.10%
0.20%
```

### 15.4 不同 Alpha 权重

比较不同综合得分方式：

```text
等权 Alpha
偏动量 Alpha
偏风险控制 Alpha
只使用动量因子
只使用风险调整动量因子
```

该部分的目标是检验策略表现是否稳定，而不是寻找一个事后最优参数。

---

## 16. 计量分析部分

除回测外，本项目还会进行简单的收益率预测回归，用于检验因子与未来收益之间是否存在统计关系。

基准回归模型为：

```text
future_return_{i,t} = β0 + β1 mom_20_{i,t} + β2 mom_60_{i,t} + β3 vol_20_{i,t} + β4 drawdown_20_{i,t} + β5 amount_change_20_{i,t} + ε_{i,t}
```

其中：

| 变量               | 含义           |
| ---------------- | ------------ |
| future_return    | 未来 ETF 收益率   |
| mom_20           | 过去 20 日收益率   |
| mom_60           | 过去 60 日收益率   |
| vol_20           | 过去 20 日年化波动率 |
| drawdown_20      | 过去 20 日最大回撤  |
| amount_change_20 | 过去 20 日成交额变化 |
| ε                | 随机误差项        |

回归方法使用 OLS，并进一步使用稳健标准误作为补充检验。

需要强调的是，该部分只用于检验统计相关性，不直接说明因果关系，也不能单独作为交易决策依据。

---

## 17. 项目结构

```text
daily-etf-cross-sectional-alpha/
│
├── README.md
├── requirements.txt
│
├── data/
│   └── README.md
│
├── notebooks/
│   ├── 01_data_collection.ipynb
│   ├── 02_factor_construction.ipynb
│   ├── 03_ic_and_group_return_analysis.ipynb
│   ├── 04_alpha_strategy_backtest.ipynb
│   └── 05_regression_analysis.ipynb
│
├── src/
│   ├── data_loader.py
│   ├── factors.py
│   ├── preprocessing.py
│   ├── backtest.py
│   ├── performance.py
│   ├── ic_analysis.py
│   └── regression.py
│
├── figures/
│   ├── equity_curve.png
│   ├── drawdown_curve.png
│   ├── turnover_analysis.png
│   ├── transaction_cost_comparison.png
│   ├── ic_series.png
│   ├── rank_ic_series.png
│   └── group_return_analysis.png
│
└── reports/
    └── Daily_ETF_Cross_Sectional_Alpha_Report.pdf
```

---

## 18. 运行方式

### 18.1 克隆项目

```bash
git clone https://github.com/zengming029-spec/daily-etf-cross-sectional-alpha.git
cd daily-etf-cross-sectional-alpha
```

### 18.2 安装依赖

```bash
pip install -r requirements.txt
```

### 18.3 依次运行 Notebook

请按照以下顺序运行：

```text
01_data_collection.ipynb
02_factor_construction.ipynb
03_ic_and_group_return_analysis.ipynb
04_alpha_strategy_backtest.ipynb
05_regression_analysis.ipynb
```

---

## 19. 依赖环境

本项目使用以下 Python 库：

```text
pandas
numpy
matplotlib
statsmodels
akshare
scipy
tqdm
jupyter
```

对应的 `requirements.txt` 可以设置为：

```text
pandas
numpy
matplotlib
statsmodels
akshare
scipy
tqdm
jupyter
```

---

## 20. 初步结果

本部分将在回测和因子检验完成后更新。

计划展示的绩效表如下：

| 策略         | 年化收益率 | 年化波动率 | Sharpe Ratio | 最大回撤 | 年均换手率 | 扣费后收益 |
| ---------- | ----: | ----: | -----------: | ---: | ----: | ----: |
| 每日调仓 Top 1 |   待更新 |   待更新 |          待更新 |  待更新 |   待更新 |   待更新 |
| 每周调仓 Top 1 |   待更新 |   待更新 |          待更新 |  待更新 |   待更新 |   待更新 |
| 每月调仓 Top 1 |   待更新 |   待更新 |          待更新 |  待更新 |   待更新 |   待更新 |
| ETF 等权基准   |   待更新 |   待更新 |          待更新 |  待更新 |   待更新 |   待更新 |

计划展示的因子检验结果如下：

| 指标                          |  结果 |
| --------------------------- | --: |
| IC 均值                       | 待更新 |
| IC 标准差                      | 待更新 |
| ICIR                        | 待更新 |
| Rank IC 均值                  | 待更新 |
| High Alpha - Low Alpha 组合收益 | 待更新 |

---

## 21. 主要结论

本部分将在实证分析完成后更新。

预期讨论内容包括：

1. Alpha 得分是否具有稳定的横截面预测能力。
2. 每日调仓是否能带来更高收益，还是主要提高了换手率和交易成本。
3. 每周调仓是否在收益和成本之间取得更好平衡。
4. 高 Alpha 组是否持续跑赢低 Alpha 组。
5. 交易成本是否会显著削弱策略表现。
6. 策略结果是否依赖特定 ETF 池或特定参数组合。

---

## 22. 项目局限

本项目存在以下局限：

1. ETF 池规模有限，横截面数量不如全 A 股股票池丰富。
2. 回测没有完全考虑市场冲击成本、买卖价差和流动性约束。
3. 策略主要基于价格和成交额数据，没有加入宏观变量、资金流或基本面变量。
4. 不同 ETF 上市时间不同，可能导致样本区间不完全一致。
5. IC 和回归结果只能说明统计相关性，不能直接说明因果关系。
6. 如果过度测试参数，策略可能存在过拟合风险。
7. ETF 本身已经是指数化产品，其 Alpha 空间可能小于个股横截面策略。

---

## 23. 后续改进方向

后续可以从以下方向改进：

1. 扩大 ETF 池，加入更多行业、主题、债券、商品和跨境 ETF。
2. 引入波动率目标策略，控制组合整体风险暴露。
3. 加入止损或最大回撤控制机制。
4. 比较动量、反转、低波动、成交额变化等单因子表现。
5. 使用样本内和样本外划分，进行更严格的外推检验。
6. 加入市场状态识别，例如趋势市、震荡市和下跌市。
7. 使用机器学习模型对多个 Alpha 因子进行非线性组合。
8. 将 ETF 策略与个股横截面 Alpha 策略进行比较。

---

## 24. 技术栈

本项目使用：

```text
Python
pandas
numpy
matplotlib
statsmodels
akshare
Jupyter Notebook
Git / GitHub
```

---

## 25. 免责声明

本项目仅用于学术研究和个人学习，不构成任何形式的投资建议。历史回测结果不代表未来收益。任何基于本项目进行的实际投资决策均需自行承担风险。


