---
layout: post
title: "axi dma Linux使用"
date: 2022-03-16
description: "Linux 关于 Xilinx官方的相关东西"
tag: Linux Xilinx
---

公司项目，需要使用zynq读取sd卡的bin文件内容，然后通过运行在linux系统上的main.c函数将数据通过axi dma写入fpga交给fpga进行处理。因此这里就具体说明以下axi dma的用法。

## 前言

这篇文章主要是讲了如何进行读/写数据去DDR通过Linux 用户空间在zynq开发板上。



## DMA

直接内存访问，或称为 DMA，是嵌入式开发的一个重要方面，因为它是一种在不占用 CPU 的情况下访问嵌入式系统的主内存（通常是 DDR）的方法，因此在读取期间将其打开以执行其他操作/将循环写入内存。 

DMA 仅允许处理器启动传输以读取/写入主系统内存，然后它会生成一个中断，指示传输完成。这使处理器可以自由地执行其他任务，直到中断调用其各自的服务程序。 虽然 DMA 的概念很简单，但要想全面了解如何在具有嵌入式处理器（例如 Xilinx Zynq SoC 中的 ARM 核心处理器）的 FPGA 上实现它，却是一项不小的壮举。

然后再增加一层额外的复杂性，即如何从 Linux 等操作系统的用户空间访问它，只会让情况变得更糟。 

因此，对于这个项目，我将首先介绍在 Vivado 中获取硬件设置的机制，以及根文件系统中内核驱动程序和 Linux 应用程序的 PetaLinux 项目设置中的基本设置。然后，我将介绍 Linux 应用程序中的代码实际上是如何完成控制 FPGA 可编程逻辑中的 AXI DMA 以执行传输的任务的功能。 



## 在petalinux添加DMA调试内核驱动程序

一般情况下，Xilinx AXI DMA内核驱动程序在Petalinux项目中启用，位于 Device Driver > DMA Engine support > Xilinx DMA Engines。

这将可以在Linux应用程序中用于查看内存空间的内存映射驱动程序（/dev/mem 从用户空间 和 CONFIG_DEVMEM 在内核空间）也在Petalinux项目中默认启用。所以你不需要为项目配置内核，但是我们可以去看一看内核DMA选项并通过“help”来获得每个信息的详细说明。

用一个用于DMA的测试客户端，可以选择作为模块内置到内核中，该模块允许使用modprobe命令对DMA驱动进行内核级测试，以便在需要的时候手动加载它。在 Device Drivers > DMA Engine suppport下，DMA Test client位于底部，可以使能它。这里就不介绍如何运行DMA测试客户端了，但这里[有一个很好的方法](https://www.kernel.org/doc/html/v4.15/driver-api/dmaengine/dmatest.html)。



## 如何使用Linux用户空间的DMA

这里可以通过自己设置的vivado工程来具体看一看使用DMA的关键。所有事情都是通过设置MM2S和S2MM通道的控制寄存器中的相关位并通过读取其各自状态寄存器中的位值来监控它们的当前状态来发生的。（每个通道都有自己的、独立的控制和状态寄存器）。通过AXI Lite接口在AXI DMA中访问控制和状态寄存器。

T AXI Lite接口将控制和状态寄存器的相关值**读/写到系统内存中的物理地址**，该地址在Vivado中的AXI DMA的地址编辑器中指定。如下图

![](/images/axi_dma/0.png)/我们可以看到这个项目的控制和状态寄存器的值将从物理地址0x40400000开始放置。每个特定的控制和状态寄存器值都将具有与该基地址的专用偏移量。例如，MM2S通道提取传输的源数据的地址位于偏移量0x18（下面将会继续解释)。这意味着Linux应用程序需要将源数据写入DDR中的物理地址0x40400018。

将axi dma配置成直接寄存器模式，这意味着给定长度的数据的传输将通过主系统内存中的源地址传输到目标地址MM2S和S2MM DMA通道。

Xilinx的PG201中针对AXI DMA IP的表2-6，列出了Direct Register模式下要读写的相关控制和状态寄存器：

![](/images/axi_dma/1.png)

