# 需求与证据

|要求|本轮证据/状态|
|---|---|
|V1–V5实际运行并与B0对照|version_comparison.csv、predictions.parquet，已完成|
|历史特征/成熟标签/月度更新|feature_provenance.json、split_manifest.json、fit_snapshots.json|
|解析公式/失败路径/时序测试|33项专项测试及independent_validation.json|
|同口径收益、成本、回撤|portfolio_ledger.csv、cost_sensitivity.csv|
|配对检验与稳健性|paired_differences.csv，5/10/20日区块、2000次bootstrap、Holm|
|显著提升才替换稳定产物|未显著，production_promotion=false|
|生产300股、供应链网络、原文认证|数据不足，未完成；见data_audit.md|
|B1历史事件对照、20日辅助标签|本轮未评估，主标签5日|
|与原论文差异及比赛官方指标|尚无已核验原文，不宣称首创或官方达标|
|全项目质量门禁|small/medium通过，heavy失败，verdict=block|
