---
layout: post
title:  "mysql 主从一致性检查和修复"
date:   2017-07-27 16:15:49 +0700
categories: jekyll update
---
本文记录 percona-toolkit 工具中的2个 pt-table-checksum 和 pt-table-sync ,用于主从数据检查和修复  

先来个直观的印象    
{% highlight doc %}
[root@localhost bin]# pt-table-checksum -uroot -p123456 -h172.16.178.152  -P3306 --no-check-binlog-format  --no-check-replication-filters --databases=tdb --tables=b   
            TS ERRORS  DIFFS     ROWS  CHUNKS SKIPPED    TIME TABLE
09-06T18:58:20      0      0       12       1       0   0.009 tdb.b
{% endhighlight %}
认识几个参数：  
--no-check-binlog-format： 默认会检查binlog-format,如果不是statment，就会报错退出，想避免该检查可以设置--no-check-binlog-format  
--no-check-replication-filters： 默认在检查到在主从复制过程中有被用..ignore..过滤掉的表，检查会中断并退出，如果想避开这个检查可以设置--no-check-replication-filters  

执行上面的语句时先打开general_log,执行关闭general_log  
主库general.log    
{% highlight doc %}
Time                 Id Command    Argument
2017-09-07T02:58:21.106484+08:00           69 Connect   root@172.16.178.1 on  using TCP/IP
2017-09-07T02:58:21.106722+08:00           69 Query     set autocommit=1
2017-09-07T02:58:21.107019+08:00           69 Query     SHOW VARIABLES LIKE 'innodb\_lock_wait_timeout'
2017-09-07T02:58:21.108266+08:00           69 Query     SET SESSION innodb_lock_wait_timeout=1
2017-09-07T02:58:21.108439+08:00           69 Query     SHOW VARIABLES LIKE 'wait\_timeout'
2017-09-07T02:58:21.109915+08:00           69 Query     SET SESSION wait_timeout=10000
2017-09-07T02:58:21.110100+08:00           69 Query     SELECT @@SQL_MODE
2017-09-07T02:58:21.110237+08:00           69 Query     SET @@SQL_QUOTE_SHOW_CREATE = 1/*!40101, @@SQL_MODE='NO_AUTO_VALUE_ON_ZERO'*/
2017-09-07T02:58:21.110391+08:00           69 Query     SELECT @@server_id /*!50038 , @@hostname*/
2017-09-07T02:58:21.110533+08:00           69 Query     SELECT @@SQL_MODE
2017-09-07T02:58:21.110634+08:00           69 Query     SET SQL_MODE='NO_AUTO_VALUE_ON_ZERO'
2017-09-07T02:58:21.110796+08:00           69 Query     SHOW VARIABLES LIKE 'version%'
2017-09-07T02:58:21.112058+08:00           69 Query     SHOW ENGINES
2017-09-07T02:58:21.112337+08:00           69 Query     SHOW VARIABLES LIKE 'innodb_version'
2017-09-07T02:58:21.113659+08:00           69 Query     SELECT @@binlog_format
2017-09-07T02:58:21.113772+08:00           69 Query     /*!50108 SET @@binlog_format := 'STATEMENT'*/
2017-09-07T02:58:21.113866+08:00           69 Query     SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ
2017-09-07T02:58:21.114018+08:00           69 Query     SHOW VARIABLES LIKE 'wsrep_on'
2017-09-07T02:58:21.115244+08:00           69 Query     SELECT @@SERVER_ID
2017-09-07T02:58:21.115391+08:00           69 Query     SHOW GRANTS FOR CURRENT_USER()
2017-09-07T02:58:21.115545+08:00           69 Query     SHOW FULL PROCESSLIST
2017-09-07T02:58:21.124158+08:00           69 Query     SHOW VARIABLES LIKE 'wsrep_on'
2017-09-07T02:58:21.125639+08:00           69 Query     SELECT @@SERVER_ID
2017-09-07T02:58:21.127622+08:00           69 Query     SHOW VARIABLES LIKE 'wsrep_on'
2017-09-07T02:58:21.128977+08:00           69 Query     SELECT @@SERVER_ID
2017-09-07T02:58:21.131398+08:00           69 Query     SHOW DATABASES LIKE 'percona'
2017-09-07T02:58:21.131813+08:00           69 Query     CREATE DATABASE IF NOT EXISTS `percona` /* pt-table-checksum */
2017-09-07T02:58:21.132587+08:00           69 Query     USE `percona`
2017-09-07T02:58:21.132777+08:00           69 Query     SHOW TABLES FROM `percona` LIKE 'checksums'
2017-09-07T02:58:21.134789+08:00           69 Query     CREATE TABLE IF NOT EXISTS `percona`.`checksums` (
     db             CHAR(64)     NOT NULL,
     tbl            CHAR(64)     NOT NULL,
     chunk          INT          NOT NULL,
     chunk_time     FLOAT            NULL,
     chunk_index    VARCHAR(200)     NULL,
     lower_boundary TEXT             NULL,
     upper_boundary TEXT             NULL,
     this_crc       CHAR(40)     NOT NULL,
     this_cnt       INT          NOT NULL,
     master_crc     CHAR(40)         NULL,
     master_cnt     INT              NULL,
     ts             TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
     PRIMARY KEY (db, tbl, chunk),
     INDEX ts_db_tbl (ts, db, tbl)
  ) ENGINE=InnoDB DEFAULT CHARSET=utf8