MM2S和S2MM通道的源地址寄存器，目标地址寄存器和传输长度寄存器很简单。因为它们只是具有写入或读取的给定值。然而，特定的控制和状态寄存器具有专用于某些状态的位。

![](/images/axi_dma/2.png)

## MM2S寄存器

对于DMA MM2S通道的控制寄存器，直接寄存器模式下AXI DMA的基本功能需要查看的主要位如下：

- bit 0 = run(1)或stop(0)	DMA MM2S通道。
- bit 2 = 设置1为软复位DMA MM2S通道。
- bit 12 = 设置为1 启动DMA MM2S通道完成时中断（IOC）标志。

然后在DMA MM2S通道的状态寄存器中监视以下位：

- bit 0 = DMA MM2S通道运行时设置为0，通道停止时设置为1.
- bit 1 = 当DMA MM2S通道不空闲时设置为0，这意味着在直接寄存器模式下传输尚未完成。通道空闲时设置为1，表示传输已经完成且DMA控制器暂停。
- bit 12 = 如果启用（控制寄存器中的bit12设置为1），当传输完成时，它将读出为1，并且AXI DMA的MM2S中断输出（mm2s_introut)也将变成高电平。

![](/images/axi_dma/3.png)

然后在DMA S2MM通道的状态寄存器中监视以下位：

- bit 0 = DMA S2MM通道运行时设置为0，通道停止时设置为1.
- bit 1 = 当DMA S2MM通道不空闲时设置为0，这意味着在直接寄存器模式下传输尚未完成。通道空闲时设置为1，表示传输已经完成且DMA控制器暂停。
- bit 12 = 如果启用（控制寄存器中的bit12设置为1），当传输完成时，它将读出为1，并且AXI DMA的S2MM中断输出（s2mm_introut)也将变成高电平。

## AXI进行传输

现在查看DMA控制器接口中哪些寄存器是相关的，下面是使用AXI DMA完成传输的步骤：

1. 通过将1写入MM2S（偏移量0x00）和S2MM（偏移量0x30)控制寄存器的bit2 来复位DMA

2. 通过将0写入MM2S（偏移量0x00）和S2MM（偏移量0x30)控制寄存器的bit0确保DMA停止

3. 通过将1写入MM2S（偏移量0x00）和S2MM（偏移量0x30)控制寄存器的bit14来启用完成中断（IOC）标志。

4. 将MM2S通道要读取的数据在DDR中的源地址写入MM2S DMA源地址寄存器。

   ![](/images/axi_dma/4.png)

5. 将S2MM通道要写入的数据DDR中的目标地址写入S2MM DMA目标地址寄存器。

![](/images/axi_dma/5.png)

6. 通过将1写入MM2S控制寄存器的bit 0（偏移量来运行DMA MM2S）通道。
7. 通过将1写入S2MM控制寄存器的bit 0（偏移量0x30）来运行DMA S2MM通道。
8. 通过将要发送到MM2S传输长度寄存器的总字节数的值写入MM2S通道的传输长度。![](/images/axi_dma/6.png)
9. 通过将要读入S2MM通道内存的总字节数的值写入S2MM缓冲区长度寄存器，写入S2MM通道的缓冲区长度。!![](/images/axi_dma/7.png)
10. 监控MM2S和S2MM通道状态寄存器中的IOC标志（bit 12）和空闲标志（bit 1）。一旦读出这两个位设置为1，则传输完成。

在Linux用户空间中，DDR中源数据位置的虚拟地址、DDR中数据的目标地址以及DMA控制器读取/写入到DDR中的位置是所有通过内存映射驱动程序映射到虚拟地址。通过字符设备文件/dev/mem，内核能够访问DDR中的物理内存地址。

然后，AXI DMA通过控制接口，MM2S DMA通道接口和S2MM DMA通道接口直接访问DDR。AXI DMA从指定源地址将数据送出DDR，并将内存映射数据转化格式，然后在AXI Stream MM2S端口上输出到AXI Stream Data FIFO的输入。然后，AXI DMA在其AXI Stream Data FIFO输出返回的数据，将数据的格式转换成内存并写入DDR中的指定目标地址。



