> **已更新的交付方向（2026-09-09）**：用户已确认使用十维 PCA 动态赋权重构 NALE 的传播比重。以下旧设计仅保留讨论历史，不能作为当前实现依据。请以 [Week1 代码 Agent 任务书](week1-nale-alpha-handoff.md) 为准；Week1 尚未完成。

# PCA 降维下一步实施计划（Week1剩余任务）

**分支**：`contest-2026`  | **日期**：2026-09-09  | **关联规格**：[`spec.md`](D:/R-FinGPTv2（国创版本）/specs/contest-2026/spec.md)

> **状态校正**：Day1（数据准备）和 Day2（PCA 提取）已完成；当前只执行 Day3–Day5，完成后才算 Week1 验收。`all_pcs.csv`、`pc5_raw.csv` 和 `pca_loadings.csv` 作为现有输入保留。

## 目标与范围

Work1 已完成因子清洗和一次 PCA 提取。本周主目标是利用 PCA 主成分构造更具时效性的 Alpha 因子：识别因子衰减速度，建立短/中期预测信号，并用真实未来收益验证 IC 是否改善。滚动拟合、无前视偏差和可复现元数据是保障条件。

- **决策变量**：清洗后的因子集合、PCA 主成分数 `K`、预测持有期、因子半衰期、时效衰减函数、PC 组合权重、是否将增强因子接入策略。
- **约束**：只使用真实行情与因子数据；时间序列按日期切分，不随机打乱；原始数据不可覆盖；保留旧策略作为稳定基准；无真实未来收益时评估必须阻断。
- **数据**：
  - green：`data/week1_pca/factors_for_pca.csv`，258 个交易日、8 个因子；
  - gold：`data/week1_pca/gold/factors_for_pca.csv`，250 个交易日、8 个因子；
  - storage：`data/week1_pca/storage/factors_for_pca.csv`，300 个交易日、20 个因子；
  - 未来收益从对应 `data/raw/**/market_prices.csv` 按持有期计算并与因子日期、股票代码严格对齐。
- **评价指标**：主指标为未来 5 日/20 日截面 IC、Rank IC、ICIR 和 IC 衰减曲线；辅助指标为命中率、换手率、年化 Sharpe、最大回撤。所有策略指标必须与当前稳定基准和 raw-factor 消融组并列。
- **交付物**：PCA 管线代码、模型元数据、滚动 PCA 产物、样本外验证报告、消融对比表、图表、测试凭证和决策记录。

## 当前事实与验收基线

已核验的 Work1 产物：green、gold 均含零方差 `rf`；storage 含 5 个零方差 `alpha_chokepoint_moat_*` 且最大相关为 1.0。标准化全样本 PCA 的 PC5 累计解释率分别约为 green 79.98%、gold 93.39%、storage 95.37%。因此 PC5 不预设为最优因子，green 的“达到 80%”至少需要 PC6。

本周通过条件：

1. 训练窗口内完成零方差/重复列处理，并保存被删除列及原因；
2. 每个验证日期只使用当日以前训练窗口拟合 scaler 与 PCA；
3. 输出 `explained_variance_ratio`、累计解释率、载荷、训练窗口、随机种子和版本信息；
4. 对每个候选 PC 或 PC 组合估计 5/20 日 IC 衰减曲线，并选择验证集上的时效参数；
5. 与 raw-factor 基线比较，至少实现 IC、Rank IC 或 ICIR 的预设实质提升，同时不能显著恶化换手和回撤；
6. 只有当全量回测相对稳定基准出现预先定义的实质提升，才允许进入策略集成；否则保留基准并记录 negative result。

## 实施阶段

### Day 1：数据口径冻结与质量审计

- 选定首个主验证板块（建议 green，日期最长且因子定义简单），gold/storage 做复核集，不混合不同板块的因子含义。
- 检查日期单调性、重复日期、股票代码、缺失/无穷值、常量列、重复列和价格可用性。
- 将全样本 `bfill` 结果标为历史产物；样本外管线改为训练窗口内的前向填充，窗口起点仍缺失则剔除该列/日期并记录。
- 产出 `reports/tables/pca_data_quality.csv` 和数据字典。

**门槛**：没有真实价格或股票代码无法对齐时，停止收益评估，先修数据契约，不使用模拟收益替代。

### Day 2：滚动 PCA 与时效特征管线