2017-09-07T02:58:21.135986+08:00           69 Query     SHOW GLOBAL STATUS LIKE 'Threads_running'
2017-09-07T02:58:21.137435+08:00           69 Query     SELECT CONCAT(@@hostname, @@port)
2017-09-07T02:58:21.137938+08:00           69 Query     SELECT CRC32('test-string')
2017-09-07T02:58:21.138095+08:00           69 Query     SELECT CRC32('a')
2017-09-07T02:58:21.138250+08:00           69 Query     SELECT CRC32('a')
2017-09-07T02:58:21.138425+08:00           69 Query     SHOW VARIABLES LIKE 'wsrep_on'
2017-09-07T02:58:21.141605+08:00           69 Query     SHOW DATABASES
2017-09-07T02:58:21.142220+08:00           69 Query     SHOW /*!50002 FULL*/ TABLES FROM `tdb`
2017-09-07T02:58:21.142651+08:00           69 Query     /*!40101 SET @OLD_SQL_MODE := @@SQL_MODE, @@SQL_MODE := '', @OLD_QUOTE := @@SQL_QUOTE_SHOW_CREATE, @@SQL_QUOTE_SHOW_CREATE := 1 */
2017-09-07T02:58:21.142868+08:00           69 Query     USE `tdb`
2017-09-07T02:58:21.143093+08:00           69 Query     SHOW CREATE TABLE `tdb`.`b`
2017-09-07T02:58:21.143499+08:00           69 Query     /*!40101 SET @@SQL_MODE := @OLD_SQL_MODE, @@SQL_QUOTE_SHOW_CREATE := @OLD_QUOTE */
2017-09-07T02:58:21.144249+08:00           69 Query     EXPLAIN SELECT * FROM `tdb`.`b` WHERE 1=1
2017-09-07T02:58:21.147607+08:00           69 Query     USE `percona`
2017-09-07T02:58:21.147831+08:00           69 Query     DELETE FROM `percona`.`checksums` WHERE db = 'tdb' AND tbl = 'b'
2017-09-07T02:58:21.149172+08:00           69 Query     USE `tdb`
2017-09-07T02:58:21.149445+08:00           69 Query     EXPLAIN SELECT COUNT(*) AS cnt, COALESCE(LOWER(CONV(BIT_XOR(CAST(CRC32(CONCAT_WS('#', `id`, `v`, CONCAT(ISNULL(`v`)))) AS UNSIGNED)), 10, 16)), 0) AS crc FROM `tdb`.`b` /*explain checksum table*/
2017-09-07T02:58:21.149831+08:00           69 Query     REPLACE INTO `percona`.`checksums` (db, tbl, chunk, chunk_index, lower_boundary, upper_boundary, this_cnt, this_crc) SELECT 'tdb', 'b', '1', NULL, NULL, NULL, COUNT(*) AS cnt, COALESCE(LOWER(CONV(BIT_XOR(CAST(CRC32(CONCAT_WS('#', `id`, `v`, CONCAT(ISNULL(`v`)))) AS UNSIGNED)), 10, 16)), 0) AS crc FROM `tdb`.`b` /*checksum table*/
2017-09-07T02:58:21.150571+08:00           69 Query     SHOW WARNINGS
2017-09-07T02:58:21.150802+08:00           69 Query     SELECT this_crc, this_cnt FROM `percona`.`checksums` WHERE db = 'tdb' AND tbl = 'b' AND chunk = '1'
2017-09-07T02:58:21.150966+08:00           69 Query     UPDATE `percona`.`checksums` SET chunk_time = '0.000746', master_crc = 'd925b6b4', master_cnt = '12' WHERE db = 'tdb' AND tbl = 'b' AND chunk = '1'
2017-09-07T02:58:21.152234+08:00           69 Query     SHOW GLOBAL STATUS LIKE 'Threads_running'
2017-09-07T02:58:21.154257+08:00           69 Quit
2017-09-07T03:03:48.532074+08:00           70 Connect   root@localhost on tdb using Socket
2017-09-07T03:03:48.536051+08:00           70 Query     show databases
2017-09-07T03:03:48.537620+08:00           70 Query     show tables
2017-09-07T03:03:48.538226+08:00           70 Field List        a 
2017-09-07T03:03:48.538657+08:00           70 Field List        b 
2017-09-07T03:03:48.539268+08:00           70 Query     set global general_log = 0
{% endhighlight %}
分析下期中几个关键步骤：  
SET @@binlog_format = 'STATEMENT'： 该工具能得出主从是否一致所依赖的就是statement基础上同样的SQL语句在主从库上各自的执行结果，主库进行检查后sql语句传给从库，从库执行一遍后，也得到自己的结果  
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ ：通过加间隙锁保证在取数据行数（replaceinto select  from ）到计算出检验值这段时间数据不会被修改  
CREATE DATABASE IF NOT EXISTS `percona`   
CREATE TABLE IF NOT EXISTS `percona`.`checksums`： 创建校验值存放的库和表  
DELETE FROM `percona`.`checksums` WHERE db = 'tdb' AND tbl = 'b' ： 把以前跑的结果删除  
EXPLAIN ：期中很多的explain用来分析选取适当大小的chunk块，测试中数据量比较小，无法看出chunk的左右  
REPLACE INTO `percona`.`checksums` (db, tbl, chunk, chunk_index, lower_boundary, upper_boundary, this_cnt, this_crc) SELECT 'tdb', 'b', '1', NULL, NULL, NULL, COUNT(*) AS cnt, COALESCE(LOWER(CONV(BIT_XOR(CAST(CRC32(CONCAT_WS('#', `id`, `v`, CONCAT(ISNULL(`v`)))) AS UNSIGNED)), 10, 16)), 0) AS crc FROM `tdb`.`b` /*checksum table*/ ： 这句是核心语句，计算选取出的chunk块中数据的校验值， session 级别binlog 是STATEMENT，同样的语句会在从机上执行一次，如果数据一致，得到的校验值也一致，该工具就是利用这个值来判断主从的数据是否一致  
UPDATE `percona`.`checksums` SET chunk_time = '0.000746', master_crc = 'd925b6b4', master_cnt = '12' WHERE db = 'tdb' AND tbl = 'b' AND chunk = '1'：master_crc，master_cnt 更新值，通过binlog传送给从机  

