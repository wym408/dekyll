---
layout: post
title:  "mysql 参数 innodb_flush_method"
date:   2017-07-27 16:15:49 +0700
categories: jekyll update
---
innodb_flush_method参数控制着innodb 数据文件和 redo log 文件的刷新方式

官方文档解释
{% highlight doc %}
Defines the method used to flush data to InnoDB data files and log files, which can affect I/O throughput.
The innodb_flush_method options for Unix-like systems include:
  fsync: InnoDB uses the fsync() system call to flush both the data and log files. fsync is the default setting.
  O_DSYNC: InnoDB uses O_SYNC to open and flush the log files, and fsync() to flush the data files. InnoDB does not use O_DSYNC directly because there have been problems with it on many varieties of Unix.
  O_DIRECT: InnoDB uses O_DIRECT (or directio() on Solaris) to open the data files, and uses fsync() to flush both the data and log files. This option is available on some GNU/Linux versions, FreeBSD, and Solaris.
{% endhighlight %}  
默认fsync,调用fsync去刷新 innodb 数据文件和redo log file，这个是要经过os buffer. 
    
O_DSYNC,理解为innodb直接打开和刷新redo log file，不经过os buffer, 用fsync 刷新数据文件，经过os buffer，官方文档中指出类unix系统中此模式下有问题，最好不用使用. 

O_DIRECT,理解为数据文件的刷新不经过os buffer,redo log file 文件刷新经过os buffer. 
 
![这里写图片描述](http://img.blog.csdn.net/20170730140731084?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd3ltNDA4/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
redo log file 落盘就能保证innodb 事务的安全性,redo log file 顺序的写入到同一个文件中，在commit时尽量保证落盘到disk中，顺序IO能保证commit的用户能比较及时的接收到执行结果，此模式由innodb_flush_log_at_trx_commit参赛控制.  

data file 在磁盘上比较分散，在执行事务完成时，不用及时刷新到磁盘（异步io，及时落盘的代价是很大的，大事务commit等待结果可能就很久很久了），因为redo log 已经落盘，数据安全性已经有了保证，data file 用后台异步线程慢慢刷新到磁盘就ok，data file 在 buffer pool里面已经缓存了一次，如果再用os buffer 再缓存一次，浪费比较大，索引类unix系统都建议使用O_DIRECT模式.  






