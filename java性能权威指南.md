# java性能权威指南

##  操作系统的工具和分析

```shell
#cpu使用率
vmstat
#磁盘使用率
iostat
#网络使用率
netstat
```

## java监控工具

```shell
#打印java进程涉及的基本类、线程、vm信息
jcmd
jcmd PID help #查看当前可进行的操作
#提供jvm活动的图形化视图，线程的使用、类的使用、GC活动
jconsole
#读取内存堆转储
jhat
#查看jvm系统属性，动态设置系统属性
jinfo
#转储java进程的站信息、显示每个线程的栈的输出
jstack
#提供GC和类装载活动的信息
jstat
#见识jvm的gui工具
jvisualvm
```

### jcmd

描述：打印java进程涉及的基本类、线程、vm信息

使用：`jcmd process_id help`获取帮助

1. **jvm调优标志** `jcmd PID VM.flags [-all]`

2. 特定平台所设置标志 `java PID -XX:+PrintFlagsFinal -version`

   结果：

   ```shell
   uintx InitialBootClassLoaderMetaspaceSize = 4194304   {product}
   uintx InitialCodeCacheSize         =        2555904   {pd product}
   uintx InitialHeapSize             :=        266338304 {product}
   ```

   解释：

   1. 输出中冒号表示标志使用的费默认值。发生这种情况原因为：
       1. 标志值直接在命令行制定
       2. 其他标志间接改变了值
       3. `jvm`自动优化计算出来的默认值
   2. 没有冒号表示默认值 `product`表示所有平台默认设置一直 `pd product`表示独立于平台

