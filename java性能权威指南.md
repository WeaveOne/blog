title: "java性能权威指南"
date: 2019-07-28 10:48:16
categories: jvm
tags: [java,jvm]

----



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

<!-- more -->

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

      `jinfo -flag -PrintGCDetails PID 修改标志位的值(只对标记为`{manageable} 的标志有效)

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

2. 多数方法第一次编译级别是3(自己测试为1)（所有方法都从级别0开始）如果方法运行得足够频繁，就会编译为级别4（级别3的代码就会被丢弃）

3. 逆编译会回到0

#### 小结

1. final不会影响应用的性能



### 垃圾收集入门

#### 垃圾收集概述

1. 新生代(Eden 和 Survivor)，老年代
2. 对象首先在新生代中分配。新生代填满时，垃圾收集器会暂停所有的应用线程，回收新生代空间。不再使用的对象被回收，仍然在使用的对象被移动到其他地方。操作称为Minor GC
3. eden空间的对象要么被移走要么被回收。所有存活的对象要么移到Survivor空间要么移动到老年代。相当于进行了一次压缩整理
4. 老年代填埋时，jvm需要找出老年代中不再使用的对象，并对他们进行回收，简单的垃圾回收直接停掉所有的应用线程。找出不再使用的对象，进行回收，接着对堆空间整理。这个过程被称为Full GC（针对老年代）。通常导致应用程序线程长时间停顿。
5. CMS和GC收集器被称为Concurrent垃圾收集器。由于它们将停止应用程序的可能降到最小，也被称为低停顿收集器
6. 评估垃圾收集器时：
   1. 单个请求会受停顿时间影响，不过其受FUllGC长时间停顿的影响更大。如果目标要尽可能地缩短响应时间，那么选择Concurrent收集器更合适。
   2. 如果平均响应时间比最大响应时间更重要，采用Throughput收集器通常就能满足要求。
   3. 使用Concurrent收集器来避免长的停顿时间也有其代价，这会消耗额外的cpu

#### GC算法

1. Serial垃圾收集器，使用单线程清理堆的内容，无论是MinorGC 还是FullGC,清理对空间时，所有应用线程都会被暂停. -XX:+UseSerialGC
2. Throughput垃圾收集器，项目Serial 使用多线程进行垃圾回收。 -XX:+UseParallelGC  -XX:+UseParallelOldGC
3. CMS收集器，设计初衷为了消除前两者收集器FullGC周期中的长时间停顿，CMS在MinorGC是会暂停所有的应用线程，并以多线程的方式进行垃圾回收。不同的是收集算法。 。在FullGC时不在暂停应用线程，而是使用若干个后台线程定期对老年代空间进行扫描及时回收其中不再使用的对象。使其成为一个低延迟收集器。应用线程旨在MIniorGC以及后台线程扫描老年代时发生极其短暂的停顿，总时长比使用Throughput收集器短得多。额外的付出代价是更高的CPU使用。除此之外，后台线程不在进行任何压缩整理工作，意味着堆会逐渐碎片化。如果后台线程无法获得完成他们任务的cpu资源或者堆变得过渡碎片化以至于无法找到空间分配。CMS就蜕化到Serial收集器行为。-XX:+UseParNewGC -XX:+UseConcMarkSweepGC默认禁用
4. G1垃圾收集器，设计初衷是为了尽量缩短处理超大堆时产生的停顿，minorGC采用暂停所有应用程序的方式。同其他收集算法一样。老年代的垃圾收集工作由后台线程完成，大多数不需要暂停应用程序。由于老年代被划分到不同的区域，G1通过讲对象从一个区域复制到另一个区域，完成对象清理工作，意味着在正常的处理过程中，G1收集器实现了堆的压缩整理（部分），G1收集器的堆不大容易发生碎片化。同CMS避免FullGC的代价是消耗额外的CPU周期。-XX:UseG1GC

#### GC调优基础

