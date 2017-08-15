---
layout: post
title:  "mysql explain 执行计划"
date:   2017-08-14 16:15:49 +0700
categories: jekyll update
---
本文记录mysql执行计划相关学习

{% highlight doc %}
(root@localhost) [tdb]> explain  select * from t1 where a=123;
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref   | rows | filtered | Extra |
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | t1    | NULL       | ref  | a             | a    | 4       | const |   19 |   100.00 | NULL  |
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-------+
{% endhighlight %}
执行计划涉及到的字段有id,select_type,table,partitions,type,possible_keys,key,key_len,ref,rows,filtered,Extra

1.id:
id列数字越大越先执行，如果说数字一样大，那么就从上往下依次执行，id列为null的就表是这是一个结果集，不需要使用它来进行查询。
{% highlight doc %}
(root@localhost) [tdb]> explain  select * from t1 where a=123 union  select * from t1 b where a=234 union all select * from t1 c where a=456 ;
+----+--------------+--------------+------------+------+---------------+------+---------+-------+------+----------+-----------------+
| id | select_type  | table        | partitions | type | possible_keys | key  | key_len | ref   | rows | filtered | Extra           |
+----+--------------+--------------+------------+------+---------------+------+---------+-------+------+----------+-----------------+
|  1 | PRIMARY      | t1           | NULL       | ref  | a             | a    | 4       | const |   19 |   100.00 | NULL            |
|  2 | UNION        | b            | NULL       | ref  | a             | a    | 4       | const |   15 |   100.00 | NULL            |
|  3 | UNION        | c            | NULL       | ref  | a             | a    | 4       | const |   11 |   100.00 | NULL            |
| NULL | UNION RESULT | <union1,2,3> | NULL       | ALL  | NULL          | NULL | NULL    | NULL  | NULL |     NULL | Using temporary |
+----+--------------+--------------+------------+------+---------------+------+---------+-------+------+----------+-----------------+
{% endhighlight %}

2.select_type:
simple：表示不需要union操作或者不包含子查询的简单select查询。单表查询为simple,有连接查询时，外层的查询为simple
{% highlight doc %}
(root@localhost) [tdb]> explain select * from t1 where a=123 ;
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref   | rows | filtered | Extra |
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | t1    | NULL       | ref  | a             | a    | 4       | const |   19 |   100.00 | NULL  |
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-------+

(root@localhost) [tdb]> explain select * from t1 a ,t3 b  where a.id=b.id and a.a=123  ;
+----+-------------+-------+------------+--------+---------------+---------+---------+----------+------+----------+-------+
| id | select_type | table | partitions | type   | possible_keys | key     | key_len | ref      | rows | filtered | Extra |
+----+-------------+-------+------------+--------+---------------+---------+---------+----------+------+----------+-------+
|  1 | SIMPLE      | a     | NULL       | ref    | PRIMARY,a     | a       | 4       | const    |   19 |   100.00 | NULL  |
|  1 | SIMPLE      | b     | NULL       | eq_ref | PRIMARY       | PRIMARY | 4       | tdb.a.id |    1 |   100.00 | NULL  |
+----+-------------+-------+------------+--------+---------------+---------+---------+----------+------+----------+-------+
{% endhighlight %}
primary：一个需要union操作或者含有子查询的select，位于最外层的单位查询的select_type即为primary。且只有一个
union：union连接的两个select查询，第一个查询select_type是PRIMARY，除了第一个表外，第二个以后的表select_type都是union
dependent union：与union一样，出现在union 或union all语句中，但是这个查询要受到外部查询的影响
union result：包含union的结果集，在union和union all语句中,因为它不需要参与查询，所以id字段为null
subquery：除了from字句中包含的子查询外，其他地方出现的子查询都可能是subquery
dependent subquery：与dependent union类似，表示这个subquery的查询要受到外部表查询的影响
derived：from字句中出现的子查询，也叫做派生表，其他数据库中可能叫做内联视图或嵌套select 这个没测出啥情况会出现。。。
{% highlight doc %}
(root@localhost) [tdb]> explain  select * from t1 where a=123 union  select * from t1 b where a=234 union all select * from t1 c where a=456 ;
+----+--------------+--------------+------------+------+---------------+------+---------+-------+------+----------+-----------------+
| id | select_type  | table        | partitions | type | possible_keys | key  | key_len | ref   | rows | filtered | Extra           |
+----+--------------+--------------+------------+------+---------------+------+---------+-------+------+----------+-----------------+
|  1 | PRIMARY      | t1           | NULL       | ref  | a             | a    | 4       | const |   19 |   100.00 | NULL            |
|  2 | UNION        | b            | NULL       | ref  | a             | a    | 4       | const |   15 |   100.00 | NULL            |
|  3 | UNION        | c            | NULL       | ref  | a             | a    | 4       | const |   11 |   100.00 | NULL            |
| NULL | UNION RESULT | <union1,2,3> | NULL       | ALL  | NULL          | NULL | NULL    | NULL  | NULL |     NULL | Using temporary |
+----+--------------+--------------+------------+------+---------------+------+---------+-------+------+----------+-----------------+

(root@localhost) [tdb]> explain  select * from t1 a where a=123 and exists (select 1 from t3 b where a.id=b.id) ;
+----+--------------------+-------+------------+--------+---------------+---------+---------+----------+------+----------+-------------+
| id | select_type        | table | partitions | type   | possible_keys | key     | key_len | ref      | rows | filtered | Extra       |
+----+--------------------+-------+------------+--------+---------------+---------+---------+----------+------+----------+-------------+
|  1 | PRIMARY            | a     | NULL       | ref    | a             | a       | 4       | const    |   19 |   100.00 | Using where |
|  2 | DEPENDENT SUBQUERY | b     | NULL       | eq_ref | PRIMARY       | PRIMARY | 4       | tdb.a.id |    1 |   100.00 | Using index |
+----+--------------------+-------+------------+--------+---------------+---------+---------+----------+------+----------+-------------+

