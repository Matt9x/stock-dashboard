# M4 全样本长周期量化大推演报告（扩域 B/C）· 正式版

- 报告日期：2026-09-19｜分支：`week2-m4-fullrun`（基于 `origin/contest-2026` = `16e8b07`）
- 任务：任务一（技术深化）——把 M4 走步评测从 6 信号日冒烟扩展到 2024–2026 完整 644 个交易日全日历周期，研究域扩至 **B（新能源与周期资源组，100 支）** 与 **C（大金融与消费医药组，100 支）**
- 评测器：`scripts/evaluate_pca_nale_integration.py`（`pca_nale_integration_eval_v1`，本轮新增 domain B/C 支持）
- 口径水印：`unified_total_return_basis_v1_declared_caliber`｜冻结复核日：**2025-04-01**（与 A 组 `m4-full-run-20260916` 相同）
- 标签：主 5 交易日、辅 20 交易日；重叠持有期**不做任何年化**（F3，产物有字段名守卫）
- 计费：0 成本对照 + 主结论双边 15.0 bp；区块 bootstrap seed 42、2000 次（10/40 交易日块 + 5/20 敏感性）
- 上游参照：A 组（99 支）`m4-full-run-20260916` 的结论——动态门控未激活、全周期无显著非线性超额

---

## 1. 扩域数据基础（新增）

| 项 | B 组 | C 组 |
|---|---|---|
| 清单 | `data/task_split/student_B_energy_materials_100.csv` | `data/task_split/student_C_finance_consumer_100.csv` |
| 技术因子来源 | CSMAR `TRD_Dalyr` 真实导出（16 字段，2024-01-02~2026-08-28，批 10 支/查询） | 同左（**当日下载额度约 20 次已用尽：80/100 支已缓存，余 20 支 `000423,002007,…` 待额度重置后重跑同一命令断点续拉**） |
| 落地行数 | 64,310 行 × 18 因子（100 支 × 644 日，停牌日缺行如实缺失） | 待续 |
| 并入 master | `enrich_master_panel_bc_technical.py --only student_B`：64,400 行 × 12 技术列，总体覆盖 97.53%（mom_60d/vol_60d 90.70% = 60 日滚动预热，其余 ≥99.70%）；A 组行与标准 9 列、`ret` 口径逐位未动 | `--only student_C`（数据落地后执行，脚本按组幂等） |
| 收益口径 | 前复权 close 日收益率（`config/data_caliber/csmar_master_close_basis.json` 声明，未经本报告改动） | 同左 |
| W-attn 注意力网络 | **不可评**：`data/raw/student_ac_crawled` 实测 0/100 有 B 组语料 ⇒ 评测器 fail-closed（配置即拒绝），报告并列登记 `NO_CORPUS`；B 域只评 `W-ind`、`W-corr` | 可评（C 组 100/100 有语料） |

## 2. 运行矩阵（run_id 隔离，任一目录已存在即拒绝覆盖）

| run_id | 域 | 设计 | 信号日（规划→应用） | 变体数 | IC 行数 |
|---|---|---|---|---:|---:|
| `m4-full-run-B-20260919` | B | 与 A 组同协议（步长 5、start 180、apply 300、门限降级 20） | 93 → 69 | 19 | 2,622 |
| `m4-full-run-B-long126-20260919` | B | **全日历长样本**（步长 1、start 0、apply 300、门限回到 Week1 默认 **126**） | 644 → 344 | 19 | 13,072 |
| `m4-full-run-C-20260919` | C | 同 B 可比口径 | — | — | 待 C 数据落地 |
| `m4-full-run-C-long126-20260919` | C | 同 B 长样本口径 | — | — | 待 C 数据落地 |

长样本口径的意义：门控训练池从 A/B 可比口径的 23 个标签成熟信号日提升到 **216 个（≥126，Week1 默认门限首次真正可达）**，从而首次在"训练样本足够长"的条件下检验门控能否激活。

## 3. 核心结论（B 域，先答任务书的问题）

