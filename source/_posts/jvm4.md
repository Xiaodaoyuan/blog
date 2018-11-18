---
title: JVM（4）：虚拟机性能监控
date: 2018-09-26 15:12:08
categories: Java
tags: JVM虚拟机
---

要进行虚拟机性能监控光有理论还不够，还需要实践。JDK为我们提供了一些命令行工具以及图形化洁面工具进行虚拟机性能监控实践。
<!--more-->
### 命令行工具
- **jps：虚拟机进程状况工具**  
用于列出正在执行的虚拟机进程。  
命令格式：jps ［options］［hostid］  
主要选项：-q   只输出LVMID，省略主类的名称  
    　　　　　-l 输出主类的全名，如果执行的是jar包，输出jar的包路径  
  　　　　　 -m   输出虚拟机进程启动时传给主类main函数的参数  
  　　　　　 -v     输出虚拟机进程启动时的JVM参数  
jps可以通过RMI协议查询开启了RMI服务的远程虚拟机进程状态，hostid为RMI注册表中注册的主机名。


- **jstat：虚拟机统计信息监视工具**  
用于监视虚拟机各种运行状态信息。  
命令格式： jstat -<option\> [-t] [-h <lines\>] <vmid\> [<interval\> [<count\>]]  
     　　jstat -gc   2764   250  20     每250ms查看一次进程ID2764的gc情况，一共查看20次  
主要选项：-class       监视类装载、卸载、总空间以及所耗费时间  
    　　  -gc           监视java堆gc状况，包括Eden区、两个Survivor区、、老年代、永久带等的容量、已用空间、GC时间合计等信息 


- **jinfo：Java配置信息工具**  
实时查看和调整虚拟机各项参数。  
命令格式：jinfo [ option ] vmid


- **jmap：Java 内存映像工具**  
用于生成堆转储快照，一般是headdump文件或者dump文件。  
命令格式：jmap [ option ] vmid  
主要选项：-dump 生成Java堆转储快照  
　　　　　-heap 显示Java堆详细信息。


- **jhat：虚拟机堆转储快照分析工具**  
jhat与jmap搭配使用，来分析jmap生成的堆转储快照。

- **jstack：Java堆栈跟踪工具**   
用于生成虚拟机当前的线程快照，一般是threaddump文件或者javacore文件。  
命令格式：jstack [ option ] vmid
---
### 图形化工具
- **JConsole：Java监视与管理控制台**
- **VisualVM：多合一故障处理工具**
