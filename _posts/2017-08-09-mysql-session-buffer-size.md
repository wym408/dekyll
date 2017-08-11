---
layout: post
title:  "mysql session 级别buffer变量"
date:   2017-07-27 16:15:49 +0700
categories: jekyll update
---
mysql 变量分为全局变量和会话变量，全局变量对整个实例有效，会话变量只对当前会话有效。  
本篇介绍几个会话级别的内存变量，这几个变量的特点是不能分配太大，因为每个会话都有可能请求分配，
如果太大会造成内存不够用  


1.join_buffer_size  
两表（或多表）join的操作需求，MySQL在完成某些join需求的时候（all row join/all index /scan join）为了减少参与join的“被驱动表”的读取次数以提高性能，需要使用到join buffer来协助完成join操作  
当join buffer 太小，mysql不会将该buffer存入磁盘文件而是先将join buffer中的结果与需求join的表进行操作，然后清空join buffer中的数据，继续将剩余的结果集写入次buffer中，如此往复，这势必会造成被驱动表需要被多次读取，成倍增加IO访问，降低效率（执行计划中如果现实using join buffer）  
2表join分配1次 join_buffer_size，3表join 分配2次join_buffer_size，所以不是每个session只分配1次  
建议适当加大join_buffer_size，比如 32M 64M,不能太大，多线程会消耗掉太多内存  
当然最好的办法是建立适当的索引，不要用到join_buffer_size空间   

2.sort_buffer_size  
排序缓存，每个线程在第一次需要用到时分配，不能太大，建议4M,8M

3.read_rnd_buffer_size
MySQL的随机读缓冲区大小，当按随机顺序读取行时，将分配一个随机读取缓冲区，先把数据读入到缓冲区，在缓冲区按照主键排序，然后再按照顺序读取聚簇树，提高查询速度，如果需要大量数据可适当的调整该值，但MySQL会为每个客户连接分配该缓冲区所以尽量适当设置该值，以免内存开销过大,建议设置在8M,16M。二级索引查询和连表查询都可以用到。  

4.tmp_table_size   
内存临时表大小，在sort，group by 时可能产生临时表，如果临时表空间需要的空间大于tmp_table_size值，则会用到swap区影响性能，此变量是线程级别，尽量适当设置该值，以免内存开销过大，建议32M,64M。避免产生过大临时表，尽量使用好索引，使进入临时表的数据量减少。


