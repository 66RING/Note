---
title: communication andr networking笔记
date: 2021-03-24
tags: 
- network
mathjax: true
---


# 数据和信号

## 模拟与数字

- 模拟数据：连续状态的信息
- 数字数据：离散状态的信息


- 复合信号
    * 单一正弦波在数据通信中没有作用，我们需要发送复合信号，复合信号由许多简单正弦波组成
    * **基础频率或第一谐波**：频率与复合信号一样的正弦波
    * **第N谐波**：频率是基础频率的N倍

- 带宽：**复合信号的带宽是信号最高频率和最低频率的差值**

- 数字信号：
    * 一个有L个电平的信号可以表示$log_2L$个位

如有4个电平，可以表示2个位：

```
电平
4    11
3    10
2    01
1    00
```

- **比特率**：一秒钟发送的位数，单位：**每秒位(bps)**
- 位长：一个位在传输介质上的距离
    * 位长=传播速度x位持续时间


### 数字信号的传输

数字信号是无穷大带宽的复合模拟信号。两种方法传输数字信号：基带传输和宽带传输

#### 基带传输

todo p59

基带传输通过通道发送数字信号，不转换成模拟信号


#### 宽带传输

todo

宽带传输或调制把数字信号转变为模拟信号。调制允许我们使用 **带通通道** ，即带宽不从0开始的通道。


## 传输减损

- 衰减
    * 克服介质的阻抗需要放热，能量损失，需要使用信号放大器放大
    * **分贝(dB)** 用于计算两种信号或同一信号在不同位置直接的相对强度
    * 功率表示：$dB = 10 \log_{10} \frac{P2}{P1}$
    * 用电压表示：$dB = 20 \log_{10} \frac{V2}{V1}$
- 失真
    * 不同频率成分在介质中有自己从传播速度，在终点时有各自的延迟
- 噪声
    * **信噪比SNR**
    * $SNR = \frac{平均信号功率}{平均噪声功率}$ 是功率比
    * 用分贝描述：$SNR_{dB} = 10 \log_{10} SNR$


## 数据速率限制
### 无噪声通道：奈奎斯特比特率

对于无噪声通过，理论上最大比特率：

$$比特率 = 2 \times 带宽 \times \log_2 L$$

但是如果电平数L很大，接收方会很难区分不同电平，从而削弱系统可靠性。**电平数应该是2的幂** 


### 噪声通道：香农容量定理

噪声通道理论上的最高数据传输速率：

$$通道容量 = 带宽 \times \log_2 {(1+SNR)}$$

无论使用多少个电平，都不可能获得比通道容量更高的数据速率


### 两者结合

todo

**香农容量定理给出了数据速率的上限，奈奎斯特公式给出所需的电平数** 

eg: 


## 性能

- 带宽
    * 以赫兹衡量：即复合信号的频率范围，即通道能通过信号的频率范围
    * bps衡量：每秒发送的位数
    * todo  p70

- 吞吐量
    * 吞吐量是发送速率的**实际衡量值**，带宽是链路的潜在衡量值


- 延迟 = 传播时间+传输时间+排队时间+处理时间
    * 传播时间 = 距离 / 传播速度
        + 一个位从源到目的的用时
    * 传输时间 = 报文长度 / 带宽
        + 从报文第一个位到达到最后一个位到达的用时
    * 排队时间
- 带宽与延迟的乘积 = 充满链路的位的个数 todo p72


# 数字传输

- 数字数据：存储在计算机中的数据
- 数字信号：数据在链路中传播的形式


## 线路编码

将数字数据转换为数字信号的过程

- 数据元素：位
- 信号元素：数据元素的载体，我们传输不会一次只传输一位(电平L能传$log_2 L$个位)
- 比率r：每个信号元素承载的数据元素的数量






