从机general.log  
{% highlight doc %}
2017-09-07T02:58:21.133855+08:00           71 Connect   root@172.16.178.1 on  using TCP/IP
2017-09-07T02:58:21.134096+08:00           71 Query     set autocommit=1
2017-09-07T02:58:21.134322+08:00           71 Query     SHOW VARIABLES LIKE 'innodb\_lock_wait_timeout'
2017-09-07T02:58:21.136416+08:00           71 Query     SET SESSION innodb_lock_wait_timeout=1
2017-09-07T02:58:21.136600+08:00           71 Query     SHOW VARIABLES LIKE 'wait\_timeout'
2017-09-07T02:58:21.138054+08:00           71 Query     SET SESSION wait_timeout=10000
2017-09-07T02:58:21.138196+08:00           71 Query     SELECT @@SQL_MODE
2017-09-07T02:58:21.138356+08:00           71 Query     SET @@SQL_QUOTE_SHOW_CREATE = 1/*!40101, @@SQL_MODE='NO_AUTO_VALUE_ON_ZERO'*/
2017-09-07T02:58:21.138474+08:00           71 Query     SELECT @@SERVER_ID
2017-09-07T02:58:21.138857+08:00           71 Query     SELECT @@server_id /*!50038 , @@hostname*/
2017-09-07T02:58:21.139004+08:00           71 Query     SHOW VARIABLES LIKE 'wsrep_on'
2017-09-07T02:58:21.140353+08:00           71 Query     SHOW GRANTS FOR CURRENT_USER()
2017-09-07T02:58:21.140531+08:00           71 Query     SHOW FULL PROCESSLIST
2017-09-07T02:58:21.140757+08:00           71 Query     SHOW SLAVE HOSTS
2017-09-07T02:58:21.142717+08:00           71 Query     SHOW VARIABLES LIKE 'wsrep_on'
2017-09-07T02:58:21.144291+08:00           71 Query     SELECT @@SERVER_ID
2017-09-07T02:58:21.146005+08:00           71 Query     SHOW VARIABLES LIKE 'wsrep_on'
2017-09-07T02:58:21.147979+08:00           71 Query     SELECT @@SERVER_ID
2017-09-07T02:58:21.150186+08:00            2 Query     CREATE DATABASE IF NOT EXISTS `percona` /* pt-table-checksum */
2017-09-07T02:58:21.152441+08:00           71 Query     SHOW TABLES FROM `percona` LIKE 'checksums'
2017-09-07T02:58:21.153075+08:00            2 Query     CREATE TABLE IF NOT EXISTS `percona`.`checksums` (
     db             CHAR(64)     NOT NULL,
     tbl            CHAR(64)     NOT NULL,
     chunk          INT          NOT NULL,
     chunk_time     FLOAT            NULL,
     chunk_index    VARCHAR(200)     NULL,
     lower_boundary TEXT             NULL,
     upper_boundary TEXT             NULL,
     this_crc       CHAR(40)     NOT NULL,
     this_cnt       INT          NOT NULL,
     master_crc     CHAR(40)         NULL,
     master_cnt     INT              NULL,
     ts             TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
     PRIMARY KEY (db, tbl, chunk),
     INDEX ts_db_tbl (ts, db, tbl)
  ) ENGINE=InnoDB DEFAULT CHARSET=utf8
