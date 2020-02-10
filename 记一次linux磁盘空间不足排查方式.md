title: "记一次linux磁盘空间不足排查方式"
date: 2019-07-28 10:48:16
categories: 问题记录
tags: [linux]

----



今天预发布环境查询sql出现：

```shell
错误代码： 1021
Disk full (/tmp/#sql_1_0.MAI); waiting for someone to free some space... (errno: 28 "No space left on device")
```

大概意思就是：磁盘不足；

接下来排查：

使用 `du -h * | sort -nr `排序查看最大占用文件夹依次类推找到占用最大文件夹进行查找

`du -sh * | sort -nr `  `du -sh`可以查看具体多大

找了半天发现是某个容器日志文件太大造成空间不足。重启docker 完毕

也可以作为docker 磁盘占用查看的方式

