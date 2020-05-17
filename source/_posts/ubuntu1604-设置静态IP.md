---
title: ubuntu1604 设置静态IP
top: false
cover: false
toc: true
mathjax: false
date: 2020-04-30 15:29:44
password:
summary:
categories: Linux
img:
keywords: IP 静态IP Ubuntu
tags:
	- IP
	- 静态IP
	- Ubuntu
---

1. 首先确保是NAT连接模式

2. 在VMware Workstation的“编辑”选项找到虚拟网络编辑器，在虚拟网络编辑器面板中找到NAT设置，获取子网掩码和网关。

	![](https://cdn.jsdelivr.net/gh/zhishuangR/myImg@master/md/20200430154842.png)

3. 在虚拟机中打开终端，输入命令`ifconfig`查看网卡信息

	![](https://cdn.jsdelivr.net/gh/zhishuangR/myImg@master/md/20200430160112.png)

4. 输入命令`sudo gedit /etc/network/interfaces`手动配置网络

	![](https://cdn.jsdelivr.net/gh/zhishuangR/myImg@master/md/20200430161332.png)

5. 输入命令`reboot`重启虚拟机

6. 重启后输入命令`ifconfig`可看到IP地址变成了我们手动设置的IP

	![](https://cdn.jsdelivr.net/gh/zhishuangR/myImg@master/md/20200430161920.png)

7. 改好了呢