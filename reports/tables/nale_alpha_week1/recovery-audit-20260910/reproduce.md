# 只读复核命令

在项目根目录PowerShell中运行。以下复制生成逻辑到内存作来源指纹比对，不调用有写盘副作用的生成函数。随机数仅用于验证模拟数据来源，不是新市场实验。

```powershell
@'
import numpy as np
import pandas as pd
rng = np.random.RandomState(42)
dates = pd.bdate_range('2024-01-02', '2026-08-28')
r = []
for d in dates.strftime('%Y-%m-%d'):
    mean, sd = ((-.0002, .009) if d < '2024-09-24' else
                (.015, .025) if d <= '2024-10-15' else
                (.0005, .011) if d <= '2024-12-31' else
                (.0006, .010) if d <= '2025-12-31' else (.0003, .009))
    r.append(rng.normal(mean, sd))
r = np.array(r)
index = 3400 * np.cumprod(1 + r)
rng.uniform(15, 120)
base = rng.uniform(180, 320)
stock = base * np.cumprod(1 + np.clip(1.35*r + .28/252 + rng.normal(0, .022, len(dates)), -.1, .1))
p = pd.read_csv('data/raw/backtest_paper_2024_2026_300stocks/market_prices.csv', index_col=0)
assert list(dates.strftime('%Y-%m-%d')) == list(p.index)
assert np.allclose(index, p['000300.SH'], rtol=0, atol=1e-10)
assert np.allclose(stock, p['300750'], rtol=0, atol=1e-10)
print('模拟指数最大误差', np.abs(index-p['000300.SH']).max())
print('模拟300750最大误差', np.abs(stock-p['300750']).max())
m = pd.read_csv('data/task_split/csmar_master/csmar_factor_panel_master.csv', dtype={'stock_code': str})
m = m.sort_values(['stock_code', 'trade_date'])
delta = (m.groupby('stock_code').close.pct_change(fill_method=None) - m.ret).abs()
print('主表', m.shape, '股票', m.stock_code.nunique(), '日期', m.trade_date.nunique())
print('close简单收益与ret差异超过10bp的行数', int((delta > .001).sum()))
f = pd.read_csv('data/task_split/factors_768d_all.csv', dtype={'code': str})
x = f.filter(regex='^dim_').to_numpy(float)
print('特征', f.shape, '中心化秩', np.linalg.matrix_rank(x-x.mean(0)))
print('行情缺少的因子代码', sorted(set(f.code)-set(m.stock_code)))
'@ | py -3 -
```

本轮实际输出：694日日期一致；误差9.094947e-13和4.547474e-13；主表(192199,22)、299股644日；308行收益口径差异；特征(300,780)、中心化秩299；缺600317。

9个旧测试和校准器独立解析反例的命令见 [quickstart](../../../../specs/contest-2026/week1-recovery-quickstart.md)。环境与完整输入哈希见audit_evidence.json。上述数字仅对这些输入哈希有效。
