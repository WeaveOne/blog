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

      `jinfo -flag PrintGCDetails PID # turn off PrintGCDetails` 修改标志位的值(只对标记为`manageable`的标志有效)

### jconsole

描述：提供jvm活动的图形化视图，线程的使用、类的使用、GC活动

