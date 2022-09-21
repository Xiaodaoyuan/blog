---
title: JVM（1）：内存结构
date: 2018-09-25 11:38:43
categories: Java
tags: JVM虚拟机
---

#### 程序计数器
　　程序计数器是内存结构中很小的一块，属于线程私有的。字节码解释器工作时就是通过改变这个计数器的值来选取下一条需要执行的指令，分支、循环、跳转、异常处理、线程恢复都要依赖这个计数器来实现。程序计数器是内存结构中唯一一个没有定义OOM的区域。
<!--more-->
#### 虚拟机栈
　　虚拟机栈也是线程私有的，主要用于存储局部变量表、操作数栈、动态链接、方法出口等信息。每一个方法的调用对应着一个栈帧在虚拟机中的入栈出栈过程。   
　　虚拟机栈中定义了StackOverflowError和OutOfMemeryError。当线程请求的栈深度大于虚拟机允许的最大深度，就会抛出SOF异常。如果虚拟机栈可以动态扩展，扩展时无法申请足够的内存，就会抛出OOM异常。

#### 本地方法栈
　　本地方法栈和虚拟机栈一样，只不过本地方法栈是为native方法服务的。hotspot虚拟机把本地方法栈和虚拟机栈合二为一了。

#### JAVA堆
　　Java堆时内存结构中最大的一块区域，它是线程共享的，主要用于存储对象的实例。Java堆可以细分为年轻代和老年代，年轻代又可以分成Eden区，From Survivor区和To Survivor区。Java堆是垃圾收集器管理的主要区域，现在的收集器基本都采用分代收集的策略。当内存不足时，Java堆中也会出现OOM异常。

#### 方法区
　　方法区也是线程共享的，主要用于存储被加载的类信息、常量、静态变量、即时编译器编译后的代码等数据。方法区中也会出现OOM异常。
#### 直接内存
　　直接内存并不是虚拟机内存结构中的一块，但是这部分内存也被频繁使用，也会导致ＯＯＭ异常。JDK1.4中引入的NIO就使用了直接内存。