---
layout: post
title: Linux platform
date: 2022-2-25
typora-root-url: ..
---

今天觉得关于linux platform设备驱动的整个流程还不是很清晰，因此这里开始进行一次比较全面的复盘和回顾。并在Mizar板子上运行相关实验。

## 前言

关于linux驱动方向上，IO的读写操作最为简单，可以通过已有的实验进行回顾。但是像I2C、SPI、LCD等这些复杂的外设的驱动就不能这么简单的去写了，Linux系统要考虑到驱动的可重用性，因此提出了驱动的分离与分层这样的软件思路，在这个思路下诞生了我们经常打交道的platform设备驱动，也叫平台设备驱动。那么这里就具体介绍一下Linux下的驱动分离与分层，以及platform框架下的设备驱动如何去写。

## Linux驱动的分离与分层

### 驱动的分隔与分离

对于Linux这样一个成熟、庞大、复杂的操作系统，代码的重用性非常重要，否则的话就会在Linux内核中存在大量无意义的重复代码。尤其是驱动程序，因为驱动程序占用了Linux内核代码量的大头，如果不对驱动程序加以管理，任由重复的代码肆意增加，那么用不了多久Linux内核的文件数量就庞大到无法接受的地步。

假如现在有三个平台A、B、C这三个平台（这里的平台说的是SOC）上都有MPU6050这个I2C接口的六轴传感器，按照我们写逻辑的I2C驱动的思路，每个平台都要有一个MPU6050的驱动，因此编写出来的最简单的驱动框架如图所示：

![](/images/linux/platform0.png)

从上图中可以看出，每种平台下都有一个主机驱动和设备驱动，主机驱动肯定是必须要的，毕竟不同的平台其I2C控制器不同。但是右侧的设备驱动就没必要每个平台都写一个，因为不管对于哪个SOC来说，MPU6050都是一样的，通过I2C接口读写数据就行了，只需要一个MPU6050的驱动程序即可。如果再来几个I2C设备，比如AT24C02、FT5206（电容触摸屏）等，按照上图所示，那么设备端的驱动将会重复的编写好几次。显然，在Linux驱动程序中的这种写法是不推荐的，那么最好的做法就是每个平台的I2C控制器都提供一个统一的接口（主机驱动），每个设备的话也只提供一个驱动程序（设备驱动），每个设备通过统一的I2C接口驱动来访问，这样就可以大大简化驱动文件，比如将上图框架转换成如图框架;

![](/images/linux/platform1.png)

实际上呢，I2C驱动设备肯定有很多种，不知MPU6050这一个，因此实际框架如图：

![](/images/linux/platform2.png)

这个就是驱动的分离。也就是将主机驱动和设备驱动分隔开来，比如I2C、SPI等等都会采用驱动分隔的方式来简化驱动的开发。在实际的驱动开发中，一般I2C主机控制器驱动已经有半导体厂家编写好了，而设备驱动一般也由设备期间的厂家编写好了，我们只需要提供设备的信息就好了，比如I2C设备的话，设备连接到哪个I2C接口，I2C的速度是多少等。相当于将设备信息从设备驱动中剥离开来，驱动使用标准方法来获取到设备信息（比如从设备树中获取信息），然后根据获取到的设备信息来初始化信息。这样就相当于驱动只负责驱动，设备只负责设备，想办法将两者进行匹配即可。这个就是Linux中的总线（bus）、驱动（driver）和设备（device）模型，也就是驱动分离。总线就是驱动和设备信息的桥梁，负责给两者搭桥，如图：

![](/images/linux/platform3.png)

当我们向系统注册一个驱动的时候，总线就会在右侧的设备中查找，看看有没有与之相配的设备，如果有的话就将两者联系起来。同样的，当向系统中注册一个设备的时候，总线就会在左侧的驱动中查找有没有与之匹配的设备，有的话也联系起来。Linux内核中大量的驱动程序都采用总线、驱动和设备模式，因此待会重点说的platform驱动就是这个思想下的产物。

### 驱动的分层