### jinfo

   描述：查看`jvm`系统属性，动态设置系统属性

   使用:

   1. `jinfo -flags PID`

   2. `jinfo -flag PrintGCDetails PID` 查看单个标志的值 `-XX: +PrintGCDetails`

      `jinfo -flag -PrintGCDetails PID 修改标志位的值(只对标记为`manageable`的标志有效)

### jconsole

描述：提供jvm活动的图形化视图，线程的使用、类的使用、GC活动

### JIT编译器

1. C1 client 编译器  C2 server编译器 没有被编译执行就是解释执行（只有相对热点数据被编译执行）

   c1编译器在前期比c2编译器快。后期则是C2 后期C2可以进行编译优化，所以提前编译不会被优化

2. 分层编译，必须是server 属性：-xx:+TieredComplilation 一般为最佳选择

#### 调优代码缓存

1. client编译和分层编译可能填满代码缓存器。
2. -xx:ReservedCodeCacheSize 可以设置代码缓存区大小；ps：并不是越大越好代码缓存设为1GB则jvm就会保留1GB内存空间，虽说是需要时才分配但是他任然是被保留的，意思是及其必须有1g内存空间存在
3. jconsole 的内存面板中的 code cache可以监控

#### 编译阈值

1. 编译基于两种jvm计数器：方法调用计数器和方法中的循环回边计数器（看做循环完成的次数）
2. 栈上替换（OSR）：jvm在循环进行时也能进行编译循环。编译结束后，jvm替换还在栈上的代码，循环的下一次迭代就会执行快得多的编译代码
3. 阈值：-XX:CompileThreshold = N触发
4. OSR trigger = (CompileThreshold * ((OnStatckReplacePercentage - InterpreterProfilePercentage)/100))
   1. ,默认值：OnStatckReplacePercentage  servcer：140 client:933
   2. 默认值：InterpreterProfilePercentage  33
5. 计数器会周期性减少，所以执行时间越长并不会导致所有代码都会被编译，执行次数少的代码可能永远都不会被编译

#### 检测编译过程

1. -XX:+PrintCompilation  开启编译打印，每当编译发生时打印一条记录
2. `jstat -compiler PID` 查看编译总数，失败总数等
3. `jstat -printcompilation PID 1000`每秒输出最后一项的编译

#### 逃逸分析

1. `-XX:+DoEscapeAnalysis` 默认为`true`

2. 逃逸分析：对象在除该方法之外的其他地方被调用

   ```java
   public Student getStudent(){
       Student s = new Student;
       // s对象被直接返回，逃逸
       return s;
   }
   ```

#### 逆优化

1. 代码被丢弃(made not entrant)：一个接口有两个是实现类，最开始调用第一个实现类使其优化。之后调用第二个这样之前的优化失效，优化也被废弃。当有更多请求时，jvm会很快终止这部分代码编译。对性能影响不大。分层编译时client编译后的代码被server编译也会把之前编译的代码丢弃
2. 僵尸代码(made zombie)：代表jvm已经回收被丢弃的代码。回收后编译器就会标记为僵尸代码。如果发现僵尸方法，意味着这些有问题的代码可以从代码缓存中移除。如果僵尸化以后再次加载并且频繁使用，jvm就需要重新编译和重新优化。

#### 分层编译级别

1. 编译级别有：

   0：解释代码

   1：简单C1编译代码

   2：受限的C1编译代码

   3：完全C1编译代码

   4：C2编译代码

2. 多数方法第一次编译级别是3(自己测试为1)（所有方法都从级别0开始）如果方法运行得足够频繁，就会编译为级别4（级别3的代码就会被斗气）

3. 逆编译会回到0

#### 小结

1. final不会影响应用的性能



### 垃圾收集入门

#### 垃圾收集概述

1. 新生代(Eden 和 Survivor)，老年代
2. 对象首先在新生代中分配。新生代填满时，垃圾收集器会暂停所有的应用线程，回收新生代空间。不再使用的对象被回收，仍然在使用的对象被移动到其他地方。操作称为Minor GC
3. eden空间的对象要么被移走要么被回收。所有存活的对象要么移到Survivor空间要么移动到老年代。相当于进行了一次压缩整理
4. 老年代填埋时，jvm需要找出老年代中不再使用的对象，并对他们进行回收，简单的垃圾回收直接停掉所有的应用线程。找出不再使用的对象，毒气进行回收，接着对堆空间整理。这个过程被称为Full GC（针对老年代）。通常导致应用程序线程长时间停顿。
5. CMS和GC收集器就是通过这种方式。被称为Concurrent垃圾收集器。由于它们将停止应用程序的可能降到最小，也被称为低停顿收集器
6. 评估垃圾收集器时：
   1. 单个请求会受停顿时间影响，不过其受FUllGC长时间停顿的影响更大。如果目标要尽可能地缩短响应时间，那么选择Concurrent收集器更合适。
   2. 如果平均响应时间比最大响应时间更重要，采用Throughput收集器通常就能满足要求。
   3. 使用Concurrent收集器来避免长的停顿时间也有其代价，这会消耗额外的cpu

#### GC算法

1. Serial垃圾收集器，使用单线程清理堆的内容，无论是MinorGC 还是FullGC,清理对空间时，所有应用线程都会被暂停. -XX:+UseSerialGC
2. Throughput垃圾收集器，项目Serial 使用多线程进行垃圾回收。 -XX:+UseParallelGC  -XX:UseParallelOldGC
3. CMS收集器，设计初衷为了消除前两者收集器FullGC周期中的长时间停顿，CMS在MinorGC是会暂停所有的应用线程，并以多线程的方式进行垃圾回收。不同的是收集算法。 。在FullGC时不在暂停应用线程，而是使用若干个后台线程定期对老年代空间进行扫描及时回收其中不再使用的对象。使其成为一个低延迟收集器。应用线程旨在MIniorGC以及后台线程扫描老年代时发生极其短暂的停顿，总时长比使用Throughput收集器短得多。额外的付出代价是更高的CPU使用。除此之外，后台线程不在进行任何压缩整理工作，意味着堆会逐渐碎片化。如果后台线程无法获得完成他们任务的cpu资源或者堆变得过渡碎片化以至于无法找到空间分配。CMS就蜕化到Serial收集器行为。-XX:UseParNewGC -XX:UseConcMarkSweepGC默认禁用
4. G1垃圾收集器，设计初衷是为了尽量缩短处理超大堆时产生的停顿，minorGC采用暂停所有应用程序的方式。同其他收集算法一样。老年代的垃圾收集工作由后台线程完成，大多数不需要暂停应用程序。由于老年代被划分到不同的区域，G1通过讲对象从一个区域复制到另一个区域，完成对象清理工作，意味着在正常的处理过程中，G1收集器实现了堆的压缩整理（部分），G1收集器的堆不大容易发生碎片化。同CMS避免FullGC的代价是消耗额外的CPU周期。-XX:UseG1GC

#### GC调优基础

1. 调整堆的大小 -xms  -xmx 原则所有jvm内存分配不超过物理内存代销
2. 

