---
layout: post
title:  "mysql timeout 相关变量详解"
date:   2016-07-27 16:15:49 +0700
categories: jekyll update
---
登录mysql,执行 show global variables like '%timeout%' ; 不看不知道，一看吓一跳，有13个timeout相关变量，学习整理下          

    
+-----------------------------+----------+      
| Variable_name               | Value    |     
+-----------------------------+----------+     
| connect_timeout             | 10       |    
| delayed_insert_timeout      | 300      |    
| have_statement_timeout      | YES      |    
| innodb_flush_log_at_timeout | 1        |    
| innodb_lock_wait_timeout    | 5        |     
| innodb_rollback_on_timeout  | OFF      |    
| interactive_timeout         | 180      |    
| lock_wait_timeout           | 1800     |     
| net_read_timeout            | 30       |    
| net_write_timeout           | 60       |    
| rpl_stop_slave_timeout      | 31536000 |     
| slave_net_timeout           | 60       |    
| wait_timeout                | 180      |         
+-----------------------------+----------+         
    
    
变量详解官方文档:[https://dev.mysql.com/doc/refman/5.7/en/server-system-variables.html][1] 
  
1.connect_timeout    
{% highlight doc %}The number of seconds that the mysqld server waits for a connect packet before responding with Bad handshake. The default value is 10 seconds.{% endhighlight %}    
mysql的基本原理应该是有个监听线程循环接收请求，当有请求来时，创建线程（或者从线程池中取）来处理这个请求。由于mysql连接采用TCP协议，那么之前势必是需要进行TCP三次握手的。TCP三次握手成功之后，客户端会进入阻塞，等待服务端的消息。服务端这个时候会创建一个线程(或者从线程池中取一个线程)来处理请求，主要验证部分包括host和用户名密码验证。host验证我们比较熟悉，因为在用grant命令授权用户的时候是有指定host的。用户名密码认证则是服务端先生成一个随机数发送给客户端，客户端用该随机数和密码进行多次sha1加密后发送给服务端验证。如果通过，整个连接握手过程完成。    
由此可见，整个连接握手可能会有各种可能出错。所以这个connect_timeout值就是指这个超时时间了。可以简单测试下，运行下面的telnet命令会发现客户端会在10秒后超时返回。   
{% highlight shell %} telnet 172.16.178.148 3306 {% endhighlight %}    
{% highlight result %} | 110 | unauthenticated user | 172.16.178.1:50296 | NULL | Connect |   10 | Receiving from client | NULL             | {% endhighlight %}    

  
2.wait_timeout interactive_timeout  
wait_timeout:  
{% highlight doc %}
The number of seconds the server waits for activity on a noninteractive connection before closing it.  
On thread startup, the session wait_timeout value is initialized from the global wait_timeout value or from the global interactive_timeout value, depending on the type of client (as defined by the CLIENT_INTERACTIVE connect option to mysql_real_connect())
{% endhighlight %}
interactive_timeout:  
{% highlight shell %} 
The number of seconds the server waits for activity on an interactive connection before closing it. An interactive client is defined as a client that uses the CLIENT_INTERACTIVE option to mysql_real_connect()
{% endhighlight %}  
这2个参数都是用于控制sleep线程被杀掉之前的等待时间，进入mysql的时候session级别 wait_timeout 根据client类型被设置为global wait_timeout(nointeractive) or global interactive_timeout(interactive) 值  
测试如下：  
{% highlight mysql %} set global interactive_timeout=3; ##设置交互超时为3秒{% endhighlight %}  
用mysql -u 重新进入mysql:  
{% highlight result %}  
(root@localhost) [(none)]> show variables like '%timeout%' ;  
+-----------------------------+----------+  
| Variable_name               | Value    |  
+-----------------------------+----------+  
| connect_timeout             | 10       |  
| delayed_insert_timeout      | 300      |  
| have_statement_timeout      | YES      |  
| innodb_flush_log_at_timeout | 1        |  
| innodb_lock_wait_timeout    | 5        |  
| innodb_rollback_on_timeout  | OFF      |  
| interactive_timeout         | 3        |  
| lock_wait_timeout           | 1800     |  
| net_read_timeout            | 30       |  
| net_write_timeout           | 60       |  
| rpl_stop_slave_timeout      | 31536000 |  
| slave_net_timeout           | 60       |  
| wait_timeout                | 3        |  
+-----------------------------+----------+  
{% endhighlight %}  
看到sessio级别 wait_timeout 被设置为 interactive_timeout值，过3s中执行 select 1 可以看到会重新连接  
{% highlight mysql %}
(root@localhost) [(none)]> select 1 ;   
ERROR 2006 (HY000): MySQL server has gone away  
No connection. Trying to reconnect...  
Connection id:    115  
{% endhighlight %}  

3.innodb_lock_wait_timeout innodb_rollback_on_timeout  
innodb_lock_wait_timeout:  
{% highlight doc %}
The length of time in seconds an InnoDB transaction waits for a row lock before giving up  
{% endhighlight %}
innodb等待事务行锁的时间，默认50s  
innodb_rollback_on_timeout:
{% highlight doc %}
InnoDB rolls back only the last statement on a transaction timeout by default. If --innodb_rollback_on_timeout is specified, a transaction timeout causes InnoDB to abort and roll back the entire transaction.  
{% endhighlight %}
事务因为innodb_lock_wait_timeout超时时，innodb_rollback_on_timeout =on 时回滚整个事务，innodb_rollback_on_timeout=off 只回滚超时的语句，默认值为off  
测试:  
数据准备  
{% highlight test %}
(root@localhost) [tdb]> show create table a \G;
*************************** 1. row ***************************
       Table: a
Create Table: CREATE TABLE `a` (
  `id` int(11) NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
1 row in set (0.00 sec)

ERROR: 
No query specified

(root@localhost) [tdb]> select * from a ;
+----+
| id |
+----+
|  1 |
|  2 |
|  3 |
+----+
{% endhighlight %}
[1]: https://dev.mysql.com/doc/refman/5.7/en/server-system-variables.html

