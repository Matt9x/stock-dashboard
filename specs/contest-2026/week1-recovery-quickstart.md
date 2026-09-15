> **2026-09-10 实际回测完成更新（优先于下方旧状态）**：已完成V1–V5及B0的98股历史标题/真实Dretwd代理实验，测试378个信号日，独立复核通过。各版Rank IC约-0.033，净Sharpe约0.885，最大回撤37.46%；逐级增益不显著，V1全程不可识别回退、V2另有22日求解失败回退。研究结果不支持替换稳定基准。生产300股统一历史输入仍不足；不能把代理研究称生产NALE全量验收。报告见 reports/tables/nale_alpha_week1/20260910-five-formulas/five_formula_report.md。PR1–6远端已核对，PR4四份缺失原始表已隔离恢复。旧前视成绩仍无效。

# Week1 接续操作说明

当前分支 `contest-2026`。先读 [接续计划](week1-recovery-plan.md) 和 [审计报告](../../reports/tables/nale_alpha_week1/recovery-audit-20260910/validation_report.md)。旧评估脚本有前视且固定写mvp目录，不将其输出用于正式绩效。

## 已存在、可以复核的命令

在仓库根目录执行；解释器按本地安装替换，示例为本轮实际执行的Windows Python launcher。

```powershell
py -3 -m pytest -q tests/test_dynamic_nale_alpha.py -p no:cacheprovider
```

2026-09-10 实际结果9 passed。其覆盖范围不包括完整历史因果性，不能用于宣告Week1完成。独立数学反例：

```powershell
@'
import numpy as np
from src.pricing.dynamic_nale_alpha import calibrate_public_calibrator
s = np.array([0., 1.])
a, c, _ = calibrate_public_calibrator(s, s)
print('actual=', (a,c), 'expected=', (0.,1.))
assert np.allclose([a,c], [0.,1.]), '旧校准器公式错误：不能由9个既有单测排除'
'@ | py -3 -
```

这个断言在当前未修复代码上预期失败；它是手算夹具，不是市场实验。修复后应通过。检查当前输入与审计哈希一致：

```powershell
@'
import hashlib, json
from pathlib import Path
p = Path('reports/tables/nale_alpha_week1/recovery-audit-20260910/audit_evidence.json')
e = json.loads(p.read_text(encoding='utf-8'))
for name, expected in e['input_hashes'].items():
    assert hashlib.sha256(Path(name).read_bytes()).hexdigest() == expected, name
print('审计输入与源码哈希匹配')
'@ | py -3 -
```

## 实施阶段命令（本轮没有执行源码修复）

先完整阅读bug合集，再登记开始；源码改动后优先尝试全量真实数据层回测与统计，前提不满足登记阻断，随后small→medium→heavy；不得拿旧凭证复用为新凭证。

```powershell
./tools/run_quality.ps1 begin-unit --name 'Week1 无前视研究恢复' --acceptance '独立校准解析解、成熟标签、未来扰动不变性、真实数据审计、全版本隔离评估；缺数据明确阻断；稳定产物不覆盖'
# 按计划R1-R5实施；真实全量回测或记录其数据阻断
./tools/run_quality.ps1 small
./tools/run_quality.ps1 medium --feature 'Week1 无前视研究恢复'
./tools/run_quality.ps1 heavy --version 'week1-recovery'
```

新CLI `scripts/evaluate_nale_alpha.py` 和配置 `config/experiments/nale_alpha_week1.json` 是设计目标，目前不存在。实现前不要将下列命令描述为可运行交付：

```powershell
py -3 scripts/evaluate_nale_alpha.py --config config/experiments/nale_alpha_week1.json --run-id week1-first-causal-run --audit-only
# 仅在数据审计通过后运行正式评估，使用未被占用的独立run_id
```

本轮的规划工具setup-plan已运行并解析路径；其模板会覆盖plan.md，调用时内存保留原文件并在finally恢复。正式入口在plan.md顶部链接本接续计划，以保留其他正在进行的工作。