(root@localhost) [tdb]> explain select * from t1 c where exists (  select 1 from t1 a where a=123 and a.id=c.id  union select 1 from t3 b where a=234 and b.id=c.id)  ;
+----+--------------------+------------+------------+--------+---------------+---------+---------+----------+-------+----------+-----------------+
| id | select_type        | table      | partitions | type   | possible_keys | key     | key_len | ref      | rows  | filtered | Extra           |
+----+--------------------+------------+------------+--------+---------------+---------+---------+----------+-------+----------+-----------------+
|  1 | PRIMARY            | c          | NULL       | ALL    | NULL          | NULL    | NULL    | NULL     | 99489 |   100.00 | Using where     |
|  2 | DEPENDENT SUBQUERY | a          | NULL       | eq_ref | PRIMARY,a     | PRIMARY | 4       | tdb.c.id |     1 |     5.00 | Using where     |
|  3 | DEPENDENT UNION    | b          | NULL       | eq_ref | PRIMARY,a     | PRIMARY | 4       | tdb.c.id |     1 |     5.00 | Using where     |
| NULL | UNION RESULT       | <union2,3> | NULL       | ALL    | NULL          | NULL    | NULL    | NULL     |  NULL |     NULL | Using temporary |
+----+--------------------+------------+------------+--------+---------------+---------+---------+----------+-------+----------+-----------------+

(root@localhost) [tdb]> explain   select  ( select id from t1 where a=435   limit 1 ),id  from t3 where a=123 ;
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref   | rows | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-------------+
|  1 | PRIMARY     | t3    | NULL       | ref  | a             | a    | 4       | const |   19 |   100.00 | Using index |
|  2 | SUBQUERY    | t1    | NULL       | ref  | a             | a    | 4       | const |    8 |   100.00 | Using index |
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-------------+
{% endhighlight %}

3.table:
显示的查询表名，如果查询使用了别名，那么这里显示的是别名，如果不涉及对数据表的操作，那么这显示为null，如果显示为尖括号括起来的<derived N>就表示这个是临时表，后边的N就是执行计>
划中的id，表示结果来自于这个查询产生。如果是尖括号括起来的<union M,N>，与<derived N>类似，也是一个临时表，表示这个结果来自于union查询的id为M,N的结果集。
{% highlight doc %}
(root@localhost) [tdb]> explain select * from t1 c where exists (  select 1 from t1 a where a=123 and a.id=c.id  union select 1 from t3 b where a=234 and b.id=c.id)  ;
+----+--------------------+------------+------------+--------+---------------+---------+---------+----------+-------+----------+-----------------+
| id | select_type        | table      | partitions | type   | possible_keys | key     | key_len | ref      | rows  | filtered | Extra           |
+----+--------------------+------------+------------+--------+---------------+---------+---------+----------+-------+----------+-----------------+
|  1 | PRIMARY            | c          | NULL       | ALL    | NULL          | NULL    | NULL    | NULL     | 99489 |   100.00 | Using where     |
|  2 | DEPENDENT SUBQUERY | a          | NULL       | eq_ref | PRIMARY,a     | PRIMARY | 4       | tdb.c.id |     1 |     5.00 | Using where     |
|  3 | DEPENDENT UNION    | b          | NULL       | eq_ref | PRIMARY,a     | PRIMARY | 4       | tdb.c.id |     1 |     5.00 | Using where     |
| NULL | UNION RESULT       | <union2,3> | NULL       | ALL    | NULL          | NULL    | NULL    | NULL     |  NULL |     NULL | Using temporary |
+----+--------------------+------------+------------+--------+---------------+---------+---------+----------+-------+----------+-----------------+
{% endhighlight %}

4.type:
依次从好到差：system，const，eq_ref，ref，fulltext，ref_or_null，unique_subquery，index_subquery，range，index_merge，index，ALL，除了all之外，其他的type都可以使用到索引，除了index_merge之外，其他的type只可以用到一个索引

system：表中只有一行数据或者是空表，且只能用于myisam和memory表。如果是Innodb引擎表，type列在这个情况通常都是all或者index  
{% highlight doc %}
(root@localhost) [test]> show create table tt1 \G;
*************************** 1. row ***************************
       Table: tt1
Create Table: CREATE TABLE `tt1` (
  `a` int(11) NOT NULL AUTO_INCREMENT,
  `b` int(11) DEFAULT NULL,
  PRIMARY KEY (`a`)
) ENGINE=MyISAM AUTO_INCREMENT=3 DEFAULT CHARSET=utf8mb4
1 row in set (0.00 sec)

ERROR: 
No query specified

(root@localhost) [test]> select * from tt1 ;
+---+------+
| a | b    |
+---+------+
| 2 |    2 |
+---+------+
1 row in set (0.00 sec)

(root@localhost) [test]> explain  select * from tt1 ;
+----+-------------+-------+------------+--------+---------------+------+---------+------+------+----------+-------+
| id | select_type | table | partitions | type   | possible_keys | key  | key_len | ref  | rows | filtered | Extra |
+----+-------------+-------+------------+--------+---------------+------+---------+------+------+----------+-------+
|  1 | SIMPLE      | tt1   | NULL       | system | NULL          | NULL | NULL    | NULL |    1 |   100.00 | NULL  |
+----+-------------+-------+------------+--------+---------------+------+---------+------+------+----------+-------+
{% endhighlight %}

const：使用唯一索引或者主键，返回记录一定是1行记录的等值where条件时，通常type是const。其他数据库也叫做唯一索引扫描
{% highlight doc %}
(root@localhost) [test]> show create table tt2 \G;
*************************** 1. row ***************************
       Table: tt2
Create Table: CREATE TABLE `tt2` (
  `a` int(11) NOT NULL AUTO_INCREMENT,
  `b` int(11) NOT NULL,
  PRIMARY KEY (`a`),
  UNIQUE KEY `b` (`b`)
) ENGINE=InnoDB AUTO_INCREMENT=3 DEFAULT CHARSET=utf8mb4
1 row in set (0.00 sec)

ERROR: 
No query specified

(root@localhost) [test]> select * from tt2 ;
+---+---+
| a | b |
+---+---+
| 1 | 1 |
| 2 | 2 |
+---+---+
2 rows in set (0.00 sec)

(root@localhost) [test]> explain select * from tt2 where b=1 ;
+----+-------------+-------+------------+-------+---------------+------+---------+-------+------+----------+-------------+
| id | select_type | table | partitions | type  | possible_keys | key  | key_len | ref   | rows | filtered | Extra       |
+----+-------------+-------+------------+-------+---------------+------+---------+-------+------+----------+-------------+
|  1 | SIMPLE      | tt2   | NULL       | const | b             | b    | 4       | const |    1 |   100.00 | Using index |
+----+-------------+-------+------------+-------+---------------+------+---------+-------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)

(root@localhost) [test]> explain select * from tt2 where a=1 ;
+----+-------------+-------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
| id | select_type | table | partitions | type  | possible_keys | key     | key_len | ref   | rows | filtered | Extra |
+----+-------------+-------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | tt2   | NULL       | const | PRIMARY       | PRIMARY | 4       | const |    1 |   100.00 | NULL  |
+----+-------------+-------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
{% endhighlight %}

