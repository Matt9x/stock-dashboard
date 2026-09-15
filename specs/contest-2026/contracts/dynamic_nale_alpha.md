# API 契约：十维 PCA 动态赋权重构 NALE (Week 1 Contract)

**版本**：v1.0  
**依据**：[`specs/contest-2026/week1-nale-alpha-handoff.md`](../week1-nale-alpha-handoff.md)  
**状态**：已冻结  

---

## 1. 核心数学契约 (Mathematical Formulation)

### 1.1 动态门控与传播公式

对于截面股票 $i$ 在交易日 $t$：
$$N_{i,t} = (W_t S_{0,t})_i, \quad D_{i,t} = N_{i,t} - S_{0,i,t}$$

十维 PCA 输入 $z_{i,k,t}$（均值 0，方差 1，降维基底冻结自初始训练期），门控线性组合：
$$u_{i,t} = b_m + \sum_{k=1}^{10} w_{k,m} z_{i,k,t}$$

传播混合系数 $\alpha_{i,t}$（工程有界范围 $[0.05, 0.75]$，初始 $u=0$ 时 $\alpha=0.40$）：
$$\alpha_{i,t} = 0.05 + 0.70 \cdot \sigma(u_{i,t}), \quad \text{其中 } \sigma(u) = \frac{1}{1 + e^{-u}}$$

最终 NALE 综合打分：
$$S_{i,t} = S_{0,i,t} + \alpha_{i,t} D_{i,t}$$

---

### 1.2 公共校准器 (Public Dimensionless Calibrator)

为了使无量纲打分映射至未来超额收益 $y_{i,t}$，先在初始训练窗利用 B0（固定 $\alpha=0.4$）打分拟合并冻结线性校准器：
$$\hat{y}_{i,t} = a + c \cdot S_{i,t}, \quad c \ge 0.005$$

训练损失函数（日期内等权，跨日期按时间衰减加权）：
$$L(\theta) = \frac{\sum_t \omega_t \frac{1}{n_t} \sum_{i=1}^{n_t} (y_{i,t} - \hat{y}_{i,t}(\theta))^2}{\sum_t \omega_t \cdot \operatorname{Var}(Y)} + \lambda \|\theta\|_2^2$$

其中 $\omega_t = 2^{-\text{age}(t)/H}$（V3 时间加权衰减，默认半衰期 $H=60$ 交易日）。

---

## 2. Python 接口定义 (`src/pricing/dynamic_nale_alpha.py`)

### 2.1 `DynamicNALEAlphaEstimator`

```python
class DynamicNALEAlphaEstimator:
    def __init__(
        self,
        n_components: int = 10,
        l2_reg: float = 0.001,
        half_life: float = 60.0,
        random_state: int = 42
    ): ...

    def fit_pca(self, factors_768_df: pd.DataFrame) -> "DynamicNALEAlphaEstimator":
        """对 768D 文本因子执行标准主成分分析并冻结投影基底。
        要求中心化后有效秩 >= 10，否则抛出 ValueError。
        """
        ...

    def transform_pca(self, factors_768_df: pd.DataFrame) -> pd.DataFrame:
        """输入 768 维特征，输出 (N, 10) 的标准化主成分矩阵 Z。"""
        ...

    def calibrate_public_calibrator(
        self,
        S0_train: np.ndarray,
        D_train: np.ndarray,
        Y_train: np.ndarray
    ) -> tuple[float, float]:
        """使用 B0 (alpha=0.40) 的训练数据拟合公共校准参数 (a, c)。
        保证 c >= 0.005。
        """
        ...

    def fit_gated_weights(
        self,
        S0_train: np.ndarray,
        D_train: np.ndarray,
        Y_train: np.ndarray,
        Z_train: np.ndarray,
        version: str = "V1",
        time_decay: bool = False,
        g_regime: np.ndarray | None = None,
        q_reliability: np.ndarray | None = None,
    ) -> np.ndarray:
        """非线性有界优化求解 theta = (b, w_1...w_10, ...)。
        参数边界限定在 [-3.0, 3.0]，零初值起步。
        """
        ...

    def predict_alpha(
        self,
        Z: np.ndarray,
        theta: np.ndarray,
        version: str = "V1",
        g: float = 0.0,
        q: float = 0.0
    ) -> np.ndarray:
        """计算每只股票的动态传播系数 alpha_i in [0.05, 0.75]。"""
        ...

    def propagate(
        self,
        S0: np.ndarray,
        W_norm: np.ndarray,
        alpha: np.ndarray | float
    ) -> np.ndarray:
        """执行图传导：S = S0 + alpha * (W_norm @ S0 - S0)。
        若股票无邻居（孤立节点），自动退化为自环令 D_i = 0，保证 S_i = S0_i。
        """
        ...
```

---

## 3. 五大版本配置矩阵 (Model Version Matrix)

| 版本代号 | 架构描述 | 训练加权 $\omega_t$ | 状态变量 $g_t$ | 可靠性 $q_t$ | 门控参数数量 |
|:---|:---|:---|:---|:---|:---|
| **B0** | 固定常量基线 ($\alpha=0.40$) | N/A | 无 | 无 | 0 |
| **B1** | 现有事件衰减/高斯时滞动态基线 | N/A | 离散事件 | 无 | 0 (规则驱动) |
| **V1** | 十维 PCA 静态有界回归 | 样本等权 | 无 | 无 | 11 ($b, w_1..w_{10}$) |
| **V2** | 十维 PCA 月度滚动回归 (21日走步) | 样本等权 | 无 | 无 | 11 |
| **V3** | 十维 PCA 时间衰减滚动回归 | $2^{-\text{age}/60}$ | 无 | 无 | 11 |
| **V4** | 市场状态条件加权回归 | $2^{-\text{age}/60}$ | MA20/MA60 - 1 | 无 | 22 |
| **V5** | 网络可靠性与状态双条件回归 | $2^{-\text{age}/60}$ | MA20/MA60 - 1 | 滚动均方误差差 | 33 |

---

## 4. 异常与边界处理约束 (Boundary & Failure Contract)

1. **数值防溢出**：对所有 $u$ 截断至 $[-15.0, 15.0]$，确保 `np.exp(-u)` 永不溢出为 Inf 或下溢为 NaN。
2. **孤立节点处理**：若 $W_{norm}$ 对应行为全 0（无产业链上下游边），自动置对角线自环 $W_{i,i}=1$，令 $D_i=0$。
3. **退化识别**：若校准器 $c=0$ 或矩阵 $D$ 全为 0，门控不可识别，系统自动回退至 B0（固定 0.40），并记录警告日志。
4. **代码保留前导零**：所有证券代码严格为 6 位数字字符串（例如 `000001`、`002594`），不得转换为整型。
