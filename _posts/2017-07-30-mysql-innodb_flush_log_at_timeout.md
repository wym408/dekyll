---
layout: post
title:  "mysql 参数 innodb_flush_log_at_timeout & innodb_flush_log_at_trx_commit"
date:   2017-07-27 16:15:49 +0700
categories: jekyll update
---
mysql innodb redo log 刷新到disk的方式  


innodb的redo log 先写入log buffer，然后再存入disk,  
其中存入disk有2种方式，以是否使用os buffer为不同，由innodb_flush_method参赛控制，在类nuix系统上一般都是用innodb_flush_method=O_DIRECT，也就是使用os buffer.

使用os buffer刷新redo log，又分为3中情况，用innodb_flush_log_at_trx_commit参数控制，值分别为0,1,2  
innodb_flush_log_at_trx_commit=1是默认值，即事务commit时，都把log buffer 刷新到os buffer，并且从os buffer中刷新到disk上.
 
innodb_flush_log_at_trx_commit=0，每隔1s把log buffer中的数据刷新到os buffer，并请从os buffer中刷新到disk上，也就是1s以内commit完的数据还在log buffer上，如果mysqld或者操作系统crash，则最多可能丢失1s的数据.
  
innodb_flush_log_at_trx_commit=2， 事务commit时，把log buffer 刷新到os buffer，每隔1s 把os buffer中数据刷新到disk 上，也就是1s以内的commit完数据存在于os buffer，如果操作系统crash 则最多可能丢失1s的数据，如果只是mysqld crash则不会丢失数据.  

![这里写图片描述](http://img.blog.csdn.net/20170730131205310?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd3ltNDA4/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)  

上面提到的1s，即innodb_flush_log_at_timeout的默认值，可用此参数修改.  
为了数据的安全性，innodb_flush_log_at_trx_commit使用默认值，则innodb_flush_log_at_timeout就没有意义了.

