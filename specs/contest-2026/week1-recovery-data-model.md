# Week1 接续数据模型

日期：2026-09-10。以下是待实施的数据契约，不能声称旧CSV已经包含这些字段。

| 实体 / 主键 | 字段 | 验证与关系 |
|---|---|---|
| DocumentVersion / document_id, revision_id | code、publish_at、available_at、retrieved_at、source_url、content_sha256、文本、版本证据 | code六位字符串；抓取时间不能代替发布时可用时间；晚修订内容不得前灌 |
| FeatureSnapshot / signal_date, code, feature_version | available_at、document_hashes、embedding_model/revision、aggregation_cutoff、dim_000…dim_767 | 截止与所有原文可用时间<=signal_cutoff；同embedding版本；有限数值、维数正确；不覆盖快照 |
| MarketObservation / date, code | open/close、volume、trade_status、adjustment_type、adjustment_version、available_at、source | 价格>0；复权/停牌/上市状态按来源核实；缺失不前向回填成可成交记录 |
| EdgeSnapshot / date, source_code, target_code, network_version | weight、evidence_id、available_at、valid_from、valid_to | 非负有限权；证据当时可用；行归一化；无邻居统一自环，原样基线另报 |
| SignalPanel / date, code | signal_cutoff、S0、N、D、PC01…PC10、pca_version、g、q、q_valid_days、fallback_reason | D=N-S0；代码映射显式连接，不能依赖输入行顺序；缺历史S0或网络拒绝正式评估 |
| ForwardLabel / signal_date, code, horizon | entry_at、exit_at、label_available_at、asset_return、benchmark_return、y_excess | t+1收盘入场、t+1+h退出；同持有期基准；y为小数；尾部不成熟不补值 |
| FitSnapshot / fit_id, model_version | cutoff、train_dates、max_label_available_at、PCA版本、a/c、theta、lambda/H、initial_theta、success/message、loss/gradient、sample_count、fallback_reason | 拟合开盘前截止前收盘；所有标签退出严格早于拟合时点；PCA只用初始训练；系数及输入全哈希 |
| Prediction / run_id, date, code, version | fit_id、S0/N/D、PC01…PC10、十维贡献、u、alpha_nale、S、y_hat、g/q、input_available_at、train_cutoff、fallback_reason | S=S0+alpha*D；y_hat=a+cS；当时参数快照；事后标签独立关联，不能回写预测 |
| ReliabilityHistory / signal_date, code | 当时a/c、individual_prediction、network_prediction、label_available_at、squared_error_difference | 仅汇总当前决策前已成熟的最近60信号日；<20日q=0并标缺失；定标参数训练期冻结 |
| RunManifest / run_id | config/hash、input/code hashes、environment、seed、split_id、version_status、quality_receipts、artifact_hashes、scope_status | 目录已存在拒绝覆盖；无输入哈希/分区/预测不可标研究完成 |

时间统一为带时区 Asia/Shanghai 时间戳，交易日字段另存日期。只有日期的公告采用下一交易日才可用的保守假设，并单独披露；无法确定原版本时不放进有效历史面板。

三维状态分别记录：engineering={planned,partial,verified}；research={blocked,valid_negative,valid_inconclusive,valid_positive}；promotion={not_eligible,eligible}。缺特征或网络时research=blocked；工程测试通过不自动迁移研究/发布状态。当前为partial/blocked/not_eligible。
