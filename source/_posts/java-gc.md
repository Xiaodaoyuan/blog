---
title: Java GC总结
date: 2019-02-19 17:43:27
categories: Java
tags: JVM虚拟机
---


#### 1. Jvm内存结构
![image](/images/jvm-struct.png)
<!--more-->
#### 2. GC介绍
优化一个Java程序，主要关注点有两个：响应时间和吞吐量。所以我们在做GC优化的时候会考虑这两个方面：当程序需要低延迟时，我们应该考虑减少GC的停顿时间；当程序需要高吞吐量时，我们应该考虑降低整体GC时间。

 ##### 2.1 JVM分代回收
  ![image](/images/gc-time.png)
  由上图可以看出大部分对象的存活时间很短，基本在一次gc后就会被回收，所以当代主流虚拟机（Hotspot VM）的垃圾回收都采用“分代回收”的算法。“分代回收”是基于这样一个事实：对象的生命周期不同，所以针对不同生命周期的对象可以采取不同的回收方式，以便提高回收效率。
  ![image](/images/heap-struct.png)
-   新生代（Young Generation）：大多数对象都在新生代中被分配，当新生代被填满后就会触发Minor GC，经过Minor GC后会回收大部分对象，所以新生代很适合采用复制算法。回收时会现将eden区和一块survivor区中的存活对象标记，然后复制到另一块survivor区，然后回收eden区和其中的一块survivor区。新生代中的对象每经过一次Minor GC，年龄会加1，当达到一定年龄后会进入老年代。这个值默认是15，可以通过参数MaxTenuringThreshold来设置。
-   老年代（Old Generation）：在新生代中经历了N次垃圾回收后仍然存活的对象，就会被放到年老代，该区域中对象存活率高。老年代的垃圾回收（又称Major GC）通常使用“标记-清理”或“标记-整理”算法。整堆包括新生代和老年代的垃圾回收称为Full GC（HotSpot VM里，除了CMS之外，其它能收集老年代的GC都会同时收集整个GC堆，包括新生代）。
-   永久代（Perm Generation）：主要存放元数据，例如Class、Method的元信息，与垃圾回收要回收的Java对象关系不大。相对于新生代和年老代来说，该区域的划分对垃圾回收影响比较小。

##### 2.2 垃圾收集算法
判断对象是否存活：
- 引用计数法：缺点是不能解决循环引用问题
- 可达性分析法：根据对象到GCRoots是否可达来判断对象是否存活，可以作为GCRoots有以下一些：虚拟机栈中引用的对象、方法区中类静态属性引用的对象、方法区中常量引用的对象、本地方法栈JNI引用的对象。

回收算法：
-  标记清除算法：会产生内存碎片
-  标记整理算法：不会产生碎片，比标记清除效率更低
-  复制算法：需要额外的内存空间

##### 2.3 STW（stop the world）
Hotspot VM判断对象是否存活使用的是可达性分析法。为了保证整个分析过程期间对象的引用关系不发生变化，需要暂停所有java执行线程，这个就叫做STW。  

**OopMap**  
由于目前的主流Java虚拟机使用的都是准确式GC，所以当执行STW后，并不需要一个不漏地检查完所有执行上下文和全局的引用位置，虚拟机应当是有办法直接得知哪些地方存放着对象引用。在HotSpot的实现中，是使用一组称为OopMap的数据结构来达到这个目的的，在类加载完成的时候，HotSpot就把对象内什么偏移量上是什么类型的数据计算出来，在JIT编译过程中，也会在特定的位置记录下栈和寄存器中哪些位置是引用。这样，GC在扫描时就可以直接得知这些信息了。

**安全点**  
HotSpot并没有为每条指令都生成OopMap，只有在特定的位置记录这个信息，这些位置称为安全点（SaftPoint）。安全点的选定基本上是以程序“是否具有让程序长时间执行的特征”为标准进行选定的。长时间执行的最明显特征就是指令序列复用，例如方法调用、循环跳转、异常跳转等。  

