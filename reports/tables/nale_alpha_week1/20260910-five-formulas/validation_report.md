# 五公式回测验证报告

结论：98股受限代理回测已执行并完成独立数值复核；整个项目发布门禁未通过，禁止宣称全项目验收或生产替换。

- 专项测试：tests/test_dynamic_nale_alpha.py、test_nale_alpha_recovery.py、test_integration_nale_alpha_recovery.py，共33 passed；增加图表后再次单跑完整流水线冒烟，1 passed in 11.77s。冒烟输入为明确标注的合成夹具，与本轮真实行情结果分开。
- small通过：.quality-state/reports/20260910171040-small-4c35cc.md。
- medium通过：.quality-state/reports/20260910171200-medium-5cd75f.md。
- heavy实际执行：.quality-state/reports/20260910171906-heavy-edb4f8.md。全套pytest 1042 passed、5 failed、15 subtests passed；依赖检查另外失败，magika 0.6.3要求Windows onnxruntime<=1.20.1，当前1.29.0。v0.0.0只是门禁格式参数，没有发布、打标签或提交。
- 五个失败节点：旧截面特征溯源哈希与原始meta不一致（000001）；data_adapter缺日历时错误提示与断言不符；daily脚本调用共享push helper的静态断言；teacher固定收益正值断言；teacher涨停簇数量6与7不符。这些不属于本轮五公式新增测试，不据此声称一定是历史已有失败，也不放宽断言或改用户数据。原始JSONL/meta/manifest哈希一致性测试通过；本轮没有使用失败的旧截面特征/旧provenance，自己重建历史embedding并保存输入哈希。
- verdict=block，质量CLI登记BUG-0024、BUG-0025；未绕过门禁，质量状态只由CLI维护。

独立复核详见independent_validation.json：222264条预测的S=S0+alpha*D及y_hat=a+c*S恒等式、alpha范围、入场退出顺序；77个拟合快照的最大标签日期严格早于拟合日期；三个真实样本五日Dretwd超额标签手算；全部版本逐日Spearman IC、ICIR，以及净值、Sharpe、最大回撤从落盘账本独立重算。图表已目视检查，两图220dpi。完成时重新验证全部源码及本轮原文/收益输入哈希未变化。

限制：账本复算证明算术一致，不证明市场可成交。未验证公告原版本、模型预训练时间点可用性、历史股票池及退市回收；这是回顾代理研究。V1全程回退，V2一次优化器ABNORMAL回退22日，不能描述为五种模型每月都成功识别。未取得显著改善，稳定量化产物保留。300股生产验收和B1仍缺数据。
