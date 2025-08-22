# attacks模块

>   一些核心攻击代码集成

### CPA

**`cat.attacks.cpa(byte_idx, plaintexts, traces, mask_scheme=None, mask=-1)->np.ndarray`**

执行相关性能量分析（Correlation Power Analysis, CPA），用于从加密设备的能量迹中恢复指定位置的密钥字节。

**参数**

- **`byte_idx`** (`int`)  
  要攻击的密钥字节索引（例如：0–15 对于 AES-128）。

- **`plaintexts`** (`numpy.ndarray`)  
  输入明文数据，形状为 `(N, L)` 或 `(N,)`，其中 `N` 是能量迹的数量，`L` 是每条明文的字节数。

- **`traces`** (`numpy.ndarray`)  
  能量迹数据，二维数组，形状为 `(N, T)`，`N` 是能量迹的数量，`T` 为每条能量迹的时间样本点个数。必须与 `plaintexts` 的数量一致。

- **`mask_scheme`** (`str`, 可选, 默认: `None`)  
  掩码防护方案类型（目前仅有`bool`可选）。  
  
- **`mask`** (`int`, `list`, 或 `numpy.ndarray`, 可选, 默认: `-1`)  
  掩码值或掩码序列，形状为 `(N,)`。  

**返回值**

- **`np.ndarray`**  
  推测出的最佳密钥值，返回形式为 `np.array([best_key])`，其中 `best_key` 是最可能的密钥字节（0–255）。

**示例**

```python
import mlscat
import numpy as np

# 示例数据
plaintexts = np.array([[0x32, 0x43, 0xf1, 0xa4], 
                       [0x32, 0x43, 0xf1, 0xa5], 
                       [0x32, 0x43, 0xf1, 0xa6]])
traces = np.random.rand(3, 1000)

# 执行 CPA 攻击
guess_key = mlscat.attacks.cpa(byte_idx=3, plaintexts=plaintexts, traces=traces)

print("推测密钥字节:", guess_key)  # 输出: 推测密钥字节
```

## DPA

**`cat.attacks.dpa(traces, plaintexts, threshold, target_byte, target_point, leakage_function)`** 

执行**差分能量分析**（Differential Power Analysis, DPA），用于从加密设备的能量迹中恢复 AES-128 的密钥字节。 该函数基于指定的泄漏模型（如汉明重量）将能量迹分为两组，计算每组在目标采样点上的均值差异，从而推测最可能的密钥候选。 

>   ⚠️ **注意**：当前实现专为 **AES-128** 设计。若用于 AES-192、AES-256 或其他算法，需手动修改中间值计算逻辑。 

**参数** 

-   **`traces`** (`numpy.ndarray`)    

    能量迹数据，形状为 `(N, T)`，其中 `N` 是迹的数量，`T` 是每条迹的采样点数。

-   **`plaintexts`** (`numpy.ndarray`) 

    对应的明文数据，形状为 `(N, 16)`（适用于 AES-128）。每条明文应为 16 字节。

- **`threshold`** (`int`)  
  用于分组的阈值。根据泄漏函数的输出值与该阈值比较，将迹划分为两组：
  - 小于阈值 → 组1
  - 大于等于阈值 → 组2
- **`target_byte`** (`int`)  
  要攻击的密钥字节索引（0–15），用于计算中间值 `SBox[plaintext[target_byte] ^ key_guess]`。
- **`target_point`** (`int`)  
  在每条能量迹中要分析的**采样点索引**（例如：810）。攻击基于该点的能量值进行分组均值分析。
- **`leakage_function`** (`str`)  
  使用的泄漏模型：
  - `'hw'`：使用**汉明重量**（Hamming Weight）作为分类依据
  - 其他值：直接使用 S 盒输出值（不推荐，通常 `'hw'` 更有效）

**返回值**

- **`candidate_key`** (`int`)  
  推测出的最可能密钥字节（0–255），对应最大均值差的密钥猜测。
- **`mean_diffs`** (`numpy.ndarray`, 形状: `(256,)`)  
  所有 256 个密钥猜测对应的两组迹在 `target_point` 处的均值差绝对值数组，可用于进一步分析或绘图。

**示例**

```python
import mlscat
import numpy as np

# 假设已采集数据
traces = np.load('traces.npy')       # shape: (2000, 15000)
plaintexts = np.load('plain.npy')    # shape: (2000, 16)

# 执行 DPA 攻击：攻击第 0 字节，在采样点 810 分析，使用汉明重量模型，阈值为 4
candidate, diffs = mlscat.attacks.dpa(
    traces=traces,
    plaintexts=plaintexts,
    threshold=4,
    target_byte=0,
    target_point=810,
    leakage_function='hw'
)

print("推测密钥字节:", candidate)  
```

## TemplateAttack

> **模板攻击**（Template Attack, TA）—— 一种高阶侧信道分析方法，基于建模阶段的统计特征对攻击阶段数据进行密钥推测。

`TemplateAttack` 类实现了经典的**基于多元高斯分布的模板攻击**，分为两个阶段：



1. **模板构建阶段**（`profiling`）：从能量迹数据中学习每个中间值类别的均值与协方差矩阵。
2. **攻击阶段**（`attack`）：使用模板对未知密钥的攻击数据进行概率推断，并计算**平均排名**（Mean Rank）以评估密钥恢复能力。

---

```python
class TemplateAttack:
    def __init__(self, X_profiling, Y_profiling, X_attack) -> None:
        """
        初始化模板攻击对象。

        参数:
            X_profiling (np.ndarray): 配置阶段的能量迹，形状为 (N_prof, T)。
            Y_profiling (np.ndarray): 配置阶段的中间值标签，形状为 (N_prof,)。
            X_attack (np.ndarray): 攻击阶段的能量迹，形状为 (N_attack, T)。

```

## PCC

计算**皮尔逊相关系数**（Pearson Correlation Coefficient）向量，用于侧信道分析中衡量中间值猜测与能量迹各采样点之间的线性相关性。

该函数对每一条迹的每个采样点分别计算与给定目标中间值序列的相关性，返回绝对值形式的相关系数数组，便于后续分析（如 CPA 攻击中寻找最大相关点）。

### 参数

-   **`targets`** (`numpy.ndarray`, 形状: `(N,)`)
     中间值模型输出序列（例如：`SBox[plaintext[i] ^ k_guess]`），长度为 `N`，对应 `N` 条能量迹。
-   **`traces`** (`numpy.ndarray`, 形状: `(N, T)`))
     能量迹数据，其中：
    -   `N`：迹的数量（必须与 `targets` 长度一致）
    -   `T`：每条迹的采样点数量

------

### 返回值

-   **`pearson_list`** (`numpy.ndarray`, 形状: `(T,)`)
     每个采样点与 `targets` 之间的皮尔逊相关系数的**绝对值**数组。
     值域范围：`[0, 1]`

    -   接近 `1`：强相关
    -   接近 `0`：弱相关或无相关

    用于定位最可能泄露密钥信息的采样点。

~~~python
import numpy as np
from mlscat.utils import pcc

# 假设已有数据
targets = np.array([123, 45, 89, 111, 67])  # 5 条迹的中间值
traces = np.random.rand(5, 1000)            # 5 条迹，每条 1000 个采样点

# 计算相关系数
correlations = pcc(targets, traces)

# 找到最大相关点
max_corr_point = np.argmax(correlations)
print("最相关采样点索引:", max_corr_point)
print("相关系数:", correlations[max_corr_point])
~~~





