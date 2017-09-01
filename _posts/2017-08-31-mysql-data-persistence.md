---
layout: post
title:  "mysql 数据落盘整理"
date:   2017-08-30 16:15:49 +0700
categories: jekyll update
---
本文记录mysql数据落盘

Redo_Log:    
WAL(write_ahead_redo_log)保证了innodb数据的持久性   
redo log 中不仅包含了表和索引数据，还包含了undo数据，所以也保证了undo数据的持久化   
Log sequence number >= Log flushed up to >= Pages flushed up to >= Last checkpoint at  
innodb_log_buffer_size 控制redo log缓冲池大小  
何时redo buffer中的数据会落地到磁盘  
1.master thread 每秒刷新到磁盘，不管事务是否提交  
2.commit 时，和 innodb_flush_log_at_trx_commit 参数值相关  
  0：不落盘  
  1：落盘，调用fsync()，不丢失事务  
  2：落到OS系统缓存，依靠操作系统再落地到磁盘，数据库crash不会丢失数据，操作系统crash会丢失数据  
3.log buffer 剩余空间小于1/2 


Binlog:  
binlog 是MySQL Server 层日志，和redo log 无直接关系，而且可以关闭  
sync_binlog = N ,控制落盘的频次，commit N 此 binlog 落盘1次  
sync_binlog = 1 ，每次提交都持久化binlog ，如果更新大量数据且binlog_rowmat=ROW,commit 是需要等待的  
当binlog 打开时，且 innodb_support_xa =1 ，用分布式事务控制 binlog 和 redo log   

redo log 是innodb 独立使用，存储的数据页的物理更改情况  
binlog 是mysql 的所有操作数据变更日志，存储的是逻辑日志  
当 innodb_support_xa =1 ，sync_binlog = 1，innodb_flush_log_at_trx_commit=1 才能保证数据是真正安全的   
sync_binlog 非1，当OS crash 时，binlog会丢失数据，造成binlog 和 innodb 数据不一致  
innodb_flush_log_at_trx_commit 非1 当 OS crash 时，innodb 会丢失数据， 造成 innodb数据和 binlog 不一致  
innodb_support_xa 非 1 时，无法用分布式事务保证 innodb 和 binlog 数据的一致性  
binlog 和 innodb 数据不一致会造成主从数据的不一致  

Data：  
checkpoint 技术  
Data脏页落盘不管对应的事务是否commit  
1.sharp point  
  数据库关闭时刷盘，即innodb_fast_shutdown=1  
2.fuzzy point
落盘的数据对应的redo log 必须已经落盘  
  .Master thread 1s 和 10s  
  .LRU 没有足够的空闲页  
  .redo log 没有足够的可复用空间,释放redo log 空间的前提是对应的 data 数据必须已经落盘   
  .脏页太多，脏页率 > innodb_max_dirty_pages_pct  

脏页落盘用到 double write 技术，增加安全性，先顺序IO写的ibddata1，再随机IO合并到idb  
1.先把 dirty page memcpy 到 2M大小的 double write buffer  
2.分2次把 double write buffer 中的数据顺序IO写入到共享表空间，每次1M，并且立刻调用fsync()  
3.再把double write buffer 中的数据离散的写入到相应  

Undo:  
没有找到undo 页落盘的详细资料  
根据 redo log 中有用WAL 方式持久化undo 数据页，猜测undo 数据页的持久化原理和 Data数据页一致  
Undo页中包含事务信息，事务ID和事务状态，用于回滚和crash recovery  

