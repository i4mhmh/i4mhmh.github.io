# 其他工具

## get_mid

**mlscat.utils.get_mid(p, k, mask_scheme=None)**

计算加密过程中某个字节的**中间值**（Intermediate Value），支持无掩码和布尔掩码（Boolean Masking）两种模式，常用于侧信道分析中的模板构建或攻击阶段标签生成。

该函数模拟 AES 加密中 `SBox[plaintext ^ key]` 操作，并可扩展支持掩码防护下的中间值计算。

### 参数说明

-   **`p`** (`int`)
     明文字节（8 位，范围 `0x00` ~ `0xFF`）。
-   **`k`** (`int`)
     密钥猜测字节（8 位，范围 `0x00` ~ `0xFF`）。
-   **`mask`** (`int`, 可选, 默认: `-1`)
     掩码值：
    -   若为 `-1`：表示**无掩码模式**，仅计算 `SBox[p ^ k]`
    -   若为其他值（如 `0x5A`）：表示使用的掩码字节，需配合 `mask_scheme` 使用
-   **`mask_scheme`** (`str`, 可选, 默认: `None`)
     掩码方案类型：
    -   `None` 或未指定：使用无掩码路径
    -   `'bool'`：启用**布尔掩码**（Boolean Masking），返回 `SBox[p ^ k] ^ mask`
    -   其他值：当前未实现，忽略

### 返回值

-   `int`

    计算得到的中间值字节，范围 0x00~0xFF，具体如下：

    -   无掩码时：`AES_Sbox[p ^ k]`
    -   布尔掩码时：`AES_Sbox[p ^ k] ^ mask`

~~~python
from mlscat.utils import get_mid

# 无掩码情况
p, k = 0x32, 0x2B
mid_unmasked = get_mid(p, k)

# 布尔掩码情况
mask = 0x5A
mid_masked = get_mid(p, k, mask=mask, mask_scheme='bool')
~~~



## get_tatgets

**mlscat.utils.get_targets(plaintexts, mask=-1)**

生成**中间值标签矩阵**（Targets Matrix），用于侧信道分析中的模板构建、CPA、DPA 或机器学习攻击。

该函数为每条明文和每个可能的密钥字节（0–255）计算对应的中间值（如 `SBox[plaintext ^ key]`），输出形状为 `(N, 256)` 的二维数组，其中每一行表示该迹在 256 个密钥猜测下的中间值类别。

支持无掩码和**布尔掩码**（Boolean Masking）两种模式。

### 参数说明

-   **`plaintexts`** (`numpy.ndarray`, 形状: `(N, 1)` 或 `(N,)`)
     明文数据数组，包含 `N` 条明文。
    -   每个元素为一个字节（`0x00` ~ `0xFF`）
    -   示例：`plaintexts[i][0]` 表示第 `i` 条迹的明文字节
-   **`mask`** (`int` 或 `numpy.ndarray`, 可选, 默认: `-1`)
     掩码控制参数：
    -   若 `mask == -1` 或为整数：表示**无掩码模式**，仅计算 `SBox[plaintext ^ key]`
    -   若 `mask` 是长度为 `N` 的一维数组（如 `np.ndarray`, shape: `(N,)`）：表示每条迹使用的**布尔掩码值**，返回 `SBox[plaintext ^ key] ^ mask[i]`

### 返回值

-   **`targets`** (`numpy.ndarray`, 形状: `(N, 256)`, 类型: `int64`)
     中间值标签矩阵，用于后续攻击：

    -   `targets[i][k]` 表示在第 `i` 条能量迹、密钥猜测为 `k` 时，S 盒输出的中间值（或其掩码后值）
    -   可作为 `TemplateAttack`、`CPA` 等算法的输入标签

    

~~~python
import numpy as np
from mlscat.utils import get_targets

# 假设已有明文数据
plaintexts = np.array([[0x32], [0x45], [0x98], [0x12]]) 

# 情况1：无掩码
targets_unmasked = get_targets(plaintexts)
print("Unmasked targets shape:", targets_unmasked.shape)  
print("First row (key=0~5):", targets_unmasked[0][:6])   

# 情况2：带布尔掩码（每条能量迹一个掩码）
masks = np.array([0x5A, 0x3F, 0x11, 0x7E])  
targets_masked = get_targets(plaintexts, mask=masks)
print("Masked targets shape:", targets_masked.shape)     
print("Masked first row:", targets_masked[0][:6])        
~~~

