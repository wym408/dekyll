---
layout: post
title:  "pt-online-schema-change 过程详解"
date:   2017-07-26 16:15:49 +0700
categories: jekyll update
---
MySQL 大表的DDL操作：加字段、索引等，在5.1之前都是非常耗时耗力的，长时间锁表，特别是会对MySQL服务产生影响，mysql在5.6版本增加了online ddl，但并不是所有的ddl都有效果，再加上5.6主从复制还是单线程，从机上重放主机大表ddl，主从复制的延迟肯定会加大，pt-online-schema-change是一款非常好用的热修改数据表结构工具，个人认为主要解决了2痛点：1.长时间锁表造成服务不可用 2.主从复制延迟造成从机数据不可信任

下载安装可以这里不表，网上可查  

使用限制： 
1.表必须有主键  
2.表上不可有after insert|delete|update 触发器  
3.表如果外键，除非使用 --alter-foreign-keys-method 指定特定的值，否则工具不予执行  

数据准备：
{% highlight doc %}
CREATE TABLE `e` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `v1` int(11) DEFAULT NULL,
  `v2` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=6 DEFAULT CHARSET=utf8mb4;

(root@localhost) [tdb]> select * from e ;
+----+------+------+
| id | v1   | v2   |
+----+------+------+
|  1 |    4 |    1 |
|  3 |    3 |    3 |
|  5 |    1 |    5 |
+----+------+------+
{% endhighlight %}  

执行前日志准备：  
清空general.log:  
{% highlight doc %}
echo > general.log
set global general_log=1 ;
{% endhighlight %}

执行加索引：
{% highlight doc %}
pt-online-schema-change  -uroot -p123456 -h172.16.178.148    --chunk-size 20000  --alter "add key(id)" D=tdb,t=e  --execute 
{% endhighlight %}
![这里写图片描述](http://img.blog.csdn.net/20170725092534361?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd3ltNDA4/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
从输出中可以大致了解执行过程：
创建一个新表，alter新表，创建触发器，copy数据，交换表，删除old表，删除触发器，最后返回成功alter的提示

那么，具体在数据库中是如何操作的呢，前面打开了general_log，现在去查看下general_log文件中的内容：
![这里写图片描述](http://img.blog.csdn.net/20170725093741913?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd3ltNDA4/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
![这里写图片描述](http://img.blog.csdn.net/20170725093758562?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd3ltNDA4/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
![这里写图片描述](http://img.blog.csdn.net/20170725093822993?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd3ltNDA4/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

账号连接进去后设置session基本的timeout和sql_mode,暂时没有看出有何影响，然后获取表的相应信息，应该是检查主键，触发器和是否有外键，最后到执行阶段：

创建新表：
CREATE TABLE `tdb`.`_e_new`

alter 新表：
ALTER TABLE `tdb`.`_e_new` add key(id)

老表增加trigger：
CREATE TRIGGER `pt_osc_tdb_e_del` AFTER DELETE ON `tdb`.`e` 
CREATE TRIGGER `pt_osc_tdb_e_del` AFTER UPDATE ON `tdb`.`e` 
CREATE TRIGGER `pt_osc_tdb_e_del` AFTER INSERTON `tdb`.`e` 

copy数据：
INSERT LOW_PRIORITY IGNORE INTO `tdb`.`_e_new` (`id`, `v1`, `v2`) SELECT `id`, `v1`, `v2` FROM `tdb`.`e` LOCK IN SHARE MODE
此处有锁，测试中看到的是 e表中（e是小表）所有记录加了共享锁，只能读不能写，此处看不出此工具的优势(影响在线修改)。其实此工具在大表copy数据时是分批copy， --chunk-size  设置的是copy数据的大小 ，  
 INSERT LOW_PRIORITY IGNORE INTO `tdb`.`_d_new` (`id`, `v1`, `v2`, `v3`) SELECT `id`, `v1`, `v2`, `v3` FROM `tdb`.`d` FORCE INDEX(`PRIMARY`) WHERE ((`id` >= '982118')) AND ((`id` <= '999999')) LOCK IN SHARE MODE 
 这样锁定就只锁定了 id 一个范围内的数据，不在范围内的不受影响，所以在大表时才能体现优势。
 
 表rename:
  RENAME TABLE `tdb`.`e` TO `tdb`.`_e_old`, `tdb`.`_e_new` TO `tdb`.`e`
这是一个原子操作，2表rename同时成功，且会短暂锁表

drop old 表:
 DROP TABLE IF EXISTS `tdb`.`_e_old`

drop trigger:
DROP TRIGGER IF EXISTS `tdb`.`pt_osc_tdb_e_del`
DROP TRIGGER IF EXISTS `tdb`.`pt_osc_tdb_e_upd`
DROP TRIGGER IF EXISTS `tdb`.`pt_osc_tdb_e_ins`

返回执行成功

查看整个过程，只有2个位置有锁，copy数据分批锁数据，每次都短时间影响少量数据，rename短暂锁表，这样主库上业务基本不受影响，从机的复制也没有大事务造成延迟。