了解完驱动的分隔和分离后，看一看驱动的分层。简单看一下驱动的分层，就像网络的7层模型，不同的层负责不同的内容。同样，Linux驱动往往也是分层的，分层的目的也是为了在不同的层负责不同的内容。例如input（输入子系统），简单介绍一下。input子系统负责管理所有跟输入相关的驱动，包括键盘、鼠标、触摸等，最底层的就是设备原始驱动，负责获取输入设备的原始值，获取到的输入时间上报给input核心层。input核心层会处理各种IO模型，并且提供file_operations操作集合。我们在编写输入设备驱动的时候只需要处理好输入事件的上报即可，至于如何处理这些上报的输入事件那是上层去考虑的。可以看出借助分层模型可以极大的简化我们的驱动的编写，对于驱动编写来说非常的友好。

## platform平台驱动模型简介

上面讲了设备驱动的分离，并且引出了总线（bus）、驱动（driver）和设备（device）模型，比如I2C、SPI、USB等总线。但是在SOC中有些外设是没有总线这个概念的，但是又要使用总线、驱动和设备模型该怎么办？为了解决问题，Linux提出了platform这个虚拟总线，相应的就有platform_driver和platform_device。

### platform总线

Linux系统内核使用bus_type结构体表示总线，此结构体定义在文件inlcude/linux/device.h，bus_type结构体内容如下：

```c
struct bus_type {					/* 设备的总线内型 */
	const char		*name;		/* 总线的名字 */
	const char		*dev_name; /* 用于子系统枚举设备 (dev->id) */
	struct device		*dev_root;/* 用作父设备的默认设备 */
	const struct attribute_group **bus_groups;/* 总线的默认属性 */
	const struct attribute_group **dev_groups;/* 总线上设备的默认属性 */
	const struct attribute_group **drv_groups;/* 总线上设备驱动程序的默认属性 */

	int (*match)(struct device *dev, struct device_driver *drv);
	int (*uevent)(struct device *dev, struct kobj_uevent_env *env);
	int (*probe)(struct device *dev);
	int (*remove)(struct device *dev);
	void (*shutdown)(struct device *dev);

	int (*online)(struct device *dev);
	int (*offline)(struct device *dev);

	int (*suspend)(struct device *dev, pm_message_t state);
	int (*resume)(struct device *dev);

	int (*num_vf)(struct device *dev);

	const struct dev_pm_ops *pm;

	const struct iommu_ops *iommu_ops;

	struct subsys_private *p;
	struct lock_class_key lock_key;
};
```

第9行的match函数很重要，这个函数完成设备和驱动之间匹配的，总线就是使用match函数来根据注册的设备来查找对应的驱动，或者根据注册的驱动来查找相应的设备，因此每一条总线都必须实现此函数。match函数有两个参数：dev和drv，这两个参数分别为device和device_drivere类型，就是设备和驱动。

platform总线是bus_type的一个具体实例，定义在文件drivers/base/platform.c，platform总线定义如下：

```c
struct bus_type platfor_bus_type = {
    .name = "platform",
    .dev_groups = platform_dev_groups,
    .match = platform_match,
    .uevent = platform_uevent,
    .pm = &platform_dev_pm_ops,
};
```

platform_bus_type就是platform平台总线，其中platform_match就是匹配函数。我们来看一下驱动和设备是如何匹配的，platform_match函数定义在文件drivers/base/platform.c中，函数内容如下：

```c
static int platform_match(struct device *dev, struct device_driver *drv)
{
	struct platform_device *pdev = to_platform_device(dev);
	struct platform_driver *pdrv = to_platform_driver(drv);

	/* When driver_override is set, only bind to the matching driver */
	if (pdev->driver_override)
		return !strcmp(pdev->driver_override, drv->name);

	/* Attempt an OF style match first */
	if (of_driver_match_device(dev, drv))
		return 1;

	/* Then try ACPI style match */
	if (acpi_driver_match_device(dev, drv))
		return 1;

	/* Then try to match against the id table */
	if (pdrv->id_table)
		return platform_match_id(pdrv->id_table, pdev) != NULL;

	/* fall-back to driver name match */
	return (strcmp(pdev->name, drv->name) == 0);
}
```

可以看出，驱动和设备的匹配方法有四种方法：

第11-12行的OF类型的匹配，也就是设备树采用的匹配方式。of_driver_match_device函数定义在文件include/linux/of_device.h中，device_driver结构体（表示设备驱动）中有一个名为of_match_table的成员变量，此成员变量保存着驱动的compatible匹配表，设备树中的每个设备节点的compatible属性会和of_match_table表中的所有成员比较，查看是否有相同的条目，如果有点话就表示设备和此驱动匹配，设备和驱动匹配成功以后probe函数就会执行。