### 3.1 动态门控在 >126 交易日长样本下**仍然不能激活**（干净的负结果）

| 证据 | `m4-full-run-B-20260919` | `m4-full-run-B-long126-20260919` |
|---|---|---|
| 门控训练池（标签已成熟信号日） | 23 | **216** |
| `gate_min_train_dates` | 20（降级设置） | **126（Week1 默认）** |
| V1/V2/V3 拟合结果（全部网络×版本） | fallback=`calibration_slope_zero`，converged=False | **fallback=`calibration_slope_zero`，converged=False** |
| 门控 α | 恒为 0.40 | 恒为 0.40 |
| 得分与 B0 关系 | **逐位相同**（独立复核 R2 断言） | **逐位相同**（独立复核 R2 断言） |

即：门控训练样本不足**不是**（或至少不是唯一）的激活障碍——把训练池扩大到 216 个信号日（≥ Week1 默认 126）后，三个门控版本在校准阶段依旧得到零斜率、回落 B0。任务书所问"动态门控能否在长样本下成功激活超越 B0 的显著非线性超额"的当前答案是**否**，且这一负结果首次是在门限不被降级的条件下取得的（A 组当时 126 不可达、只能降级到 20）。

### 3.2 无任何 Holm 校正后显著项（并列全变体）

- 可比口径：全部 38 个 (网络, 变体, 视界) 组合 Holm p = 1.0000（最高原始 t = 2.09，W-corr α=0.75 h5）。
- 长样本口径：全部 Holm p ≥ 0.1901；名义 t 最高 2.81（W-corr α=0.75 h5，IC 0.0381），Holm 后 0.1901，不显著。
- **B0（W-corr, α=0.40）**：h5 IC 可比口径 0.0364（t=1.57）/ 长样本 0.0317（t=2.67，Holm 0.2693）；没有任何变体在多重校正后超越 B0 显著。

### 3.3 组合层面（显式计费，逐期、不年化）

| 口径 | 最优每期净收益（15bp 双边） | B0 (W-corr h5) 净收益 | 95% 区间是否跨 0 |
|---|---|---|---|
| 可比（65–68 期） | W-corr α=0.75 h20：0.01463 [−0.00940, 0.03856]；α=0.75 h5：0.01061 [0.00160, 0.02030] | 0.00605 [−0.00122, 0.01375] | B0 跨 0 |
| 长样本（324–339 期） | W-corr α=0.75 h20：0.00935 [−0.00928, 0.02735] | 0.00403 [−0.00129, 0.00971] | 跨 0 |

⚠️ α=0.75 的区间不跨 0 属**事后网格最优**（19 变体中挑选），未做多变体校正，不得作为"显著超额"引用；B0 与全部门控的区间均跨 0。0 成本对照最大每期毛收益 0.01654（可比）/ 0.01085（长样本），均来自 W-corr α=0.75。

### 3.4 并列发现（不得只报赢家）

1. **W-ind 网络对 α 完全不敏感**（IC 逐位等于 α=0）：机理解析——S0 经行业哑变量+对数市值中性化后，各行业均值≈0；W-ind 行内传播等价于 `S = (1−α)·S0 + α·(行业均值) ≈ (1−α)·S0`，而 Pearson/Spearman IC 对正线性缩放不变 ⇒ IC 恒等。这与 A 组"传播项被中性化吸收"的观察一致，此处给出解析解释。
2. **固定高 α 仍系统性优于低 α 与门控**（W-corr：α=0.75 > α=0.60 > α=0.40 > α=0）：说明"60 日相关网络 + 传播"包含信息，但校准型门控学不到它——与 A 组的并列发现同构。
3. 排除日全部为样本尾部**标签未成熟日**（可比口径 h5 1 天 / h20 4 天；长样本 h5 5 天 / h20 20 天，截至 2026-08-28），未排除日有效股数中位 100（最小 98）——非数据缺陷，F10 设计如此。
4. C 组同源结论**尚未产生**（数据待续），本报告不对 C 域做任何预判。

## 4. 独立复核（AGENTS.md「Agent 独立复核」条目，`scripts/review_m4_fullrun.py`）