eq_ref：出现在要连接过个表的查询计划中，驱动表只返回一行数据，且这行数据是第二个表的主键或者唯一索引，且必须为not null，唯一索引和主键是多列时，只有所有的列都用作比较时才会出现eq_ref
{% highlight doc %}
(root@localhost) [tdb]> show create table t1 \G;
*************************** 1. row ***************************
       Table: t1
Create Table: CREATE TABLE `t1` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `a` int(11) NOT NULL,
  `b` int(11) DEFAULT NULL,
  `c` int(11) DEFAULT NULL,
  `name` varchar(50) NOT NULL,
  `status` tinyint(4) NOT NULL,
  `createtime` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  KEY `a` (`a`,`b`,`name`,`c`)
) ENGINE=InnoDB AUTO_INCREMENT=100003 DEFAULT CHARSET=utf8mb4
1 row in set (0.00 sec)

ERROR: 
No query specified

(root@localhost) [tdb]> show create table t3 \G;
*************************** 1. row ***************************
       Table: t3
Create Table: CREATE TABLE `t3` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `a` int(11) NOT NULL,
  `b` int(11) DEFAULT NULL,
  `c` int(11) DEFAULT NULL,
  `name` varchar(50) NOT NULL,
  `status` tinyint(4) NOT NULL,
  `createtime` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  KEY `a` (`a`,`b`,`name`,`c`)
) ENGINE=InnoDB AUTO_INCREMENT=100003 DEFAULT CHARSET=utf8mb4
1 row in set (0.00 sec)

ERROR: 
No query specified

(root@localhost) [tdb]> explain select * from t1 left join t3 on t1.id=t3.id where t1.a=123 ;
+----+-------------+-------+------------+--------+---------------+---------+---------+-----------+------+----------+-------+
| id | select_type | table | partitions | type   | possible_keys | key     | key_len | ref       | rows | filtered | Extra |
+----+-------------+-------+------------+--------+---------------+---------+---------+-----------+------+----------+-------+
|  1 | SIMPLE      | t1    | NULL       | ref    | a             | a       | 4       | const     |   19 |   100.00 | NULL  |
|  1 | SIMPLE      | t3    | NULL       | eq_ref | PRIMARY       | PRIMARY | 4       | tdb.t1.id |    1 |   100.00 | NULL  |
+----+-------------+-------+------------+--------+---------------+---------+---------+-----------+------+----------+-------+
2 rows in set, 1 warning (0.00 sec)

(root@localhost) [tdb]> explain  select * from t1 where a=123 and exists (select 1 from t3 where t1.id=t3.id) ;
+----+--------------------+-------+------------+--------+---------------+---------+---------+-----------+------+----------+-------------+
| id | select_type        | table | partitions | type   | possible_keys | key     | key_len | ref       | rows | filtered | Extra       |
+----+--------------------+-------+------------+--------+---------------+---------+---------+-----------+------+----------+-------------+
|  1 | PRIMARY            | t1    | NULL       | ref    | a             | a       | 4       | const     |   19 |   100.00 | Using where |
|  2 | DEPENDENT SUBQUERY | t3    | NULL       | eq_ref | PRIMARY       | PRIMARY | 4       | tdb.t1.id |    1 |   100.00 | Using index |
+----+--------------------+-------+------------+--------+---------------+---------+---------+-----------+------+----------+-------------+
{% endhighlight %}

ref：不像eq_ref那样要求连接顺序，也没有主键和唯一索引的要求，只要使用相等条件检索时就可能出现，常见与辅助索引的等值查找。或者多列主键、唯一索引中，使用第一个列作为等值查找也会出现，总之，返回数据不唯一的等值查找就可能出现。
{% highlight doc %}
(root@localhost) [test]> show create table tt2 \G ;
*************************** 1. row ***************************
       Table: tt2
Create Table: CREATE TABLE `tt2` (
  `a` int(11) NOT NULL AUTO_INCREMENT,
  `b` int(11) NOT NULL,
  PRIMARY KEY (`a`),
  KEY `b` (`b`)
) ENGINE=InnoDB AUTO_INCREMENT=3 DEFAULT CHARSET=utf8mb4
1 row in set (0.00 sec)

ERROR: 
No query specified

(root@localhost) [test]> explain  select *from tt2 where b=1 ;
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref   | rows | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-------------+
|  1 | SIMPLE      | tt2   | NULL       | ref  | b             | b    | 4       | const |    1 |   100.00 | Using index |
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-------------+

(root@localhost) [test]> show create table tt3 \G;
*************************** 1. row ***************************
       Table: tt3
Create Table: CREATE TABLE `tt3` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `a` int(11) NOT NULL,
  `b` int(11) NOT NULL,
  `c` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `a` (`a`,`b`)
) ENGINE=InnoDB AUTO_INCREMENT=14 DEFAULT CHARSET=utf8mb4
1 row in set (0.00 sec)

ERROR: 
No query specified

(root@localhost) [test]> explain select * from tt3 where a=1 ;
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref   | rows | filtered | Extra |
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | tt3   | NULL       | ref  | a             | a    | 4       | const |    3 |   100.00 | NULL  |
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-------+
{% endhighlight %}

fulltext：很少用到，暂时空着  

ref_or_null：与ref方法类似，只是增加了null值的比较 
{% highlight doc %}
(root@localhost) [test]> show create table tt2 \G;
*************************** 1. row ***************************
       Table: tt2
Create Table: CREATE TABLE `tt2` (
  `a` int(11) NOT NULL AUTO_INCREMENT,
  `b` int(11) DEFAULT NULL,
  PRIMARY KEY (`a`),
  KEY `b` (`b`)
) ENGINE=InnoDB AUTO_INCREMENT=3 DEFAULT CHARSET=utf8mb4
1 row in set (0.00 sec)

ERROR: 
No query specified

(root@localhost) [test]>  explain  select *from tt2 where b=2 or   b is null;
+----+-------------+-------+------------+-------------+---------------+------+---------+-------+------+----------+--------------------------+
| id | select_type | table | partitions | type        | possible_keys | key  | key_len | ref   | rows | filtered | Extra                    |
+----+-------------+-------+------------+-------------+---------------+------+---------+-------+------+----------+--------------------------+
|  1 | SIMPLE      | tt2   | NULL       | ref_or_null | b             | b    | 5       | const |    2 |   100.00 | Using where; Using index |
+----+-------------+-------+------------+-------------+---------------+------+---------+-------+------+----------+--------------------------+
{% endhighlight %}