**安全区域**  
程序不执行的时候，也就是没有分配CPU的时候，典型的就是线程sleep或者blocked状态时，这个时候不能响应中断请求。JVM不太可能等到线程重新分配CPU然后走到安全点。这样就需要安全区域解决这个问题。安全区域是指在一段区域内引用关系不会发生变化，在这中间任何一个地方GC都是安全的。我们可以把Safe Region看成是扩展的safe point。
##### 2.4 垃圾收集器
- Serial和Serial Old收集器：单线程的收集器，分别对应新生代和老年代。client模式下默认的收集器。这两个的特点是简单、易实现、效率高。
- ParNew收集器：Serial收集器的多线程版，可以充分的利用CPU资源，减少回收的时间。
- Parallel Scavenge和Parallel Old收集器：并行的多线程垃圾收集器，目标是达到一个可控制的吞吐量。提供了两个参数用于精确控制吞吐量，分别是控制最大垃圾收集停顿时间的-XX:MaxGCPauseMillis参数以及直接设置吞吐量大小的-XX:GCTimeRatio参数。
- CMS收集器：基于标记清除算法。目标是降低GC停顿时间。
- G1收集器：可预测的停顿时间。

##### 2.5 常用GC参数设置

参数 | 说明
---|---
-Xms、-Xmx |设置堆的最小内存和最大内存。设置成一样，固定堆的大小，避免堆内存动态变化带来的性能损失。
-Xmn | 设置堆中新生代的大小。
-XX:NewRatio |设置新生代与老年代的比值
-XX:SurvivorRatio|设置eden区和survivor区的比值
-XX:PretenureSizeThreshold|设置直接进入老年代的对象大小阈值
-XX:MaxTenuringThreshold|设置新生代对象晋升到老年代的年龄
-XX:+UseSerialGC|使用Serial+Serial Old收集器
-XX:+UseParNewGC|使用ParNew+Serial Old收集器
-XX:+UseConcMarkSweepGC|使用ParNew+CMS收集器
-XX:+UseParallelGC|使用Parallel+PS MarkSweep收集器
-XX:+UseParallelOldGC|使用Parallel+Paralle Old收集器
-XX:+UseG1GC|使用G1 收集器
-XX:GCTImeRatio| 设置GC时间占总时间百分比
-XX:MaxGCPauseMillis|设置最大停顿时间
-XX:ParallelGCThreads|设置并行GC时回收线程数
-XX:PrintGCDetails|输出GC详细日志
-XX:+UseAdaptiveSizePolicy |动态调整堆中各个区域的大小以及进入老年代的年龄

##### 2.6 GC日志
每一种回收器的日志格式都是由其自身的实现决定的，换而言之，每种回收器的日志格式都可以不一样。但虚拟机设计者为了方便用户阅读，将各个回收器的日志都维持一定的共性。 

**Yong GC**
![image](/images/yong-gc-log.png)
**Full GC**
![image](/images/full-gc-log.png)

#### 3. CMS收集器
##### 3.1 CMS介绍
CMS收集器是一款老年代的并发垃圾收集器，它的主要目标是降低GC停顿时间。对于要求服务器响应时间的应用上适合采用CMS收集器。开启CMS的参数是-XX:+UseConcMarkSweepGC.
##### 3.2 CMS过程
- 初始标记：暂停程序所有线程任务，然后确定从GC Roots可达的对象集，最后重新恢复程序。
- 并发标记：在程序执行时并发标记所有可达的对象。
- 并发预清理：这个阶段会追溯第二阶段中改变的对象引用。
- 重新标记：暂停应用程序，重新标记并发阶段可以发生的对象引用关系。
- 并发清理：并发清理不可达的对象。
- 并发重置：重置CMS收集器的数据结构，等待下一次垃圾回收。

##### 3.3 CMS缺点
- CMS是基于标记清除算法的，所以会产生内存碎片。
- 需要更多的CPU资源，并发阶段CMS线程和用户并发执行，这样就需要更多的CPU。
- CMS的另一个缺点是它需要更大的堆空间。因为CMS标记阶段应用程序的线程还是在执行的，那么就会有堆空间继续分配的情况，为了保证在CMS回收完堆之前还有空间分配给正在运行的应用程序，必须预留一部分空间。也就是说，CMS不会在老年代满的时候才开始收集。相反，它会尝试更早的开始收集，已避免上面提到的情况：在回收完成之前，堆没有足够空间分配！默认当老年代使用68%的时候，CMS就开始行动了。 – XX:CMSInitiatingOccupancyFraction =n 来设置这个阀值。

#### 4. G1收集器
##### 4.1 G1介绍
G1是一种服务器端的垃圾收集器，应用在多处理器和大容量内存环境中，在实现高吞吐量的同时，尽可能的满足垃圾收集暂停时间的要求。G1和CMS的目标都是降低停顿时间。G1相比CMS的优势在于没有内存碎片，可以实现可预测的停顿时间。
##### 4.2 G1的几个概念
- **Region**  