第15-16行，ACPI匹配方式。

第19-20行，id_table匹配，每个platform_driver结构体有一个id_table成员变量，顾名思义，保存了很多id信息，这些id信息存放着这个platform驱动所支持的驱动类型。

第23行，如果第三种匹配方式的id_table不存在的话，就直接比较驱动和设备的name字段，看看是不是相等，如果相等的话就匹配成功了。

对于支持设备数的Linux内核版本，一般设备驱动为了兼容性都支持设备树和无设备树两种匹配方式。也就是第一种匹配方式都会存在，第三种和第四种只要存在一种就可以，一般用的最多的还是第四种。

### platform驱动

platform_driver结构体表示platform驱动，此结构体定义在文件include/linux/platform_device.h中，内容如下：

```c
struct platform_driver{
   int (*probe)(struct platform_device *);
    int (*remove)(struct platform_device *);
    void (*shutdown)(struct platform_device *);
    int (*suspend)(struct platform_device *,pm_message_t state);
    int (*resume)(struct platform_device *);
    struct device_driver driver;
    const struct platform_device_id *id_table;
    bool prevent_deferred_probe;
};
```

probe函数：当驱动与设备匹配成功后，函数probe就会执行，非常重要的函数！一般驱动的提供者会编写，如果自己要编写一个全新的驱动，那么probe就需要自行实现。

driver成员，为device_driver结构体变量，Linux内核里面大量使用到了面向对象的四类，device_driver相当于基类。提供了最基础的驱动框架。platform_driver继承了这个基类，然后在此基础上又添加了一些特有的成员变量。

id_table表，也就是上一节讲解platform总线匹配驱动和设备的时候采用的第三种方法，id_table是个表（也就是数组），每个元素的类型为platform_device_id，platform_device_id结构体内容如下：

```c
struct platform_device_id{
    char name[PLATFORM_NAME_SIZE];
    kernel_ulong_t driver_data;
};
```

device_driver结构体定义在include/linux/device.h，device_driver结构体内容如下：

```c
struct device_driver {
	const char		*name;
	struct bus_type		*bus;

	struct module		*owner; db代表什么
	const char		*mod_name;	/* used for built-in modules */

	bool suppress_bind_attrs;	/* disables bind/unbind via sysfs */
	enum probe_type probe_type;

	const struct of_device_id	*of_match_table;
	const struct acpi_device_id	*acpi_match_table;

	int (*probe) (struct device *dev);
	int (*remove) (struct device *dev);
	void (*shutdown) (struct device *dev);
	int (*suspend) (struct device *dev, pm_message_t state);
	int (*resume) (struct device *dev);
	const struct attribute_group **groups;

	const struct dev_pm_ops *pm;

	struct driver_private *p;
};
```

of_match_table就是采用设备树的时候驱动使用的匹配表，同样是数组，每个匹配项都为of_device_id结构体类型，此结构体定义在文件include/linux/mod_devicetable.h中，内容如下：

```c
struct of_device_id{
    char name[32];
    char type[32];
    char compatible[128];
    const void *data;
};
```

对于设备而言，就是通过设备节点的compatible属性值和of_match_table中每个项目的compatible成员变量进行比较，如果有相等的就表示设备和此驱动匹配成功。

==在编写platform驱动的时候，首先定义一个platform_driver结构体变量，然后实现结构体中的各个成员变量，重点是实现匹配方法以及probe函数。当驱动和设备匹配成功以后，probe函数就会执行，具体的驱动程序在probe函数里面编写，比如字符设备驱动等等。==

当我们定义并初始化号platform_driver结构体变量以后，需要在驱动入口函数里面调用platform_driver_register函数向Linux内核注册一个platform驱动。

`int platform_driver_register(struct platform_driver *drv);`

函数参数和返回值含义如下：

driver:要注册的platform驱动。

返回值:负数,失败；0，成功

还需要在驱动卸载函数中通过platform_driver_unregister函数卸载platform驱动，platform_driver_unregister函数原型如下

`void platform_driver_unregister(struct platform_driver *drv);`

函数参数和返回值含义如下：

drv：要卸载的platform驱动。

返回值：无。

### platform设备

