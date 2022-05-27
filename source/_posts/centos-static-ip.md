---
layout: post
title: "CentOs 设置静态IP方法"
date: 2016-04-12
excerpt: "在做项目时由于公司局域网采用自动获取IP的方式，导到每次服务器重启主机IP都会变化。为了解决这个问题，参考了很多资料都没有效果，为了以后还会出现这种问题，所以将整个过程记录下来。"
categories: 
- Linux
tags: 
- Linux
comments: true
---

在做项目时由于公司局域网采用自动获取IP的方式，导到每次服务器重启主机IP都会变化。为了解决这个问题，我参考了

## 修改网卡配置　
编辑：vi /etc/sysconfig/network-scripts/ifcfg-eth0

````
DEVICE=eth0 
ONBOOT=yes 
HWADDR=22:78:e3:3e:11
BOOTPROTO=static 
IPADDR=192.168.56.103 
NETMASK=255.255.255.0 
BROADCAST=192.168.56.255
GATEWAY=192.168.56.0
````
DEVICE=eth0 #描述网卡对应的设备别名，例如ifcfg-eth0的文件中它为eth0  
BOOTPROTO=static #设置网卡获得ip地址的方式，可能的选项为static，dhcp或bootp，分别对应静态指定的 ip地址，通过dhcp协议获得的ip地址，通过bootp协议获得的ip地址  
BROADCAST=192.168.56.255 #对应的子网广播地址  
HWADDR=00:07:E9:05:E8:B4 #对应的网卡物理地址  
IPADDR=192.168.56.103 #如果设置网卡获得 ip地址的方式为静态指定，此字段就指定了网卡对应的ip地址  
NETMASK=255.255.255.0 #网卡对应的网络掩码  
## 修改网关配置
编辑：vi /etc/sysconfig/network　增加 GATEWAY 配置  
NETWORKING=yes(表示系统是否使用网络，一般设置为yes。如果设为no，则不能使用网络，而且很多系统服务程序将无法启动)  
HOSTNAME=centos(设置本机的主机名，这里设置的主机名要和/etc/hosts中设置的主机名对应)  
GATEWAY=192.168.56.103(设置本机连接的网关的IP地址。)  
我在修改这里打开编辑时前三项已经默认有了所以只增加了GATEWAY  
重启网络服务  
执行命令：  

````
service network restart
````
## 遇到的问题
>错误：弹出界面eth0: 错误：没有找到合适的设备：没有找到可用于链接System eth0 的
清除/etc/udev/rules.d/70-persistent-net.rules 里的内容，重启机器，会生成新的内容
文件下记录着网卡对应mac地址信息,将 mac 地址拷贝到ifcfg-eth0的HWADDR，重新启动网卡，成功。
    
>错误：设备 eth0 似乎不存在, 初始化操作将被延迟  
与上面的问题一样，重新操作一遍即可