两个 B 域 run 均通过 R1–R5 全部独立断言（不复用评测器内部计算）：

- **R1** `alpha_0.00` 得分 ≡ 面板 `S0`（max|Δ|=0），且跨网络逐位相同；
- **R2** 6 个 (网络×门控版本) fallback 声明与得分一致：gate ≡ b0 逐位相同；
- **R3** 抽样 (股票, 信号日) 的 `label_5` 由 master close 序列按日历轴独立连乘重算，逐位一致；
- **R4** 抽样 (网络, 变体, 信号日) 的 Pearson/Spearman IC 用 numpy 独立实现重算，一致（<1e-10）；
- **R5** 抽样组合的逐期多空（腿=20、换手计费）独立重建，均值与 `portfolio.csv` 一致（<1e-9）。

**局限（如实记录）**：① W-corr/W-attn 相关网络基于 60 日窗口，横截面 100 支的相关性估计含噪声；② 换手为腿内名单更替近似，未建模冲击成本与涨跌停不可成交；③ B 域无语料 ⇒ 任何 attention 结论缺失而非为零；④ C 组 20 支数据未到，C 域结论待续；⑤ 长样本口径 h20 标签逐日重叠 20 倍，显著性依赖区块 bootstrap（10/40 交易日块 + 5/20 敏感性），非独立样本推断。

## 5. 学术级 Bootstrap 显著性图集（每 run 一套，PNG 300dpi + PDF）

- 图 1 `fig1_ic_forest_h{5,20}`：主变体均值 IC 区块 bootstrap 95% CI 森林图；
- 图 2 `fig2_bootstrap_dist_h{5,20}`：bootstrap 均值抽样分布直方图 + CI 边界（10/40 交易日块）；
- 图 3 `fig3_ic_timeseries_h{5,20}`：逐信号日累计 IC；
- 图 4 `fig4_portfolio_ci_h{5,20}`：多空组合每期净收益 95% CI；
- 图 5 `fig5_holm_heatmap_h{5,20}`：Holm 校正后 p 热图。

目录：`reports/figures/pca_nale_integration/<run_id>/`。图由 `scripts/plot_m4_bootstrap_significance.py` 以**与评测器同算法同种子重放抽样**生成，并对 `bootstrap.csv` 的 point_mean/CI 做逐项一致性断言（重放与表脱节即拒绝出图）；图内无任何年化字样（守卫词表复用评测器）。

## 6. 复现命令

```bash
# B 域（已完成）
python scripts/evaluate_pca_nale_integration.py --run-id m4-full-run-B-20260919 --domain B --networks W-ind,W-corr
python scripts/evaluate_pca_nale_integration.py --run-id m4-full-run-B-long126-20260919 --domain B --networks W-ind,W-corr \
    --signal-step 1 --start-index 0 --first-apply-index 300 --gate-min-train-dates 126
# 独立复核与图
python scripts/review_m4_fullrun.py --run-id <run_id>
python scripts/plot_m4_bootstrap_significance.py --run-id <run_id>
# C 域（待 CSMAR 当日额度重置后；80/100 支已缓存，仅需 2 批下载）
python scripts/build_csmar_daily_panel_factors.py --cohort student_C
python scripts/enrich_master_panel_bc_technical.py --only student_C
# 然后按 B 的两条命令以 --domain C --run-id m4-full-run-C-20260919 / -C-long126-20260919 执行
```

输入哈希、冻结复核、门控拟合明细见各 run 的 `manifest.json`（`input_sha256`、`verification_frozen_at`、`gate_fits`）。

## 7. 纪律声明

- 本报告不含任何年化字段；重叠持有期只报每期统计与 bootstrap 区间。
- 负面结果如实并列：门控未激活（216 训练日条件下的新证据）、无 Holm 显著项、B0 组合区间跨 0。
- 全部结论可由上列命令与 manifest 哈希复现；`reports/tables/pca_nale_integration/<run_id>/` 为唯一权威数字来源，本报告文字仅为转述。