platform驱动已经写好后，我们还需要platform设备，否则的话单单一个驱动也做不了什么。platform_device这个结构体表示platform设备，这里我们要注意，如果内核支持设备树的话就不要使用platform_device来描述设备。当然，如果一定要用platform_device来描述设备信息的话也是可以，platform_device结构体定义在文件include/linux/platform_deivce.h，结构体内容如下：

```c
struct platform_device {
	const char	*name;
	int		id;
	bool		id_auto;
	struct device	dev;
	u32		num_resources;
	struct resource	*resource;

	const struct platform_device_id	*id_entry;
	char *driver_override; /* Driver name to force a match */

	/* MFD cell pointer */
	struct mfd_cell *mfd_cell;

	/* arch specific additions */
	struct pdev_archdata	archdata;
};
```

第2行，name表示设备名字，要和所使用的platform驱动的name字段相同，否则的话设备就无法匹配到对应的驱动。比如对应的platform驱动的name字段为“xxx-gpio"，那么此name字段也要设置为”xxx-gpio"。

num_resources表示资源数量，一般是resource资源的大小。

resource表示资源，也就是设备信息，比如外设寄存器。Linux内核使用resource结构体表示资源，resource结构体内容如下：

```c
struct resource {
	resource_size_t start;
	resource_size_t end;
	const char *name;
	unsigned long flags;
	unsigned long desc;
	struct resource *parent, *sibling, *child;
};
```

start和end分别表示资源的起始和终止信息，对于内存类的资源，就表示内存起始和终止地址，name表示资源名字，flags表示资源类型，可选的资源类型都定义在文件include/linux/ioport.h里面。

在不支持设备树的Linux版本中，用户需要编写platform_device变量来描述设备信息，然后使用platform_device_register函数将设备信息注册到Linux内核中，此函数原型如下所示：

`int platform_device_register(struct platform_device *pdev)`

函数参数和返回值含义如下：

pdev:要注册的platform设备。

返回值:负数,失败；0，成功。

如果不再使用platform的话可以通过platform_device_unregister函数注销掉相应platform设备，platform_device_unregister函数原型如下：

`void platform_device_unregister(struct platform_device *pdev)`

函数参数和返回值含义如下：

pdev：要注销的platform设备。

返回值：无。

## 设备树下的platform驱动

### 设备树下的platform驱动简介

platform驱动框架分为总线、设备和驱动，其中总线不需要我们这些驱动程序员去管理，这个是Linux内核提供的，我们在编写驱动的时候只要关注与设备和驱动的具体实现即可。**在没有设备树的Linux内核下，我们需要分别编写并注册platform_device和platform_driver**，分别代表设备和驱动。在使用设备树的时候，设备的描述被放到了设备树中，因此，**platform_device就不需要我们去编写了，我们只需要实现platform_driver即可**，当内核在解析设备树的时候会自动帮我们创建一个platform_device对象；在编写基于设备树的platform驱动的时候我们需要注意以下几点

#### 1 在设备树中创建设备节点

首先我们需要在设备树中创建设备节点来描述设备信息，重点是要设置好compatible属性的值，因为platform总线需要通过设备节点compatible属性值来匹配驱动！这点要切记。

```
led{
	compatible = "mizar,led";
	status = "okay";
	defalut-state = "on";
	led-gpio = <& gpio0 7 GPIO_ACTIVE_HIGH>;
};
```

#### 2 编写platform驱动的时候要注意兼容属性

在使用设备树的时候，platform驱动会通过of_match_table来保存兼容性值，也就是表明此驱动兼容哪些设备。所以of_match_table将会尤为重要

```c
static const struct of_device_id leds_of_match[] = {
    {.compatible = "mizar,led"},
    {/*Sentinel*/}
};

MODULE_DEVICE_TABLE(of,leds_of_match);

static struct paltform_driver leds_platform_driver = {
    .driver = {
        .name = "zynq-led",
        .of_match_table = leds_of_match,
    },
    .probe = leds_probe,
    .remove = leds_remove,
}
```

## 总结

对platform_devicce 和 platform_driver有一个全局的认识，由于现在都是基于设备树的编程，因此，我们只需要编写platform_driver，完成相应的.probe 和 .remove函数即可完成大部分的platform驱动编写。

## 说明

本文档参考了:

正点原子的linux开发手册--领航者ZYNQ之嵌入式Linux开发指南V1.5.2

