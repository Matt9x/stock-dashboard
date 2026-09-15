# Week1 接续接口契约（待实施）

本契约从属于 `../week1-nale-alpha-handoff.md`。旧 `dynamic_nale_alpha.md` 中与它冲突的下界、损失、参数边界和日程不再作为接续实现依据。项目是离线Python批处理，本轮不虚构REST/GraphQL服务。

## 输入审计与切分

`audit_inputs(config) -> AuditResult`：验证行情、特征、网络、名称映射、源版本、输入哈希和日期覆盖。缺真实历史或来源时状态为BLOCKED，输出缺口；不自动补造。数据错误、重复(date,code)、NaN/Inf、代码丢前导零抛明确异常。

`build_labels(prices, benchmark, horizon=5) -> ForwardLabelPanel`：标签为 P[t+1+h]/P[t+1]-1，超额标签减去同持有期基准；signal在t收盘之后。交易日由明确日历查找，不能以自然日偏移替代。缺入/退出价格或未成熟尾部不计算标签。

`split_and_schedule(panel, config) -> SplitManifest`：同一信号日期全部股票同分区；验证/测试边界剔除跨越下一分区预测截止的训练标签。每月首交易日开盘前、截止前收盘，选最近126个成熟信号日。验证选lambda/H，测试前冻结配置。

## 模型

`fit_pca(training_features)`：仅训练期，full SVD，有效中心化秩>=10；保存均值/尺度/列顺序/模型版本；transform时列集合一致后显式对齐，不接受乱序导致语义错配。未来样本不能改变已有基底。

`calibrate_public_calibrator(scores, returns, date_ids) -> (a,c,diagnostics)`：日期内有效股票等权、日期等权；方差与协方差使用相同权重约定；c=max(0,cov/var)。零分数方差则c=0、a=加权y均值，报告不可识别；负斜率同理。所有版本共享同次拟合校准器，不强制正下界。

`fit_gated_weights(...) -> FitSnapshot`：损失为日期加权的股票内均方误差 + lambda*||theta||²，theta包含b与所有条件系数；不得未声明地除以Var(y)，不额外约束theta在[-3,3]。零初值；记录求解状态/目标值/梯度；c=0、多数D=0或优化失败时回退B0并记原因。主版本支持逐期Z[T,N,10]，不能将静态Z[N,10]复制成“历史文本面板”。

`predict(snapshot, signal_panel) -> Predictions`：严格只读当时可用输入，alpha=.05+.70*expit(u)，S=S0+alpha*D，y_hat=a+cS。B0固定.4。B1调用真实既有事件规则，事件缺失则N/A；禁止把另一个离散度启发式命名为B1。

`compute_reliability(history, cutoff)`：用当时保存的个股/网络预测与截止前成熟标签；最近60信号日误差差，<20日置0并记录；定标训练期冻结后clip[-3,3]。训练预热是前推预测，不用样本内拟合误差。不得读取当前尚待预测的y。

## 评估与输出

`evaluate(predictions, realized_labels, ledger, config)`：每日>=20股才算IC；ICIR均值/样本标准差，不年化，零方差N/A。方向基于y_hat而非S，真实零收益和零预测单列、报告覆盖和多数类；逐级日期区块bootstrap2000次seed42，Holm及区块敏感性；费用、换手、正幅度MDD依统一净值账本。收盘选股不得取得同日已发生收益；历史区间只能用当时snapshot。

拟实施CLI：`python scripts/evaluate_nale_alpha.py --config <json> --run-id <id> --audit-only`；通过审计后去掉`--audit-only`评估。退出码：0=本阶段成功（可能负结果），2=数据/成熟度阻断，1=运行错误。manifest同时记录scope_status，避免audit-only退出0被当成MVP完成。目录已存在返回错误，禁止静默覆盖旧mvp目录。

完整运行输出沿用总任务书§10；额外保存交易账本、每日指标和校准器快照。数据阻断时仅输出审计、缺口与blocked manifest，绩效表不生成成功数值。
