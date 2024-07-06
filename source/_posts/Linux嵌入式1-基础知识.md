---
title: Linux嵌入式1-启动开发环境
date: 2024-01-14 11:16:38
tags:
categories: Linux嵌入式学习
toc: true
---

# 应用开发环境搭建：

开发板移植uboot：完成网络移植

服务器安装nfs和tftp

windows、服务器、开发板需要处在同一网段，使用虚拟需要添加网卡开启桥接模式，***关闭防火墙***

挂载zImage和dtb之前先使用nfs和tftp测试

完成配置后 uboot在emmc中，zImage和dtb使用tftp挂载，根文件系统使用nfs挂载

最后验证交叉编译工具



安装nfs过程出现的问题，挂载失败，检查是服务器nfs版本为4，uboot只支持2（原文链接：https://blog.csdn.net/qq_42212668/article/details/125250873）

![image-20231007210103881](http://wochaoaidahaide.oss-cn-beijing.aliyuncs.com/img/image-20231007210103881.png)

## 配置过程中的常用命令：

~~~shell
setenv ipaddr 192.168.1.50
setenv ethaddr b8:ae:1d:01:00:00
setenv gatewayip 192.168.1.1
setenv netmask 255.255.255.0
setenv serverip 192.168.1.253
saveenv

nfs启动文件系统：
setenv bootargs 'console=ttymxc0,115200 root=/dev/nfs nfsroot=192.168.1.107:/home/wujing/linux/nfs/alientrootfs,proto=tcp rw ip=192.168.1.50:192.168.1.107:192.168.1.1:255.255.255.0::eth0:off'

tftp挂载
setenv bootcmd 'tftp 80800000 zImage; tftp 83000000 imx6ull-14x14-emmc-4.3-800x480-c.dtb; bootz 80800000 - 83000000'
~~~



# 应用开发环境启动：

启动Ubuntu，mobaX连接开发板，检查Ubuntu的IP地址和开发板uboot中设置的tftp服务器地址是否一致，不一致使用以下命令修改：

```shell
setenv serverip 192.168.1.253
```

![image-20240114122139754](http://wochaoaidahaide.oss-cn-beijing.aliyuncs.com/img/image-20240114122139754.png)

**检查虚拟机网络设置，VMnet1为net模式，用来虚拟机上网，VMnet0为桥接模式，用来连接开发板挂载。**

检查无误后在uboot中输入boot启动



nfs挂载根文件系统目录 ：/home/wujing/linux/nfs/alientrootfs

应用程序源码存放目录：/home/wujing/Desktop/alitenk-test



常用命令：

```uboot
printenv #查看环境变量
boot #启动linux
```

使能Ubuntu环境变量

```shell
source /opt/fsl-imx-x11/4.1.15-2.1.0/environment-setup-cortexa7hf-neon-poky-linux-gnueabi
```