unique_subquery：用于where中的in形式子查询，子查询返回不重复值唯一值  value IN (SELECT primary_key FROM single_table WHERE some_expr)
5.1在使用unique_subquery，5.7不知why    
{% highlight doc %}
5.1
mysql> show create table tbyunmu_order \G;
*************************** 1. row ***************************
       Table: tbyunmu_order
Create Table: CREATE TABLE `tbyunmu_order` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `user_id` int(11) NOT NULL,
  `pro` varchar(20) COLLATE utf8_unicode_ci DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `user_id` (`user_id`)
) ENGINE=InnoDB AUTO_INCREMENT=9 DEFAULT CHARSET=utf8 COLLATE=utf8_unicode_ci
1 row in set (0.00 sec)

ERROR: 
No query specified

mysql> explain 
    -> select * from test.tbyunmu_order where id in (select id from test.tbyunmu_order where user_id=1) ;
+----+--------------------+---------------+-----------------+-----------------+---------+---------+------+------+-------------+
| id | select_type        | table         | type            | possible_keys   | key     | key_len | ref  | rows | Extra       |
+----+--------------------+---------------+-----------------+-----------------+---------+---------+------+------+-------------+
|  1 | PRIMARY            | tbyunmu_order | ALL             | NULL            | NULL    | NULL    | NULL |    8 | Using where |
|  2 | DEPENDENT SUBQUERY | tbyunmu_order | unique_subquery | PRIMARY,user_id | PRIMARY | 4       | func |    1 | Using where |
+----+--------------------+---------------+-----------------+-----------------+---------+---------+------+------+-------------+

5.7
(root@localhost) [tdb]> show create table tbyunmu_order \G;
*************************** 1. row ***************************
       Table: tbyunmu_order
Create Table: CREATE TABLE `tbyunmu_order` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `user_id` int(11) NOT NULL,
  `pro` varchar(20) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `user_id` (`user_id`)
) ENGINE=InnoDB AUTO_INCREMENT=9 DEFAULT CHARSET=utf8mb4
1 row in set (0.00 sec)

ERROR: 
No query specified

(root@localhost) [tdb]> explain  select * from tbyunmu_order where id in (select id from tbyunmu_order where user_id=1);
+----+-------------+---------------+------------+--------+-----------------+---------+---------+----------------------+------+----------+-------------+
| id | select_type | table         | partitions | type   | possible_keys   | key     | key_len | ref                  | rows | filtered | Extra       |
+----+-------------+---------------+------------+--------+-----------------+---------+---------+----------------------+------+----------+-------------+
|  1 | SIMPLE      | tbyunmu_order | NULL       | ref    | PRIMARY,user_id | user_id | 4       | const                |    2 |   100.00 | Using index |
|  1 | SIMPLE      | tbyunmu_order | NULL       | eq_ref | PRIMARY         | PRIMARY | 4       | tdb.tbyunmu_order.id |    1 |   100.00 | NULL        |
+----+-------------+---------------+------------+--------+-----------------+---------+---------+----------------------+------+----------+-------------+
{% endhighlight %}

index_subquery：该联接类型类似于unique_subquery。可以替换IN子查询，但只适合下列形式的子查询中的非唯一索引：value IN SELECT key_column FROM single_table WHERE some_expr)
同样，只在5.1中测出来了，5.7不知why  
{% highlight doc %}
5.1
mysql> show create table test.tbyunmu_order  \G;
*************************** 1. row ***************************
       Table: tbyunmu_order
Create Table: CREATE TABLE `tbyunmu_order` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `user_id` int(11) NOT NULL,
  `pro` varchar(20) COLLATE utf8_unicode_ci DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `user_id` (`user_id`)
) ENGINE=InnoDB AUTO_INCREMENT=9 DEFAULT CHARSET=utf8 COLLATE=utf8_unicode_ci
1 row in set (0.00 sec)

ERROR: 
No query specified

mysql> explain  select * from test.tbyunmu_order where user_id in (select user_id from test.tbyunmu_order a where id>4);
+----+--------------------+---------------+----------------+-----------------+---------+---------+------+------+--------------------------+
| id | select_type        | table         | type           | possible_keys   | key     | key_len | ref  | rows | Extra                    |
+----+--------------------+---------------+----------------+-----------------+---------+---------+------+------+--------------------------+
|  1 | PRIMARY            | tbyunmu_order | ALL            | NULL            | NULL    | NULL    | NULL |    8 | Using where              |
|  2 | DEPENDENT SUBQUERY | a             | index_subquery | PRIMARY,user_id | user_id | 4       | func |    4 | Using index; Using where |
+----+--------------------+---------------+----------------+-----------------+---------+---------+------+------+--------------------------+

5.7
(root@localhost) [tdb]> show create table tbyunmu_order  \G;
*************************** 1. row ***************************
       Table: tbyunmu_order
Create Table: CREATE TABLE `tbyunmu_order` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `user_id` int(11) NOT NULL,
  `pro` varchar(20) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `user_id` (`user_id`)
) ENGINE=InnoDB AUTO_INCREMENT=9 DEFAULT CHARSET=utf8mb4
1 row in set (0.00 sec)

ERROR: 
No query specified

(root@localhost) [tdb]> explain  select * from tbyunmu_order where user_id in (select user_id from tbyunmu_order a where id>4);
+----+-------------+---------------+------------+-------+-----------------+---------+---------+---------------+------+----------+-------------------------------------+
| id | select_type | table         | partitions | type  | possible_keys   | key     | key_len | ref           | rows | filtered | Extra                               |
+----+-------------+---------------+------------+-------+-----------------+---------+---------+---------------+------+----------+-------------------------------------+
|  1 | SIMPLE      | a             | NULL       | index | PRIMARY,user_id | user_id | 4       | NULL          |    8 |    75.00 | Using where; Using index; LooseScan |
|  1 | SIMPLE      | tbyunmu_order | NULL       | ref   | user_id         | user_id | 4       | tdb.a.user_id |    1 |   100.00 | NULL                                |
+----+-------------+---------------+------------+-------+-----------------+---------+---------+---------------+------+----------+-------------------------------------+
{% endhighlight %}

range：索引范围扫描，常见于使用>,<,is null,between ,in ,like等运算符的查询中
{% highlight doc %}
(root@localhost) [tdb]> show create table t1 \G;
*************************** 1. row ***************************
       Table: t1
