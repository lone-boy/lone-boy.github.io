---
layout: post
title: CMakeLists
data: 2022-03-05
categories:CMakeLists
typora-root-url: ..
---

学一学CMakeLists这个玩意吧。看uhd源码的时候意识到大型工程都是使用make命令就可以编译。而cmakelist是非常重要的一个玩意，这里我们将会将Make、Makefile、CMake和CMakeLists的作用以及之间的联系。

## Make

一个.c或者.cpp文件从源文件到目标文件的过程叫做编译，但是一个项目不可能只存在一个文件，这就涉及到多个文件的编译问题，在编译的过程中必然涉及到某个文件的先编译，某个文件的后编译。构建过程就是安排文件的编译先后关系。

Make就是一种构建工具，属于GNU项目。



## Makefile

make命令执行时，需要一个makefile文件，告诉make命令如何去编译和链接程序。makefile规则后续再说。



## Cmake和CmakeLists.txt

虽然Make和Makefile简化了手动构建的过程，但是编写makefile文件仍然是一个很麻烦的工作，因此就有了CMake工具。CMake工具用于生成Makefile

文件，而如何生成Makefile文件则由CMakeLists.txt文件指定。

例如这里有一个hello.cpp。

我们编写一个CMakeLists.txt文件，具体内容如下。

```cmake
PROJECT(HELLP)

SET(SRC_LIST hello.cpp)

MESSAGE(STATUS "this is BINARY dir" ${HELLO_BINDARY_DIR})
MESSAGE(STATUS "this is SOURCE dir" ${HELLO_SOURCE_DIR})
MESSAGE(STATUS "this is PROJECT_SOURCE" ${PROJECT_SOURCE_DIR})

ADD_EXECUTABLE(hellp.out ${SRC_LIST})
```

执行完cmake命令后就可以运行。

**因此三者之间的一个关系如图**





## CMakeLists命令写法



