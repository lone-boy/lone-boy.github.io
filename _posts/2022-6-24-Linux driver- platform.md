---
layout: post
title: Linux driver-platform-pwm
data: 2022-06-15
description: "Linux驱动学习"
tag: Linux驱动
typora-root-url: ..
---
之前学习Linux的很多记录都没了，现在通过新的方式把学习的东西记录下来。从驱动的框架包括很多子系统的学习都会记录。这里尝试写一个自己的驱动程序用来控制自己的Vivado IP核模块。

## 前言

在Vivado软件中，我们可以通过创建和封装IP核的方式来自定义IP核，支持将当前工程和工程中的模块封装，当然也可以创建一个带有AXI4接口的IP核，用于PS和PL端之间的数据通信。这一次的系统框图如下：

![image-0](/images/linux/image-0.png)

这里，Breath LED IP核是自定义的IP核，PS通过AXI接口可以为LED IP模块配置从而控制PL的LED灯。



## 硬件设计

1. 创建一个新的IP核，在Manage IP中选择创建新的IP核工程，创建完后，开始创建打包IP核，这里需要选择一个带有AXI4接口的IP核。

2. 开始编辑IP核。主要是完成呼吸灯的配置，然后呼吸灯可以有寄存器控制，主要是呼吸灯的开关状态，使能状态，以及步长可以设置。

   | .sw_ctrl    | .set_en      | .set_freq_step |
   | ----------- | ------------ | -------------- |
   | slv_reg0[0] | slv_reg1[31] | slv_reg[9:0]   |

   寄存器0的最低位控制呼吸灯的开关状态，寄存器1的最高位数据是呼吸灯频率的设置有效信号(set_en)，寄存器地址1对应数据的低10位控制呼吸灯频率的步长（set_freq_step)。
