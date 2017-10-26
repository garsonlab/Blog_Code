---
title: Unity中使用c++
date: 2017-3-13
tags:
- Unity
- c++
categories: UnityScript
---


Unity具有跨平台特性，所以一般若是要使用c++分为四种情况：Windows、Android、MacOS以及IOS，对应使用生成的库文件后缀分别为”.dll”、”.so”、”.bundle”、”.a”；

## 库文件生成
### DLL
在Windows下生成dll的方式非常简单，一般来说，对我们这种写代码的来说都会安装vs，直接在vs中新建相应的类库，然后添加你需要的cpp文件和头文件即可。我试了此种方法，但是用vs生成的会包含一些我不需要的头文件等，而我又类似于有洁癖，不想在我需要的代码之外看见其他的，并且，我懒得去除0.0 所以换了一种生成方式——CMake。
度娘了一下使用方法，各种分文件夹、各种list文件，一阵头疼。我需要编译的文件不多，也就五六个“cpp”，六七个“.h”，所以直接放在了一个文件夹下。层级为：

	Root——bin //放置生成dll  
		|— build//放置cmake生成的工程  
		|— lib//放置源码  
		      |—|—CmakeLists.txt  
		|— CMakeLists.txt  

Root下CMakeLists.txt的写法：

	cmake_minimum_required(VERSION3.0)  
	PROJECT (Your_Project_Name)  
	ADD_SUBDIRECTORY(lib)
 第一行版本，第二行设置自己的工程名，第三行加入lib。

Lib下CmakeLists.txt的写法：
``` bash
	set(PROJECT_NAME" Your_Project_Name ")  
	SET(SRC  
	    ****.cpp#多个cpp文件按此方式写  
	    ****.cpp  
	    ****.cpp)  
	    ADD_LIBRARY(${PROJECT_NAME} SHARED ${SRC}) #想得到动态库，参数就是SHARED  
	    install (  
	             TARGETS           ${PROJECT_NAME}  
	             DESTINATION"../bin"  
	             )
```

接着，打开提前安装的cmake工具（cmake-gui），选择Source Code为Root, binaries选择build，点击左下角Configure按钮，如果出现“CMAKE_INSTALL_PREFIX”最好还是选择Root吧，然后，选择左下角的Generate按钮，Done完之后在build文件夹下用VS打开一个“vcxproj”文件，右键在相应的项目上生成就可以得到DLL。

### SO
.so文件用于安卓上，所以我们使用Android NDK来编译。在Windows上编译我的环境为ndk+Cygwin，亲自在虚拟机Linux上只需要下载相应的ndk即可。具体的安装配置步骤找度娘即可。
NDK编译首先需要编写Android.mk文件，具体编写为:

``` bash
 	LOCAL_PATH:= $(call my-dir)  
   
    include $(CLEAR_VARS)  
   
    LOCAL_MODULE    := Your_Project_Name  
    LOCAL_SRC_FILES := \  
    ****.cpp \  
    ****.cpp \  
    ****.cpp \  
    ****.cpp  
   
    APP_STL := stlport_static  
    include $(BUILD_SHARED_LIBRARY)
```

 以上指令只看名字也能猜出大概的意思，把所有源文件和mk文件放在一个文件夹下，下一步就是直接编译了，先切换到指定的目录，CygWin执行指令：$NDK_ROOT/ndk-build，结果并不是我所需要的，提示为：No Such File or Directy #include<vector>

因为我的源文件用了系统的类库，但是在此并没找到，Google说要加上一句:
``` bash
	APP_STL :=stlport_static  
```
然并卵！又试了N中网上盛传的解决方案都没卵用。然后我就看看ndk自带的demo，模仿着又在文件夹下添加了一个Application.mk
``` bash
APP_PLATFORM := android-9  
APP_ABI := all  
APP_STL := stlport_static  
```

还有由于很多安全限制，许多函数的接口上，必须用“const”修饰，一般你在vs上是不会报错的…在此会提示：Error: No much function for call ***
 __int64在linux下也会有问题，要改成相应的。
………一大堆vs下没有的bug来袭…
然后，没有然后了，就是执行成功了，恭喜你获得.so文件一个。
（另，在linux下，源文件必须放在小写的jni目录下，否则不识别，我也不造为什么0.0）


### bundle
复制你使用ndk时修改的那一大堆在Windows下不会报错但Android下一大波bug的源文件到你的mac下，使用XCode新建一个MacOS下的bundle项目，代码添加进去，直接build即可。

### a
使用方式同bundle，在iOS下选择 Cocoa TouchStatic Library 新建，然后拷贝代码，执行，获得.a。

### 注
另，导出dll时， 在要导出的头文件下首先添加
``` cpp
#pragma once  
#define DllExport  extern "C" __declspec( dllexport )//宏定义,  
```

然后， 在需要导出的类或方法前，添加 **DLLExport**，类似：
``` cpp
DllExport MyClass * NewMyClass(); //导出一个方法  
```

而在除了dll的其他导出上， 不需要以上的定义， 而是在需要导出的类或函数前后做如下的定义写法
``` cpp
#pragma once  
#ifdef __cplusplus  
extern "C" {  
#endif  
  
//要导出的函数或类  
  
#ifdef __cplusplus  
}  
#endif  
```

##  使用
在Unity中创建文件夹“Plugins”，
###直接把把dll扔进去即可，或者创建个“x86_64”的文件夹装dll。
###在该文件夹下创建Android/Libs，把生成的armeabi-v7a和x86两个文件夹拷贝进来即可。
###在该文件夹下创建IOS文件夹，把.a放进去
###bundle文件同dll一样处理
###代码
``` csharp
const string DLL_NAME="*****"; //android和ios下类库前会自动加lib的，但此处我们用的是不写的
[DllImport(DLL_NAME)]
public static extern IntPtr Methord();//IntPtr用来接收指针
[DllImport(DLL_NAME)]
public static extern int M1(IntPtrpath);
[DllImport(DLL_NAME)]
public static extern void M2(IntPtr path, int mask=0x01);
```

导出使用全都类似这样。接着你只需要直接在unity中使用就可以了。