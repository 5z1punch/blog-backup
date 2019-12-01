---
title: '施耐德plc开发速成——21天从入门到炸掉核电厂[3]'
date: '2016-02-08T00:00:00.000Z'
categories: industrial_control
tags:
  - industrial
  - program
---

# 2016-2-8-施耐德plc开发速成——21天从入门到炸掉核电厂\[3\]

接下来学习一下`Modbus协议`。 主要讲的是我在工作中经常用到的`TCP/IP-Modbus`。 MODBUS 是 OSI 模型第 7 层上的应用层报文传输协议,它在连接至不同类型总线或网络的设备之间提供客户机/服务器通信。

### 1. 总体描述：

MODBUS 协议定义了一个与基础通信层无关的简单协议数据单元\(PDU\)。特定总线或网络上的 MODBUS 协议映射能够在应用数据单元\(ADU\)上引入一些附加域。

上图为通用modbus帧。 如果请求正确且成功传输并执行，则返回操作码和数据相应，如果出错，则返回差错码和异常码。

### 2. 数据模型：

### 3. TCP/Modbus 帧的构成：

#### 3.1 MBAP Header :

在帧的开头是一个7个字节的协议头，MBAP\(Modbus Application Header\) 头，这个头由以下几部分组成： 1\) Transaction Identifier \(2 bytes\): 事务标识符，2 个字节，这是由客户端指定的，用来唯一标识数据包的id号。由于网络和服务器链路的原因，这些请求可能不会按照发出的顺序先后到达。 2\) Protocol Identifier \(2 bytes\) : 协议标识符，2 个字节，在modbus服务中，这个字段始终为0x0000，其他值预留给以后的扩展使用。 3\) Length \(2 bytes\): 数据长度，2 个字节， 这个长度是紧随在此字段后的，该包的剩余字节数。高位补零。 4\) Unit Identifier \(1 byte\): 单元标识符，1 个字节，这个字段由客户端设置。其目的是在串行链路通信中，与服务器通过网关或者网桥连接时，目的ip的地址标识了网桥本身的地址，而unit identifier这个字段是在该网关的串行链路子网中标识从站的地址。 分配串行链路上MODBUS从站设备地址为1~247\(10进制\) ，地址0作为广播地址。 如果在tcp/ip网络上进行通讯，这个字段是没有意义的，并且为了放置ip自动分配造成的麻烦，通常要设置为0xff 或者 0x00 ， 均表示 该字段无效。

#### 3.2 请求实体 -&gt; 简单协议数据单元\(PDU\):

1\) function code \(1 byte\): 功能码，1个字节，表示要执行的操作。 一共有三类功能码： a\) 公共功能码： 被良好定义，保证唯一，公开证明，具有可用一致性测试的功能码，也就是官方功能码，只有modbus维护组织可以改变。 b\) 用户定义功能码：用户可自己定义，不能保证唯一。 c\) 保留功能码：一些公司对传统产品通常使用的功能码,并且对公共使用是无效的功能码。 三种功能码的定义范围： 

2\) 数据请求 ，长度不定。

### 4. 功能码描述：

#### 4.1 常用公共功能码：

#### 4.2 完整功能码描述：

参考[modbus协议中文完整版](http://wenku.baidu.com/link?url=Gbb3q3qraz6XcGsWHs_i6mXjuum9y1uS2kd3-UEnCnTBXfN6q1xYKT2yWpazEgLFqeubyzL6tun_5vRQ1DbQuMC8gDt7HyegX3MUX2sqQ1C) 的第一部分:`Modbus 协议` 的第6小章`功能码描述`

### 5. 参考文献：

[simplymodbus-TCP](http://www.simplymodbus.ca/TCP.htm) [intro\_modbustcp](http://www.prosoft-technology.com/kb/assets/intro_modbustcp.pdf) [modbus协议中文完整版](http://wenku.baidu.com/link?url=Gbb3q3qraz6XcGsWHs_i6mXjuum9y1uS2kd3-UEnCnTBXfN6q1xYKT2yWpazEgLFqeubyzL6tun_5vRQ1DbQuMC8gDt7HyegX3MUX2sqQ1C)

## To Be Continue

#### By TheSeven