G1的各代存储地址是不连续的，每一代都使用了n个不连续的大小相同的Region，每个Region占有一块连续的虚拟内存地址。如下图所示：
![image](/images/G1-gc-layout.png)  
在上图中，我们注意到还有一些Region标明了H，它代表Humongous，这表示这些Region存储的是巨大对象（humongous object，H-obj），即大小大于等于region一半的对象。为了减少连续H-objs分配对GC的影响，需要把大对象变为普通的对象，建议增大Region size。  
一个Region的大小可以通过参数-XX:G1HeapRegionSize设定，取值范围从1M到32M，且是2的指数。如果不设定，那么G1会根据Heap大小自动决定。总的region数量不超过2048个。

- **SATB**  

全称是Snapshot-At-The-Beginning，由字面理解，是GC开始时活着的对象的一个快照。它是通过Root Tracing得到的，作用是维持并发GC的正确性。 

SATB抽象的说就是在一次GC开始的时候是活的对象就被认为是活的，此时的对象图形成一个逻辑“快照”（snapshot）；然后在GC过程中新分配的对象都当作是活的。其它不可到达的对象就是死的了。 

如何知道哪些对象是一次GC开始之后新分配的：每个region记录着两个top-at-mark-start（TAMS）指针，分别为prevTAMS和nextTAMS。在TAMS以上的对象就是新分配的，因而被视为隐式marked。 

在并发标记阶段，有可能一个死亡对象被重新引用，这样就有可能导致活对象被回收。SATB通过write barrier 将引用记录下来，回收的时候会扫描这个引用记录。

当然，很可能有对象在snapshot中是活的，但随着并发GC的进行它已经死了，但SATB还是会让它活过这次GC，在下一次GC中再回收，这样就会造成浮动垃圾。
- **Remembered Sets（RSets）** 

全称是Remembered Set，是辅助GC过程的一种结构，典型的空间换时间工具。还有一种数据结构也是辅助GC的：Collection Set（CSet），它记录了GC要收集的Region集合，集合里的Region可以是任意年代的。
G1中每个Region都有一个RSet，RSet记录了其他Region中的对象引用本Region中对象的关系，GC的时候可以通过扫描RSet来避免全堆的扫描。

- **Pause Prediction Model**  

Pause Prediction Model即停顿预测模型。G1通过Pause Prediction Model来满足用户定义的目标停顿时间，并根据指定的目标停顿时间来选择要收集的region数量。 

目标停顿时间可以通过参数-XX:MaxGCPauseMillis来指定，默认是200ms。  

G1收集器之所以能建立可预测的停顿时间模型，是因为它可以有计划地避免在整个Java堆中进行全区域的垃圾收集。G1跟踪各个Region里面的垃圾堆积的价值大小（回收所获得的空间大小以及回收所需时间的经验值），在后台维护一个优先列表，每次根据允许的收集时间，优先回收价值最大的Region（这也就是Garbage-First名称的来由）。

##### 4.3 G1 GC模式
G1提供了两种GC模式，Young GC和Mixed GC，两种都是完全Stop The World的。
- **Yong GC**  
G1的Yong GC和其他收集器的Minor GC过程类似，将年轻代region中存活的对象复制移动到Survivor区中的region，或者晋升到老年代。为了下一次Yong GC，还会动态调整Eden区和Survivor区的大小。

- **Mixed GC**  
除了年轻代region中的存活对象，还会根据global concurrent marking统计得出收集收益高的若干老年代Region。在用户指定的开销目标范围内尽可能选择收益高的老年代Region。  
在G1通过几个Mixed  GC回收足够的老年代region之后，会恢复执行Yong GC直到下一global 次concurrent marking完成。

Mixed GC不是Full GC，它只能回收部分老年代的Region，如果Mixed GC实在无法跟上程序分配内存的速度，导致老年代填满无法继续进行Mixed GC，就会使用Full GC来收集整个GC heap。

##### 4.4 标记周期过程
- **初始标记（Initial mark）**  
标记那些会存在对老年代引用的survivor regions (root regions)，会发生STW，伴随一次Yong GC发生。
- **扫描根引用区（Root region scanning ）**  
扫描 Survivor到老年代的引用，该阶段必须在下一次 Young GC 发生前结束。这个阶段不能发生Yong GC，如果中途Eden区真的满了，也要等待这个阶段结束才能进行Young GC。
- **并发标记（Concurrent marking）**  
寻找整个堆的存活对象。这个阶段和程序并发运行的，可以被Yong GC打断。
- **重新标记（Remark）**  
会发生STW，标记最后可能存活的对象。这个阶段使用了SATB。
- **清理（Cleanup）**  
对存活对象和空Region进行统计（STW）；  
清除Remembered Sets（STW）；  
清除空Region并加入到free list（并发执行）。

