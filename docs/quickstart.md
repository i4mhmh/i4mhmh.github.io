# Quickstart

Mlscat 快速上手指南

>   First of all

~~~python
import mlscat as cat
~~~



## 获取中间值

若要直接获取中间值，分为已知掩码以及未知掩码两种情况。

~~~python
# 掩码未知


~~~



## CPA

若要尝试使用Mlscat进行一次CPA攻击，仅需通过以下几行代码。

假设攻击的密钥字节`byte_num`为`2`，

~~~python
# 掩码未知

# plaintexts.shape=[m,16]	 AES-128
# traces.shape = [m,n] 

byte_num = 2
cpa(byte_num, plaintexts, traces)
~~~

