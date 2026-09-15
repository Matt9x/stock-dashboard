# 交接书：768D/PCA 因子流水线 → NALE 传播层集成（M1.5）

- 起草：2026-09-14，Codex 侧代码 Agent
- 触发：用户指令「`docs/handoff_pca_to_nale_integration.md` 开工」。该文件此前不存在（全 `D:\` 搜索、git 全分支与历史均无），经用户确认由本 Agent 依现状自拟后再动源码。
- 定位：本文件是 **M1.4（PCA 复合因子周度多空回测，2026-09-14 交付）与 Week1（十维 PCA 动态赋权重构 NALE）之间的集成与数据前置层**。
  它**不取代** [`specs/contest-2026/week1-nale-alpha-handoff.md`](../specs/contest-2026/week1-nale-alpha-handoff.md) 的公式、五版本定义与三层验收，也不取代 [`week1-recovery-plan.md`](../specs/contest-2026/week1-recovery-plan.md) 的 R1–R7 阻断判定；它规定"在真实数据上把 PCA 因子喂进 NALE"所需先做什么、禁止什么、怎么算完成。
- 阅读前置：`PROJECT_SHARED_MEMORY.md` 顶部 2026-09-14 / 09-12 / 09-10 三条更正、`WORKFLOW.md`、`bug合集/INDEX.md`。

---

## 0. 一句话目标

把 768 维语义嵌入按 **as-of 语料**降成的 **PC01~PC10** 合成逐日 S0（裁决 D3：统一 10 维，不再用 M1.4 的 5 维固定权重合成），喂进 NALE 传播层 `S=S0+α(W_norm·S0−S0)`，在**声明式统一总收益口径**（裁决 D1 修订版，M0 已交付，见 F2″）、**点时可用的网络 W**（裁决 D2：W-text 与 W-ind 双主）、**只使用已成熟标签**的前提下，检验"沿网络传播是否带来相对 α=0 不传播的增量"，并给出**可发布或不发布**的结论（负结果算完成）。

## 1. 独立核验事实（本轮全部由本 Agent 亲自重算，非转述）

> 编号说明：初稿的 F2/F3/F4 已被 F2′/F2″/F3 取代（F4 关于 `market_value` 的判断并入 F2′ 第 2 条与
> F2″ 的"踩过的坑 (a)"），F11 顺延为 F14。引用时以本文件现状为准。

复现脚本（全部只读，不落数据产物）：
`scratch/probe_pca_nale_state.py`、`scratch/probe_ret_caliber.py`、`scratch/probe_ret_divergence.py`、
`scratch/probe_mv_return_caliber.py`、`scratch/probe_bc_contamination.py`、`scratch/probe_pit_corpus_coverage.py`、
`scratch/probe_total_return_sources.py`、`scratch/probe_adjustment_hypothesis.py`、
`scratch/probe_forward_adjustment_identity.py`、`scratch/verify_repair_quality.py`、
`scratch/verify_return_basis_invariants.py`、`scratch/debug_contradicted.py`。

> 诚实记录：前五个探针服务于初稿 F2 的**错误假设**（以为 B/C 被填入伪崩盘），
> 它们同时也在 `debug_order_invariance.py` 处暴露了我自己脚本的一个 bug（把 `cohort` 列并入了待比较的帧）。
> 结论以 `probe_adjustment_hypothesis.py`、`probe_forward_adjustment_identity.py`、
> `verify_repair_quality.py`、`debug_contradicted.py` 四项复核为准。

### F1 两条流水线"同名不同物"

- `alpha_composite_nale` 在 `src/pricing/factor_neutralization.py:302` 只是一个**固定权重线性组合**的名字：PC1~PC5 经清洗后按 `+0.15 / −0.15 / +0.35 / +0.20 / −0.35`（`:241-247`）加权再 Z-score。**没有任何图传播**。
- NALE 传播是另一件事：`src/analysis/scoringv3.py:116-158` 的 `S_NALE=(1−α)S0+α(W_norm S0)`，α 固定 `0.4`（`:54`）；`src/graph/nale_alpha_adapter.py:124` 的 `S0+αD`。
- 结论：M1.4 的名字里带 `nale`，但 NALE 从未参与。**集成工作量为真增量，不是重命名。**

### F2′ 收益口径真相（本条**推翻并取代**本文件初稿 F2 的结论）

初稿曾断言"M1.4 用 `close.pct_change()` 填 B/C，于是每个除权日变成 −10%~−67% 的假崩盘"。
**该因果链方向反了**，已由四项独立检验推翻：

1. **越限跳空只出现在 A 组。** 按板块涨跌停（主板 ±10%、创业板/科创板 ±20%，容差 0.5pp）判定：
   A 组 12 行越限（`002594` 2025-07-29 单日 −66.9%、`688012` 2026-05-29 −34.3% 等），
   **B 组 0/64,300、C 组 0/64,300**。若 B/C 与 A 同为不复权，按 A 组发生率期望各约 12.2 行，
   P(观测到 0) ≈ 5.1×10⁻⁶（每组）。
2. **复权恒等式。** CSMAR 约定 `TurnoverRate = Volume/流通股本`、`MarketValue = 成交价 × 流通股本`，
   故比值 `(market_value/close) ÷ (volume/turnover_rate)` 在存真实成交价时应恒 ≈1。
   实测：A 组中位 **1.0040→1.0001（全样本平坦，组内标准差中位 0.0015）**；
   B 组 **1.0623→1.0000**、C 组 **1.0795→1.0000**，81%/88% 的股票首末相对漂移 >2%，
   且末端精确收敛到 1.000 —— 这正是**前复权**的定义特征。高分红样本 `000001` 比值 1.1912→1.0002，
   量级与其股息率吻合。
3. **来源链路已就地核验（B 组）**。A 组 `data/task_split/student_a_delivery/02_脚本/build_csmar_daily_panel_factors.py:167-168`
   写明 `ret = Dretwd`、`close = Clsprc`（不复权）；B/C 组
   `scripts/fetch_student_{b,c}_csmar_data.py:77-81` 与两份 manifest 指向 CSMAR **`TRD_FwardQuotation.ClosePrice`（复权行情表）**。
   并且 B 组的原始导出就在库内：`data/raw/pr_sources/pr4-dd94761/trd_fward_quotation.csv`
   （sha256 `b818d4cf…`，69,400 行，`Symbol` 全为 B 组 100 支），
   与主面板 B 组 `close` 在 **64,400/64,400 行逐行完全相等（max\|Δ\|=0.0）**，
   且 `ClosePrice` 带 3 位小数（复权重算指纹）而 A 组 `Clsprc` 为 1~2 位。
   **C 组原始导出缺失**（`data/task_split/student_c/sources/` 不存在）⇒ B 已升级为"导出级核验"，C 仍属表名+恒等式推断，两者证据等级不同，须分别披露。
4. 因此"前复权"结论此前只被 `recovery-audit-20260910` 撤回为"缺元数据"，
   本轮补上了它当时缺的**可计算判据**（1–3 项），但**仍未获得文档级确认**（原始导出与字段字典不在库内）。

**真正的缺陷在反方向**：`close` 列跨组异质（A 不复权、B/C 前复权），于是凡拿 `close` 直接算收益的代码，
对 A 组会把除权日读成暴跌并丢掉分红：

- `scripts/evaluate_ashare_pca_factors.py:145,157-164` 对**全部 299 支**用
  `close.shift(-5)/close − 1` 作前瞻收益 → A 组 99 支含 12 根 −17.6%~−66.9% 伪崩盘、
  159 行 `|Dretwd − close收益| > 1%`，年化系统性低估 **3.11%**（实测日均背离 1.234×10⁻⁴）。
  → **`reports/tables/ashare_pca_factors/` 的 Rank IC / ICIR / 分层收益（含 `alpha_composite_nale`
  IC +0.0144、年化 ICIR 2.24、PC5 p=0.004）在 A 组上不可信，须以统一口径重做后方可引用。**
- 反过来，`src/pricing/pca_backtest.py:99-104` 的 `eff_ret` = A 用 `Dretwd`、B/C 用前复权日收益，
  **两边都是含息总收益**，其口径混用问题远小于初稿判断。M1.4 汇总数字仍不可用，
  但作废理由是 F3/F7（全样本 fit、静态截面当逐日信号、重叠收益逐年化、`transaction_cost` 形参未生效、
  换手恒为 0），**不是**"B/C 被填入伪崩盘"。

### F2″ 已交付的修复（M0）

`src/data/return_basis.py` + `config/data_caliber/csmar_master_close_basis.json`
+ `scripts/audit_return_basis.py`，产物 `reports/tables/pca_nale_integration/m0-baseline-v1/`：

- **口径以声明为准、由复核背书**（不由收益反推）：`declared_caliber` / `verify_caliber` / `compute_return_basis`。
  实测判定：99 支 unadjusted 全部 `confirmed`、200 支 forward_adjusted 中 155 `confirmed`、45 `inconclusive`、
  **0 `contradicted`**；统一口径覆盖 **191,999/192,199 = 99.8959%**（仅 B/C 各缺首日），
  统一后**越限伪收益行数 = 0**，`max|basis_return| = 0.20035`。
- **为什么必须声明驱动**：实测把 2025-06-30 之后的 `close/market_value` 扰动后重算，
  **历史行的 `basis_return` 会改变** —— 全样本推断本身就是一条前视通道（与 F7 同源）。
  声明式口径先验固定后，逐行只依赖 `(t-1, t)`，时间可回放性已由
  `tests/test_return_basis.py::test_future_perturbation_cannot_change_past_basis` 锁定。
- **踩过的两个坑（留此存照，防回退）**：
  (a) 曾试图"市值背离即修复"，实测 52 行中 40 行原本价收益与基准仅差 ~1e-7，
      替换成市值增速反而凭空造出 5%~16% 新误差 → 修复判据收紧为"仅越限日"，后又整体废弃
      （不复权序列的恒等式比值本就平坦，不含除权信息）；
  (b) 曾用恒等式漂移反驳"不复权"声明，误把 36 支 A 组股票判为 `contradicted` ——
      漂移其实来自**解禁/增发/回购**改变流通股本，不构成反证 → 改为 `inconclusive`，
      并新增真反证（`close` 收益与申报总收益逐行全等 ⇒ 该 close 已复权）。

### F3 M1.4 其余可复算缺陷（与口径无关，仍然成立）

`pca_backtest.py:146-151` 全样本 fit `StandardScaler+PCA`；`:160-164` 取每支**最后**市值做中性化；
`:245` 再平衡日用"行号整除 5"而非交易日历；`:266-276` 成交性过滤在日期缺失时**回退为全集**；
`:279` 同一静态 α 复用到全部 129 个调仓日；`:194,311` 的 `transaction_cost` 形参**从未生效**；
`:350-353` 把**重叠的 5 日收益逐日化**后 ×√252 算年化波动与夏普，与任务书 §8"重叠持有期收益
不能直接当独立日收益年化"冲突。测试侧只断言范围与一致性（夏普通过区间为 `(−2,2)`），
不校验因果性与样本外价值；`tests/` 与 `tools/` 中**没有任何一处引用** `src/pricing/nale_alpha_*`。

### F5 文本语料有时间戳，但覆盖不对称——PIT 重建的硬边界

- `data/raw/student_ac_crawled/*.jsonl`：200 个文件，79,225 条可解析 `publish_time`（77,254 公告 + 1,971 新闻），**A 组 100 支 + C 组 100 支，B 组 0 支**。
- `publish_time` 跨度 2020-01~2026-09，但 2020–2021 仅 165 条、**2022/2023 为 0**，2024 起才成量（2024:27,312 / 2025:29,839 / 2026:21,909）。
- 逐月累计"PIT 可用股票数"：2024-01 仅 1 支，2024-02 起 185 支，2024-03 起 193 支（共 200）。
- 结论：**真实可用的 PIT 文本面板窗口 ≈ 2024-03 → 2026-08，且样本域上限是 A∪C=200 支（扣掉 CSMAR 缺失后 199）**。B 组永久缺原始语料，只能：(a) 联网补抓（有先例脚本），或 (b) 显式排除并披露。不得用今日截面回填冒充。

### F6 网络边证据完全不存在 → NALE 的 W 目前无合法输入

- `data/` 全部文件按 `supply|chain|供应链|relation|edge|graph|network|边|图谱|网络|产业链` 检索：**0 命中**。
- `src/data/nale_alpha_panel.py:22-29` 要求的 5 个时间戳键（`edge_source_published_at` / `edge_available_at` / `edge_evidence_id` 等）覆盖率 **0%**；`src/graph/nale_alpha_network.py:62-63` 拒收 `is_observed=False` 的边（除非 `allow_fixture=True`）。
- 现存唯一"网络"：夹具 5 节点闭环（`scripts/evaluate_nale_alpha.py:145-150`）、60 日收益相关代理（`src/graph/sector_graph_engine.py:65`）、手写 τ/σ/η 先验（`src/graph/temporal_constants.py:204`）。
- **陷阱**：`src/graph/sector_graph_engine.py:209-221` 先播种 `corr=0.5` 再查表，而阈值为 `>=0.40` → **相关性缺失时同板块边无条件成立并伪造 ρ=0.5**。任何复用该模块产出 W 的路径必须先修此缺陷。

### F7 静态截面被当日频信号使用（M1.4 的前视根源）

`factors_768d_all.csv` 是 300×780 **无日期**截面，300 个 `retrieved_at_utc` 全落在 2026-09-07T15:47Z~09-09T02:17Z（约 34 小时）。`pca_backtest.py:146-151` 在**全样本**上 fit `StandardScaler+PCA`，`:160-164` 取每支**最后**市值做中性化，`:279` 把同一 α 复用到全部 129 个调仓日。这是任务书 §6 明令禁止的"把截面复制到 2024–2026 每天"。

### F8 传播层在生产上是空的；NALE 骨架 fail-closed

- `calculate_nale_score` 的**非测试调用方为 0**（`align_gfca_coordinates` 仅 1 处：`unified_pipeline_runner.py:136`）；`calculate_temporal_nale_score` 非测试调用方 0；`SupplyChainGraph()` 实例化为**零边**。→ 传播层可视为绿地，但也意味着"保护既有传播基线"这一说法不成立，B0 必须新跑。
- `scripts/evaluate_nale_alpha.py:242-293`：`blocked_real_data` 退出码 2，随后**唯一可执行路径是生成夹具**；`config/experiments/nale_alpha_week1.json:4-6` 为 `run_enabled: false` / `empirical_status: blocked_unverified_historical_data`。
- 重复实现风险：`propagate_nale` 有两份（`nale_alpha_adapter.py:67` 与 `dynamic_nale_alpha.py:35`，签名不同），有界门控也两份。**接线前必须先定唯一权威模块**，否则"新路径是否真生效"无从证明（任务书 §2）。

### F9 既有对外文档中的两处虚假显著性（集成时不得引用，须另案更正）

- `reports/tables/backtest_paper_2024_2026_300stocks/accuracy_and_performance_report.md:20` 的 `t=3.92 (p<0.01)` 来自 `scripts/build_2024_2026_300stocks_backtest.py:704` 的**硬编码字面量**，且该脚本产出的 694 日价格已被证实是 seed42 随机模拟。
- `reports/static_vs_temporal_nale_comparison.md` §4 宣称"Rank IC 提升均超 +35%、t 突破 3.0"，其数据源 `docs/data/quantitative/temporal_nale_comparison.json` 实际为 10d `0.0481→0.0477`（−0.7%）、t 均 <1.2、`overall_superior: false`。

### F10 停牌/退市缺口与 `shift(-h)` 交易日历错配（M0 后续必修，实测）

- 面板 192,199 行 ≠ 299×644=192,356，缺 **357 行**，全部落在 `student_A`：
  `601989 中国重工` 264 行（末日 2025-08-12，被 `600150 中国船舶` 吸并退市）+ 12 支共 93 个停牌日
  （`600150` 14 天、`002185/002049/688521/688766/688126` 各 10 天、`688012` 9 天、`688981` 6 天…）。
  复现：`scratch/verify_panel_gaps.py`。
- **缺口是"整行不存在"，不是"单元格为 NaN"**。因此 `compute_return_basis` 看不到任何停牌信息，
  下游若按行序偏移取标签就会出错：`scripts/evaluate_ashare_pca_factors.py:157` 的
  `groupby(code).close.shift(-5)` 数的是**可用行**而非交易日 ——
  对停牌 14 天的 `600150`，所谓"5 日前瞻收益"实际横跨约 19 个交易日；
  对 `601989`，其尾部 5 行静默变 NaN 后整支被 `dropna` 吃掉。
- `pca_backtest.py` 走的是 date×code 透视，缺行成为 NaN 后被 `mean()` 跳过，
  后果不同但同样静默：**腿内实际持仓数在停牌日无声缩水**，而测试只断言 `min_leg ≥ 10`。
- 契约要求（写入 §3 并在 M0.2 实现）：所有前瞻标签必须建立在**对齐到面板完整交易日历**的序列上，
  停牌日显式生成 `status='suspended'` 的占位行且 `basis_return=NaN`，
  `h` 日标签严格按交易日历计数；跨停牌/退市的标签必须记拒绝原因，不得静默丢弃整支。

### F12 文本侧溯源对账失败：全池只有 A 组的 768 维向量能与盘上语料对上（实测）

复现：`scratch/verify_cohort_backing.py`、`scratch/verify_provenance_reconciliation.py`、
`scratch/verify_hash_reproducibility.py`（全部只读）。

| 组 | 因子表声明公告 | 盘上实有公告 | 交割比 | `input_sha256` 与抓取 sidecar 一致 |
|---|---:|---:|---:|---|
| student_A | 40,999 | 40,999 | **100.0%** | **100/100** |
| student_B | 195,050 | **0** | **0.0%** | 无 sidecar 可对（`student_ac_crawled` 无 B 组文件，`student_b/sources` 不存在） |
| student_C | 172,968 | 36,255 | **21.0%** | **0/100**（100/100 支的 `announcement_count` 与哈希都不合，比值 1.77×~9.13×，中位 4.85×） |
| 全池 | 409,017 | 77,254 | **18.9%** | — |

- 新闻数同样不合：因子表把 C 组 100 支全部记为 `news_count = 0`，而 sidecar 与 `.jsonl` 实测 C 组有 991 条新闻；
  A 组 980 条完全对平。
- **哪一侧为真已可判定**（直接数 `.jsonl` 条目）：`.meta.json` sidecar 与真实行数 A 100/100、C 100/100 全合；
  因子表与真实行数 **A 100/100 合、C 0/100 合**（C 声明 172,968 公告 vs 实有 36,255；声明新闻 0 vs 实有 991）。
  `因子表 CSV == provenance JSON` 200/200 全等 ⇒ 同源写出。
  故**被高估的是因子表及其 provenance，不是抓取 sidecar**；BUG-0025 的 `000001` 单支失败是整个 C 组
  100 支系统性失配的表现，根因已定位、未修复前不得 `resolve`。复现：`scratch/verify_scanner_and_c_source.py`。
- 哈希配方未记录：`.meta.json` 的 `raw_text` 是 2,746 字符**截断摘要**，我用 8 种候选规范化
  （`raw_text` 直取 / 换行归一 / strip / GBK / `json.dumps(sort_keys)` / `.jsonl` 原行拼接 / 标题拼接 /
  标题+内容拼接 / 逐条排序 JSON）对 12 支抽样**无一能复现** `input_sha256`（含 A 组）。
  ⇒ 即便 A 组"三处一致"也只证明同源写出，不证明哈希覆盖所声明语料 —— **整条链在仓库内不可第三方复算**。

**对本里程碑的后果**

1. `PROJECT_SHARED_MEMORY.md` §二"审计结论：零 Mock、零硬编码、哈希与原始语料 100% 对应"
   与实测冲突（实际对应率 18.9%，C 组哈希 0/100），已按"保留原件 + 追加更正"处理。
2. **M2 的 as-of 文本面板不能声称"复现 768 维因子表的输入"**：以盘上语料重建的
   `available_at` 面板是一个**新的、自洽的特征族**（A 组 41,979 条 / C 组 37,246 条，均带真实 `publish_time`），
   与静态表的 409,017 条声明无包含关系可证。二者须分别命名（建议 `pca_nale_s0_*` vs 历史 `factors_768d_*`）。
3. B 组既无文本又无 `ret`，三重缺失叠加，D1 的"研究域 A∪C=199 支"结论不变且更强。
4. 任何引用 C 组静态向量的分组结论（含 M1.4 的 `student_C` 夏普 +0.2347）在溯源上降级为
   "输入不可核验的工程产物"，不得进入网申/PPT 实证口径。

### F13 管道离群组是 A，不是 B（更正先前两份清点的互斥说法）

`pd.crosstab(cohort_key, feature_source)` 实测：

| cohort | `fastembed_nlp_extracted_strict` | `llm_deepseek_extracted_strict` |
|---|---:|---:|
| student_A | **100** | 0 |
| student_B | 0 | **100** |
| student_C | 0 | **100** |

三份 provenance JSON 与之一致（A `llm_backend = fastembed_nlp_distilled`；B/C `= deepseek-chat`；
三者 `embedding_backend` 同为 `jinaai/jina-embeddings-v2-base-zh`、`fallbacks_allowed = false`）。
所以**同一个 768 维 PCA 基底混合了两种摘要后端**：A 组走本地 NLP 蒸馏，B/C 走 DeepSeek 摘要，
离群组是 **A**。叠加 F12（A 组语料可对平、C 组不合、B 组没有）后，
"按 cohort 分组读 PCA 结果"必须同时带这两个混杂因子，否则会把管道差异误读成板块差异。

### F14 对两份 Agent 清点报告的核对结论（防止错误口径回流）


同一任务的独立清点报告（`.agents` 会话）与本轮实测存在以下差异，**以本轮重算为准**：

| 其结论 | 实测 | 判定 |
|---|---|---|
| "128,800 行收益被替换成**不复权** `close.pct_change()`，是最高风险 8 行代码" | B/C 的 `close` 本身就是前复权（`TRD_FwardQuotation`），0/64,300 越限、恒等式收敛到 1.000 | **错**，见 F2′ |
| "A 组 `ret` 与 `close` 收益中位差 2.4e-7 ⇒ 二者同口径，缺交叉验证" | 308 行差 >0.1%、全为正、最大 67.3pp，集中在除权除息季 | **错**（它据此认为无分红信息） |
| "A 组 `ret` 由 `close.pct_change()` 现算（`build_panel_from_tencent.py`）" | 主面板 A 组来自 `build_csmar_daily_panel_factors.py:167-168` 的真实 `Dretwd` | **错**（引用了另一条交付链路的脚本） |
| "`ret` 逐支全有或全无，0 partial-missing" | 成立；但"缺失 357 单元格"实为 357 **整行**（停牌/退市），见 F10 | 需精确化 |
| "768 维嵌入无点时历史；唯一面板形态是 29×98×768 代理 npz" | 成立（已复核 shapes 与 98 支全在 300 池内） | **对** |
| "`data/` 下网络边证据 0 命中；`calculate_nale_score` 非测试调用方 0" | 成立 | **对** |
| "测试强在内部一致性、完全不管因果与样本外" | 成立（夏普通过区间 `(-2,2)`） | **对** |

第二份清点报告（会话 `97f241e3`，同一任务的另一次独立扫描）另有三条需要定案：

| 其结论 | 实测 | 判定 |
|---|---|---|
| "`scoringv3` 的固定 α=0.4 传播**已接入 5 个生产 runner**，是要保护的稳定基线" | 5 个 runner 只 `import` + 构造 `GFCAScoringEngine(...)`；全仓对 `.calculate_nale_score(` 的调用只有 `tools/compare_temporal_nale.py:184` 与 `tests/test_temporal_nale.py:57`，`.calculate_temporal_nale_score(` 仅测试；`unified_pipeline_runner.py:136` 只调 `align_gfca_coordinates` | **错**：传播层生产调用为 **0**，M1 无"既有传播基线"可保护，B0 必须新跑 |
| "B/C 原始导出在库内：`data/raw/pr_sources/pr4-dd94761/trd_fward_quotation.csv` 与主面板 B `close` 逐行相等" | 复核成立：69,400 行、`Symbol` 全为 B 组 100 支，与主面板 B `close` **64,400/64,400 行 max\|Δ\|=0.0**；但同目录**无 C 组导出**（`student_c/sources/` 不存在） | **对（仅 B）**，已据此把 B 组口径从"推断"升级为"导出级核验"，C 组仍为推断，二者证据等级已分开标注 |
| "唯一面板形态嵌入是 29×98×768 代理，98 支全属 student_A；另有 2,842 条 `{date, code, available_at, document_hashes, text_sha256}` 的 PIT 账本" | 复核成立：npz shapes `(29, 98, 768)`、months 2024-04-01→2026-08-03、98 支 `cohort_key` 全为 `student_A`；`feature_provenance.json` 2,842 条、29 个日期、`available_at` 范围 2024-01-19→2026-08-03 | **对**，且是 M2 的正面资产：该账本形状可直接作为 `pca_nale_s0_signal` 的 PIT 契约模板（但只覆盖 A 组 98 支、且为标题级代理，不得当生产面板） |
| "IC 汇总的 `n_trading_days` 是 639 而非记忆中的 644" | 复核成立（`factor_ic_summary_5d.csv` 六行全部 639） | **对**，已写入作废说明 |
| "`alpha_composite_nale` 的区块 bootstrap p 值其实不显著" | 复核成立：p = **0.060**；其余子因子 p = 0.614/0.474/0.170/0.196 均不显著，只有 PC5 的 0.004 显著 | **对**，"表现卓越/工业级可用"的措辞超出证据支持，已写入作废说明 |
| "BUG 未解决数是 23 条而非 ~13 条" | 与 `bug合集/INDEX.md`（25 行，23 未解决）及 `catalog.json`（`next_id: 26`）一致 | **对** |
| "§8 前置修复项 (1)：67% 的 `ret` 由 unadjusted-vs-adjusted `close` 合成" | 与本报告自身 §4.2 的"B/C close = 前复权"矛盾；实测 B/C 为前复权，见 F2′ | **错**（沿用初稿口径，已在 F2′ 推翻） |

### F15 主面板"特征族按组割裂"：跨组统一因子截面在当前数据下不存在（实测，已登记 BUG-0027）

复现：`scratch/probe_m2_inputs3.py`（只读）。

`data/task_split/csmar_master/csmar_factor_panel_master.csv` 的 299 支 192,199 行里，**特征族按组互斥**：

| 列族 | A 组（99 支 / 63,399 行） | B/C 组（200 支 / 128,800 行） |
|---|---:|---:|
| `mom_5d/20d/60d`、`vol_20d/60d`、`turnover_20d` | 有值（57,558~63,003） | **全 NaN（0）** |
| `amihud`、`price_pos`、`amplitude`、`gap`、`hit_limit`、`abnormal_trd` | 有值（63,300~63,399） | **全 NaN（0）** |
| `pe_ttm`、`pb`、`roe` | **全 NaN（0）** | 有值（112,650 / 117,200 / 128,800） |
| 申报总收益 `ret` | 有值（63,399） | **全 NaN（0）** |

即所谓 "master" 面板是**三份不同交付的纵向拼接、同名不同物**（与 F1 同一性质）：任何"跨 299 支统一"
的因子截面都会得到整列 NaN，而这一点在既有报表里是**静默**的。

同时：A 组申报 100 支，面板只有 **99** 支 —— `600317` **整支 644 行完全缺失**
（`universe_300_assigned.csv` 含该代码且 `cohort_key=student_A`）。此前只记为"299/300 差异"，
本轮完成逐支归因（即 M0.4 的结论）。

后果与处置：as-of 面板工厂已按此 **fail-closed**（技术族只接受 A 组研究域；任一特征列在给定域上全缺
即抛错），故不存在"静默产出全 NaN 假面板"的通道。跨组建模（例如"199 支统一技术+基本面因子"）
在补齐数据前**不可行**；M2 的真实冒烟因此只在 A 组 99 支上进行，并如实标注域边界。


---

### F16 C 组原始输入在仓库与 GitHub 全域均不存在（本轮实测，含联网抓取授权后复核）

**结论：那条 heavy 失败无法靠改代码修复，缺的就是输入本身。** 搜索范围与结果：

| 范围 | 结果 |
|---|---|
| 本地分支（`contest-2026`/`main`/`pr-1`/`pr-2`/`pr-4`/`pr-5-student-c`/`016-…`/`teacher-framework-refactor`） | 仅 **PR4** 含 `data/task_split/student_b/sources/csmar_raw/`（B 组原始 CSMAR 导出，即 B 组口径得以"导出级核验"的来源）；**无任何 C 组 sources** |
| GitHub 远端（用户授权联网后 `ls-remote` + `fetch`） | `refs/pull/*` 共 **9 条**：`pull/1..7/head` + `pull/6/merge` + `pull/7/merge`，**不存在 #8+**；`origin/main` 已同步到 6866e04（2026-09-13，为已物理隔离的开源分支，不含赛事数据） |
| PR5（student C）全树 | 只有 `factors_768d_student_C.csv`、`factors_768d_student_C_provenance.json`、`student_c/csmar/*`（面板+parquet+manifest）、`SUBMISSION_SCOPE.md`、`data/school_factors/README.md`；**无 jsonl、无语料、无 sources/** |
| 全仓可达对象 | **从来没有任何 `*.jsonl` 或 `*crawled*` 路径被提交**（语料一直是本地未入库文件） |
| 不可达对象 / reflog / LFS | 2 个 stash-like WIP 提交（与语料无关）、31 个不可达对象均无命中；**未使用 Git LFS** |

**关键对照（判定"配方对、输入缺"）**：
* PR5 的 C 因子表与主分支 C 因子表**逐值相同**（`input_sha256` 均为 `7e67fb0e…`、公告合计均 172,968、`news_count` 均 0）；
* 二者对盘上 sidecar（`data/raw/student_ac_crawled/*.meta.json`）**0/100 不符**；A 组则 100/100 相符；
* 盘上语料与 sidecar **自洽**（按 PR5 的 `_corpus_hash`：`json.dumps(items, ensure_ascii=False, sort_keys=True, separators=(",",":"))` 重算 == sidecar 记录，故同文件的另一条测试通过）；
* C 组自己的 manifest 指向一个**不存在的目录**：`data/task_split/student_c/sources/csmar_raw/trd_fward_quotation_filling_calendar.csv`。

⇒ **哈希配方正确，缺的是那份约 172,968 条公告的输入语料**（盘上只有 36,255 条，比例 4.77×），且它比 A 组多出的部分不在仓库任何位置。

**要解锁这条阻塞，需要补入（三样任一份都推进）**：
1. `data/task_split/student_c/sources/csmar_raw/trd_fward_quotation.csv`（+ `…_filling_calendar.csv`）——可把 C 组口径由"表名+恒等式推断"升级为 **导出级核验**（与 B 组同等级）；
2. 能对出 `7e67fb0e…` 的 C 组语料（100 支、约 172,968 条公告）；
3. 同一次抓取的 C 组 sidecar（`<code>.meta.json`，其 `input_sha256` 应与因子表一致）。
补入后我会用 PR5 的配方**独立重算**哈希与逐支条数，三者一致才允许 `heavy` 放行；任何不一致一律继续按阻塞处理，**不得改写因子表哈希来"对上"**（那会把"来自更大语料"伪装成"来自盘上语料"，属伪造溯源）。

## 2. 目标与非目标

**目标**

1. ✅（M0 已交付）统一 299 股日频总收益口径，产出带哈希溯源的 `return_basis_audit.md`；
   研究域边界见 D1：收益侧全 299 覆盖，文本侧上限 A∪C=199 支。
2. 构造 **as-of 日频 S0 面板**：文本按 `publish_time < available_at` 聚合 → 训练期定标的 Z-score +
   full SVD PCA（**PC01~PC10 冻结基底**，裁决 D3）→ MAD 去极值 → 行业哑变量 + **当日**对数流通市值
   OLS 残差 → 截面 Z-score；标签严格按交易日历计数并显式处理停牌/退市行（F10）。
3. 构造 **至少两种 PIT 可辩护的网络 W**（见 §5），并声明各自的经济学解释与证据局限。
4. 走步评测传播增益：`α=0`（不传播）vs `α=0.4`（B0）vs α 网格 vs Week1 动态 α（V1/V2/V3 复用 `src/pricing/nale_alpha_models.py`），同一数据、同一分割、同一损失。
5. 三层验收判定（工程完成 / 扩大研究 / 替换稳定版本），未达替换条件即保留稳定基准并登记阻断。

**非目标**

- 不追求"全球首创"表述；不做 V4/V5（需先导证据，属 Week1 R5）；不替换稳定量化产物；不把夹具结果写成实证；不改 `bug合集/`、`.quality-state/`（只能经 CLI）。

## 3. 数据契约（新增，逐条对应 F2′/F5/F6/F7）

`pca_nale_s0_signal` 每行最小必填：
`code(6位字符串)`, `signal_date`, `available_at`, `raw_corpus_version`, `embedding_model_version`, `embedding_truncation_date`, `feature_window_start`, `s0_version`, `pca_version`, `neutralization_version`, `S0`, `price_close`, `available_flags`。

`pca_nale_edge_evidence`（W 的输入账本）：
`edge_id`, `src_code`, `dst_code`, `relation_type`, `source_published_at`, `available_at`, `valid_from`, `valid_to`, `is_observed`, `evidence_id`, `edge_version`。缺 `available_at` 的边**禁止**进入任何真实实验组。

`return_basis`（口径登记表，全池统一）：
`code`, `series_id`, `price_source_id`, `adjustment`(`equal_weighted_total_return` 或显式声明的替代), `dividend_treatment`, `fx`(`CNY`), `calendar_source`, `hash`。**同一次实验内禁止混用两种口径。**

`trading_calendar`（面板完整交易日历，644 日）+ 逐支占位：面板缺行时必须生成
`status ∈ {suspended, delisted, not_listed}` 的占位行（`basis_return=NaN`），
`h` 日前瞻标签一律按该日历计数，**禁止**用 `groupby(code).shift(-h)` 之类"按可用行偏移"
的写法（F10 实测会使 5 日标签实际横跨约 19 个交易日）。

## 4. 分层计划

- **M0 口径修复与数据门禁（无建模）**
  - M0.1 **（已完成）** 全池统一到含息总收益：A 组用申报 `Dretwd`、B/C 用前复权 `close` 日收益率，
    不复权且无申报总收益一律 `unavailable`；禁止 `close.pct_change()` 与 `ret` 无声明混填（实现 `src/data/return_basis.py`）。
    未解锁项：C 组原始导出与 CSMAR 字段字典级确认、面板 357 个**整行**缺口（停牌/退市，全在 A 组）逐支归因。
  - M0.2 基准序列（等权市场基准）独立来源 + 复现说明。
  - M0.3 文本语料版本化：B 组缺语料的处置（补抓或排除）；A/C 语料按 `publish_time` 分月切段并落 SHA256 清单。
    **升级为硬前置（F12）**：先解决"因子表声明 409,017 条 vs 盘上 77,254 条"与 C 组 100/100 哈希失配；
    在解决前，以盘上语料重建的 as-of 面板必须命名为**新特征族**（`pca_nale_s0_*`），
    不得称"复现 768 维因子表输入"；`.meta.json.raw_text` 截断 + 哈希配方未记录 ⇒ 需补 `hash_recipe` 才能第三方复算。
  - M0.4 `600317` 等 299/300 差异逐支调查结论。
  - **出口条件**：口径审计报告 + 缺口清单；任一未达即整段实证标 `BLOCKED_DATA`，不得进入 M2 之后。
- **M1 权威模块收敛**：定 `propagate_nale` 与有界门控的唯一实现与模块归属（建议以 `src/graph/nale_alpha_adapter.py` 为传播权威，`src/pricing/nale_alpha_models.py` 为拟合权威），删除/重导出重复实现；同时修 `sector_graph_engine.py:209-221` 相关性缺失兜底缺陷（缺证据即拒边）。
- **M2 as-of 面板工厂**：`src/data/pca_nale_asof_panel.py`（新增），复用 `factor_neutralization` 的清洗原语但改为**逐信号日截面 + 当日市值 + 训练期定标 PCA**；纯函数、夹具可测。
- **M3 网络工厂**：`src/graph/pca_nale_networks.py`（新增），产出 `W_t` + 覆盖率/孤立点/去边敏感性报告。
- **M4 走步评测**：`scripts/evaluate_pca_nale_integration.py`（新增），只读消费 M2/M3，输出隔离至 `data/processed/pca_nale_integration/<run_id>/`、`reports/tables/pca_nale_integration/<run_id>/`、`reports/figures/pca_nale_integration/<run_id>/`；禁止无 run_id 目录与就地覆盖（补 M1.4 的 `ashare_pca_backtest/` 缺陷）。
- **M5 独立复核与门禁**：手算小样本 + 梯度有限差分 + 时间往返测试（改未来数据不改变过去输出）；`small → medium → heavy`；全量数据层回测与上一稳定版本多维对比，未见质变不覆盖基准。

## 5. 网络 W 的合法来源候选（须并列报告，不得只报赢家）

| 代号 | 构造 | PIT 处理 | 解释 | 已知风险 |
|---|---|---|---|---|
| W-ind | 同 `sub_industry` 共现（`factors_768d_all.csv` 的行业字段，实测 **6 类各 50 支**、100% 非空，可作静态结构证据；本文件早期版本误写"38 类"，已按实测更正） | 静态结构假设（`is_observed=False`）；行业若随时间变动须按 `valid_from/valid_to` 生效 | 同行信息外溢 | 与中性化所用的行业哑变量同源 → 传播项可能被正交化吸收，须做去共线消融 |
| W-corr | 截至 t 的过去 60 日相关 ρ>=阈值（窗口右端**含** t） | 只用 t 及以前已实现收益；缺相关**不得**默认通过（修 F6 陷阱） | 收益联动/共同暴露 | 与"网络=供应链"的经济叙事不符，须改标题口径为"相关网络传播" |
| W-attn（取代不可 as-of 的 W-text） | 截至 t 的过去 60 日日频**文本对数计数**滚动相关（逐条 `publish_time` + 次日可用） | 天然 PIT；随时间演化 | 关注度/信息扩散联动 | 稀疏月份须报覆盖率；零方差节点被拒边（实测 13/99 孤立） |
| W-text（嵌入相似） | **不可 as-of ⇒ `NOT_ASOF`，拒绝产出** | 768 维向量 `retrieved_at_utc` 全晚于 2026-09-07，且本机无嵌入模型缓存 | — | 任何早于该日使用即前视；替代物见 W-attn |
| W-supply | 真实供应链边账本 | 需要 `source_published_at/available_at` | NALE 原意 | **当前不存在**；除非补到可溯源数据，否则只做消融不做主结论 |

主实验默认 **W-attn 与 W-ind 双网络并列**；W-corr 作为第三对照（其结果不得包装成供应链证据）；
W-text（嵌入族）与 W-supply 均标 `NOT_ASOF`/`NOT_EVALUABLE`，必须与主结论并列出现。

## 6. 公式与实现边界（继承任务书 §3–§5，此处只补集成约定）

- S0 = 当日 `alpha_composite_nale`（截面 Z-score 后），N = W_norm S0，D = N − S0，S = S0 + α·D；α ∈ [0.05, 0.75]，u=0 ⇒ α=0.4。
- 十维门控输入 z：由 as-of 面板的 PC01~PC10 标准化得到（注意：M1.4 只用了 PC1~PC5；**PC6~PC10 是否纳入须先冻结**，二者不可同时宣称"十维"与"五成分合成"）。
- 子因子缺失**禁止静默填 0**（修 `factor_neutralization.py:267-273`），缺失即整行标不可用并计数。
- 合成权重：M1.4 的 `+0.15/−0.15/+0.35/+0.20/−0.35` 来自全样本回归证据 → **属前视**。集成实验中该权重只允许作为"历史固定先验基线"，任何学习版本必须用已成熟标签滚动重估，并并列报告。
- 无邻居行统一置自环（N=S0），同时另报原样生产基线差异。

## 7. 评测与统计

- 日截面 Pearson IC / Spearman Rank IC，≥20 只有效股票/日，不足记缺失；ICIR 默认不年化。
- 主标签 5 交易日、辅 20 日；`t+1` 收盘入场、`t+1+h` 收盘退出为研究代理，冻结口径后不得择优切换；停牌/涨跌停/退市用真实交易状态过滤（`Trdsta`/`LimitStatus` 仅 A 组存在——B/C 缺此字段本身即 M0 缺口，须在缺口清单中列明）。
- 配对区块 bootstrap：seed 42、2000 次、5 日标签区块 10 日、20 日标签区块 40 日，报区块长度敏感性；多版本比较用 Holm。
- 换手与净夏普用同一可执行策略，**显式计费**（基准 0 成本仅作为对照，主结论至少给双边千 1.5 一档）。
- 重叠持有期收益**禁止**直接逐日化年化（修 F3）。
- 报告全部版本与全部网络，含退化与回退原因逐日记录。

## 8. 验收标准（可观察）

**V-工程（必须全绿才算"集成完成"）**

1. `return_basis_audit.md`：全池口径唯一、逐支有来源哈希、混用为 0。
2. as-of 面板工厂在夹具上满足：把任一 `t'>t` 的未来数据整体替换，`≤t` 的 PCA、系数、q、S0、S **逐元素不变**（往返测试）。
3. 每种 W 输出覆盖率、孤立点数、去 10%/50% 最强边的符号翻转率与 IC 变化。
4. `propagate_nale` 唯一权威实现，重复实现已消除并有导入断言（`assert` 模块身份）。
5. 新产物全部落 `<run_id>` 隔离目录；重跑不改已发布 manifest；不覆盖稳定基准。
6. 测试：本轮新增测试强断言（BUG-0021 扫描 0 error）；`small` 绿；随后 `medium --feature`、`heavy --version`；`verdict` 归档。

**V-研究（允许负结果，但必须给出结论）**

7. 至少 100 支 × ≥200 个真实信号日、非夹具的走步结果；主 Rank IC 与配对差值 95% CI 齐备。
8. 明确回答："传播相对 α=0 是否有增量"；若 CI 跨 0，结论即"证据不足"，并给出需要多少样本。

**V-发布（三条同时满足才允许替换稳定版本；任一不满足即保留旧版）**

9. 保守条件下主 Rank IC 相对不传播基线的配对差值 95% CI 下界 > 0；
10. 净夏普（含费用）相对稳定正值基线提升 ≥ 10%；
11. 最大回撤绝对改善 ≥ 5pp。

**同时必须新增**

12. `literature_difference.md` 更新：补 NALE 论文题名/版本/公式号（未核验前不得宣称首创），并说明"NALE 命名因子 ≠ NALE 传播"的更正。
13. `requirements_traceability.csv`：官方命题阈值原文件、页码、原句、指标分母、核验状态。

## 9. 禁止事项

1. 禁止用原始 `close` 比率（`pct_change` / `shift(-h)/close-1`）直接当日收益或前瞻标签——A 组 `close` 不复权（F2′）；
   也禁止在**同一实验内**混用未声明的两种收益口径。所有收益必须来自 `src/data/return_basis.py` 的 `basis_return`。
2. 禁止用今日 768D 截面回填 2024–2026 逐日面板后称无前视（F7）。
3. 禁止把 694 日模拟价格、夹具、代理研究写成生产 NALE 实证。
4. 禁止引用 F9 两处 t 值/提升幅度作为证据。
5. 禁止直接编辑 `.quality-state/`、`bug合集/`；改源码前必须 `begin-unit`。
6. 禁止覆盖 M1.4 及更早稳定产物；如需作废，走"新增作废说明 + 保留原件"。
7. 禁止在未修 `sector_graph_engine` 缺失兜底前复用其相关网络输出。

## 10. 用户裁决（2026-09-14，已冻结；正文相应条目按此执行）

- **D1 取数授权 = 不联网，口径改为"声明式统一总收益"（实测后修订，取代初稿的"不复权价收益 + 除权掩码"）。**
  实测证明 B/C 的 `close` 已是前复权价（见 F2′），因此不需要"降级为价收益"，而是把三组统一到
  **含息总收益**：A 用申报 `Dretwd`，B/C 用前复权 `close` 日收益率，`unadjusted 且无申报总收益`
  一律 `unavailable`（不编造）。实现见 F2″；口径声明落在
  `config/data_caliber/csmar_master_close_basis.json`（逐支带 `source_table/adjustment/provenance`），
  由 `verify_caliber` 用涨跌停约束与复权恒等式独立背书。
  - 产物水印：`unified_total_return_basis_v1_declared_caliber`；
  - M1.4 的 `ashare_pca_backtest/` 与 `ashare_pca_factors/` 两份产物按 §9-6 **原件保留**，
    但 `ashare_pca_factors/`（IC 白皮书）因用原始 `close` 算前瞻收益，**结论作废待重算**；
    `ashare_pca_backtest/` 作废理由改为 F3（前视与年化口径），不再是"伪崩盘"；
  - **研究域**：收益侧 299 支全覆盖（99.8959% 行可用）；**文本 as-of 侧仍受 F5 限制，
    PIT 因子面板上限是 A∪C=199 支（B 组无原始语料）**，两个域的可用性必须分别声明，不得混为一谈。
- **D2 主网络 = W-text 与 W-ind 双主并列，W-corr 作对照，W-supply 标 `NOT_EVALUABLE`。** 三者均须先修 `sector_graph_engine.py:209-221` 的"缺相关即通过"兜底缺陷（F6）。
- **D3 PCA 维数 = 统一 10 维。** S0 与门控输入都走 PC01~PC10（训练期定标、full SVD、符号与列序锁定）；**放弃**以 5 维固定权重合成作为 S0。
  直接推论：本实验与 M1.4 的 `alpha_composite_nale` **不可直接比较**，实验内基线重定为"同一 10 维 S0、α=0 不传播"；`+0.15/…/−0.35` 那套全样本权重只作为历史先验基线单列报告（其前视性见 F7/§6）。
- **D4 门禁 = 授权 bootstrap + begin-unit。** 实测源码哈希 `bb7363f3fb…` ≠ 最近 small 凭证 `c8af115b09…`（M1.4 收尾在 09:17 凭证后又新增 `tools/verify_pca_backtest_empirical.py`、`tests/test_challenger_pca_backtest_stress.py` 等），故先 `bootstrap` 建新基线，再 `begin-unit`。

## 11. 执行进度与下一步

**已完成（2026-09-14）**

- ✅ `bootstrap` 建立新质量基线（13/13 `small` 全绿，凭证 `.quality-state/reports/20260914132017-small-77c71c.md`）；
- ✅ `begin-unit` → M0 实现 → `small` 通过（凭证 `…/20260914140121-small-83add8.md`，当前 `small: 有效`）；
- ✅ M0 交付：`src/data/return_basis.py`、`config/data_caliber/csmar_master_close_basis.json`、
  `scripts/audit_return_basis.py`、`tests/test_return_basis.py`（36 项全绿、弱断言扫描 0 命中）；
  审计产物以 `reports/tables/pca_nale_integration/m0-baseline-v2/` 为**当前权威版本**
  （`m0-baseline-v1/` 是 B 组口径证据升级前的历史 run，按不可覆盖原则原样保留；两版统一口径统计量一致：
  覆盖率 99.8959%、年化偏差 3.1107%、越限伪收益 0 行）；run 目录拒绝覆盖已实测；
- ✅ 对 M1.4 IC 白皮书就地放置 `reports/tables/ashare_pca_factors/INVALIDATION_NOTICE.md`（原件保留）；
- ✅ `PROJECT_SHARED_MEMORY.md` 顶部同步（含被推翻的因果判断与作废数字清单）；
  其 §二"哈希与原始语料 100% 对应"一条已按"保留原件 + 追加更正"处理（实测交割率 18.9%）。
- ✅ 经门禁 CLI 登记 **BUG-0026**（`data`/heavy，评分 21/22 = S 级）：F12 的语料交割与哈希失配；
  同时为 **BUG-0025** 定位根因（不是 `000001` 孤例，而是整个 C 组 100/100 系统性失配）——
  两条均**未修复**，禁止 `resolve`。
  登记后 `bug合集` 哈希变化，`small` 凭证按设计失效（`status`：*"Bug 合集与最近一次 small 测试不一致"*），
  因此下一个单元必须重新 `begin-unit`（即重新全量阅读 bug 合集）再跑 `small`；这是门禁本来的意图，不是回归。

**下一步（每项均须独立 `begin-unit → small`）**

1. ~~**M1 权威模块收敛**~~ → **M0.5 门禁扫描器修正（已完成 2026-09-14）**：
   `tools/assert_scanner.py` 两类假阳性已修 —— (a) `_MODULE_ASSERT_METHODS` 补
   `assert_frame_equal / assert_series_equal / assert_index_equal / assert_extension_array_equal / …`
   （pandas 断言此前既不计入分母、又被判 `no-assert`）；(b) 用例收集改为只认模块级函数与 unittest 类方法，
   测试内部的 `def test_*` 辅助函数不再当用例。
   实测：`errors 2 → 0`、`total_asserts 3321 → 3341`、`weak_ratio 0.45% → 0.39%`、`scan passed False → True`；
   回归锁定 `tests/test_assert_scanner_false_positives.py`（9 项，含"真无断言仍须报 error"、
   "类内裸 assertTrue 仍须报 warn"两条**反向能力**用例，防止把扫描器修成假绿灯）；
   门禁自测 `tests/test_quality_system.py` 70 项仍全绿；单元凭证 `…/20260914144221-small-80aa24.md`。
   **未做（刻意）**：不改 5% 阈值、不把 `weak_assert_check` 的 fail-open（`quality_gate.py:2086-2087`）
   改成 fail-closed —— 实测该 fail-open 分支目前**未被触发**（`from tools import assert_scanner` 可导入，
   扫描器真在跑），是否收紧属门禁策略，需另行裁定。
   修后 `verdict` 仍为 `block`，但阻塞项已换成**真实**的两条：
   09-10 遗留的 heavy 失败凭证（`股票全量回归测试` 5 failed）与 `Python 依赖一致性`
   （`magika 0.6.3` 要求 `onnxruntime<=1.20.1`，实装 1.29.0，即 BUG-0024）。
1. ~~**M1 权威模块收敛**~~ → **已完成 2026-09-14（代码单元 `UNIT-20260914-150259-a0f6c1`，凭证
   `.quality-state/reports/20260914151622-small-3975bd.md`；本条文字由文档同步单元
   `UNIT-20260914-151935-b2c677` 落在同一源码哈希上）**：
   传播权威定于 `src/graph/nale_alpha_adapter.propagate_nale_vectorized`（`S = S0 + α·(S0 @ W − S0)`），
   `src/pricing/dynamic_nale_alpha.propagate_nale` 改为**模块级别名再导出**（非 def 包装，故 `is` 同一性成立，
   后人无法悄悄分叉）；字典式 `nale_alpha_adapter.propagate_nale` 内部改为调用同一权威核并加等式自证。
   同时修 `sector_graph_engine.py` 的"缺相关即通过"兜底：删除 `corr = 0.5` 与
   `stock_corr_with_leader = 0.5` 两个静默默认值 —— 缺证据（无矩阵 / 不在矩阵 / NaN）**拒绝该 peer 并计数**，
   龙头相关性无证据时为 `None` 且**不得**推断 `follower_catchup`；新增字段
   `corr_missing_evidence_count` / `leader_corr_with_stock` / `leader_corr_missing` 显式透出证据缺口
   （未知板块兜底分支同步同一 schema）。
   独立复核（不依赖门禁退出码）：`tests/test_nale_propagation_authority.py`（18 项）逐位比对两条入口、
   手算两点图 `S0=[0,1]→S=[0.5,0.5]`、α=0 不动点、孤立行自环、非法输入拒绝路径守恒；
   `tests/test_sector_graph_missing_corr.py`（9 项）覆盖矩阵缺失 / 整行 NaN / 代码不在索引 / 负相关 / 真实高相关正向路径；
   `probe_m1_mutation_check.py` **反向能力自检**：同一夹具下旧兜底逻辑会给出
   `follower_catchup + 1.25% 溢出收益 + 2 个 peer`，修复后为 `divergent + 0.0% + 0 个 peer + missing=2`，
   证明夹具对该缺陷敏感（不是假绿灯）。门禁实测新增后弱断言 0 error（3408 条断言，占比 0.4%）。
   **注意本轮实测确认**：传播层生产调用为 0，故 M1 不需要"保护既有传播基线"，B0 须新跑。
1b. **门禁治理缺口（需用户裁定，属控制文件改动）**：`config/data_caliber/*.json` 不在
   `.quality-gates.json` 的 `source_roots`（`src/tests/tools/docs/start_local.py`）内，
   **故修改口径声明不会使 `small` 凭证失效** —— 而它恰恰是当前全部收益口径的地基。
   建议把 `config/data_caliber/csmar_master_close_basis.json` 加入 `additional_control_files`；
   本轮已在每次审计产物的 `run_manifest.json` 里钉死其 SHA256（v2 记录 `cb37cf817e045880…`）作为替代保障。
   **补充实测（2026-09-14，M1 单元期间意外发现）**：`source_extensions` 只有
   `[.py, .js, .html, .css, .json]`，故 `docs/*.md` 与根目录 `PROJECT_SHARED_MEMORY.md`
   **完全不参与 `source_hash`**（只有 `README.md`/`QUALITY_WORKFLOW.md`/`项目规则.md` 因列入
   `additional_control_files` 才被绑定）。实测：同一 `source_hash`（`dab6ac15…`）下改写本文件正文，
   `small` 凭证不失效。含义：本交接书与全局记忆属于**不受门禁保护的控制性文字**，
   其内容变更需靠人工评审与独立复核，不能依赖凭证绑定。**副作用记录**：本轮为对齐哈希曾把本条改写
   临时回退再重新落笔，属多余动作（哈希本就未变）；如实记录，避免后人误以为文档受哈希保护。
2. ~~**M2 as-of S0 面板工厂**~~ → **已完成 2026-09-14（单元 `UNIT-20260914-154005-978deb`）**：
   新增 `src/data/pca_nale_asof_panel.py`（纯函数、夹具可测）+ `tests/test_pca_nale_asof_panel.py`（45 项全绿）。
   交付内容：交易日历对齐（`status ∈ {trading, suspended, delisted, not_listed}` 占位行）；
   **按日历计数**的 h 日前瞻标签（`label_h` + `label_ok_h` + `label_reason_h`，逐例记拒绝原因）；
   逐信号日截面（训练期定标 `StandardScaler`+full-SVD `PCA(10)` → PC 截面 Z → 等权合成 →
   MAD 去极值 → 行业哑变量 + 当日对数流通市值 OLS 残差 → 截面 Z-score）；PCA 分量符号按
   "最大绝对载荷项为正"对齐（`align_pca_signs`，手算用例锁定）；缺失特征**整行标不可用并计数**，绝不填 0；
   输出满足 §3 契约列（含 `available_flags` / `s0_version` / `pca_version` / `neutralization_version`）；
   `S0_pc5_prior` 仅作"历史固定先验基线"并列（版本号写明 `lookahead_derived`），主口径 `S0` 不使用任何标签。
   独立复核：时间往返测试（改 cutoff 之后的特征/价格/市值/收益，此前所有信号日的 `S0` 与 PC 得分逐位不变）；
   跨停牌 5 日标签**手算乘积**逐位比对 + 反向守卫（证明"按可用行偏移"会给出跨过停牌日的假数字）；
   中性化残差对行业哑变量与对数市值的正交性用 `lstsq` 独立验证；符号对齐用手造载荷手算 +
   引擎内不变量双测；`NOT_ASOF` 族（静态 768 维嵌入）在任何配置下被拒绝。
   真实数据冒烟（`scratch/smoke_m2_real.py`，只读）：A 组 99 支 × 89 个信号日（每 5 日取点）= 8,746 行，
   全部可用；`S0` 均值 0、标准差 0.995、范围 [−3.54, +4.05]；10 个 PC 累计解释方差均值 96.77%；
   `S0` 与 `S0_pc5_prior` 的截面相关系数仅 **0.09**（⇒ D3 判定二者不可互称，必须并列报告）；
   标签可用性 1 日 8743/8746、5 日 8636/8746、10 日 8526/8746、20 日 8306/8746，
   拒绝原因全部可归因（`insufficient_calendar_after` / `suspended_inside_horizon` / `delisted_inside_horizon`）；
   构建耗时 7.6s/89 个信号日（全量 444 日约 38s，工程可接受）。
   **同时登记 BUG-0027**（见 F15）：主面板特征族按组割裂，跨组统一因子截面在当前数据下不可得。

2b. **M2 遗留（下一单元处理）**：
   (a) M4 CLI 必须在真实运行中改用**训练期** `verify_caliber` 结论，不得用全样本复核表（当前冒烟脚本为省事
   传了全样本表，属前视通道，已在脚本注释与本条标明，不得复制进 CLI）；
   (b) 文本族（`text_flow_v1`）目前只有夹具级端到端用例，尚未在真实语料上跑通并报告覆盖率——
   需与 M3 的 W-text 一起做（同一份语料、同一套 as-of 规则）；
   (c) `industry` 来自 `universe_300_assigned.csv` 的静态 `sub_industry`，属"静态结构假设"，
   在 W-ind 处必须显式声明；
3. ~~**M3 网络工厂**~~ → **已完成 2026-09-14（单元 `UNIT-20260914-163520-66302d`，凭证
   `.quality-state/reports/20260914164434-small-89e27f.md`）**：新增 `src/graph/pca_nale_networks.py`
   + `tests/test_pca_nale_networks.py`（38 项全绿）。三类可评网络：
   `industry_cooccurrence`（W-ind，静态假设，`is_observed=False`）、
   `return_correlation`（W-corr，只用截至 t 的已实现收益）、
   `attention_correlation`（W-attn，日频文本对数计数滚动相关，逐条 `publish_time` + 次日可用）；
   两条显式拒绝产出：`text_embedding_similarity`（`NOT_ASOF`，本机无嵌入模型、向量全晚于 2026-09-07）、
   `supply_chain_edges`（`NOT_EVALUABLE`，仓库无源边账本）。归一化与传播**复用 M1 权威实现**
   （为此在 `nale_alpha_adapter` 暴露公开 `normalize_network`）；**缺证据一律拒边并分别计数**
   （重叠不足/零方差/非有限 ρ/低于阈值），绝不回落 0.5（F6 陷阱），并有反向守卫用例断言权重里不出现 0.5。
   产出覆盖率/密度/度分布/孤立点统计、**去边敏感性**（权威传播核 α=0.4，手算三点图逐位校验）、
   满足 §3 契约的边账本（含 `available_at` 不早于 `valid_from` 的校验；空边账本拒绝下结论）。
   独立复核：手算三节点相关网络、阈值**边界含等号**、窗口只取声明区间、时间往返（改未来收益/文本不改变
   此前网络，含 W-attn）、孤立点传播恒等（N=S0）、参数与非法输入 fail-closed、并列报告结构。
   真实数据冒烟（`scratch/smoke_m3_real.py`，只读；域 = A 组 99 支，信号日 2025-04-02，语料 79,225 条）：
   * W-ind：2,401 边（= C(50,2)+C(49,2) 精确吻合）、密度 0.4949、覆盖 1.000、孤立 0、度 48~49；
   * W-corr（60 日、ρ≥0.40、min_overlap 40）：1,520 边、密度 0.3133、覆盖 1.000、度 1/30/74、
     权重 0.4001~0.9547、低于阈值 3,331 对、窗口含 NaN 98 对、重叠不足 0；
   * W-attn（同参数）：387 边、密度 0.0798、覆盖 **0.869（13/99 孤立）**、零方差 98 对、低于阈值 4,366 对；
   * 去边敏感性（移除权重最大的边）：W-corr 5%（76 边）max|ΔS|=0.809、55 节点变化；W-ind 5%（120 边）
     max|ΔS|=0.573；W-attn 5%（19 边）max|ΔS|=0.172。
   **局限**：W-attn 的 13 个孤立点在传播中等价于不传播（N=S0），M4 必须分网络分别报告覆盖度；
   W-ind 与中性化行业哑变量同源，必须做去共线消融后才能下"传播增益"结论；C 组无技术因子，
   故跨组网络与 S0 不能同域，跨域结论一律不下。
   同时更正本文件 §5 的行业分类数（"38 类" → 实测 **6 类各 50 支**）。
4. ~~**M4 走步评测**~~ → **已完成 2026-09-14（单元 `UNIT-20260914-170510-acf02b`）**：
   `scripts/evaluate_pca_nale_integration.py` + `tests/test_evaluate_pca_nale_integration.py`（43 项全绿），
   门禁 `small`（凭证 `20260914180655-small-caa922`）与 `medium --feature "PCA-NALE 集成"`
   （凭证 `20260914180745-medium-593edc`）均通过。
   * **run_id 强制与隔离**：`--run-id` 必填且只允许 `[A-Za-z0-9._-]+`；三个产物目录
     （`data/processed|reports/tables|reports/figures/pca_nale_integration/<run_id>/`）任一已存在即拒绝运行，
     实测同 run_id 二次运行被拒（测试锁定）；无 run_id 目录一个字节都不写。
   * **口径复核点时冻结**：`frozen_verification` 只用 `trade_date <= 冻结日`（= 首个应用信号日的前一交易日）
     的数据算 `verify_caliber`，冻结日写入 manifest；测试用"改未来 close/market_value"证明历史结论逐位不变。
   * **同数据同切分**：测试断言所有变体的 (股票, 信号日) 集合完全一致，且 `alpha_0.00` 与面板 `S0` 逐位相等。
   * **变体（共 28 个）**：α=0 / B0 α=0.40 / α 网格（0.05、0.20、0.60、0.75）/ 动态门控 V1·V2·V3
     （**按网络分别拟合**，训练段只用标签已成熟的信号日，V3 用真实交易日年龄做时间权重）/
     W-ind 节点标签置换安慰剂；三条网络各一套。
   * **统计**：日截面 Pearson/Spearman IC（<20 只有效股记缺失并计数）、ICIR 不年化、配对区块 bootstrap
     （seed 42；区块 10/40 交易日 + 5/20 敏感性，按信号步长换算为信号日块）、Holm 校正；
     **字段名守卫**：任何含 `annual` 的字段在落盘前直接抛错（修 F3）。
   * **组合**：多空各取 `max(10, ceil(0.2×有效股数))`，显式计费（0 成本对照 + 双边 15bp 主档），
     换手按腿内名单更替比例估计并逐期扣费，只报每期均值/标准差/区间——**不做任何年化**。
   * **产物**：`asof_panel.csv.gz`、`variant_scores.csv.gz`、`edge_evidence.csv.gz`（§3 边账本）、
     `metrics_by_variant.csv`、`ic_series.csv`、`bootstrap.csv`、`portfolio.csv`、`network_diagnostics.csv`、
     `variant_index.csv`、`report.md`、`ic_by_variant.png`、`manifest.json`（输入 SHA256、语料清单哈希、
     版本串、冻结日、网络覆盖度、门控拟合摘要）。
   * **真实冒烟**（`m4-smoke-20260914a`；A 组 99 支，计划 93 个信号日、应用 6 个，语料 79,225 条）：
     28 个变体、336 行 IC、0 个被排除的信号日；W-corr 覆盖 1.000、W-attn 平均覆盖 0.911（孤立点最多 15）。
   * ⚠️ **必须如实记录的负面结果**：9 次门控拟合（3 网络 × V1/V2/V3）**全部回落**
     `fallback_reason = calibration_slope_zero`（B0 校准斜率退化为 0），`GateFit.alpha` 因此恒为 0.4 ——
     V1/V2/V3 与 B0 **数值完全相同**（`alpha_min = alpha_max = 0.40`）。即**本样本上动态 α 不可识别**，
     不得宣称任何"动态门控增益"。直接原因之一是训练段只有 23 个信号日
     （`gate_min_train_dates` 已被迫由 Week1 默认 126 降至 20，属**降级设置**，已写入 manifest 与报告）。
   * ⚠️ 冒烟期 6 个信号日的 IC（−0.14~+0.10）**只是管线连通性检查，不构成任何实证结论**，禁止引用。

5. ~~**M5 独立复核与门禁升级**~~ → **已完成 2026-09-14（两个单元）**：
   * **M5 独立复核**（`UNIT-20260914-185638-a2fc0d`）：新增 `tests/test_pca_nale_integration_review.py`
     （12 项全绿，独立重建夹具、不复用其它测试的 fixture）：跨模块一致性（M4 的 `alpha_0.00` ≡ 面板 `S0`
     ≡ α=0 的权威核结果；`b0` ≡ 用 M3 重建同一 W-ind 网络 + 权威核传播，误差 <1e-12 且已注明
     1e-16 级 BLAS 非确定性来源）、CLI 级时间往返（改远处未来 ⇒ 断点前的 as-of 面板/变体得分/IC/网络诊断
     **逐列逐位不变**；并加**反向守卫**证明同一改动确实改变了断点之后的行，避免假绿灯）、
     spy 截获 `compute_return_basis` 收到的 verification 确认窗口止于冻结日（防全样本复核回流）、
     手算两点传播与三点图去边敏感性、门控目标函数**闭式损失**与**中心差分梯度**校验、
     产物隔离拒绝发生在任何写出之前、报告不隐藏孤立点。
   * **M5b 遗留 heavy 失败归因与修复**（`UNIT-20260914-191407-28efb4`）：
     先复现全部 5 项失败并逐条定位根因，再按证据修复，**没有一处是靠放宽断言过关**：
     - `test_validate_teacher_framework` ×2 ⇒ **工具缺陷（未来数据泄漏）**：
       `docs/data/kline/001258.json` 已由 252 根（`LAST_DATE=2026-08-13`）长到 **268 根（2026-09-04）**，
       而 `run_three_actions` / `list_limit_up_events` **未夹断到模块自己声明的数据截止日**，
       于是涨停聚簇 6→7 簇、触发后"20 日窗口"跨到 08-24 之后，收益 +17.28%→−3.46%。
       修复：新增 `clamp_to_last_date` / `report_lookahead_drift`，四个统计函数统一夹断（缺失 `LAST_DATE`
       即 fail-closed），空表边界行为保持 `[]`；**测试断言一字未改**，夹断后自动回到文档化数值。
     - `test_data_adapter` ⇒ **诊断口径错误**：本地目录根本没有数据时却报"缺交易日历"，
       把调用方引向错误方向。修复：仅当本地确有数据、缺日历才要求 `expected_trading_dates`；
       无数据时报准确的覆盖错误。两路径均 fail-closed，并补反向守卫用例（有数据+缺日历仍必须报日历要求）。
     - `test_git_push_with_fallback` ⇒ **断言过窄**：原断言要求每日脚本**字面包含**助手路径，
       但两个脚本接线方式不同（`daily_morning.ps1` 直接调用；`daily_local.ps1` → `tools/daily_routine.py`
       → 第 9/9 步调用同一助手）。改为**传递性校验**（直接调用或经 routine 调用）并保留原始
       `git push origin main` 禁令断言 + 新增反向守卫（删掉 routine 里的助手调用必须失败）。
     - `test_adversarial_m4_data_provenance` ⇒ **不可修复，如实保留为阻断**：期望哈希
       `d0e166b0…`（provenance JSON）与盘上重算 `7e67fb0e…` 不一致，属 BUG-0026（哈希配方未记录）
       与 BUG-0025（C 组 100/100 失配）的同一根因；该测试本身就是完整性检查，**必须继续失败**，
       禁止跳过或放宽。
   * **门禁结果**：修复后 `heavy` 由 **5 failed → 1 failed**（1,383 passed / 157s，凭证
     `.quality-state/reports/20260914193341-heavy-84d005.md`）；`small`（`…192936-small-3f471e`）与
     `medium`（`…193014-medium-bdb96a`）均通过。
   * ⚠️ **门禁机制发现（需用户裁定）**：`heavy` 每次失败都会**自动登记一个新 Bug**
     （本次新增 BUG-0029，字段与 BUG-0028 同类），而 bug 合集哈希是凭证绑定项 ⇒ 每次 heavy 失败都会
     把刚通过的 `small`/`medium` 凭证判为失效，形成"重跑 heavy → 新 Bug → 凭证失效"的循环。
     当前处置：**不再重复触发 heavy**，只把阻塞项登记清楚；是否让 stage 级失败复用同一条记录，
     属门禁策略改动，需另行裁定。
   * **当前发布判定（未达 §8-V 三条）**：`verdict = block`，阻塞项为
     ①`test_adversarial_m4_data_provenance`（BUG-0026/BUG-0025 溯源链）；②`Python 依赖一致性`
     （BUG-0024：`magika 0.6.3` 要求 `onnxruntime<=1.20.1`、实装 1.29.0）；③heavy 无通过凭证。
     **稳定基准保持不变**，未覆盖任何既有量化产物。
6. 若需重算 M1.4 IC：改 `evaluate_ashare_pca_factors.py` 用 `basis_return` 累乘 + 训练期定标 PCA，
   新结果落 `reports/tables/ashare_pca_factors/<run_id>/`，不覆盖 `m0` 之前的旧目录。

6. ~~**M6 C 组静态向量溯源作废（输入全域不可得）**~~ → **已完成 2026-09-14（用户裁定执行）**：
   新增 `reports/tables/pca_nale_integration/C_PROVENANCE_INVALIDATION.md`
   + `tests/test_c_cohort_provenance_invalidation.py`（6 项全绿，把"作废"变成**可执行断言**）。
   * **数字全部实测**（`scratch/probe_invalidation_numbers.py` 与上述测试双重锁定）：
     A 组三方（重算 / sidecar / 因子表）**100/100** 自洽（配方正确）；
     C 组"重算 == sidecar"**100/100**、"因子表 == sidecar"**0/100**、"因子表 == 重算"**0/100**；
     C 组公告 sidecar **36,255** vs 表 **172,968**（**4.7709×**）、新闻 sidecar **991** vs 表 **0**；
     逐条记录 37,246 = 盘上 jsonl 行数（自洽）。
   * **作废范围**：C 组 768 维向量的 `input_sha256`/`announcement_count`/`news_count` 不可核验；
     一切以 C 组静态向量为输入的分组结论（含 M1.4 `student_C` 夏普 +0.2347 一类数字）
     **禁止进入网申/PPT/对外报告与主结论**。
   * **未作废**：A 组 768 维向量（100/100 有效）；C 组 CSMAR 日频面板（前复权、恒等式复核通过）；
     C 组逐条 `publish_time` 语料（可用作**新特征族** `text_flow_v1`，但不得称"复现因子表输入"）。
   * **防静默漂移**：该测试实时重算并断言"C 组表与盘上不一致"，故任何试图**改写因子表哈希去凑 sidecar**
     的做法会立刻被拒；将来补全输入后需**人工显式更新**测试与说明（三条解除条件已写入说明第 5 节）。
   * **解除所需输入**：`data/task_split/student_c/sources/csmar_raw/trd_fward_quotation.csv`（+ filling calendar）、
     能对出 `7e67fb0e…` 的 C 组语料（100 支、约 172,968 条公告）、同批次 C 组 sidecar。三者补齐并按配方
     独立复算一致后方可解除。

## 12. 复核方法与局限

- 本节全部数值由本 Agent 亲自编写只读探针（§1 列表）在根树 `data/task_split/csmar_master/…`、`data/task_split/factors_768d_all.csv`、`data/raw/student_ac_crawled/` 上重算得到，并与另一独立 Agent 的静态清点结果交叉比对；不一致处以本 Agent 重算为准（例如"ret 是否等于 `close.pct_change()`"，重算结论为**不等**，A 组 `ret` 含真实分红送转信息）。
- 局限：(a) 未联网核验 CSMAR 额度与 `Dretwd` 可得性；B 组口径改由**库内原始导出**逐行核验（见 F2′ 第 3 条），
  C 组仍依赖表名 + 统计恒等式，未获文档级确认；(b) 未重跑 M1.4 回测本身（其产物按 §9-6 保留原样，仅作口径作废说明）；
  (c) F9 两处结论取自对报告与脚本的静态对读，尚未逐字节复算其生成日志；(d) 语料 `publish_time` 的**真实性**
  依赖抓取时源站标注，残余风险为源站回填/修订，须在 M0.3 用 URL 与内容哈希抽样核验；
  (e) 前复权基准日未知 —— 日收益率对统一重基不变（因此 `basis_return` 与 MA20/MA60 这类比率型状态变量安全），
  但任何使用 `close` **绝对水平**的特征仍依赖基准日，使用前须单独核验。
