---
title: 低功耗蓝牙 | 简单介绍
date: 2018-03-30 01:03:52
tags: 低功耗蓝牙
categories: 学习笔记
toc: true
---

由于最近的项目中需要用到使用低功耗蓝牙的设备，所以学习一下
<!--more-->

## 低功耗蓝牙历史
低功耗蓝牙（Bluetooth Low Energy, BLE) 的诞生是为了满足低功耗工作的场景，它既能够满足传输适量的数据的需求，功耗又比较低。  
低功耗蓝牙（有时又叫Bluetooth Smart）正式提出于**蓝牙4.0核心规范**，而最早是由诺基亚提出的，当时被称作"Wibree"，后来在2010年BSIG（Bluetooth Special Interest Group）就将Wibree合并到了原有的蓝牙规范当中，作为4.0核心规范的一部分。由于BLE并不能向前兼容原有的蓝牙协议，所以不能将BLE等同于蓝牙。

## 低功耗蓝牙特点
BLE的理论传输速度最高为是1Mbps，当然这肯定是理想条件下的，实际条件下传输速度一般为5-10kbps，典型的传输距离大概在2-5m。

## 低功耗蓝牙整体架构
BLE整体架构主要由三部分组成，应用（Application）、主机（Host）和控制器（Controller）。  
应用指的就是用户创建的带有与蓝牙协议交互的接口的应用；主机覆盖了蓝牙协议栈的上层；控制器覆盖了蓝牙协议栈的下层。**主机**可以通过HCI（Host Controller Interface)与控制器进行通信，顾名思义，HCI就是Host与Controller进行通信的接口，有了HCI，Controller就可以与很多Host进行交互，只要Host满足HCI的相关规范。  

Host包含了以下层级：
- Generic Access Profile (GAP)
- Generic Attribute Profile (GATT)
- Logical Link Control and Adaptation Protocol (L2CAP)
- Attribute Protocol (ATT)
- Security Manager (SM)
- Host Controller Interface (HCI), Host side  

Controller包含了以下层级：
- Host Controller Interface (HCI), Controller side
- Link Layer (LL)
- Physical Layer (PHY)

下面将分别介绍一些层的功能
### Physical Layer （PHY）
PHY层包含了模拟通信电路，用来调制和解调模拟信号，并将模拟信号转化为数字信号。BLE可以使用40个通道来进行传输，其中37个通道用于传输连接数据，而最后三个通道用于传输广播数据。  
BLE使用跳频技术，在每次连接事件到达的时候，BLE的传输信号就会通道之间进行“跳跃”，也就是选择不同的信号传输频率。而选择的通道值会在连接建立的时候在两个设备之间进行传输。

### Link Layer
Link Layer是与PHY层直接通信的层。Link Layer定义了以下角色：
- 广播者（Advertiser）：发送广播包的设备
- 扫描者（Scanner）：扫描广播包的设备
- 主设备（Master）：初始化连接并管理连接的设备
- 从设备（Slave）：接收连接请求并且跟随主设备的时序的设备

同时Link Layer也定义了**蓝牙设备地址**，一个48位的唯一标识符，类似于IP协议中的MAC地址。  
Link Layer也负责连接的建立、连接间隔的管理和数据的加密

### Host Controller Interface (HCI)
HCI允许CPU来通过串口（UART或者USB）来控制BLE设备。经典的应用场景就是电脑或者手机作为Host，然后一个小的蓝牙适配器作为Controller，通过USB插到电脑上。

### ATT
ATT是一个简单的C/S（Client/Server)协议，一个用户端向服务端发起请求，然后服务端向客户端发送数据。在蓝牙协议中，当一个请求还在处理的时候，服务端无法接收其他的请求。服务端的数据在蓝牙协议中被称之为attribute，每一个attribute都含有一个16位的attribute handle、一个UUID、一组权限和数据值。handle用于获取数据的标识符，标识是哪个数据，UUID用来指定数据的类型和性质。  

在ATT协议中，当用户端需要读写数据，在发送请求的时候需要带上对应attribute的handle值，服务端依照响应的操作类型返回数据或者返回响应结果。

### L2CAP
L2CAP层有两个功能，一个是将上层协议数据封装成标准的BLE格式或者从BLE包中解包，另一个是分段和重组，即将很大的包划分成多个段，每一个段最大为27个字节。
L2CAP负责转发ATT和SMP协议，ATT是BLE中最基本的负责数据交换的协议，而SMP是负责在节点之间产生和分发安全密钥的协议。

### GATT
GATT是在ATT上层的一个协议，它定义了Server中的数据是如何组织和如果和在不同设备之间交换的。在GATT中，数据被划分为一个个**Service**，每个Service都有若干个**characteristics**，每一个characteristic都是一个数据和相应的描述信息的集合。比如，如果我们想要获取温度数据，那么对应的Characteristic就会包含一些metadata（比如，温度的单位是摄氏度还是华氏度，以及温度的值）。不同的Services具有不同的UUID，有一些通用的Services可以在蓝牙协议官方网站上找到对应的信息。

在GATT中，不同类型的Service被分成**GATT Profiles**，每个profile包含多个services。同时，characteristic也包含一个对应的UUID。

### GAP
GAP层负责控制广播和连接，它规定了设备如何执行设备的发现、连接、安全建立等操作


## 低功耗蓝牙网络拓扑
BLE设备可以担任两种角色，中心设备（Central Device）和外设（Peripheral Devices）。中心设备通常是手机或者电脑这些处理能力较强的设备，而外设通常为一些传感器或者低功耗设备。  

一个BLE设备可以发送两种类型的数据，广播包（Advertising Packets)和扫描响应包（Scan Response Data)。外设周期性发送广播包，使自己可以被其他设备发现，当其他设备接收到广播包的时候，它们可以发送扫描响应包来向发送广播包的设备请求数据。  

一个BLE设备只能使用两种方式与其他设备进行通信：广播（Broadcasting）和连接（Connections）。
### 广播
广播是将数据发送给所有正在监听数据的设备，当使用广播的时候，有两种角色：广播者（Broadcaster）和观察者（Observer）。广播者周期性的发送非连接性质的广播包给任何想要连接它的设备，同时观察者循环扫描区域以便接收广播包；当观察者接收到了广播包之后，就可以发送扫描响应包来请求数据。**广播是唯一一种设备可以一次向多个节点发送数据的方式**
![](http://p3jggzq4i.bkt.clouddn.com/gsbl_0103.png)
### 连接
连接是一种两个设备之间永久的，周期性的数据交换方式，**中心设备**扫描**连接性质**的广播包，在适当的时候初始化一个连接。当连接建立之后，中心设备就可以建立时序，进行周期性的数据交换。当连接建立之后，中心设备和外设通常会协商**连接事件**（connection event），一个连接事件在确定的一个时间点进行数据交互。这种通信方式是节省能耗的关键，两个设备可以进行启动、交换数据、然后休眠的过程，直到下一个连接事件到来。
![](http://p3jggzq4i.bkt.clouddn.com/gsbl_0104.png)