Create Table: CREATE TABLE `t1` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `a` int(11) NOT NULL,
  `b` int(11) DEFAULT NULL,
  `c` int(11) DEFAULT NULL,
  `name` varchar(50) NOT NULL,
  `status` tinyint(4) NOT NULL,
  `createtime` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  KEY `a` (`a`,`b`,`name`,`c`)
) ENGINE=InnoDB AUTO_INCREMENT=100003 DEFAULT CHARSET=utf8mb4
1 row in set (0.01 sec)

ERROR: 
No query specified

(root@localhost) [tdb]> explain  select * from t1  where a in (100,200) ;
+----+-------------+-------+------------+-------+---------------+------+---------+------+------+----------+-----------------------+
| id | select_type | table | partitions | type  | possible_keys | key  | key_len | ref  | rows | filtered | Extra                 |
+----+-------------+-------+------------+-------+---------------+------+---------+------+------+----------+-----------------------+
|  1 | SIMPLE      | t1    | NULL       | range | a             | a    | 4       | NULL |   23 |   100.00 | Using index condition |
+----+-------------+-------+------------+-------+---------------+------+---------+------+------+----------+-----------------------+
1 row in set, 1 warning (0.00 sec)

(root@localhost) [tdb]> explain  select * from t1  where a between 100 and 200 ;
+----+-------------+-------+------------+-------+---------------+------+---------+------+------+----------+-----------------------+
| id | select_type | table | partitions | type  | possible_keys | key  | key_len | ref  | rows | filtered | Extra                 |
+----+-------------+-------+------------+-------+---------------+------+---------+------+------+----------+-----------------------+
|  1 | SIMPLE      | t1    | NULL       | range | a             | a    | 4       | NULL | 1025 |   100.00 | Using index condition |
+----+-------------+-------+------------+-------+---------------+------+---------+------+------+----------+-----------------------+

(root@localhost) [tdb]> explain  select * from t1  where a  > 100 and a< 200 ; 
+----+-------------+-------+------------+-------+---------------+------+---------+------+------+----------+-----------------------+
| id | select_type | table | partitions | type  | possible_keys | key  | key_len | ref  | rows | filtered | Extra                 |
+----+-------------+-------+------------+-------+---------------+------+---------+------+------+----------+-----------------------+
|  1 | SIMPLE      | t1    | NULL       | range | a             | a    | 4       | NULL | 1002 |   100.00 | Using index condition |
+----+-------------+-------+------------+-------+---------------+------+---------+------+------+----------+-----------------------+
{% endhighlight %}

index_merge：表示查询使用了两个以上的索引，最后取交集或者并集，常见and ，or的条件使用了不同的索引，官方排序这个在ref_or_null之后，但是实际上由于要读取所个索引，性能可能大部分时间都不如range
{% highlight doc %}
(root@localhost) [tdb]> show create table t1 \G;
*************************** 1. row ***************************
       Table: t1
Create Table: CREATE TABLE `t1` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `a` int(11) NOT NULL,
  `b` int(11) DEFAULT NULL,
  `c` int(11) DEFAULT NULL,
  `name` varchar(50) NOT NULL,
  `status` tinyint(4) NOT NULL,
  `createtime` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  KEY `a` (`a`,`b`,`name`,`c`),
  KEY `b` (`b`)
) ENGINE=InnoDB AUTO_INCREMENT=100003 DEFAULT CHARSET=utf8mb4
1 row in set (0.00 sec)

ERROR: 
No query specified

(root@localhost) [tdb]> explain select * from t1 where a=123 or b=123 ;
+----+-------------+-------+------------+-------------+---------------+------+---------+------+------+----------+------------------------------------+
| id | select_type | table | partitions | type        | possible_keys | key  | key_len | ref  | rows | filtered | Extra                              |
+----+-------------+-------+------------+-------------+---------------+------+---------+------+------+----------+------------------------------------+
|  1 | SIMPLE      | t1    | NULL       | index_merge | a,b           | a,b  | 4,5     | NULL |   29 |   100.00 | Using sort_union(a,b); Using where |
+----+-------------+-------+------------+-------------+---------------+------+---------+------+------+----------+------------------------------------+
{% endhighlight %}

index：索引全表扫描，把索引从头到尾扫一遍，常见于使用索引列就可以处理不需要读取数据文件的查询、可以使用索引排序或者分组的查询。
{% highlight doc %}
(root@localhost) [tdb]> show create table t1 \G;
*************************** 1. row ***************************
       Table: t1
Create Table: CREATE TABLE `t1` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `a` int(11) NOT NULL,
  `b` int(11) DEFAULT NULL,
  `c` int(11) DEFAULT NULL,
  `name` varchar(50) NOT NULL,
  `status` tinyint(4) NOT NULL,
  `createtime` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  KEY `a` (`a`,`b`,`name`,`c`),
  KEY `b` (`b`)
) ENGINE=InnoDB AUTO_INCREMENT=100003 DEFAULT CHARSET=utf8mb4
1 row in set (0.00 sec)

ERROR: 
No query specified

(root@localhost) [tdb]> explain select count(b) from t1  ;
+----+-------------+-------+------------+-------+---------------+------+---------+------+-------+----------+-------------+
| id | select_type | table | partitions | type  | possible_keys | key  | key_len | ref  | rows  | filtered | Extra       |
+----+-------------+-------+------------+-------+---------------+------+---------+------+-------+----------+-------------+
|  1 | SIMPLE      | t1    | NULL       | index | NULL          | b    | 5       | NULL | 99489 |   100.00 | Using index |
+----+-------------+-------+------------+-------+---------------+------+---------+------+-------+----------+-------------+
{% endhighlight %}

all：这个就是全表扫描数据文件，然后再在server层进行过滤返回符合要求的记录。看到这个要小心了  
{% highlight doc %}
(root@localhost) [tdb]> show create table t1 \G;
*************************** 1. row ***************************
       Table: t1
Create Table: CREATE TABLE `t1` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `a` int(11) NOT NULL,
  `b` int(11) DEFAULT NULL,
  `c` int(11) DEFAULT NULL,
  `name` varchar(50) NOT NULL,
  `status` tinyint(4) NOT NULL,
  `createtime` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  KEY `a` (`a`,`b`,`name`,`c`),
  KEY `b` (`b`)
) ENGINE=InnoDB AUTO_INCREMENT=100003 DEFAULT CHARSET=utf8mb4
1 row in set (0.01 sec)

ERROR: 
No query specified

(root@localhost) [tdb]> explain  select *from t1 where c=123 ;
+----+-------------+-------+------------+------+---------------+------+---------+------+-------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows  | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+------+---------+------+-------+----------+-------------+
|  1 | SIMPLE      | t1    | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 99489 |    10.00 | Using where |
+----+-------------+-------+------------+------+---------------+------+---------+------+-------+----------+-------------+
{% endhighlight %}

