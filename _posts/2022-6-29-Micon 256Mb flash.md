---
layout: post
title: "Micon 用户手册"
date: 2022-6-29
description: "Flash 手册"
tag: 芯片手册
typora-root-url: ..
---

## Micon 256Mb flash

### 简介

今天在一块板子上进行擦除flash操作时，发现里面东西擦不干净，而且读取的也有问题。所以，通过从用户手册来寻找原因。

256Mbit的flash就是32KB的大小。可以看下图，一共有511个扇区，一个大扇区有64KB，每个子扇区4KB。

![image-0](/images/chips/image-0.png)



![image-2](/images/chips/image-2.png)

### 状态寄存器

状态寄存器位可以使用读状态寄存器或写状态寄存器命令。当状态寄存器启用/ 禁用位（第 7 位）设置为 1 且 W# 驱动为低电平，状态寄存器非易失性位变为只读且不会执行写状态i寄存器操作。退出此硬件保护模式的唯一方法是驱动 W#为高电平。

![image-1](/images/chips/image-1.png)



### 命令定义

![image-3](/images/chips/image-3.png)

![image-4](/../../../.config/Typora/typora-user-images/image-20220629174317645.png)

![image-20220629174426892](/images/chips/image-4.png)

![image-20220629174515152](/images/chips/image-5.png)

![image-20220629174539502](/images/chips/image-6.png)





## Winbond(w25q128 jv)

### 简介

这款flash芯片似乎没有password功能。但是也可以做到保护作用。老样子，介绍一下这款芯片的参数，128Mbit flash，16Mbytes flash。一共有256个Block，每个Block是64KB，对应的地址就是000000h-FFFFFFh。

### 状态寄存器

|           S7            |       S6       |         S5         |         S4         |         S3         |         S2         |         S1         |        S0         |
| :---------------------: | :------------: | :----------------: | :----------------: | :----------------: | :----------------: | :----------------: | :---------------: |
|           SRP           |      SEC       |         TB         |        BP2         |        BP1         |        BP0         |        WEL         |       BUSY        |
| STATUS REGISTER PROTECT | SECTOR PROTECT | TOP/BOTTOM PROTECT | BLOCK PROTECT BITS | BLOCK PROTECT BITS | BLOCK PROTECT BITS | WRITE ENABLE LATCH | WRITE IN PROGRESS |

**S0：**只读位，当设备执行页编程，高速页编程，扇区擦除，块擦除，芯片擦除，写状态寄存器或编成安全寄存器指令，在此期间，设备忽略其它指令，知道擦除或写状态寄存器完成后，BUSY位会被清楚为0状态，表示器件已经准备好接受进一步指令。

**WEL：**只读位。执行一个写使能寄存器的时候，当器件被禁止写时，WEL状态为被清除为0。上电时或执行以下任何指令后会出现写禁止状态：禁止写、页编程、四页编程、扇区擦除、块擦除、芯片擦除、写状态寄存器、擦除安全寄存器和程序安全寄存器。

**Block Protect Bits（BP2，BP1，BP0）：**可写，块保护位（BP2、BP1、BP0）是状态寄存器（S4、S3 和 S2）中的非易失性读/写位，提供写保护控制和状态。 可以使用写状态寄存器指令设置块保护位（参见交流特性中的 t W）。 可以保护存储器阵列的全部、无或部分免受编程和擦除指令的影响（参见状态寄存器存储器保护表）。 块保护位的出厂默认设置为 0，没有阵列保护。

**Top/Bottom Block Protect (TB)：**可写，非易失性顶部/底部位 (TB) 控制块保护位 (BP2、BP1、BP0) 是否保护阵列的顶部 (TB=0) 或底部 (TB=1)，如状态寄存器中所示 内存保护表。 出厂默认设置为 TB=0。 根据 SRP、SRL 和 WEL 位的状态，可以使用写状态寄存器指令设置 TB 位。

**Sector/Block Protect Bit (SEC)：**可写，非易失性扇区/块保护位 (SEC) 控制块保护位 (BP2, BP1, BP0) 是保护顶部 (TB=0) 中的 4KB 扇区 (SEC=1) 还是 64KB 块 (SEC=0) 或数组的底部 (TB=1)，如状态寄存器存储器保护表中所示。 默认设置为 SEC=0。

**Complement Protect (CMP)：**可写，补码保护位 (CMP) 是状态寄存器 (S14) 中的非易失性读/写位。 它与 SEC、TB、BP2、BP1 和 BP0 位配合使用，为阵列保护提供更大的灵活性。 一旦 CMP 设置为 1，之前由 SEC、TB、BP2、BP1 和 BP0 设置的阵列保护将被反转。 例如，当 CMP=0 时，可以保护顶部的 64KB 块，而阵列的其余部分则不受保护； 当 CMP=1 时，顶部的 64KB 块将不受保护，而阵列的其余部分将变为只读。 有关详细信息，请参阅状态寄存器存储器保护表。 默认设置为 CMP=0。

**Status Register Protect (SRP, SRL)：**为 W25Q128JV 提供了三个状态配置寄存器。 Read Status Register-1/2/3 指令可用于提供有关闪存阵列可用性的状态、设备是写使能还是禁用、写保护状态、Quad SPI 设置、安全寄存器锁定状态、 擦除/编程挂起状态，输出驱动强度，写状态寄存器指令可用于配置设备写保护功能，四线SPI设置，安全寄存器OTP锁定，输出驱动。 对状态寄存器的写访问由非易失性状态寄存器保护位（SRP、SRL）的状态、写使能指令以及在标准/双 SPI 操作期间的 /WP 引脚控制。



