# 泄露模型

通常，泄露模型的选择往往会极大程度上影响后续攻击的结果，具体来讲，当泄露模型对目标中间值信息 泄露特征刻画精度较高时，由刻画误差带来的噪声水平较低，一定程度上能减少攻击人员成功恢复密钥需要的能量迹数目。

>   为了统一代码风格，所有涉及转换的方法的输入均以数组级进行操作

## HW function

~~~python
cat.leakage.hw(mid: list[int]) -> list[int]
~~~

返回输入中间值对应的汉明重量.

**参数**

*   mid: 输入的十进制中间值数组

**Example**

~~~python
>>> result = cat.leakage.hw([191, 221, 13, 16])
>>> result
[7, 6, 3, 1]
~~~



## lsb function !暂未实现

~~~python
cat.leakage.lsb(mid: list[int]) -> list[int]
~~~

返回中间值的最低位



