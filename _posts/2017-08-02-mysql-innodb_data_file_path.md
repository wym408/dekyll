---
layout: post
title:  "mysql 参数 innodb_data_file_path & innodb_file_per_table"
date:   2017-07-27 16:15:49 +0700
categories: jekyll update
---
innodb_data_file_path：共享表空间位置和大小设定  
innodb_file_per_table： 独立表空间 


当启用了 innodb_file_per_table ，单表数据和索引数据被存在了自己的表空间里，但是共享表空间仍然在存储其它的 InnoDB 内部数据：  
1.数据字典，也就是 InnoDB 表的元数据  
2.变更缓冲区  
3.双写缓冲区  
4.撤销日志  
如果没有启用 innodb_file_per_table，则此空间中还包含：  
5.表数据  
6.表索引  

默认值：ibdata1:12M:autoextend  
名字是 ibdata1 ,初始化大小12M,需要时自动扩容增大  

启用innodb_file_per_table优点：  
1.每个表都有自已独立的表空间  
2.每个表的数据和索引都会存在自已的表空间中   
3.表空间可回收(delete 不能回收)   
  drop table  
  truncate table  
  alter table engine=innodb,顺便可以整理碎片  
4.delete 造成的碎片不会影响其他表，而且有机会整理碎片  
启用innodb_file_per_table ，合理调大innodb_open_files    

共享表空间不支持空间收缩，所以如果ibdata1占用了大量的空间，可能会造成问题。  
造成ibdata1文件过大比较常见的有2个原因：  
1.innodb_file_per_table 没有启用，所有数据和索引都存在此空间中，所以建议都要启用innodb_file_per_table  
2.大事务或一个长时间的事务没有结束，造成此空间中存放了大量的undo页 ，结束掉事务可以阻止此空间继续增大   

如何回收共享表空间：
暂时还没有一个快速的方案，只能先mysqldump出数据，然后停止 MySQL 并删除所有数据库、ib_logfile*、ibdata1* 文件。
当你再启动 MySQL 的时候将会创建一个新的共享表空间。然后恢复逻辑备份。相当于把dump出来的文件导入到一个新的mysql
实例中  

查看共享表空间中数据分类  
mysql自带工具innochecksum可以查看innodb表空间中数据,缺点是必须离线才能使用，即停止实例，或者copy数据文件  
{% highlight ruby %}
innochecksum  --page-type-summary  ibdata1_bk 

File::ibdata1_bk
================PAGE TYPE SUMMARY==============
#PAGE_COUNT     PAGE_TYPE
===============================================
     102        Index page
      17        Undo log page
       2        Inode page
       0        Insert buffer free list page
     628        Freshly allocated page
       1        Insert buffer bitmap
      15        System page
       2        Transaction system page
       1        File Space Header
       0        Extent descriptor page
       0        BLOB page
       0        Compressed BLOB page
       0        Other type of page
===============================================
Additional information:
Undo page type: 6 insert, 11 update, 0 other
Undo page state: 0 active, 17 cached, 0 to_free, 0 to_purge, 0 prepared, 0 other

[root@mysql1 tdb]# cp c.ibd c.ibd_bk
[root@mysql1 tdb]# 
[root@mysql1 tdb]# innochecksum  --page-type-summary  c.ibd_bk  

File::c.ibd_bk
================PAGE TYPE SUMMARY==============
#PAGE_COUNT     PAGE_TYPE
===============================================
   41695        Index page
       0        Undo log page
       1        Inode page
       0        Insert buffer free list page
    1050        Freshly allocated page
       3        Insert buffer bitmap
       0        System page
       0        Transaction system page
       1        File Space Header
       2        Extent descriptor page
       0        BLOB page
       0        Compressed BLOB page
       0        Other type of page
===============================================
Additional information:
Undo page type: 0 insert, 0 update, 0 other
Undo page state: 0 active, 0 cached, 0 to_free, 0 to_purge, 0 prepared, 0 other
{% endhighlight %}

留一个学习连接,检查innodb内部结构的工具：[innodb_ruby][1]  

[1]: https://github.com/jeremycole/innodb_ruby