5.possible_keys
查询可能使用到的索引都会在这里列出来
{% highlight doc %}
(root@localhost) [tdb]> show create table t1 \G ;
*************************** 1. row ***************************
       Table: t1
Create Table: CREATE TABLE `t1` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `a` int(11) NOT NULL,
  `b` int(11) DEFAULT NULL,
  `c` int(11) DEFAULT NULL,
  `name` varchar(50) NOT NULL,
  `status` tinyint(4) NOT NULL,
  `createtime` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  KEY `a` (`a`,`b`,`name`,`c`),
  KEY `b` (`b`),
  KEY `a_2` (`a`),
  KEY `a_3` (`a`,`c`),
  KEY `a_4` (`a`,`b`,`c`)
) ENGINE=InnoDB AUTO_INCREMENT=100003 DEFAULT CHARSET=utf8mb4
1 row in set (0.00 sec)

ERROR: 
No query specified

(root@localhost) [tdb]> explain select * from t1 where a=123 ;
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref   | rows | filtered | Extra |
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | t1    | NULL       | ref  | a,a_2,a_3,a_4 | a    | 4       | const |   19 |   100.00 | NULL  |
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-------+
{% endhighlight %}

6.key
查询真正使用到的索引，select_type为index_merge时，这里可能出现两个以上的索引，其他的select_type这里只会出现一个。
{% highlight doc %}
(root@localhost) [tdb]> show create table t1 \G ;
*************************** 1. row ***************************
       Table: t1
Create Table: CREATE TABLE `t1` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `a` int(11) NOT NULL,
  `b` int(11) DEFAULT NULL,
  `c` int(11) DEFAULT NULL,
  `name` varchar(50) NOT NULL,
  `status` tinyint(4) NOT NULL,
  `createtime` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  KEY `a` (`a`,`b`,`name`,`c`),
  KEY `b` (`b`),
  KEY `a_2` (`a`),
  KEY `a_3` (`a`,`c`),
  KEY `a_4` (`a`,`b`,`c`)
) ENGINE=InnoDB AUTO_INCREMENT=100003 DEFAULT CHARSET=utf8mb4
1 row in set (0.00 sec)

ERROR: 
No query specified
    
(root@localhost) [tdb]> explain select * from t1 where a=123 ;
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref   | rows | filtered | Extra |
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | t1    | NULL       | ref  | a,a_2,a_3,a_4 | a    | 4       | const |   19 |   100.00 | NULL  |
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-------+

(root@localhost) [tdb]> explain select * from t1 where a=123 or b=123 ;
+----+-------------+-------+------------+-------------+---------------+------+---------+------+------+----------+------------------------------------+
| id | select_type | table | partitions | type        | possible_keys | key  | key_len | ref  | rows | filtered | Extra                              |
+----+-------------+-------+------------+-------------+---------------+------+---------+------+------+----------+------------------------------------+
|  1 | SIMPLE      | t1    | NULL       | index_merge | a,b           | a,b  | 4,5     | NULL |   29 |   100.00 | Using sort_union(a,b); Using where |
+----+-------------+-------+------------+-------------+---------------+------+---------+------+------+----------+------------------------------------+
{% endhighlight %}

7.key_len
用于处理查询的索引长度，如果是单列索引，那就整个索引长度算进去，如果是多列索引，那么查询不一定都能使用到所有的列，具体使用到了多少个列的索引，这里就会计算进去，没有使用到的列，这里不会计算进去。留意下这个列的值，算一下你的多列索引总长度就知道有没有使用到所有的列了。要注意，mysql的ICP特性使用到的索引不会计入其中。另外，key_len只计算where条件用到的索引长度，而排序和分组就算用到了索引，也不会计算到key_len中。  
在这里 key_len 大小的计算规则是：
一般地，key_len 等于索引列类型字节长度，例如int类型为4 bytes，bigint为8 bytes  
如果是字符串类型，还需要同时考虑字符集因素，例如：CHAR(30) UTF8则key_len至少是90 bytes，utf8mb4 则key_len至少是120  
若该列类型定义时允许NULL，其key_len还需要再加 1 bytes  
若该列类型为变长类型，例如 VARCHAR（TEXT\BLOB不允许整列创建索引，如果创建部分索引也被视为动态列类型），其key_len还需要再加 2 bytes  
sql能用上符合索引中几个字段，再开一篇另讲  
{% highlight doc %}
(root@localhost) [tdb]> show create table t1 \G;
*************************** 1. row ***************************
       Table: t1
Create Table: CREATE TABLE `t1` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `a` int(11) NOT NULL,
  `b` int(11) DEFAULT NULL,
  `c` int(11) DEFAULT NULL,
  `name` varchar(50) NOT NULL,
  `status` tinyint(4) NOT NULL,
  `createtime` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  KEY `a` (`a`,`b`,`name`,`c`)
) ENGINE=InnoDB AUTO_INCREMENT=100003 DEFAULT CHARSET=utf8mb4
1 row in set (0.00 sec)

ERROR: 
No query specified

(root@localhost) [tdb]> 
(root@localhost) [tdb]> explain  select * from t1 where a=1 ;
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref   | rows | filtered | Extra |
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | t1    | NULL       | ref  | a             | a    | 4       | const |   15 |   100.00 | NULL  |
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-------+
4

(root@localhost) [tdb]> explain  select * from t1 where a=1 and b=1 ;
+----+-------------+-------+------------+------+---------------+------+---------+-------------+------+----------+-------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref         | rows | filtered | Extra |
+----+-------------+-------+------------+------+---------------+------+---------+-------------+------+----------+-------+
|  1 | SIMPLE      | t1    | NULL       | ref  | a             | a    | 9       | const,const |    1 |   100.00 | NULL  |
+----+-------------+-------+------------+------+---------------+------+---------+-------------+------+----------+-------+
9=4+(4+1)

(root@localhost) [tdb]> explain  select * from t1 where a=1 and b=1 and name='aa';
+----+-------------+-------+------------+------+---------------+------+---------+-------------------+------+----------+-------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref               | rows | filtered | Extra |
+----+-------------+-------+------------+------+---------------+------+---------+-------------------+------+----------+-------+
|  1 | SIMPLE      | t1    | NULL       | ref  | a             | a    | 211     | const,const,const |    1 |   100.00 | NULL  |
+----+-------------+-------+------------+------+---------------+------+---------+-------------------+------+----------+-------+
211=4+(4+1)+(50*4+2)
{% endhighlight %}

