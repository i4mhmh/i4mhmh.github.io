# 防护措施

## Desynchronization

**mlscat.security.desynchronization(traces, window)**

对功率迹数据施加**随机去同步化**（Random Desynchronization），用于模拟或增强对抗实际环境中因时间抖动、防护机制（如随机延迟）引起的信号偏移。

该函数为每条迹引入一个在 `[0, window]` 范围内的随机延迟，在时间轴上向右移动信号，并使用**零填充**（zero-padding）处理移位后空出的部分。

### 参数说明

-   **`traces`** (`numpy.ndarray`, 形状: `(m, n)`)
     原始二维功率迹数组，其中：

    -   `m`：迹的数量
    -   `n`：每条迹的采样点数

    示例：`traces[i]` 表示第 `i` 条功率迹。

-   **`window`** (`int`)
     去同步化的最大延迟范围（以采样点为单位）。每条迹将被随机延迟 `0` 到 `window` 个点（含）。

    -   若 `window = 0`：无延迟，返回原始数据。
    -   推荐值：根据采样率和系统时钟波动估计（如 1–10 个点）。

------

### 返回值

-   **`delay_traces`** (`numpy.ndarray`, 形状: `(m, n)`)
     经过去同步化处理后的迹数组。每条迹向右移动 `random_num ∈ [0, window]` 个点，左侧补 `random_num` 个 `0`。

    >   ⚠️ **注意**：由于使用**零填充**，移位后迹的起始部分会引入 `0` 值，暂时不知道如何优化

    

~~~python
import numpy as np
from mlscat.security import desynchronization

# 创建模拟迹数据 (5 条迹，每条 100 个采样点)
original_traces = np.array([
    np.sin(np.linspace(0, 4*np.pi, 100)) + 0.1*np.random.randn(100) 
    for _ in range(5)
])  # shape: (5, 100)

# 施加去同步化，最大延迟为 5 个采样点
desync_traces = desynchronization(traces=original_traces, window=5)
~~~

## shuffling

**mlscat.preprocessing.shuffling(traces, times)**

对每条能量迹进行**随机点交换**（Random Point Swapping），模拟硬件或软件中实现的“**随机重排防护**”（Random Instruction Shuffling）机制。

该函数在每条迹内部随机选择两个采样点并交换其值，重复 `times` 次，从而扰乱功率迹的时间顺序，增加侧信道攻击难度

### 参数说明

-   **`traces`** (`numpy.ndarray`, 形状: `(m, n)` 或 `(n,)`)
     原始能量迹数组：

    -   二维：`(m, n)` — `m` 条能量迹，每条 `n` 个采样点
    -   一维：`(n,)` — 单条能量迹

    示例：`traces[i][t]` 表示第 `i` 条能量迹在时间点 `t` 的功耗值。

-   **`times`** (`int`)
     每条能量迹执行随机交换的次数。

    -   `times = 0`：无交换，返回原始数据
    -   `times` 越大，重排程度越高，时间结构破坏越严重
    -   推荐值：`10 ~ 100`（根据能量迹长度调整）

    >   💡 提示：通常 `times` 与迹长度 `n` 成正比（如 `times = n // 10`）

------

### 返回值

-   **`shuffled_traces`** (`numpy.ndarray`, 形状与输入 `traces` 相同)
    经过 `times` 次随机交换后的功率迹数据。
    每条迹独立进行重排，能量迹之间不交叉。

~~~python
import numpy as np
from mlscat.preprocessing import shuffling

# 创建模拟能量迹数据 (2 条迹，每条 200 个点)
original_traces = np.array([
    np.sin(0.1 * np.arange(200)) + 0.5 * np.random.rand(200),
    np.cos(0.1 * np.arange(200)) + 0.3 * np.random.rand(200)
])  # shape: (2, 200)

# 应用随机重排：每条能量迹交换 30 次
shuffled_traces = shuffling(traces=original_traces, times=30)
~~~

