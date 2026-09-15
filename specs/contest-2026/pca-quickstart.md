> **已更新的交付方向（2026-09-09）**：用户已确认使用十维 PCA 动态赋权重构 NALE 的传播比重。以下旧设计仅保留讨论历史，不能作为当前实现依据。请以 [Week1 代码 Agent 任务书](week1-nale-alpha-handoff.md) 为准；Week1 尚未完成。

# PCA Week1剩余任务 快速启动

1. 先检查产物和真实行情：

```powershell
python -c "import pandas as pd; p=pd.read_csv('data/week1_pca/factors_for_pca.csv',index_col=0); print(p.shape); print(p.isna().sum().sum())"
```

2. 运行现有 Work1 PCA 作为描述性基线（不用于 OOS 结论）：

```powershell
python scripts/day2_pca_extraction.py --input data/week1_pca/factors_for_pca.csv --output-dir data/week1_pca/baseline --n-components 5
```

3. 实现后运行滚动验证：

```powershell
python scripts/run_rolling_pca_validation.py --dataset green --train-days 126 --horizons 5,20 --components 1,2,3,4,5,6
```

4. 检查以下文件后再讨论是否集成：

```text
data/processed/pca_week1/green/rolling_fits.jsonl
reports/tables/pca_data_quality.csv
reports/tables/pca_oos_ablation.csv
reports/tables/pca_stability.csv
reports/pca_validation_report.md
```

若真实价格无法完成日期/股票对齐，命令应返回明确错误并停止，不生成性能结论。


