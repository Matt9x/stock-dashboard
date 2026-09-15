> **旧契约，已停止作为实现依据**：当前输入为股票×日期的十维 PCA 历史面板，权重通过回归学习。使用 [Week1 代码 Agent 任务书](../week1-nale-alpha-handoff.md)；旧文中测试期主成分必须正交的表述不适用。

# PCA 滚动管线契约

## 接口

```python
def fit_transform_rolling_pca(
    factors: pd.DataFrame,
    *,
    train_days: int,
    validation_dates: pd.DatetimeIndex,
    n_components: int | float,
    drop_constant: bool = True,
    reference_loadings: pd.DataFrame | None = None,
) -> tuple[pd.DataFrame, list[dict]]:
    """只用历史训练窗拟合 PCA，并返回验证期得分与窗口元数据。"""
```

## 前置条件

- 日期索引唯一、递增；`train_days >= 30`；
- 输入不含 NaN/Inf；常量列被删除或显式记录；
- `n_components` 不得超过当前窗口有效因子数；
- 不允许从验证区间估计填充值、标准化参数或载荷。

## 后置条件

- 输出索引与验证日期严格一致；
- 各窗口主成分两两相关绝对值 `< 0.01`（数值容差内）；
- 元数据包含窗口边界、输入列、解释率、载荷、符号对齐和版本；
- 因子数不足、日期错位、真实收益缺失时抛出可识别异常，不回退到 Mock 数据。

## 性能与测试

- 20 因子×252 日的单窗口拟合目标 `< 50ms`，以本机基准记录实际值；
- 测试覆盖正常输入、边界窗口、常量列、重复列、NaN/Inf、因子数不足、日期错位、符号翻转和 OOS 日期泄漏；
- 集成 smoke test 必须验证真实 CSV → 滚动 PCA → 指标表的完整链路。

