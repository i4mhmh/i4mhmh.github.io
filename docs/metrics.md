# 评估指标

## calc_ge_sr

**mlscat.metrics.calc_ge_sr(predictions, targets, num_traces, key, times=1000, interval=1)**

计算**密钥恢复的成功率**（Success Rate, SR）和**平均排名**（Guessing Entropy, GE 风格排名），用于评估侧信道攻击（如 CPA、DPA、模板攻击）的性能。

该函数通过多次随机重采样攻击预测结果，统计在不同迹数下正确密钥的排名变化和成功恢复概率，生成平滑的评估曲线。

---

```python
def calc_ge_sr(
    predictions: list,
    targets: np.ndarray,
    num_traces: int,
    key: int,
    times: int = 1000,
    interval: int = 1
) -> tuple:
```

计算**密钥恢复的成功率**（Success Rate, SR）和**平均排名**（Guessing Entropy, GE 风格排名），用于评估侧信道攻击（如 CPA、DPA、模板攻击）的性能。

该函数通过多次随机重采样攻击预测结果，统计在不同迹数下正确密钥的排名变化和成功恢复概率，生成平滑的评估曲线。

### 参数说明

-   **`predictions`** (`list` of `dict` or `list` of `np.ndarray`)
     攻击过程中每条迹的预测概率输出列表，长度为 `N`（迹总数）。
     每个元素应为：

    -   字典：`{中间值类别 -> 概率}`，或
    -   数组：长度为 256 的概率向量（索引为密钥猜测）

    示例：`predictions[i][k]` 表示第 `i` 条迹在密钥猜测 `k` 下的似然或相关值。

-   **`targets`** (`numpy.ndarray`, 形状: `(N, 256)`)
     预先计算的每条迹在每个密钥猜测下的**中间值类别标签**。
     用于将 `predictions[i]` 映射到对应密钥猜测的概率。
     例如：`targets[i][k] = SBox(plaintext[i][byte] ^ k)`

-   **`num_traces`** (`int`)
     每次实验中用于攻击的最大迹数。结果将按 `interval` 间隔分段统计。

-   **`key`** (`int`)
     实际正确的密钥字节值（0–255），用于计算其在候选列表中的排名和是否成功恢复。

-   **`times`** (`int`, 可选, 默认: `1000`)
     重复实验的次数（蒙特卡洛模拟），用于平均随机性影响，提高评估稳定性。

-   **`interval`** (`int`, 可选, 默认: `1`)
     统计结果的时间间隔。例如 `interval=5` 表示每 5 条迹报告一次成功率和排名。

     

    输出点数为：`n_i = num_traces // interval`

------

### 返回值

返回一个包含两个数组的元组：

-   **`success_rate`** (`numpy.ndarray`, 形状: `(n_i,)`)
     每个间隔点上的**平均成功率**。
     `success_rate[i]` 表示使用前 `(i+1)*interval` 条迹时，正确密钥排在第一名的概率（即 `rank < 1`，因为索引从 0 开始）。
-   **`all_keys_rank`** (`numpy.ndarray`, 形状: `(n_i,)`)
     每个间隔点上的**平均密钥排名**（Guessing Entropy 风格）。
     `all_keys_rank[i]` 表示使用前 `(i+1)*interval` 条迹后，正确密钥的平均排名（越低越好）。

>   ⚠️ 注意：返回值已除以 `times`，即为**平均值**。

>   食用方法

~~~python
sr, rank = calc_ge_sr(
    predictions=predictions,
    targets=targets,	// you know how to calculate this array, right?
    num_traces=100,
    key=true_key,
    times=100,
    interval=5
)
~~~

## add_gaussian_noise

**mlscat.preprocessing.add_gaussian_noise(traces, std, mean=0)**

为能量迹数据添加**高斯白噪声**（Additive White Gaussian Noise, AWGN），用于：
- 模拟真实测量环境中的噪声干扰
- 增强训练数据的鲁棒性（数据增强）
- 评估密码设备或攻击算法在低信噪比下的表现

该函数对每条迹独立添加均值为 `mean`、标准差为 `std` 的正态分布噪声。

### 参数说明

-   **`traces`** (`numpy.ndarray`, 形状: `(m, n)` 或 `(m,)`)
     原始功率迹数组，支持：

    -   二维：`(m, n)` — `m` 条迹，每条 `n` 个采样点
    -   一维：`(n,)` — 单条迹

    数据类型不限，输出将保留原始语义。

-   **`std`** (`float`)
     噪声的**标准差**（Standard Deviation），控制噪声强度或“噪声水平”。

    -   `std = 0`：无噪声，返回原始数据
    -   `std` 越大，添加的噪声越强，信噪比（SNR）越低
    -   示例建议值：
        -   轻度噪声：`std = 0.1 * np.std(traces)`
        -   中度噪声：`std = 0.3 * np.std(traces)`
        -   强噪声：`std = 0.5 * np.std(traces)`

-   **`mean`** (`float`, 可选, 默认: `0`)
     噪声的均值。通常设为 `0` 表示对称扰动。若设备存在系统性偏移，可设为非零值。

------

### 返回值

-   **`noise_traces`** (`numpy.ndarray`, 形状与输入 `traces` 相同)

~~~python
import numpy as np
from mlscat.security import add_gaussian_noise

# 创建模拟迹数据 (3 条迹，每条 1000 个点)
original_traces = np.array([
    2 * np.sin(2 * np.pi * 0.02 * np.arange(1000)) + 
    0.5 * np.sin(2 * np.pi * 0.05 * np.arange(1000))
    for _ in range(3)
])  # shape: (3, 1000)

# 添加高斯噪声：均值0，标准差0.3
noisy_traces = add_gaussian_noise(traces=original_traces, std=0.3, mean=0)
~~~

## shuffling

**mlscat.preprocessing.shuffling(traces, times)**

对每条功率迹进行**随机点交换**（Random Point Swapping），模拟硬件或软件中实现的“**随机重排防护**”（Random Instruction Shuffling）机制。

该函数在每条迹内部随机选择两个采样点并交换其值，重复 `times` 次，从而扰乱功率迹的时间顺序，增加侧信道攻击难度。

> ⚠️ 注意：此方法仅用于**模拟**软件/指令级随机重排对能量迹的影响，不改变原始数据语义，但会破坏时间相关性。