1. 调整堆的大小 -xms  -xmx 原则所有jvm内存分配不超过物理内存 - 1G
2. 如果jvm发现使用初始的堆大小，频繁发生GC，他就尝试增大堆的空间，知道jvm的GC的频率回归到正常范围，或者到达上限值。
3. 如果GC消耗太长时间，可以设置Xmx增大堆大小，经验法则是完成FullGC后，应该释放出70%的空间。可使用jconsole来查看
4. 尽量考虑通过调整GC算法的性能目标，而非微调堆的大小来改善程序性能

#### 代空间的调整

1. 代空间的分配具有两面性：如果新生代分配比较大，GC频率就低，新生代晋升老年的对象也少，但是这样老年代相对较小，比较容易填满，会更频繁的FullGC

   ```properties
   -XX:NewRatio = N # 设置新生代与老年代的空间占用比率
   -XX:NewSize = N #设置新生代空间的初始大小
   -XX:MaxNewSize=N # 设置新生代空间的最大大小
   -XmnN 将NewSize and MaxNewSize设定为同一值的快捷方法
   Initial Young GenSize = Inital Heap Size /(1 + NewRatio)
   ```

2. 新生代空间的大小是初始堆大小的33%，NewSize不设置的话有NewRatio计算出

3. 如果堆扩张，新生代也扩张直到MaxNewSize。默认情况下，新生代的最大值也是由NewRatio的值设定

#### 永久代和元空间的调整

1. 永久代（java7）：`-XX:PermSize=N` `-XX:MaxPermSize=N`
2. 元空间(java8):`-XX:MetaspaceSize=N` `-XX:MaxMetaspaceSize=N`
3. 如果程序在启动时发生大量的FullGC(因为需要载入数据量巨大的类)通常都是由于永久代或者元空间发生了大小调整，因此为了改善启动速度，增大初始值是个不错的注意
4. 永久代也会经历垃圾回收

#### 控制并发

1. 启动线程数：`-XX:+UseParallelGCThreads=N`
2. jvm会在每个cpu上运行一个线程，最多同时运行8个。一旦超出就会调整算法`threads` = 8+((N-8) * 5 / 8)
3. 如果机器同时运行了多个jvm实例，限制所有jvm使用的线程总数是不错的主意

#### 垃圾回收工具

1. 开启gc日志：-verbose:gc 或 -XX:+PrintGC推荐使用-XX:PrintGCDetails
2. 输出到文件：-xloggc:filename
3. 日志文件限制：-XX:UseGCLogfileRotation  -XX:NumerOfGCLogfiles = N -XX:GCLogfileSize = N默认无限制
4. 工具：GC Histogram(搜不到)
5. jstat -gcutil pid 1000

### 垃圾收集算法

#### 堆大小自适应调整和静态调整