2017-09-07T02:58:21.154565+08:00           71 Query     SELECT CONCAT(@@hostname, @@port)
2017-09-07T02:58:21.161967+08:00           71 Query     SHOW TABLES FROM `tdb` LIKE 'b'
2017-09-07T02:58:21.162410+08:00           71 Query     /*!40101 SET @OLD_SQL_MODE := @@SQL_MODE, @@SQL_MODE := '', @OLD_QUOTE := @@SQL_QUOTE_SHOW_CREATE, @@SQL_QUOTE_SHOW_CREATE := 1 */
2017-09-07T02:58:21.162647+08:00           71 Query     USE `tdb`
2017-09-07T02:58:21.162869+08:00           71 Query     SHOW CREATE TABLE `tdb`.`b`
2017-09-07T02:58:21.163348+08:00           71 Query     /*!40101 SET @@SQL_MODE := @OLD_SQL_MODE, @@SQL_QUOTE_SHOW_CREATE := @OLD_QUOTE */
2017-09-07T02:58:21.163802+08:00           71 Query     EXPLAIN SELECT * FROM `tdb`.`b` WHERE 1=1
2017-09-07T02:58:21.166210+08:00            2 Query     BEGIN
2017-09-07T02:58:21.166575+08:00            2 Query     DELETE FROM `percona`.`checksums` WHERE db = 'tdb' AND tbl = 'b'
2017-09-07T02:58:21.167117+08:00            2 Query     COMMIT
2017-09-07T02:58:21.167582+08:00            2 Query     BEGIN
2017-09-07T02:58:21.167967+08:00            2 Query     REPLACE INTO `percona`.`checksums` (db, tbl, chunk, chunk_index, lower_boundary, upper_boundary, this_cnt, this_crc) SELECT 'tdb', 'b', '1', NULL, NULL, NULL, COUNT(*) AS cnt, COALESCE(LOWER(CONV(BIT_XOR(CAST(CRC32(CONCAT_WS('#', `id`, `v`, CONCAT(ISNULL(`v`)))) AS UNSIGNED)), 10, 16)), 0) AS crc FROM `tdb`.`b` /*checksum table*/
2017-09-07T02:58:21.167987+08:00            2 Query     COMMIT /* implicit, from Xid_log_event */
2017-09-07T02:58:21.168637+08:00            2 Query     BEGIN
2017-09-07T02:58:21.168762+08:00           71 Query     SHOW SLAVE STATUS
2017-09-07T02:58:21.168811+08:00            2 Query     UPDATE `percona`.`checksums` SET chunk_time = '0.000746', master_crc = 'd925b6b4', master_cnt = '12' WHERE db = 'tdb' AND tbl = 'b' AND chunk = '1'
2017-09-07T02:58:21.168834+08:00            2 Query     COMMIT /* implicit, from Xid_log_event */
2017-09-07T02:58:21.170218+08:00           71 Query     SELECT MAX(chunk) FROM `percona`.`checksums` WHERE db='tdb' AND tbl='b' AND master_crc IS NOT NULL
2017-09-07T02:58:21.170625+08:00           71 Query     SELECT CONCAT(db, '.', tbl) AS `table`, chunk, chunk_index, lower_boundary, upper_boundary, COALESCE(this_cnt-master_cnt, 0) AS cnt_diff, COALESCE(this_crc <> master_crc OR ISNULL(master_crc) <> ISNULL(this_crc), 0) AS crc_diff, this_cnt, master_cnt, this_crc, master_crc FROM `percona`.`checksums` WHERE (master_cnt <> this_cnt OR master_crc <> this_crc OR ISNULL(master_crc) <> ISNULL(this_crc))  AND (db='tdb' AND tbl='b')
2017-09-07T02:58:21.171067+08:00           71 Quit
2017-09-07T03:03:53.821364+08:00           72 Connect   root@localhost on tdb using Socket
2017-09-07T03:03:53.825534+08:00           72 Query     show databases
2017-09-07T03:03:53.827534+08:00           72 Query     show tables
2017-09-07T03:03:53.828234+08:00           72 Field List        a 
2017-09-07T03:03:53.828896+08:00           72 Field List        b
{% endhighlight %} 
分析下期中几个关键步骤：   
SHOW SLAVE HOSTS： 检查主从同步延迟，如果大于 max-lag(默认1s)，主机下一次的chunk  replace into 先等待，直到主从延迟小于max-lag再继续执行
CREATE DATABASE IF NOT EXISTS `percona`   
CREATE TABLE IF NOT EXISTS `percona`.`checksums  
DELETE FROM `percona`.`checksums` WHERE db = 'tdb' AND tbl = 'b'  
上面三句是主机上通过statement形式的binlog传送到从机上  
REPLACE INTO `percona`.`checksums` (db, tbl, chunk, chunk_index, lower_boundary, upper_boundary, this_cnt, this_crc) SELECT 'tdb', 'b', '1', NULL, NULL, NULL, COUNT(*) AS cnt, COALESCE(LOWER(CONV(BIT_XOR(CAST(CRC32(CONCAT_WS('#', `id`, `v`, CONCAT(ISNULL(`v`)))) AS UNSIGNED)), 10, 16)), 0) AS crc FROM `tdb`.`b` /*checksum table*/ ：这句也是binlog传送过来的，验证主从数据一致性的核心语句  
SELECT CONCAT(db, '.', tbl) AS `table`, chunk, chunk_index, lower_boundary, upper_boundary, COALESCE(this_cnt-master_cnt, 0) AS cnt_diff, COALESCE(this_crc <> master_crc OR ISNULL(master_crc) <> ISNULL(this_crc), 0) AS crc_diff, this_cnt, master_cnt, this_crc, master_crc FROM `percona`.`checksums` WHERE (master_cnt <> this_cnt OR master_crc <> this_crc OR ISNULL(master_crc) <> ISNULL(this_crc))  AND (db='tdb' AND tbl='b') ： 期中master_crc是通过binlog传送过来的主机值，this_crc是从机上用replace into select from 计算出来的，两值不等表明主从数据不一致  

从上面的分析大概能看出，pt-table-checksum一次只针对一个表，而且会根据表的大小以及当前系统的繁忙程度，计算出一次检查中能包含的数据行数，来尽量避免对线上服务的影响，如果在执行过程中遇到突发的负载增加，还会自动的将检查停下来等待，所以即使面对成千上万的数据库和表时，它也能顺利的进行检查  

在检查过程中，工具会随时对主从连接情况进行检查，如果从库延迟太大，主从复制中断，检查会停下来等待；这里需要注意的是主从复制过滤，因为这种情形下，主从数据库中的库表存在情况不一致，检查过程中的执行语句会与当前的主从复制过程冲突导致主从复制进程失败，所以如果有过滤存在，需要指定参数--no-check-replication-filters  

在一个块的数据被检查之前，会先执行explain操作，来确定执行该检查的安全性，如果太大不能在指定时间内完成检查的话就会将该块数据跳过，另外，如果主库上整表的数据特别少或干脆是空表，并不会直接将整表当做一个块去检查，而是会再去从库，确定从库中也是有同样少的数据，避免从库表数据太多却被当成一个块执行造成的从库数据阻塞  

另外还有一些安全保护设置，在上面的执行流程中已经列出来了，如设置innodb_lock_wait_timeout=1，如果锁等待超过1S，就放弃此次执行  

在执行过程中如果遇到任何异常，可随时中断进程，如kill或CTRL-C，不会造成任何影响，后面想从此次中断继续检查时，简单的采用--resume就可以  

在从机上删除1行数据
{% highlight doc %}
delete from b where id=1 ;  

[root@localhost ~]# pt-table-checksum -uroot -p123456 -h172.16.178.152  -P3306 --no-check-binlog-format  --no-check-replication-filters --databases=tdb --tables=b 
            TS ERRORS  DIFFS     ROWS  CHUNKS SKIPPED    TIME TABLE
09-07T09:52:29      0      1       12       1       0   0.275 tdb.b
{% endhighlight %}
TS ：完成检查的时间戳。  
ERRORS ：检查时候发生错误和警告的数量。  
DIFFS ：不一致的chunk数量。  
ROWS ：比对的表行数。  
CHUNKS ：被划分到表中的块的数目。  
SKIPPED ：由于错误或警告或过大，则跳过块的数目。  
TIME ：执行的时间。  
TABLE ：被检查的表名。  

DIFFS =1 ，说明数据不一致

修复数据：  
{% highlight doc %}
 pt-table-sync  --replicate=percona.checksums  --databases=tdb --tables=a h=172.16.178.152,u=root,p=123456 h=172.16.178.153,u=root,p=123456 --print    
REPLACE INTO `tdb`.`a`(`id`, `v`) VALUES ('1', 'a') /*percona-toolkit src_db:tdb src_tbl:a src_dsn:h=172.16.178.152,p=...,u=root dst_db:tdb dst_tbl:a dst_dsn:h=172.16.178.153,p=...,u=root lock:1 transaction:1 changing_src:percona.checksums replicate:percona.checksums bidirectional:0 pid:374 user:root host:localhost.localdomain*/;
{% endhighlight %}
pt-table-sync 工具   
--replicate=percona.checksums：之前产生的不一致信息  
--databases=tdb --tables=a： 需要修复的库和表  
h=172.16.178.152,u=root,p=123456： 前面一个是主库信息  
h=172.16.178.153,u=root,p=123456： 后面一个是从库信息，也可以省略  
--print：展示修复sql，不执行  
--execute: 直接修复  

主机general.log  
{% highlight doc %}
 SELECT /*rows in chunk*/ `id`, `v`, CRC32(CONCAT_WS('#', `id`, `v`, CONCAT(ISNULL(`v`)))) AS __crc FROM `tdb`.`a` FORCE INDEX (`PRIMARY`) WHERE (`id` >= '1') AND (((`id` >= '1')) AND ((`id` <= '3340'))) ORDER BY `id` FOR UPDATE
{% endhighlight %}
从库general.log  
{% highlight doc %}
  SELECT db, tbl, CONCAT(db, '.', tbl) AS `table`, chunk, chunk_index, lower_boundary, upper_boundary, COALESCE(this_cnt-master_cnt, 0) AS cnt_diff, COALESCE(this_crc <> master_crc OR ISNULL(master_crc) <> ISNULL(this_crc), 0) AS crc_diff, this_cnt, master_cnt, this_crc, master_crc FROM percona.checksums WHERE master_cnt <> this_cnt OR master_crc <> this_crc OR ISNULL(master_crc) <> ISNULL(this_crc)

 SELECT /*rows in chunk*/ `id`, `v`, CRC32(CONCAT_WS('#', `id`, `v`, CONCAT(ISNULL(`v`)))) AS __crc FROM `tdb`.`a` FORCE INDEX (`PRIMARY`) WHERE (`id` >= '1') AND (((`id` >= '1')) AND ((`id` <= '3340'))) ORDER BY `id` LOCK IN SHARE MODE
{% endhighlight %}
找到chunk块不一致的数据，然后把期中的所以数据拉出来对比，主从不一致的数据对从库进行修复  

上面只是打印了修复sql，也可以直接修复，修复完再检查 

{% highlight doc %}
[root@localhost ~]# pt-table-sync  --replicate=percona.checksums  --databases=tdb --tables=a h=172.16.178.152,u=root,p=123456 h=172.16.178.153,u=root,p=123456 --execute  

[root@localhost ~]# pt-table-checksum -uroot -p123456 -h172.16.178.152  -P3306 --no-check-binlog-format  --no-check-replication-filters --databases=tdb --tables=a           
            TS ERRORS  DIFFS     ROWS  CHUNKS SKIPPED    TIME TABLE
09-07T11:02:02      0      0  6815744      12       0   6.106 tdb.a
{% endhighlight %}
修复完检查 DIFFS=0

 

