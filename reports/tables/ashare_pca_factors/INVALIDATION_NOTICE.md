# 作废说明：本目录 IC 指标待以统一收益口径重算（不删除原件）

- 日期：2026-09-14
- 范围：`factor_ic_summary_5d.csv`、`quintile_returns_5d.csv`、`ashare_pca_factor_whitepaper_5d.md`
- 状态：**结论作废，原件保留为历史证据**（依 `docs/handoff_pca_to_nale_integration.md` §9-6）

## 作废理由

本目录的前瞻收益由 `scripts/evaluate_ashare_pca_factors.py:145,157-164` 计算：

```
fwd_ret_5d = close.shift(-5) / close - 1     # 对全部 299 支一视同仁
```

而 `data/task_split/csmar_master/csmar_factor_panel_master.csv` 的 `close` 列**跨组不同复权口径**
（独立复核实证，见 `docs/handoff_pca_to_nale_integration.md` F2′）：

| 组 | `close` 实际口径 | 来源链路 | 证据 |
|---|---|---|---|
| student_A（99 支） | **不复权** `Clsprc` | `student_a_delivery/02_脚本/build_csmar_daily_panel_factors.py:167-168`（`ret=Dretwd, close=Clsprc`） | 12 行日收益越过板块涨跌停（最大 −66.9%）；`1.0040→1.0001` 全样本平坦 |
| student_B/C（200 支） | **前复权** `TRD_FwardQuotation.ClosePrice` | `scripts/fetch_student_{b,c}_csmar_data.py:77-81` 及两份 manifest | 0/64,300 行越限（若不复权期望约 12.2 行，P(0)≈5×10⁻⁶）；恒等式比值 `1.0623/1.0795 → 恰好 1.000` |

因此对 A 组 99 支，本目录的 5 日/20 日前瞻收益把**除权除息日读成了真实暴跌**，
并整体丢失分红收益：

- 12 行 −17.6%~−66.9% 的伪崩盘落进前瞻收益窗口；
- A 组 `Dretwd` 与 `close` 日收益的均值背离 1.234×10⁻⁴/日 = **年化 3.11%** 的系统性低估；
- 159 行背离超过 1%。

跨组比较（`alpha_composite_nale` 全池 IC vs 分组 IC）因此**混用了两种收益定义**。

- 可比行数与有效交易日：本目录 `factor_ic_summary_5d.csv` 记录 `n_trading_days = 639`，
  而 `PROJECT_SHARED_MEMORY.md` 称"644 个交易日"，两者不一致（缺行为停牌/退市整行，见交接书 F10）。
- **即便不考虑口径问题，综合因子的显著性也被表述过度**：本目录自身记录的
  `alpha_composite_nale` 区块 bootstrap `p = 0.060`，**在 5% 水平下不显著**
  （年化 ICIR 2.24 属"量级尚可"，但记忆中"表现卓越/工业级可用"的措辞超出该 p 值支持；
  只有 PC5 `alpha_institutional_gaming` 的 p = 0.004 在 1% 水平显著，而它正是受口径污染最重的一列）。
- 其余子因子 p 值：momentum 0.614、retail_divergence 0.474、institutional_value 0.170、northbound_flow 0.196，
  全部不显著，却在下一次集成实验中被当作"已验证有效"的合成权重依据（`+0.15/−0.15/+0.35/+0.20/−0.35`）。

## 具体影响面

本目录全部指标（Mean Rank IC、年化 ICIR、胜率、Block Bootstrap p 值、5 分组单调性与 Q5−Q1 超额）
均建立在上述混口径标签上，**不得作为实证引用**，包括：

- `alpha_composite_nale` Mean Rank IC +0.0144 / 年化 ICIR 2.24 / 胜率 54.93% / Q5−Q1 +0.129%
- `alpha_institutional_gaming`(PC5) Mean Rank IC −0.0225 / 年化 ICIR −3.52 / p = 0.004 / 单调性 −0.90

`PROJECT_SHARED_MEMORY.md` 2026-09-14 顶部条目引用的上述数字同待重算后更新。

## 重算路径（已就绪）

统一口径与审计已交付并通过质量门禁 `small`：

- 模块：`src/data/return_basis.py`（`declared_caliber` / `verify_caliber` / `compute_return_basis`）
- 声明：`config/data_caliber/csmar_master_close_basis.json`
- 审计：`scripts/audit_return_basis.py` → `reports/tables/pca_nale_integration/m0-baseline-v1/`
- 测试：`tests/test_return_basis.py`（36 项，弱断言扫描 0 命中）

重算要求：把 `evaluate_ashare_pca_factors.py` 的 `close.shift(-h)/close - 1` 换成
`return_basis` 的 `basis_return`（A 组用申报 `Dretwd`、B/C 用前复权日收益）按日累乘，
并且 PCA / 中性化市值必须改为**训练期定标 + 当日截面**，不得沿用全样本 fit（F3/F7）。

## 复核方法与局限

- 全部数字由 `scratch/probe_*.py`、`scratch/verify_*.py` 只读探针在本仓库根树实测，未联网取数。
- 局限：B/C 的"前复权"判定依赖来源表名 + 两条可计算判据，**尚未获得 CSMAR 字段字典级确认**
  （原始导出不在库内）；`inconclusive` 的 45 支 B/C 股票窗口内未发生可观察公司行为，
  其口径属证据不足而非证据支持。
