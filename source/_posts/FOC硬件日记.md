---
title: FOC硬件日记（正在更新）
date: 2024-01-20 23:33:48
tags: 
categories: FOC学习
toc: true
---

# 1.20

硬件设计，参考STC的无感BLDC方案，但是主控更换为STM32，三相逆变器采用FD6288Q，使用mos桥方式支持大电流（考虑小电流drv8313方案，只支持2A电流，仍然需要加mos驱动，考虑成本选择FD6288Q）。考虑加入电流环，电流检测采用INA199A1DCKR。下图为三相逆变电路及电流检测：

![image-20240120234842699](http://wochaoaidahaide.oss-cn-beijing.aliyuncs.com/img/image-20240120234842699.png)

存在验证的问题：

问题1：FD6288官方手册外围电路中需要增加自举电阻，参考开源方案并没有加入自举电阻，

问题2：电流检测放在半桥的下桥接地，理论上放在哪儿无影响

问题3：电流检测压差采用分压电阻得到1.65V，参考STM32F103C的FOC方案设计，C系列无VREF

---

单片机选型使用STM32F103ZET6（理论上做6路FOC电机都没问题，大炮打蚊子），最初选型考虑STM32G和STM32F4系列，最终由于价格和学习基础理论，采用ZET6（主要因为手头有剩的）。单片机VREF采用3.3V，可能会出现精度问题，暂不考虑。电源设计如下：

![image-20240120235458613](http://wochaoaidahaide.oss-cn-beijing.aliyuncs.com/img/image-20240120235458613.png)

电路中5V仅作为FD6288Q芯片供电，为了电路简单，使用AMS1117（手头有剩的），MCU和其他电路的3.3V区分，分别使用两个RT9013稳压。完整电路还包括显示，串口，交互按键，暂未完成。

# 1.22

参考柠檬FOC项目，又看到开源博主说FD6288的最低供电为12V，但是看芯片手册输入电压为4~24V，原STC无感电路设计采用5V，电机高速长时间运行时，ams1117处于温热状态，考虑更换为buck电路：

![image-20240122230156017](http://wochaoaidahaide.oss-cn-beijing.aliyuncs.com/img/image-20240122230156017.png)

FD6288Q仍然使用5V供电。

# 1.24

完成原理图设计和PCB大致布局，完善USB串口，交互按键和LED指示灯。完整原理图如下：

![image-20240124210149669](http://wochaoaidahaide.oss-cn-beijing.aliyuncs.com/img/image-20240124210149669.png)

![image-20240124210213483](http://wochaoaidahaide.oss-cn-beijing.aliyuncs.com/img/image-20240124210213483.png)

电机接口考虑使用铜皮开窗，使用香蕉头和电机连接，或直接焊接。