### 状态寄存器2

|             S15             |        S14         |             S13             |             S12             |             S11             |   S10    |     S9      |          S8          |
| :-------------------------: | :----------------: | :-------------------------: | :-------------------------: | :-------------------------: | :------: | :---------: | :------------------: |
|             SUS             |        CMP         |             LB3             |             LB2             |             LB1             |   (R)    |     QE      |         SRL          |
| SUSPEND STATUS(Status Only) | COMPLEMENT PROTECT | SECURITY REGISTER LOCK BITS | SECURITY REGISTER LOCK BITS | SECURITY REGISTER LOCK BITS | Reserved | QUAD ENABLE | STATUS REGISTER LOCK |

**Erase/Program Suspend Status (SUS)：**只读，暂停状态位是状态寄存器 (S15) 中的只读位，在执行擦除/编程暂停 (75h) 指令后设置为 1。 SUS 状态位通过擦除/编程恢复 (7Ah) 指令以及断电、上电周期清除为 0。

**Security Register Lock Bits (LB3, LB2, LB1)：**安全寄存器锁定位（LB3、LB2、LB1）是状态寄存器（S13、S12、S11）中的非易失一次性编程 (OTP) 位，可为安全寄存器提供写保护控制和状态。 LB3-1 的默认状态为 0，安全寄存器解锁。 LB3-1 可以使用写状态寄存器指令单独设置为 1。 LB3-1 为一次性可编程 (OTP)，一旦设置为 1，对应的 256 字节安全寄存器将永久变为只读。

**Quad Enable (QE) ：**Quad Enable (QE) 位是状态寄存器 (S9) 中的非易失性读/写位，可启用 Quad SPI 操作。 当 QE 位设置为 0 状态（具有订购选项“IM”和“JM”的部件号的出厂默认设置）时，/HOLD 启用，器件在标准/双 SPI 模式下运行。 当 QE 位设置为 1（部件号的出厂固定默认值，订购选项为“IQ/IN”和“JQ”），Quad IO2 和 IO3 引脚被启用，/HOLD 功能被禁用时，器件在 标准/双/四 SPI 模式。



### 状态寄存器3

|            S23            |          S22           |          S21           |   S20    |   S19    |           S18           |   S17    |   S16    |
| :-----------------------: | :--------------------: | :--------------------: | :------: | :------: | :---------------------: | :------: | :------: |
|         HOLD/RST          |         DRV 1          |         DRV 0          |  （R）   |  （R）   |           WPS           |    R     |    R     |
| / HOLD or /RESET Function | Output Driver Strength | Output Driver Strength | Reserved | Reserved | Write Protect Selection | Reserved | Reserved |

**Write Protect Selection (WPS)：**WPS 位用于选择应使用的写保护方案。 当 WPS=0 时，器件将使用 CMP、SEC、TB、BP[2:0] 位的组合来保护存储器阵列的特定区域。 当 WPS=1 时，设备将利用单个块锁来保护任何单个扇区或块。 在器件上电或复位后，所有单个块锁定位的默认值为 1。

**Output Driver Strength (DRV1, DRV0)：**DRV1 和 DRV0 位用于确定读取操作的输出驱动器强度。

| DRV，DRV0 | Driver Strength |
| :-------: | :-------------: |
|    0,0    |      100%       |
|    0,1    |       75%       |
|    1,0    |       50%       |
|    1,1    |       25%       |

**Reserved Bits：**有一些保留的状态寄存器位可以读出为“0”或“1”。 建议忽略这些位的值。 在“写状态寄存器”指令期间，保留位可以写为“0”，但不会有任何影响。

下图是状态寄存器对应的保护空间。

![image-7](/images/chips/image-7.png)



## 关于winbond 在linux下 uid的读取

寄存器0x4b是winbond芯片读取uid。

在spi-nor.c文件中加入以下信息：

```c
u8 uid[20];
	tmp = nor->read_reg(nor,0x4B,uid,20);
	if(tmp < 0){
		dev_dbg(nor->dev, "error %d reading UID\n", tmp);
		return ERR_PTR(tmp);
	}
	dev_info(nor->dev,"SPI-NOR-UniqueID %*phN\n",16,&uid[4]);
```

加入位置为，spi_nor_read_id函数的最开始即可。

新版内核修改：

```c
u8 uid[20];
if(nor -> spimem){
		struct spi_mem_op op = 
			SPI_MEM_OP(SPI_MEM_OP_CMD(0x4b,1),
					SPI_MEM_OP_NO_ADDR,
					SPI_MEM_OP_DUMMY(0,4),
					SPI_MEM_OP_DATA_IN(8,uid,1)
						);
		tmp = spi_mem_exec_op(nor->spimem, &op);
		if(tmp < 0){
			dev_dbg(nor->dev, "error %d reading UID\n", tmp);
			return ERR_PTR(tmp);
		}
		dev_info(nor->dev,"SPI-NOR-UniqueID %*phN\n",16,&uid[4]);
	} else{
		tmp = nor->read_reg(nor,0x4b,uid,20);
		if(tmp < 0){
			dev_dbg(nor->dev, "error %d reading UID\n", tmp);
			return ERR_PTR(tmp);
		}
		dev_info(nor->dev,"SPI-NOR-UniqueID %*phN\n",16,&uid[4]);
	}
```

