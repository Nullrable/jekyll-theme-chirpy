---
title: MySQL order by
author: nhsoft.lsd
categories: [MySQL]
tags: [MySQL,数据库]
pin: false
---

# 1.  innodb_sort_buffer_size
  在创建InnoDB索引时用于指定对数据排序的排序缓冲区的大小。利用这块内存把数据读进来进行内部排序然后写入磁盘。这个参数只会在创建索引的过程中被使用，不会用在后面的维护操作；在索引创建完毕后innodb_sort_buffer会被释放。
  这个值也控制了在执行online DDL期间DML产生的临时日志文件。
  默认 1048576 bytes (1MB).

# 2. sort_buffer_size
   mysql 5.6 默认大小256k，阿里云默认是1M，mysql5.7默认是1M

# 3. 三种排序模式 sort_mode
   < sort_key, rowid >对应的是MySQL 4.1之前的“原始排序模式”
   < sort_key, additional_fields >对应的是MySQL 4.1以后引入的“修改后排序模式”
   < sort_key, packed_additional_fields >是MySQL 5.7.3以后引入的进一步优化的”打包数据排序模式”


# 4. max_length_for_sort_data
   是 MySQL 中专门控制用于排序的行数据的长度的一个参数。
   它的意思是，如果单行的长度超过这个值，MySQL 就认为单行太大，要换一个算法（rowid）


# 5. 总结有三种算法
* a. 查询结果本身就是无序的，分rowid排序，packed_additional_fields（全字段排序），需要用到sort_buffer
* b. 查询结果本身就是有序的，则无需sort_buffer, 直接返回结果


# 6. mysql5.6引进“优先队列排序算法”


一些验证手段

/* 打开 optimizer_trace，只对本线程有效 */
SET optimizer_trace='enabled=on';

/* @a 保存 Innodb_rows_read 的初始值 */
select VARIABLE_VALUE into @a from  performance_schema.session_status where variable_name = 'Innodb_rows_read';

/* 执行语句 */
select city, name,age from m where city='杭州' order by name desc limit 1000;

/* 查看 OPTIMIZER_TRACE 输出 */
SELECT * FROM information_schema.OPTIMIZER_TRACE\G

/* @b 保存 Innodb_rows_read 的当前值 */
select VARIABLE_VALUE into @b from performance_schema.session_status where variable_name = 'Innodb_rows_read';

/* 计算 Innodb_rows_read 差值 */
select @b-@a;
