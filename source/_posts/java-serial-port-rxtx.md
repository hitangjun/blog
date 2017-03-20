---
title: Java串口通信RXTX
date: 2010-08-08 10:55:18
description: 
categories:
- Java
tags:
- Java
toc: true
author: John Tang
comments:
original:
permalink: 
---

### Java串口通信RXTX 
RXTX是个提供串口和并口通信的开源java类库，由该项目发布的文档均遵循LGPL协议。

该项目的主页位于 [http://users.frii.com/jarvi/rxtx/index.html](http://users.frii.com/jarvi/rxtx/index.html)。 　　

RXTX项目提供了Windows,Linux,Mac os X,Solaris操作系统下的兼容javax.comm串口通讯包API的实现，
为其他研发人员在此类系统下研发串口应用提供了相当的方便。 

这里是他的WIKI主页 [http://rxtx.qbang.org/wiki/index.php/Main_Page](http://rxtx.qbang.org/wiki/index.php/Main_Page) 

下载 [http://rxtx.qbang.org/pub/rxtx/rxtx-2.1-7-bins-r2.zip ](http://rxtx.qbang.org/pub/rxtx/rxtx-2.1-7-bins-r2.zip )

在WINDOWS上安装 [http://rxtx.qbang.org/wiki/index.php/Installation_on_MS-Windows ](http://rxtx.qbang.org/wiki/index.php/Installation_on_MS-Windows )

### 使用RXTX Examples 


- Two way communcation with the serial port 
- Event based two way Communication 
- Parallel Communications 
- Discovering comm ports 
- Discovering available comm ports 
- Writing "Hello World" to a USB to serial converter 


### 在eclipse里如何使用RXTX呢？ 

This is how I add and use RXTX in Eclipse for Win32 Projects, there are probably other ways but it works for me. 

1. Copy RXTXcomm.jar, rxtxSerial.dll and rxtxParallel.dll files to the lib directory of your project 复制RXTXcomm.jar, rxtxSerial.dll 和 rxtxParallel.dll文件到项目lib文件夹 
2. Under Project | Properties | Java Build Path | Libraries 
3. click Add JARs... Button 在工程属性下将lib包加入项目编译路径 
4. Select the RXTXComm.jar from lib directory 
5. Jar should now be in the Build Path 
6. expand the RXTXComm.jar entry in the list and select "Native Library Location" 添加后，单击RXTXComm.jar展开,选择Native Library Location,选中rxtxSerial.dll 和 rxtxParallel.dll所在的目录，然后应用修改,OK。 
7.  Select the project lib directory and apply

[附件](http://d.download.csdn.net/down/2605644/tangjunchf)里有WIKI上所有的源码，

另外com.hitangjun.rxtx.util.demo包下有一个基于RXTX应用的实例，

使用的是观察者模式 不是很明白RXTX是如何和串口通信的，串口的设置参数是如何应用，以及是如何处理收发数据的。