1. 设置-XX:MaxGCPauseMillis=N和-XX:GCTimeRatio=N 
2. MaxGCPauseMillis标志用语设定应用可承受的最大停顿时间，可以设置为0或者最小值，但是会影响MinorGC和FUllGC。如果值非常小那么老年代会非常小会频繁的FullGC.
3. GCTimeRatio标志可以设置在垃圾回收上花费的时间(与应用线程的运行时间相比较。是一个百分比![1561952165118](C:\Users\user\AppData\Roaming\Typora\typora-user-images\1561952165118.png)
4. GCTimeRatio默认为99  带入公式得到0.99意味着99%是应用程序运行时间，1%是垃圾回收。
5. GCTimeRatio值可以按期望百分比计算公式![1561952424071](C:\Users\user\AppData\Roaming\Typora\typora-user-images\1561952424071.png)
6. MaxGCPauseMillis（Throughput收集器默认没设置）标志优先级比-Xms和-Xmx高，如果设置该值，新生代、老年代会随之进行调整，直到满足对应停顿时间的模板。一旦这个目标达成，堆的总容量就开始逐渐增大，直到运行时间的比率达到设定值。这两个目标都达成后，jvm会尝试缩减堆的大小，尽可能以最小的堆的大小来满足。
7. 由于默认情况不设置停顿时间目标，通常自动堆调整的效果是堆的大小会持续增大，知道满足设置的GCTimeRatio目标。根据经验垃圾回收消耗总时间占总时间的3%到6%效果相当不错。
8. 采用动态调整是进行堆调优极好的入手点。

#### G1垃圾收集器（默认2048个分区）

1. 一种工作在堆内不同分区上的并发收集器。分区既可以归属于老年代，也可以归属于新生代
2. ，默认情况下，一个堆被划分为2048个分区。专注于垃圾最多的分区
3. eden空间耗尽会触发G1垃圾收集器进行（xx个分区填满后触发）
4. 收集活动：
   1. 新生代垃圾收集
   2. 后台收集，并发周期
      1. 至少一次（很可能多次）新生代垃圾收集。在将Eden空间中的分区标记完全释放之前，新的分区已经开始分配
      2. 标记老年代找出包含最多的垃圾分区。
      3. 并发周期包含多个阶段
         1. 初始-标记阶段（initial-mark），暂停所有应用程序-部分源于初始-标记阶段也会进行新生代垃圾收集
         2. 扫描根分区（root region）不需要暂停应用程序。不能发生新生代垃圾收集。如果扫描根分区时，新生代空间用尽，新生代垃圾收集必须等待跟扫描结束才能完成，意味着新生代垃圾收集的停顿时间更长。
         3. 并发标记阶段，完全后台运行，并发标记阶段可以被中断，这个阶段可能发生新生代垃圾收集
         4. 重新标记阶段和正常清理阶段，都会暂停应用线程
         5. 这时垃圾定位未完成，真正做的事情是定位出哪些老的分区可回收垃圾最大
   3. 混合式垃圾收集
      1. 不仅进行正常的新生代垃圾收集，同时回收扫描线程标记的分区
   4. 必要时FullGC
5. 4种情况触发fullGC
   1. 并发模式失效：在标记周期，老年代在周期完成之前被填满。
   2. 晋升失败：混合式垃圾回收，清理老年代分区，在老年代空间在垃圾回收释放出足够内存之前就会被耗尽。
   3. 疏散失败：新生代垃圾收集时，Survivor空间和老年代中没有足够的空间容纳所有的幸存对象。
   4. 巨型对象分配失败：分配巨大对象的应用程序
   5. 以上3中一种解决方案是增加堆的大小。

#### G1垃圾收集器调优

1. 主要目标是避免发生并发模式失败或者疏散失败
2. 也适用于频繁发生的新生代垃圾收集
3. 优化方案：
   1. 增加总的堆空间大小或者调整老年代，新生代之间的比列来增加老年代空间大小
   2. 增加后台线程数目（需考虑cpu）
   3. 以更高的频率进行G1的后台垃圾收集活动
   4. 以混合式垃圾回收周期中完成更多的垃圾收集工作
4. G1垃圾收集器调优模板之一是尽量简单，为达到这个目标，G1收集器最主要的调优只通过呀一个标志进行：-XX:MaxGCPauseMillis=N
5. MaxGCPauseMillis默认值为200ms 如果G1收集器发生时空停顿的时长超过该值，G1就会自动弥补
6. 如果减小参数值，达到停顿时间目标，新生代大小相应减小，频率更加频繁，为达到停顿时间目标，混合式GC收集的老年代分区数也会减少，这会增大并发模式失败发生的几率。通常：
   1. 调整G1垃圾收集的后台线程数
      1. 设置线程数ParallelGCThreads
      2. 并发运行阶段可以使用ConcGCThreads设置运行线程数。计算方法如下：![1562033997277](C:\Users\user\AppData\Roaming\Typora\typora-user-images\1562033997277.png)
   2. 调整G1垃圾收集器运行的频率
      1. 如果G1收集器更早的启动垃圾收集，能是有效的手段。G1垃圾收集周期通常在堆的占用比率参数`-XX:InitiatingHeapOccupancyPercent=N`设定的比率时启动。默认值为45。参数值的依据是整个堆的使用情况
      2. InitiatingHeapOccupancyPercent是个常数，G1收集器自身不会为了达到停顿时间目标而修改这个参数值，如果设置过高，应用程序会陷入fullGC之中，因为并发阶段没有足够的时间在剩下的堆空间被填满之前完成垃圾收集，如果过小，应用程序又会以超过实际需要的节奏进行大量的后台处理。因此G1收集器要避免频繁进行后台清理，并发周期结束之后，检查下堆的大小，确保它的值大于此时堆的大小
   3. 调整G1收集器的混合式垃圾收集周期
      1. 混合式垃圾回收周期中尽量处理更多的分区
      2. 处理的工作量取决于三个因素
         1. 如果分区的垃圾占用比达到35%这个分区就被标记为可垃圾回收`-XX:G1MixedGCLiveThresholdPercent=N`
         2. 回收分区时的最大混合式GC周期数，通过参数`-XX:G1MixedGCCountTarget=N`调节，默认值为8.减少该参数值可以帮助解决晋升失败为问题，代价是混合GC停顿时间更长,另一方面如果停顿时间过长，可以增加这个参数的值。减少每次混合式GC周期的工作量。
         3. GC停顿可忍受的最大时长MaxPauseMillis标志设定，如果G1收集器能够收集超过八分之一标记的老年代分区，增大它的值可能收集更多的老年代分区，反之能够更早的启动并发周期，

#### 高级调优

1. 晋升Survivor空间
   1. 保存生命周期短，无法满足晋升老年代的条件，所以被划分为Eden和Survivor，让新生代内有更多机会被回收
   2. 新生代垃圾收集时，对象从Eden空间移动到Survivor空间0，接着下一次垃圾收集中，活跃对象会从S0和E移动到S1之后，E和S0被清空，下一次垃圾回收从S1和E移到S0如此反复。
   3. 两种情况下，对象被移动到老年代：第一，S空间大小实在太小。新生代垃圾收集时，S空间被填满，E空间剩下的活跃对象直接进入老年代，第二，对象在Survivor空间中经历的GC周期数有个上限，超过上限就会移动到老年代。晋升阈值
   4. Survivor空间是新生代空间的一部分跟堆内的其他区域一样，jvm可以对他动态调节。Survivor空间的初始大小由`-XX:InitialSurvivorRatio=N`标志决定，初始暂用比率默认为8,大约占10%的新生代空间![1562118152283](C:\Users\user\AppData\Roaming\Typora\typora-user-images\1562118152283.png)
   5. jvm可以增大Survivor空间的大小，上限可以通过`-XX:MinSurvivorRatio=N`设置，默认为3，也就是20%![1562118157891](C:\Users\user\AppData\Roaming\Typora\typora-user-images\1562118157891.png)
   6. 为了保持Survivor空间的大小为某个固定值，可以使用SurvivorRatio参数设定为期望的值，同时关闭`UseAdaptiveSizePolicy`,会同时影响新生代和老年代
   7. 默认情况下Survivor空间调整之后要保证垃圾回收之后有50%是空闲的。通过标志`-XX:TargetSurvivorRatio=N`设置
   8. 晋升阈值通过`-XX:InitialTenuringThreshold=N`标志可以设置初始阈值Throughput和G1默认值为7，CMS为6.jvm最终会在1和最大晋升阈值（`-XX:MaxTenuringThreshold=N`）之间选择合适的值，Throughput默认最大为15 CMS为6
   9. 晋升统计信息`-XX:+PrintTenuringDistribution`标志可以在GC日志中增加这部分信息
   10. 最重要的是观察MinorGC中是否存在由于Survivor空间过小，对象直接晋升到老年代的情况，要尽量避免这种情况。如果大量的短期对象最终填满老年代，会导致频繁FullGC
   11. 最好的办法是 增大堆的大小。为了避免对象晋升到老年代，调整晋升阈值或者空间的阿晓可以避免、
2. 分配大对象
   1. TLAB 线程本地分配缓冲区
   2. TLAB -XX:+PrintTLAB 开启gc日志（slow allocs堆上分配），如果存在大量对象分配发生在TLAB之外，就应该考虑TLAB进行调整了
   3. 增大TLAB可以通过增大Eden空间，也可以指定TLAB大小`-XX:TLABSize=N`默认为0（表示使用动态计算得出的）。这个标志只能设置ResizeTLAB初始大小，为了避免每次GC都调整TLAB大小，可以使用`-XX:-ResizeTLAB`大多数平台为true，这是调整TLAB充分提升对象分配性能最简单的方法，也是最有效的方法
   4. 一旦新的对象无法适配到当前的TLAB中，jvm就会抉择，到底堆上分配还是回收当前TLAB，取决于阈值，一旦超过就会堆上分配没有就回收，这个值是动态计算得出的，默认初始值为TLAB大小的1%或者由参数`-XX:TLABWastTargetPercent=N`设定，每当发生堆上分配这个值就会增大，增量由`-XX:TLABWasterIncrement=N`设定（默认为4）。这样能够避免达到TLAB空间占用的阈值，从而持续地堆上分配对象。随着百分比（TargetPercent）的增大，回收的几率也增加，调整TLABWastTargetPercent参数的结果往往同时伴随着TLAB空间大小的调整。虽然可以调整，但是效果不那么确定。
   5. 推荐应用程序使用小型对象的做法
3. 巨型对象
   1. 对TLAB空间中无法分配的对象，jvm尽量尝试在Eden空间分配，如果Eden无法容纳，就只能在老年代中分配空间，，这种会打乱该对象正常回收周期，如果是一个短期存在的对象，还会对垃圾收集造成负面的影响。、
   2. 尽量不出现巨型对象
4. G1分区的大小
   1. 每个分区大小都是固定的，分区大小不是动态变化，具体的值是启动时，依据堆大小的最小值得出的，分区大小最小为1mb。如果堆的最小值超过2GB`分区大小 = 1<<log(初始堆的大小/2048)`简言之，初始划分堆时，分区的大小是2的最小N次幂，使其结果最接近于2048个分区。分区大小最小1mb，最大不超过32mb![1562123290403](C:\Users\user\AppData\Roaming\Typora\typora-user-images\1562123290403.png)
   2. G1分区的大小可以通过`-XX:G1HeapRegionSize=N`设置（正常情况，默认为0,）。设定的参数值应该是2的幂。否则会向下圆整到最接近2的幂
   3. 调整分区大小，让分区数量接近2048个

### 堆内存最佳实践

#### 堆分析

1. 堆直方图 `jcmd PID GC.class_histogram`输出仅包含活跃对象 显示的是JNI语法![1562727546264](C:\Users\user\AppData\Roaming\Typora\typora-user-images\1562727546264.png)

2. `jmap -histo PID`输出包含被回收的死对象 `jmap -histo:live PID` 在显示直方图之前强制FULLGC

3. 堆分析工具 

   1. `jhat file`
   2. `jvisualvm`
   3. mat 保留对象（retained size）内存指的是回收该对象可以释放的内存量。包含该对象的大小及其指向另一个对象的大小，但不包括另一个对象也被其他对象引用的对象（即共享对象）
      1. 浅对象包含该对象的大小和指向另一个对象引用的大小
      2. 深对象包含该对象及其他所有对象的大小包括共享对象

4. 内存溢出错误

   1. 原生内存不足

      jvm没有原生内存可用，跟堆无关

      错误信息：

      `Exception in thread "main" java.lang.OutOfMemoryError unable to create new native thread`

   2. 永久代或元空间内存不足

      与堆无关，有两种情况：

      1. 应用使用的类太多，超出了永久代的默认容纳范围。解决方案是增加永久代的大小（元空间也如此）
      2. 涉及类加载器的内存泄漏，常出现出javaEE应用服务器中

      错误信息：

      `Exception in thread "main" java.lang.OutOfMemoryError: Metaspace`

   3. 堆内存不足

      错误消息：

      `Exception in thread "main" java.lang.OutOfMemoryError: java heap space`

      1. 自动堆转储`-XX:+HeapDumpOnOutOfMemoryError`默认为false，打开会自动在抛出`OutOfMemoryError`是创建堆转储`-XX:HeapDumpPath=<path>`该标志制定写入位置。默认会在当前工作目录下生成`java_pod<pid>.hprof`文件
      2. `-XX:+HeapDumpAfterFullGC`运行一次FullGC后生成一个堆转储文件
      3. `-XX:+HeapDumpBeforeFullGC`FullGC之前生成

   4. 达到GC的开销限制

      错误信息：

      `Exception in thread "main" java.lang.OutOfMemoryError: GC overhead limit exceeded`

      1. 花在FullGC上的时间超出了-XX：GCTimeLimit=N标志指定的值，默认为98，意思为 如果98%的时间花在GC上，则该条件满足
      2. 一次FullGC回收的内存量少于`-XX:GCHeapFreeLimit=N`标志指定的值。默认值为2，意思是：如果FullGC期间释放的内存不足堆的2%，则该条件满足
      3. 上面两个条件连续5ciFullGC都成立（这个数值无法调整）
      4. `-XX:+UseGCOverhead-Limit`标志的值为true（默认true）

      上面四个条件必须都满足。

5. 减少内存使用

   1. 减少对象大小，最简单的方式就是让对象小一些。下面是两种方式：
      1. 减少实例变量的个数（效果明显）
      2. 减少实例变量的小（不明显）
   2. 引用对象64位中占8个字节 32位占四个字节
   3. 去掉对象中的实例字段，有助于减少对象的大小

6. 延迟初始化

   1. 如果操作使用不频繁，那延迟初始化最合适。如果很常用，实际没有节省内存，而又有轻微的性能损失
   2. 只有当初始化这些字段的几率很低时，延迟初始化才有性能方面的好处。如果一般情况下都会初始化这些字段，那实际上也不会节省内存。
   3. 避免过时引用 参考`ArrayList() remove` 方法，可以主动地将其设置为null
   4. 小结
      1. 只有当常用的代码路径不会初始化某个变量时，才去考虑延迟初始化改变量
      2. 一般不会再存在线程安全的代码上引入延迟出初始化，这样会加重现有的同步成本
      3. 如果使用了线程安全的对象，应该使用双重检查锁。并配置`volatile`关键字

### 对象周期管理

	#### 对象池

1. 对象池对象越多对GC越不友好
2. 对象池必然是同步的，如果对象要频繁地移除和替换，对象池上可能会存在大量的竞争，其结果是，访问对象池可能比初始化新对象还慢
3. 限流，对于稀缺资源的访问，线程池可以起到限流作用

#### 局部变量

1. 生命周期管理：线程局部变量要比池中管理对象更容易，成本更低。
2. 基数性：通常会伴生线程数与保存的可重用对象数之间的一一对应关系。保存的对象数不可能超过线程数
3. 同步：线程局部变量只能用于一个线程之内，所以不需要同步

#### 小结

1. 对象重用适合初始化成本高昂，数量比较少的一组对象

### 软引用、弱引用

#### 软引用

1. 如果对象以后有很大机会重用，可以使用软引用，软引用所引用的对象一定不能有其他的强引用。
2. 当强引用移除时，软引用不会立即释放，具体看下面
3. 设置`-XX:SoftRefLRUPolicyMSPerMB=N`默认值值为1000，表示当jvm内存达到75%是，会回收1024秒没有访问到的对象

#### 弱引用

1. 当问题中的所引用对象会同时被几个线程使用时，应该考虑弱引用
2. 当强引用被移除时，弱引用会立即释放

### 原生内存

#### 原生NIO缓冲区

1. 原生字节缓冲区支持原生代码和java代码在不复制的情况下共享数据。
2. `-XX:MaxDirectMemorySize=N`直接字节缓冲区

#### 原生内存跟踪

1. `-XX:NativeMemoryTracking=off|summary|detail`默认off。如果开启使用：`jcmd pid VM.native_memory summary scale=MB`获取信息