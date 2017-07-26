---   
layout: post  
title:  "mysql 5.7.16 安装"   
date:   2017-07-26 16:15:49 +0700   
categories: jekyll update   
---      
准备工作
配置文件     
安装    
   
------------------   
   
**准备工作**     
1.centos6 虚拟机(安装过程不表，有空再补)   
2.mysql 5.7.16 二进制tar包   
下载地址:[https://downloads.mysql.com/archives/community/](https://downloads.mysql.com/archives/community/)   
   wget [https://cdn.mysql.com/archives/mysql-5.7/mysql-5.7.16-linux-glibc2.5-x86_64.tar.gz](https://cdn.mysql.com/archives/mysql-5.7/mysql-5.7.16-linux-glibc2.5-x86_64.tar.gz)   
  下载完在 /root   
  选择mysql版本，操作系统以及操作系统位数，如下图:   
![这里写图片描述](http://img.blog.csdn.net/20170724085348517?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd3ltNDA4/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)   
     
**配置文件**   
测试机器配置：2核cpu，2G内存 虚拟机   
参考姜大神最佳配置:[https://github.com/jdaaaaaavid/mysql_best_configuration/blob/master/my.cnf](https://github.com/jdaaaaaavid/mysql_best_configuration/blob/master/my.cnf)   
   
vim /etc/my.cnf   
   
[mysqld]   
datadir=/home/data/mysqldata #数据文件目录   
user=mysql                   #server端启动用户   
port=3306                      
autocommit = 1   
character_set_server=utf8mb4   
sql_mode = "STRICT_TRANS_TABLES,NO_ENGINE_SUBSTITUTION,NO_ZERO_DATE,NO_ZERO_IN_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER"   
transaction_isolation = READ-COMMITTED  #建议用RC(间隙锁原因)    
explicit_defaults_for_timestamp = 1   
max_allowed_packet = 16777216   
   
\# connection #   
interactive_timeout = 180  #指的是mysql在关闭一个交互的连接之前所要等待的秒数   
wait_timeout = 180  #指的是MySQL在关闭一个非交互的连接之前所要等待的秒数   
lock_wait_timeout = 1800 # 数据结构ddl操作的锁的等待时间   
skip_name_resolve = 1   
max_connections = 512   
max_connect_errors = 1000000   
   
\# table cache performance settings   
table_open_cache = 4096   
table_definition_cache = 4096   
table_open_cache_instances = 128   
   
\# session memory settings #   
read_buffer_size = 16M   
read_rnd_buffer_size = 16M   
sort_buffer_size = 16M   
tmp_table_size = 32M   
join_buffer_size = 128M   
thread_cache_size = 64   
   
\#######################   
log_error = error.log   
slow_query_log = 1   
slow_query_log_file = slow.log   
log_queries_not_using_indexes = 1   
log_slow_admin_statements = 1 # alter    
log_slow_slave_statements = 1   
log_throttle_queries_not_using_indexes = 10  # per minutes limit   
long_query_time = 1   
min_examined_row_limit = 100   
binlog-rows-query-log-events = 1   
log-bin-trust-function-creators = 1   
log-slave-updates = 1   
general_log_file = general.log   
   
\###############password##############   
\#plugin_load = "validate_password.so"   
\#validate_password_policy=MEDIUM   
\#validate-password=FORCE_PLUS_PERMANENT   
\#validate_password_dictionary_file=/home/data/mysqldata/password.dicionary   
   
   
\#####innodb#######   
innodb_page_size = 16384   
innodb_buffer_pool_size = 1G #服务器内存的百分70左右   
innodb_buffer_pool_instances = 2   
innodb_buffer_pool_load_at_startup = 1   
innodb_buffer_pool_dump_at_shutdown = 1   
innodb_lru_scan_depth = 4096   
innodb_lock_wait_timeout = 5 #innodb 事务锁等待时间   
innodb_io_capacity = 400 #测试机比较烂,可根据服务器调整   
innodb_io_capacity_max = 2000  #同上   
innodb_flush_method = O_DIRECT   
innodb_file_format = Barracuda   
innodb_file_format_max = Barracuda   
innodb_undo_logs = 128   
innodb_undo_tablespaces = 3   
\#hhd   
innodb_flush_neighbors=1   
\#ssd   
\#innodb_flush_neighbors=0   
innodb_log_file_size=1G   
innodb_purge_threads = 4   
innodb_large_prefix = 1   
innodb_thread_concurrency = 64   
innodb_print_all_deadlocks = 1   
innodb_strict_mode = 1   
innodb_sort_buffer_size = 67108864   
innodb_write_io_threads = 8   
innodb_read_io_threads = 8    
innodb_file_per_table = 1   
innodb_autoinc_lock_mode = 2   
innodb_online_alter_log_max_size=1G   
innodb_open_files=4096   
   
server-id=1     #主从集群内不能重复   
log_bin=binlog  #binlog前缀名   
expire_logs_days=7   #binlog 保留时间   
   
\# replication settings #   
master_info_repository = TABLE   
relay_log_info_repository = TABLE   
sync_binlog = 1   
gtid_mode = on   
enforce_gtid_consistency = 1   
log_slave_updates   
binlog_format = ROW   
binlog_rows_query_log_events = 1   
relay_log = relay.log   
relay_log_recovery = 1   
slave_skip_errors = ddl_exist_errors   
slave-rows-search-algorithms = 'INDEX_SCAN,HASH_SCAN'   
   
[mysqld-5.7]   
log_timestamps=system   
log_error_verbosity=2   
slave_parallel_type=LOGICAL_CLOCK   
slave_parallel_workers=16   
slave_preserve_commit_order=on   
   
[mysql]   
prompt=(\\\u@\\\h) [\\\d]>\\\\_   
   
[client]   
user=root   
password=123456   
default-character-set=utf8mb4   
   
\#[mysqld_safe]   
\#log-error=/var/log/mysqld.log   
\#pid-file=/var/run/mysqld/mysqld.pid   
\#   
   
[mysqld80]   
basedir=/usr/local/mysql80/   
plugin_dir=/usr/local/mysql80/lib/plugin   
datadir=/home/data/mysqldata80   
socket=/tmp/mysql.sock80   
port=3305   
   
[mysqld_multi]   
mysqld=mysqld_safe   
mysqladmin=/usr/local/mysql/bin/mysqladmin   
log=/usr/local/mysql/mysqld_multi.log   
user=root   
pass=123456   
   
[mysqld3307]   
datadir=/home/data/mysqldata3307   
port=3307   
socket=/tmp/mysql.sock3307   
   
[mysqld3308]   
datadir=/home/data/mysqldata3308   
port=3308   
socket=/tmp/mysql.sock3308   
   
[mysqld56]   
basedir=/usr/local/mysql56/   
plugin_dir=/usr/local/mysql56/lib/plugin   
datadir=/home/data/mysqldata56   
socket=/tmp/mysql.sock56   
port=3309   
   
安装   
--   
官方文档   
https://dev.mysql.com/doc/refman/5.7/en/binary-installation.html   
   
1. 增加用户组和用户   
groupadd mysql   
useradd -r -g mysql -s /bin/false mysql   
2. 解压tar包，新建软连接   
cd /usr/local   
tar zxvf /root/mysql-5.7.16-linux-glibc2.5-x86_64.tar.gz    
ln -s /usr/local/mysql-5.7.16-linux-glibc2.5-x86_64 mysql   
3. 创建mysql数据目录(配置文件中datadir)   
cd mysql   
mkdir -p /home/data/mysqldata    
chmod 750  /home/data/mysqldata    
chown -R mysql:mysql  /home/data/mysqldata    
![这里写图片描述](http://img.blog.csdn.net/20170724133605494?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd3ltNDA4/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)   
4. 修改mysql文件夹下文件所有者   
chown -R mysql:mysql .   
![这里写图片描述](http://img.blog.csdn.net/20170724133853767?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd3ltNDA4/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)   
5. 初始化数据库(5.7.6 以及后续版本与5.7.5 以及前版本不同，详细见官方文档)   
bin/mysqld --initialize --user=mysql   
bin/mysql_ssl_rsa_setup    
![这里写图片描述](http://img.blog.csdn.net/20170724134252915?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd3ltNDA4/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)   
初始化完成后,mysql给root用户生成了一个临时密码，在error.log(/etc/my.cnf中  log_error = error.log) 中，可以 cat error.log | grep password 查看    
![这里写图片描述](http://img.blog.csdn.net/20170724134919094?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd3ltNDA4/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)   
6. 启动数据库   
bin/mysqld_safe --user=mysql &   
启动完成后可用 ps -ef | grep mysql 命令查看   
有2个线程,mysqld_safe 和 mysqld，如果没有请移步  error.log 查看   
![这里写图片描述](http://img.blog.csdn.net/20170724135329988?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd3ltNDA4/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)   
7. 用root用户的临时密码登录数据库，并且修改密码(<font color=red size=4 face=“黑体”> 注意：</font>修改之前其他操作数据库会报错)   
登录：bin/mysql -uroot -p':FlC!6hBHN2%'    
修改密码：SET PASSWORD = PASSWORD('123456');   
![这里写图片描述](http://img.blog.csdn.net/20170724140516455?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd3ltNDA4/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)   
重新登录：bin/mysql -uroot -p'123456'   
![这里写图片描述](http://img.blog.csdn.net/20170724140702795?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd3ltNDA4/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)   
show databases ;  没有test库，test库有安全隐患被移除了(任何用户对test库都拥有所有权限，可以随便搞挂服务器，建议其他版本的数据库安装好后删除test库)   
到此数据库已经安装完成,可以开始正常的工作了   
![这里写图片描述](http://img.blog.csdn.net/20170724141707912?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd3ltNDA4/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)   
9.配置环境变量和增加开机启动，方便使用和管理   
vim /etc/profile  增加环境变量   
export PATH=/usr/local/mysql/bin:$PATH   
source /etc/profile 生效环境变量   
调用 /usr/local/mysql/bin 目录下可执行文件不用先cd到此目录下，可以直接使用   
![这里写图片描述](http://img.blog.csdn.net/20170724142319760?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd3ltNDA4/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)   
   
cp support-files/mysql.server /etc/init.d/mysqld  增加服务   
service mysqld start|stop|restart   
![这里写图片描述](http://img.blog.csdn.net/20170724143117248?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd3ltNDA4/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)   
   
chkconfig --add mysqld 开机启动   
chkconfig --list  |grep mysql 检查是否设置成功   
![这里写图片描述](http://img.blog.csdn.net/20170724143438942?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd3ltNDA4/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)   
   
结语：   
mysql数据库配置文件中包含了大量内容，计划写一个系列，加油
