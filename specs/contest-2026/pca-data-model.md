> **已更新的交付方向（2026-09-09）**：用户已确认使用十维 PCA 动态赋权重构 NALE 的传播比重。以下旧设计仅保留讨论历史，不能作为当前实现依据。请以 [Week1 代码 Agent 任务书](week1-nale-alpha-handoff.md) 为准；Week1 尚未完成。

# PCA Week 2 数据模型

## FactorMatrix

| 字段 | 类型 | 规则 |
|---|---|---|
| `date` | datetime | 交易日、单调递增、无重复 |
| `asset_id` | string | 若为截面面板必须在日期内唯一 |
| `factor_values` | float 向量 | 训练/验证边界检查 NaN、Inf、单位和范围 |
| `source_file` | string | 指向真实 raw/processed 来源 |

## RollingPCAFit

| 字段 | 类型 | 规则 |
|---|---|---|
| `fit_start`, `fit_end` | datetime | 只包含验证日前历史 |
| `input_columns` | list[string] | 删除列及原因另存 |
| `scaler_mean`, `scaler_scale` | float 向量 | 只由训练窗估计 |
| `components` | float 矩阵 | 行为主成分、列为输入因子 |
| `explained_variance_ratio` | float 向量 | 降序、和不超过 1 |
| `cumulative_variance_ratio` | float 向量 | 单调不降 |
| `sign_alignment` | object | 参考窗口、翻转列、内积 |
| `software_version`, `random_seed` | string/int | 可复现元数据 |

## OOSMetric

| 字段 | 类型 | 规则 |
|---|---|---|
| `dataset`, `horizon`, `component_set` | string/int | 标识板块、未来收益期、PC 组合 |
| `n_obs`, `coverage` | int/float | 报告有效样本量与覆盖率 |
| `ic`, `rank_ic`, `icir`, `hit_rate` | float | 仅验证/测试区间计算 |
| `turnover`, `sharpe`, `max_drawdown` | float | 含明确交易成本假设 |
| `comparison_to_baseline` | object | 与稳定基准、raw-factor 消融并列 |