##### 4.5 G1参数

参数 | 说明
---|---
-XX:+UseG1GC | 开启G1收集器
-XX:G1HeapRegionSize=n | 设置每个region大小
-XX:MaxGCPauseMillis=200 |设置最大停顿时间
-XX:G1NewSizePercent=5 |设置新生代占堆的百分比
-XX:G1MaxNewSizePercent=60 | 设置新生代占堆的最大百分比
-XX:ParallelGCThreads=n |设置STW后并行工作的线程数（通常和cpu核心数一样）
-XX:ConcGCThreads=n |设置并发标记时工作的线程数（通常约等于1/4ParallelGCThreads）
-XX:InitiatingHeapOccupancyPercent=45|设置触发标记周期时堆占用阈值
-XX:G1MixedGCLiveThresholdPercent=85|设置触发包含老年代region的Mixed GC的堆占用阈值
-XX:G1HeapWastePercent=5|设置浪费的堆内存百分比，当可回收百分比小于这个时，不会触发Mixed GC
-XX:G1OldCSetRegionThresholdPercent=10|设置在Mixed GC中要回收的老年代region上限
-XX:G1MixedGCCountTarget=8|设置Mixed GC的目标数量
-XX:G1ReservePercent=10|设置空闲内存的百分比，以减少内存空间溢出的风险

##### 4.6 G1 使用建议
- 避免通过-Xmn或者-XX:NewRatio等相关参数设置新生代的大小，因为这会让设置的停顿时间MaxGCPauseMillis失效。
- 当优化GC时，总存在延迟与吞吐量之间的权衡。G1的目标是10%的垃圾收集时间，Parallel收集器的目标是1%的垃圾收集时间。当使用G1时想要更高的吞吐量，就需要适当增加延迟时间。设置MaxGCPauseMillis过低的话会让吞吐量降低。
- 优化Mixed GC：  
使用-XX:InitiatingHeapOccupancyPercent参数改变触发标记时阈值；  
使用XX:G1MixedGCLiveThresholdPercent and -XX:G1HeapWastePercent这两个参数控制触发Mixed GC时机；  
使用-XX:G1MixedGCCountTarget and -XX:G1OldCSetRegionThresholdPercent这两个参数调整Mixed GC回收老年代region数量。


#### 5. Jvm性能监控工具
- **[jps：虚拟机进程状况工具](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jps.html)**  
列出正在执行的虚拟机进程。  
 jps  [options]   [hostid]   
-q参数 只输出LVMID，省略主类的名称。  
-l参数输出主类的全名，如果执行的是jar包，输出jar的包路径。   
-m参数输出虚拟机进程启动时传给主类main函数的参数。  
-v参数输出虚拟机进程启动时的JVM参数。

- **[jstat:虚拟机统计信息监视工具](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jstat.html)**  
监视虚拟机各种运行状态信息。  
jstat [ generalOption | outputOptions vmid [ interval[s|ms] [ count ] ]  
主要参数有class、compiler、gc、gccapacity、gcutil等。
- **[jinfo:Java配置信息工具](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jinfo.html)**  
实时查看和调整虚拟机各项参数。  
jinfo [ option ] pid

- **[jmap:Java 内存映像工具](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jmap.html)**  
jmap [ options ] pid  
-dump:[live,] format=b, file=filename  
-heap  
-histo[:live]
- **[jhat:虚拟机堆转储快照分析工具](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jhat.html)**  
jhat与jmap搭配使用，来分析jmap生成的堆转储快照。  
jhat [ options ] heap-dump-file
- **[jstack:Java堆栈跟踪工具](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jstack.html)**  
输出当前java进程中的线程堆栈快照。  
jstack [ options ] pid

#### 6. GC优化
##### 6.1 技巧
- 减少进入老年代的对象数量。  
老年代GC相对来说会比新生代GC更耗时，因此，减少进入老年代的对象数量可以显著降低Full GC的频率。所以需要设置合理的新生代大小。
- 减少Full GC的时间。  
  通过减少老年代内存大小可以降低Full GC时间，但是这样会导致Full GC频率升高甚至出现OOM，所以要设置一个合理的老年代大小。

##### 6.2 过程
- 监控GC状态
- 分析监控结果后决定是否需要优化GC。  
- 设置GC类型/内存大小（可以在多台服务器上试验不同参数）
- 分析结果（将结果满意的参数应用到所有服务器上）