8.ref
如果是使用的索引常数等值查询，这里会显示const，如果是连接查询，被驱动表的执行计划这里会显示驱动表的关联字段，可能还有其他情况，没测出来  
{% highlight doc %}
(root@localhost) [tdb]> show create table t1 \G;
*************************** 1. row ***************************
       Table: t1
Create Table: CREATE TABLE `t1` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `a` int(11) NOT NULL,
  `b` int(11) DEFAULT NULL,
  `c` int(11) DEFAULT NULL,
  `name` varchar(50) NOT NULL,
  `status` tinyint(4) NOT NULL,
  `createtime` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  KEY `a` (`a`,`b`,`name`,`c`)
) ENGINE=InnoDB AUTO_INCREMENT=100003 DEFAULT CHARSET=utf8mb4
1 row in set (0.00 sec)

ERROR:
No query specified

(root@localhost) [tdb]>
(root@localhost) [tdb]> explain  select * from t1 where a=1 ;
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref   | rows | filtered | Extra |
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | t1    | NULL       | ref  | a             | a    | 4       | const |   15 |   100.00 | NULL  |
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-------+

(root@localhost) [tdb]> show create table tbyunmu_order  \G;
*************************** 1. row ***************************
       Table: tbyunmu_order
Create Table: CREATE TABLE `tbyunmu_order` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `user_id` int(11) NOT NULL,
  `pro` varchar(20) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `user_id` (`user_id`)
) ENGINE=InnoDB AUTO_INCREMENT=9 DEFAULT CHARSET=utf8mb4
1 row in set (0.00 sec)

ERROR:
No query specified

(root@localhost) [tdb]> explain  select * from tbyunmu_order where user_id in (select user_id from tbyunmu_order a where id>4);
+----+-------------+---------------+------------+-------+-----------------+---------+---------+---------------+------+----------+-------------------------------------+
| id | select_type | table         | partitions | type  | possible_keys   | key     | key_len | ref           | rows | filtered | Extra                               |
+----+-------------+---------------+------------+-------+-----------------+---------+---------+---------------+------+----------+-------------------------------------+
|  1 | SIMPLE      | a             | NULL       | index | PRIMARY,user_id | user_id | 4       | NULL          |    8 |    75.00 | Using where; Using index; LooseScan |
|  1 | SIMPLE      | tbyunmu_order | NULL       | ref   | user_id         | user_id | 4       | tdb.a.user_id |    1 |   100.00 | NULL                                |
+----+-------------+---------------+------------+-------+-----------------+---------+---------+---------------+------+----------+-------------------------------------+
{% endhighlight %}

9.rows
这里是执行计划中根据统计信息估算的扫描行数，不是精确值，但是比较有参考意义，这个数值很大，说明可能需要大量的IO  
{% highlight doc %}
(root@localhost) [tdb]> show create table t1 \G;
*************************** 1. row ***************************
       Table: t1
Create Table: CREATE TABLE `t1` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `a` int(11) NOT NULL,
  `b` int(11) DEFAULT NULL,
  `c` int(11) DEFAULT NULL,
  `name` varchar(50) NOT NULL,
  `status` tinyint(4) NOT NULL,
  `createtime` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  KEY `a` (`a`,`b`,`name`,`c`)
) ENGINE=InnoDB AUTO_INCREMENT=100003 DEFAULT CHARSET=utf8mb4
1 row in set (0.00 sec)

ERROR: 
No query specified

(root@localhost) [tdb]> explain  select * from t1 where a=123 ;
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref   | rows | filtered | Extra |
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | t1    | NULL       | ref  | a             | a    | 4       | const |   19 |   100.00 | NULL  |
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)

(root@localhost) [tdb]> explain  select * from t1 where c=123 ;
+----+-------------+-------+------------+------+---------------+------+---------+------+-------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows  | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+------+---------+------+-------+----------+-------------+
|  1 | SIMPLE      | t1    | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 99489 |    10.00 | Using where |
+----+-------------+-------+------------+------+---------------+------+---------+------+-------+----------+-------------+

使用索引和不使用索引预估扫描的行数
{% endhighlight %}

10.filtered
它指返回结果的行占需要读到的行(rows列的值)的百分比,实际使用中发现这个值不是很准，猜测和统计信息相关。  

11.extra
这个列可以显示的信息非常多，有几十种,包含不适合在其他列中显示但十分重要的额外信息

using filesort：排序时无法使用到索引时，就会出现这个。常见于order by和group by语句中。  
把数据拉入server层，在内存中排序，如果超过sort_buffer_size 空间，则要使用swap区，所以执行计划中看到 using filesort，要引起注意，拉入server层的数据量不能太多  
{% highlight doc %}
(root@localhost) [tdb]> show create table t1 \G;
*************************** 1. row ***************************
       Table: t1
Create Table: CREATE TABLE `t1` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `a` int(11) NOT NULL,
  `b` int(11) DEFAULT NULL,
  `c` int(11) DEFAULT NULL,
  `name` varchar(50) NOT NULL,
  `status` tinyint(4) NOT NULL,
  `createtime` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  KEY `a` (`a`,`b`,`name`,`c`)
) ENGINE=InnoDB AUTO_INCREMENT=100003 DEFAULT CHARSET=utf8mb4
1 row in set (0.00 sec)

ERROR: 
No query specified

(root@localhost) [tdb]> explain  select * from t1 where a=123 order by c ;
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+---------------------------------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref   | rows | filtered | Extra                                 |
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+---------------------------------------+
|  1 | SIMPLE      | t1    | NULL       | ref  | a             | a    | 4       | const |   19 |   100.00 | Using index condition; Using filesort |
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+---------------------------------------+
1 row in set, 1 warning (0.00 sec)

(root@localhost) [tdb]> explain  select c,count(a)  from t1 where a=123 group by c ;  
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-----------------------------------------------------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref   | rows | filtered | Extra                                                     |
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-----------------------------------------------------------+
|  1 | SIMPLE      | t1    | NULL       | ref  | a             | a    | 4       | const |   19 |   100.00 | Using where; Using index; Using temporary; Using filesort |
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-----------------------------------------------------------+
{% endhighlight %}

using temporary：表示使用了临时表存储中间结果,常见于排序和分组查询,临时表占用tmp_table_size空间，如果超出，也要使用swap区，所以执行计划中看到 using temporary，要引起注意，拉入server层的数据量不能太多。使用filesort和temporary的话会很吃力，WHERE和ORDER BY的索引经常无法兼顾，如果按照WHERE来确定索引，那么在ORDER BY时，就必然会引起Using filesort，这就要看是先过滤再排序划算，还是先排序再过滤划算。
{% highlight doc %}
(root@localhost) [tdb]> show create table t1 \G;
*************************** 1. row ***************************
       Table: t1