- 在 `src/pricing/factor_orthogonalization.py` 之上新增可复用的滚动拟合封装（建议 `src/pricing/rolling_pca.py`），接口接收训练窗口、验证区间、`K`/解释率阈值和缺失处理策略。
- 训练窗口只拟合 `StandardScaler` 与 PCA；验证窗口只调用 `transform`。
- 默认候选：`K ∈ {1, 2, 3, 4, 5, 6}`，以及累计解释率 `80%/90%/95%`；当有效因子数小于 K 时返回明确错误，不把最后一个 PC 重命名为 PC5。
- 对每个 PC 生成时效信号：`alpha_t = pc_t × exp(-ln(2) × age / half_life)`，并比较半衰期 3、5、10、20 个交易日；动量偏离项只作为候选增强项，必须通过验证集选择。
- 处理 PCA 符号不定：以训练首个窗口的载荷为参考，对后续窗口按载荷内积执行符号对齐，并记录对齐状态。
- 保存每个窗口的 scaler 均值/尺度、载荷、特征值、解释率、输入列和版本元数据。

### Day 3：因子衰减与时效参数选择

- 以真实未来收益计算各 PC 在预测日后 1、3、5、10、20 日的 IC/Rank IC，绘制因子衰减曲线。
- 比较半衰期、动量项和 PC 组合权重；参数只在训练/验证区间选择，测试区间保持冻结。
- 比较相邻滚动窗口的载荷余弦相似度、主成分子空间相似度和符号翻转次数。
- 输出三板块的解释率曲线、载荷热力图、PC 得分分布和窗口稳定性表。
- 对高相关/重复因子做敏感性分析：保留全部有效列 vs 删除重复列，报告结果差异。
- 对 PC1…PCk 做载荷解释；不得仅依据编号宣称“情绪”“机构博弈”等金融含义。

### Day 4：真实收益样本外验证与消融

- 从真实 `market_prices.csv` 计算 `forward_return_{5d,20d}`，使用发布日期/交易日对齐，避免把未来价格带入特征窗口。
- 按时间滚动计算截面 IC、Rank IC、ICIR、命中率、覆盖率和有效样本数，并给出 bootstrap 或 Newey-West 置信区间（按数据长度选择）。
- 至少比较：稳定基准、raw factors、PC1、PC2、PC3、PC4、PC5、累计解释率 K、时效增强 PC 组合；记录各预测窗口的 IC、ICIR、交易成本、换手、Sharpe、最大回撤。
- 只在验证集上选择 K 和是否启用 PC5；测试区间只做一次最终确认。

### Day 5：决策与回测接入评审

- 生成 `reports/tables/pca_oos_ablation.csv`、`reports/tables/pca_stability.csv`、`reports/pca_validation_report.md`。
- 独立复核一个小样本：手算标准化、协方差特征值排序、PC 正交性和未来收益日期偏移。
- 以稳定基准为保护线：若没有预先定义的实质提升，PCA 仅作为研究产物，不替换线上得分；若通过，提交集成变更前的 `begin-unit` 验收标准。

## 质量门禁与执行顺序

这是文档与实验计划阶段，不修改源码，故不运行 `begin-unit`。一旦进入实现：先完整阅读 `bug合集`，执行 `tools/run_quality.ps1 begin-unit`，再按 `small → medium → heavy` 顺序运行；任何失败由质量门禁记录，不能手改状态。

## 项目结构决策

采用现有 Python 单体结构：

```text
src/pricing/factor_orthogonalization.py  # 现有静态 PCA 基础接口
src/pricing/rolling_pca.py               # 本周新增滚动拟合封装
scripts/day2_pca_extraction.py           # 保留为 Work1 复现入口
scripts/run_rolling_pca_validation.py    # 本周实验入口
data/week1_pca/                          # 不覆盖的 Work1 产物
data/processed/pca_week1/                # 滚动 PCA 生成物
reports/figures/pca/                     # 图表
reports/tables/                          # 指标与质量表
tests/                                   # 单元、契约、集成与 smoke test
```

## 复杂度跟踪

| 项目 | 原因 | 更简单方案为何不足 |
|---|---|---|
| 滚动 PCA + 符号对齐 | 防止全样本拟合前视偏差并保证时间可比 | 单次全样本 PCA 无法支持样本外结论 |
| 多 K 消融 | PC5 不是理论上必然最优 | 只测 PC5 无法区分编号效应与真实增量 |




