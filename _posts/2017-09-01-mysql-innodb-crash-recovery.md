---
layout: post
title:  "MySQL InnoDB crash recovery"
date:   2017-08-14 16:15:49 +0700
categories: jekyll update
---
本文记录MySQL InnoDB crash recovery 相关过程，涉及到redo log，undo log，binlog  

基础知识：  
LSN：Log sequence number ，可以理解为数据库创建以来产生的redo log 的量，值越大表示数据库更新越多，也可以理解为redo log 产生的时间，值越大表示产生的时间越晚。每个数据页上也有一个LSN，表示最后更新时的LSN，值越大表示越晚被修改。比如数据页A LSN 为100，数据页B LSN 为200，checkpoint LSN 为150，系统LSN 为300 ，系统当前已经更新到300，LSN小于150的数据页A已经被刷新到磁盘，LSN大于200的页B还在buffer中  
Log sequence number（LSN1）：当前系统LSN最大值，新的事务日志LSN将在此基础上生成（LSN1+新日志的大小）     
Log flushed up to（LSN2）：当前已经写入日志文件的LSN  
Pages flushed up to（LSN3）：当前最旧的脏页数据对应的LSN，写Checkpoint的时候直接将此LSN写入到日志文件  
Last checkpoint at（LSN4）：当前已经写入Checkpoint的LSN  
LSN1>=LSN2>=LSN3>=LSN4  
LSN3和LSN4 的区别没有看懂，暂时按照一个相同含义来理解  

Redo log: InnoDB 修改一条数据，先写Redo log，再写数据页，在Redo log 成功后就可以返回给用户成功。写数据页随机IO是后台进程慢慢进行，不影响用户体验，写Redo log 是顺序IO（非严格顺序IO，Redo log file 前2KB保存了一些其他信息比如checkpoint需要随机IO），速度很快，用户体验很好。Redo log 是对没有刷新到磁盘数据页的保护，在数据页刷新到磁盘后对应的Redo log 数据库就可以被重用了。Redo log 中同样包含了Undo log 信息，Redo log 的持久化同样保护了undo log，undo log 不需要立即持久化。Redo log 的持久化有3个触发点：  
1.Master 线程，1s 和 10s 循环都会持久化Redo log ，不管是否commit  
2.commit时，保证相应事务的Redo log 持久化（innodb_flush_log_at_trx_commit=1）  
3.redo_log_buffer 剩余空间小于1/2  
1和3 触发点会把没有commit的redo log 持久化。数据库 crash 后可用Redo log 把没有持久化的数据页和undo log 数据页恢复出来，保证数据不会丢失。  

Undo log：InnoDB支持事务，有 rollback 操作，同时为了挺高并发支持MVCC功能，这2个功能都依赖 undo log 。为了实现统一的管理，与redo日志不同，undo日志在Buffer Pool中有对应的数据页，与普通的数据页一起管理，依据LRU规则也会被淘汰出内存，后续再从磁盘读取。与普通的数据页一样，对undo页的修改，也需要先写redo日志。 undo log 中保存了事务ID 和 状态，这个对后续的crash recovery 很关键。

checkpoint：数据库为了提高性能，在内存中修改了数据页后，不会立刻刷新到磁盘，而是通过checkpoint机制。checkpoint之前的数据页一定已经刷新到磁盘，对应的redo log 就没用了（redo log 文件循环使用，这部分可以被覆盖）。checkpoint 之后的脏页可能刷新到了磁盘，也可能没有刷新到磁盘，所以checkpoint之后的redo log 在crash recovery时需要用到。InnoDB已经脏页情况定期推荐checkpoint，从而减少数据库崩溃后恢复需要的时间，chenkpoint存在redo log file 第一个文件的头部。checkpoint 触发条件有4点：  
1.Master 1s 循环  
2.LRU 没有足够的空闲页  
3.redo log 没有足够的可复用空间,释放redo log 空间的前提是对应的 data 数据必须已经落盘  
4.脏页太多，脏页率 > innodb_max_dirty_pages_pct  

binlog: MySQL binlog 是server层控制，不光记录InnoDB的数据变更，还记录其他引擎的数据变化。binlog 可以用来做主从复制，使从机和主机数据一致，从而实现负载均衡，读写分离，高可用等功能。binlog在commit是持久化（sync_binlog=1）,持久化后这个数据就可以发送到从机。如果binlog记录成功了，redo log 没有持久化，此时数据库crash，再恢复起来时，就会主从数据不一致。因此InnoDB commit时使用了XA事务处理redo log 和binlog 。步骤：  
1. redo log  prepare  
2. binlog 持久化  
3. redo log 持久化  
这个地方看到过其他版本（1. InnoDB prepare 2.binlog 持久化 3.修改事务对应的信息，日志写入redo log buffer，再fsync 这个版本感觉没有理解，后续再慢慢参悟）  

crash recovery：  
1.前滚  
实际就是执行redo log，checkpoint前的数据已经落盘，所以只要把redo log 中LSN从Last checkpoint 到Log flushed up to 拿出来重放就可以，这部分redo 对应的数据页没有落,redo 根据数据中的space id +page no 等信息把对应的数据页读到buffer ，再把redo 重放，恢复出最新的数据页。redo log 执行时不考虑事务状态信息，全部重复，考虑到redo log 落盘的数据中是有未commit数据，所以执行完，数据处于超前状态。所以才有下个阶段：回滚    

2.回滚  
通过undo log 可以构造出无提交事务链表，有insert_undo_list 和 update_undo_list 2个列表 trx_list
如果没打开binlog，则可以直接全部rollback  
如果打开了binlog,情况稍微复杂，通过最后一个binlog文件构建出binlog 已经提交的 xid 列表（binlog文件切换时要求上个文件对应的所有redo log 已经落盘），分不同情况确定是rollback or commit  
trx_list 中有事务tid，而 xid 列表中没有事务tid ： 回滚事务  
trx_list 中有事务tid，而 xid 列表中也有事务tid ： 提交事务

通过上面2个阶段才最终完成了recovery过程，数据状态恢复到crash前的状态   