Create Table: CREATE TABLE `t1` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `a` int(11) NOT NULL,
  `b` int(11) DEFAULT NULL,
  `c` int(11) DEFAULT NULL,
  `name` varchar(50) NOT NULL,
  `status` tinyint(4) NOT NULL,
  `createtime` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  KEY `a` (`a`,`b`,`name`,`c`)
) ENGINE=InnoDB AUTO_INCREMENT=100003 DEFAULT CHARSET=utf8mb4

(root@localhost) [tdb]> explain  select c,count(a)  from t1 where a=123 group by c ;
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-----------------------------------------------------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref   | rows | filtered | Extra                                                     |
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-----------------------------------------------------------+
|  1 | SIMPLE      | t1    | NULL       | ref  | a             | a    | 4       | const |   19 |   100.00 | Using where; Using index; Using temporary; Using filesort |
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-----------------------------------------------------------+
{% endhighlight %}

using where: 表示存储引擎返回的记录并不是所有的都满足查询条件，需要在server层进行过滤。查询条件中分为限制条件和检查条件，5.6之前，存储引擎只能根据限制条件扫描数据并返回，然后server层根据检查条件进行过滤再返回真正符合查询的数据,看到using where 看下有多大的数据量要被拉到server层，如果大的话可以考虑建立更合适的索引.  
using index condition:5.6.x之后支持ICP特性，可以把检查条件也下推到存储引擎层，不符合检查条件和限制条件的数据，直接不读取，这样就大大减少了存储引擎扫描的记录数量。extra列显示using index condition,过滤工作都在存储引擎层完成，性能不错  
{% highlight doc %}
(root@localhost) [tdb]> show create table t1 \G;
*************************** 1. row ***************************
       Table: t1
Create Table: CREATE TABLE `t1` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `a` int(11) NOT NULL,
  `b` int(11) DEFAULT NULL,
  `c` int(11) DEFAULT NULL,
  `name` varchar(50) NOT NULL,
  `status` tinyint(4) NOT NULL,
  `createtime` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  KEY `a` (`a`,`b`,`name`,`c`)
) ENGINE=InnoDB AUTO_INCREMENT=100003 DEFAULT CHARSET=utf8mb4
1 row in set (0.00 sec)

ERROR: 
No query specified

(root@localhost) [tdb]> explain  select *  from t1 where a=123  and status=-1   ;
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref   | rows | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-------------+
|  1 | SIMPLE      | t1    | NULL       | ref  | a             | a    | 4       | const |   19 |    10.00 | Using where |
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-------------+

(root@localhost) [tdb]> explain  select *  from t1 where a=123  and c=345  ;
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-----------------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref   | rows | filtered | Extra                 |
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-----------------------+
|  1 | SIMPLE      | t1    | NULL       | ref  | a             | a    | 4       | const |   19 |    10.00 | Using index condition |
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-----------------------+
{% endhighlight %}

using index：查询时不需要回表查询，直接通过索引就可以获取查询的数据。性能不错  
{% highlight doc %}
(root@localhost) [tdb]> show create table t1 \G;
*************************** 1. row ***************************
       Table: t1
Create Table: CREATE TABLE `t1` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `a` int(11) NOT NULL,
  `b` int(11) DEFAULT NULL,
  `c` int(11) DEFAULT NULL,
  `name` varchar(50) NOT NULL,
  `status` tinyint(4) NOT NULL,
  `createtime` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  KEY `a` (`a`,`b`,`name`,`c`)
) ENGINE=InnoDB AUTO_INCREMENT=100003 DEFAULT CHARSET=utf8mb4
1 row in set (0.00 sec)

ERROR: 
No query specified

(root@localhost) [tdb]> explain select a,b from t1 where a=123 ;
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref   | rows | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-------------+
|  1 | SIMPLE      | t1    | NULL       | ref  | a             | a    | 4       | const |   19 |   100.00 | Using index |
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-------------+
{% endhighlight %}

using intersect：表示使用and的各个索引的条件时，该信息表示是从处理结果获取交集
using union：表示使用or连接各个使用索引的条件时，该信息表示从处理结果获取并集
{% highlight doc %}
(root@localhost) [tdb]> show create table t1 \G;
*************************** 1. row ***************************
       Table: t1
Create Table: CREATE TABLE `t1` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `a` int(11) NOT NULL,
  `b` int(11) DEFAULT NULL,
  `c` int(11) DEFAULT NULL,
  `name` varchar(50) NOT NULL,
  `status` tinyint(4) NOT NULL,
  `createtime` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  KEY `a` (`a`),
  KEY `b` (`b`)
) ENGINE=InnoDB AUTO_INCREMENT=100003 DEFAULT CHARSET=utf8mb4
1 row in set (0.00 sec)

ERROR: 
No query specified

(root@localhost) [tdb]> 
(root@localhost) [tdb]> 
(root@localhost) [tdb]> explain  select * from t1 where a=1 and b=2 ;
+----+-------------+-------+------------+-------------+---------------+------+---------+------+------+----------+-----------------------------------+
| id | select_type | table | partitions | type        | possible_keys | key  | key_len | ref  | rows | filtered | Extra                             |
+----+-------------+-------+------------+-------------+---------------+------+---------+------+------+----------+-----------------------------------+
|  1 | SIMPLE      | t1    | NULL       | index_merge | a,b           | b,a  | 5,4     | NULL |    1 |   100.00 | Using intersect(b,a); Using where |
+----+-------------+-------+------------+-------------+---------------+------+---------+------+------+----------+-----------------------------------+
1 row in set, 1 warning (0.00 sec)

(root@localhost) [tdb]> 
(root@localhost) [tdb]> 
(root@localhost) [tdb]> explain  select * from t1 where a=1 or  b=2 ;   
+----+-------------+-------+------------+-------------+---------------+------+---------+------+------+----------+-------------------------------+
| id | select_type | table | partitions | type        | possible_keys | key  | key_len | ref  | rows | filtered | Extra                         |
+----+-------------+-------+------------+-------------+---------------+------+---------+------+------+----------+-------------------------------+
|  1 | SIMPLE      | t1    | NULL       | index_merge | a,b           | a,b  | 4,5     | NULL |   21 |   100.00 | Using union(a,b); Using where |
+----+-------------+-------+------------+-------------+---------------+------+---------+------+------+----------+-------------------------------+
{% endhighlight %}


