# 五公式真实行情与历史标题代理回测

**范围：受限回顾性研究；不是生产供应链NALE、不是300股完整验收。**

统一口径：98股，644个原始交易日；信号测试2025-02-05至2026-08-20，净值结算至2026-08-28。

V1冻结十维系数；V2月首滚动；V3训练时间衰减；V4叠加市场状态交互；V5叠加成熟历史网络可靠性交互。

|版本|Rank IC|ICIR|净Sharpe|累计净收益|最大回撤|
|---|---:|---:|---:|---:|---:|
|B0|-0.033185|-0.1094|0.8854|47.81%|37.46%|
|V1|-0.033185|-0.1094|0.8854|47.81%|37.46%|
|V2|-0.033189|-0.1095|0.8854|47.81%|37.46%|
|V3|-0.033186|-0.1094|0.8854|47.81%|37.46%|
|V4|-0.033185|-0.1094|0.8858|47.85%|37.46%|
|V5|-0.033169|-0.1094|0.8848|47.76%|37.46%|

逐级配对差异及日期区块bootstrap/Holm见paired_differences.csv；主区块10日，另报5/20日。

V1–V5全数披露，负结果允许；优化c=0或不可识别时按规范回退B0。参数选择仅使用被purge的验证期，测试前存档；测试按冻结月首日程更新。

## 结果解释与独立复核

五公式均已实际执行。未观察到显著或实质提升：主10日区块下五项逐级比较的Holm调整p值均为1，5/20日敏感性也未显著。全部版本Rank IC约-0.033，不能因组合累计净收益约47.8%就认定预测有效。

V1初始c=0，不可识别，378日全部回退；所有版本公共校准有165日c=0。V2另有2026-03-02拟合返回ABNORMAL（梯度范数2.15e-10），按预定失败规则回退22日，未在看到测试成绩后修改求解容差。V3–V5虽有非零动态参数，排序及交易影响很小。方向准确率50.74%，低于测试标签多数方向基准56.45%。

独立复核222264条预测的S/y_hat恒等式、alpha边界、77次拟合时间，手算3个真实五日超额标签，逐日重算六版本Spearman IC/ICIR，从账本独立重算净值、Sharpe及回撤，全部一致，见independent_validation.json。该复核不消除数据代理和真实成交限制。

## 数据与执行限制

- historical announcement titles and publication dates are accepted as recorded; original revisions and source URLs are not verified
- date-only publications become available the following calendar day; monthly features use only publications before the month starts
- the embedding model was trained later than some historical observations; pretrained-model hindsight remains
- missing return is an untradable stale valuation day; holdings cannot transact; delisting recovery is unknown
- total-return marks use Dretwd, not unadjusted Clsprc
- daily ranking portfolio executes no earlier than next close; costs are a frozen research assumption, limit-order feasibility is unverified
- current A stock membership causes survivorship bias; this is not a point-in-time historical universe
- five variants share identical proxy S0, causal return network and portfolio rules; production S0 and supply-chain network are unavailable
- B1 is not evaluated because historical event and industry-lag evidence is unavailable
- full production 300-stock backtest is blocked; no production promotion regardless of proxy results

B1：不可评估，未用离散度规则冒充事件模型。旧MVP的IC/收益数字有前视，不作有效绩效比较。稳定生产产物未覆盖。

是否可以替换稳定版本：**否**。原生产历史输入和300股统一口径未齐备；本结果仅检验共同代理管线中的五个门控公式。

图表：
- [net_value_and_drawdown.png](../../../figures/nale_alpha_week1/20260910-five-formulas/net_value_and_drawdown.png)
- [paired_rank_ic_intervals.png](../../../figures/nale_alpha_week1/20260910-five-formulas/paired_rank_ic_intervals.png)
