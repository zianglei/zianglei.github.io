---
title: 配置教程 | 树莓派安装和配置"华为E8372"4G网卡
date: 2018-06-10 21:44:00
tags:
  - 配置教程
  - 树莓派
  - 反对无脑教程
categories:
  - [配置教程,树莓派]
toc: true
---

由于最近需要保证树莓派能够连接4G网，又能建立局域网，所以就想到了了4G无线网卡。在JD上选了一款[华为的E8372无线4G上网卡](https://item.jd.com/5148678.html)，在根据网上的教程配置的过程中发现有一些出入和不足，所以写一个完整的教程

## 预备知识
市面上的USB 4G无线上网卡都具有两种模式，一种是存储模式，即插上电脑就是一个U盘，一般里面存的是驱动程序；另一种模式是网卡模式，即插上电脑之后电脑就能通过它上网。由于市面上的无线网卡都有Windows驱动，所以插上电脑就自动安装驱动，自动切换到网卡模式，而由于Linux上没有对应的驱动，所以无线网卡无法自动切换模式，所以需要我们手动切换。

## 切换步骤
切换模式需要的软件是usb-modeswitch

首先确认树莓派是否识别到网卡
```bash
lsusb
```
结果类似下图
![](http://p3jggzq4i.bkt.clouddn.com/2018-06-10-19-15-44.png)

可以看到第一条就是我们的网卡，前面的ID为12d1:1f01

查看磁盘的ID号
```bash
ls -al /dev/disk/by-id/
```

结果如下
![](http://p3jggzq4i.bkt.clouddn.com/2018-06-10-21-18-26.png)

可以看到usb网卡的ID号为sr0，挂载sr0并查看里面的内容可以发现存储的是windows上的驱动程序，证明usb网卡现在是存储模式

挂载之后可以看到里面的内容
```bash
sudo mount /dev/sr0 /mnt
```

结果如下
![](http://p3jggzq4i.bkt.clouddn.com/2018-06-10-21-20-49.png)

安装usb-modeswitch，会自动安装对应的依赖usb-modeswitch-data（有的教程会让单独安装这个，其实不用）
```bash
sudo apt-get install usb-modeswitch -y
```

然后根据[官方的文档](http://www.draisberghof.de/usb_modeswitch/#usage)，我们需要修改`/lib/udev/rules.d/40-usb_modeswitch.rules`,添加我们的型号，让usb-modeswitch能够识别我们的usb网卡并自动转换模式
**注：在这个文件中已经包含了很多网卡型号，在修改文件之前可以先重启树莓派，使用`lsusb`查看usb网卡的ID是否发生了变化，如果发生变化，则不用修改该文件，网卡已经可以正常使用，可以跳过后面的步骤了**

打开该文件，然后加入
`ATTRS{idVendor}=="12d1", ATTRS{idProduct}=="1f01", RUN+="usb_modeswitch '%b/%k'"
`
此处以E8372为例，如果是其他的网卡，则需要将`12d1`替换为之前`lsusb`看到的ID中冒号前面的部分，将`1f01`替换为冒号后面的部分。

加入之后的文件如图

![](http://p3jggzq4i.bkt.clouddn.com/2018-06-10-21-32-53.png)

然后重启树莓派，使用`lsusb`命令查看ID是否发生变化
![](http://p3jggzq4i.bkt.clouddn.com/2018-06-10-21-34-01.png)

然后再使用`ifconfig`命令查看是否多了一个网口，在此处是多了一个eth1

![](http://p3jggzq4i.bkt.clouddn.com/2018-06-10-21-35-05.png)

到此就表明usb网卡已经配置成功，可以通过usb网卡愉快地网上冲浪啦~

**注：之前在网上查到的教程在将usb网卡的模式转换了之后会出现多个/dev/ttyUSB，然后还需要安装拨号软件进行手动拨号，而我使用的这个网卡并没有出现/dev/USB,并且可以直接上网。猜测原因为这个USB网卡已经自动完成了拨号上网的操作。**

如果在上述步骤配置完成后出现了多个/dev/ttyUSB（通过`ls /dev/tty*`命令查看），可以参考下面的“参考资料”的第二个链接进行进一步的配置。

参考资料：
[1] https://www.raspberrypi.com.tw/tag/usb_modeswitch/
[2] https://zhoujianshi.github.io/articles/2017/Linux%20%E4%BD%BF%E7%94%A8USB%203G%E4%B8%8A%E7%BD%91%E5%8D%A1%EF%BC%88wvdial%E7%9A%84%E4%BD%BF%E7%94%A8%EF%BC%89/index.html


