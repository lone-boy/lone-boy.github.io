---
layout: post
title: "petalinux 编译相关文件保存方法"
date: 2022-06-21
description: "Petalinux东西"
tag: Petalinux
typora-root-url: ..
---

尽管petalinux在编译一些很小的工程时非常便捷，但是一旦当我们需要多次修改bit流文件或者是修改设备书这种清卡情况，就非常麻烦。因为每一次都需要去编译，这个过程过于长久。因此，这里提供一种新的方式来提高我们制作系统的效率。



## 前言

首先先了解以下petalinux的基本流程。从命令入手：

- petalinux-create --type project --name <> --template zynq

这个命令没啥说的，主要就是创建了一个petalinux项目，后面所有的操作都是基于这个命令完成。

- petalinux-config --get-hw-description=.

这个命令主要是通过配置hdf文件。=.可以说的是路径。

- petalinux-build

这个命令主要是对配置好的petalinux文件进行编译。

- petalinux-package --boot --fsbl --u-boot --force

这个命令主要是制作BIN文件，用来引导启动uboot。

当然，有的时候也会根据自己的需求去配置，会用到petalinux-config -c <u-boot kernel rootfs>等等命令，这个就不详细说了。



## 步骤

首先peatlinux主要是每一次都要去load uboot和内核源代码，再加上编译需要的时间，就会变得很长。然后最要命的是每次petalinux编译完之后就会清理内核源代码和uboot源代码。因此，首先是要将源代码保存下载，具体方法如下：

```sh
echo 'RM_WORK_EXCLUDE += "linux-xlnx"' >> project-spec/meta-user/conf/petalniuxbsp.conf
echo 'RM_WORK_EXCLUDE += "u-boot-xlnx"' >> project-spec/meta-user/conf/petalniuxbsp.conf
ecgo ''>> project-spec/meta-user/conf/petalniuxbsp.conf

```

上面就是让petalinux不删除源码，这里使用一个脚本文件来创建petalinux工程并完成硬件描述文件的读取。

```sh
#!/bin/sh
help () {
    echo "Error : need \$projectname \$hw_bsp"
    echo "  eg :"
    echo "      $0 ax_project ~/hw-vivado-bsp-for-petalinux/some_packakge"
    exit 1
}

if [ -z "$1" ]; then
	help
fi
if [ -z "$2" ]; then
	help
fi

## 删除源目录
rm $1 -rf

echo "$2"
## 创建项目
petalinux-create --type project --template zynq --name $1

## 导入硬件信息
cd $1
petalinux-config --get-hw-description $2

## 使PetaLinux不删除源码
echo 'RM_WORK_EXCLUDE += "linux-xlnx"'>> project-spec/meta-user/conf/petalinuxbsp.conf

echo 'RM_WORK_EXCLUDE += "u-boot-xlnx"'>> project-spec/meta-user/conf/petalinuxbsp.conf

echo ''>> project-spec/meta-user/conf/petalinuxbsp.conf

```



### 获取petalinux源代码以及配置文件

Petalinux工程编译后，在```build/tmp/work-shared```目录下，根据对应芯片目录下的`kernel-source`中能找到源代码。

这里我会将该目录的代码单独拷贝出来，以后都可以基于该kernel进行开发。但是请注意，这个内核程序是没有编译过的，因此，我们得先通过配置文件使用传统的方式去配置该内核并且完成编译的工作。这里，将`kernel-build-artifacts`文件夹下的所有东西复制到拷贝过来的内核中。如果不复制过来，就不能编译该内核了。



### 设备树的获取

设备的一般使用sdk就可以获取，首先我们先去xilinx的github上下载设备树源代码链接如下https://github.com/Xilinx/device-tree-xlnx。接下来我们可以打开Vivado工程进入sdk。当然也有一种命令行的方法通过hsi命令来获得设备树，后续有机会会写。下载完设备树后记得切换到对应的vivado版本。

这里进入sdk后，先导入我们刚刚下载的设备树库，在Xilinx->Repositoris里可以选择导入外部库，如图

![image-0](/images/petalinux/image0.png)

上面的是导入库到当前工作空间，别的工程就不会导入。下面是导入到全局空间中，这样后面所有的工程都不需要导入该库文件。

点击New，选择设备树代码目录点击apply和ok即可。

接下来创建设备树工程如图

![image1](/images/petalinux/image1.png)

在Board Support Package OS选择device_tree即可。