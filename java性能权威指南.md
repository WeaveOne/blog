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

