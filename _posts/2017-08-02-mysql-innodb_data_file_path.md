---
layout: post
title:  "mysql 参数 innodb_data_file_path"
date:   2017-07-27 16:15:49 +0700
categories: jekyll update
---
共享表空间位置和大小设定  

当启用了 innodb_file_per_table ，单表数据和索引数据被存在了自己的表空间里，但是共享表空间仍然在存储其它的 InnoDB 内部数据：  
1.数据字典，也就是 InnoDB 表的元数据  
2.变更缓冲区  
3.双写缓冲区  
4.撤销日志  
如果没有启用 innodb_file_per_table，则此空间中还包含：
5.表数据
6.表索引

默认值：ibdata1:12M:autoextend

共享表空间不支持空间收缩，所以如果ibdata1占用了大量的空间，可能会造成问题。
造成ibdata1文件过大比较常见的有2个原因：
1.innodb_file_per_table 没有启用，所有数据和索引都存在此空间中
2.很久之前开启了事务，没有结束，造成此空间中存放了大量的undo页 ，结束掉事务可以阻止此空间继续增大

如何回收共享表空间：
暂时还没有一个快速的方案，只能先mysqldump出数据，然后停止 MySQL 并删除所有数据库、ib_logfile*、ibdata1* 文件。
当你再启动 MySQL 的时候将会创建一个新的共享表空间。然后恢复逻辑备份。相当于把dump出来的文件导入到一个新的mysql
实例中



