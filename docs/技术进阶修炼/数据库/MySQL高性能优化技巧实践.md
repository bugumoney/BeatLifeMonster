# MySQL高性能优化技巧实践

### InnoDB 缓冲池

InnoDB 引擎在内存中有一个缓冲池用于缓存数据和索引，这有助于你更快地执行 MySQL 查询语句。选择合适的内存大小需要一些重要的决策并对系统的内存消耗有较多的认识，你需要考虑：
- 其它的进程需要消耗多少内存，这包括你的系统进程，页表，套接字缓冲。
- 你的服务器是否专门用于 MySQL，还是你运行着其它非常消耗内存的服务。

在一个专用的机器上，你可能会把 60-70％ 的内存分配给`Innodb_buffer_pool_size`，如果你打算在一个机器上运行更多的服务，你应该重新考虑专门用于 innodb_buffer_pool_size 的内存大小，此时需要关注一下几个指标，InnoDB缓冲池可用页面的数量，使用了多少，总计多少，以及缓冲池的使用率。通过这些指标数据判断数据库的健康状况以及调节内存。

- Innodb_buffer_pool_free
- Innodb_buffer_pool_total
- Innodb_buffer_pool_used
- Innodb_buffer_pool_utilization

```sql
show global status like 'Innodb_buffer_pool%';
+---------------------------------------+--------------+
| Variable_name                         | Value        |
+---------------------------------------+--------------+
| Innodb_buffer_pool_pages_data         | 24108        |
| Innodb_buffer_pool_bytes_data         | 394985472    |
| Innodb_buffer_pool_pages_dirty        | 948          |
| Innodb_buffer_pool_bytes_dirty        | 15532032     |
| Innodb_buffer_pool_pages_flushed      | 759467581    |
| Innodb_buffer_pool_pages_free         | 0            |
| Innodb_buffer_pool_pages_misc         | 467          |
| Innodb_buffer_pool_pages_total        | 24575        |
| Innodb_buffer_pool_read_ahead_rnd     | 0            |
| Innodb_buffer_pool_read_ahead         | 40166721     |
| Innodb_buffer_pool_read_ahead_evicted | 2303581      |
| Innodb_buffer_pool_read_requests      | 960676643611 |
| Innodb_buffer_pool_reads              | 4728901556   |
| Innodb_buffer_pool_wait_free          | 0            |
| Innodb_buffer_pool_write_requests     | 11117197708  |
+---------------------------------------+--------------+
```



### MySQL 进程数/最大连接数

`threads_connected`：当前客户端已连接的数量；
这个值会少于预设的值，但你也会监控到这个值较大，它可保证客户端是处在活跃状态。如果`threads_connected == max_connections`时，数据库系统就不能提供更多的连接数了，这时如果程序还想新建连接线程，数据库系统就会拒绝，如果程序没做过多的错误处理，就会出现报错信息。

`threads_running`： 处于激活状态的线程的个数；
如果数据库超负荷了，你将会得到一个正在（查询的语句持续）增长的数值。这个值也可以少于预先设定的值。这个值在很短的时间内超过限定值是没问题的。当threads_running值超过预设值时并且该值在5秒内没有回落时，要同时监视其他的一些值。

`max_connections`：当前服务器允许多少并发连接；
MySQL 服务器允许有 SUPER 权限的用户在最大连接之外再建立一个连接。只有当执行 MySQL 请求的时候才会建立连接，执行完成后会关闭连接并被新的连接取代。但要记住，过多的连接会导致内存的使用量过高并且会锁住你的 MySQL 服务器。一般小网站需要100-200 的连接数，而较大可能需要500-800甚至更多。此值很大程度上取决于你 MySQL 的使用情况。

```sql
show status like '%threads%';
+------------------------+-------+
| Variable_name          | Value |
+------------------------+-------+
| Threads_cached         | 1     |
| Threads_connected      | 33    |
| Threads_created        | 32765 |
| Threads_running        | 16    |
+------------------------+-------+
```



### MySQL临时表和内存表

临时表可以使用任何存储引擎，临时表只在单个连接中可见，当连接断开时，临时表也会消失。MySQL最初会将临时表创建在内存中，当数据变的太大后，就会转储到磁盘上。内存表是指用MEMORY引擎创建的表。表结构存在于磁盘，数据放在内存中。

如果起初在内存中创建的临时表变的太大，MySQL会自动将其转成磁盘上的临时表。内存中的临时表由`tmp_table_size`和`max_heap_table_size`两个参数决定，这与创建MEMORY引擎的表不同。MEMORY引擎的表由`max_heap_table_size`参数决定表的大小，并且它不会转成到在磁盘上的格式。

每次创建临时表（包括内存上和磁盘上），`Created_tmp_tables`都会增加，如果是在磁盘上创建临时表（包括从内存上转成磁盘的），`Created_tmp_disk_tables`也会增加，`Created_tmp_files`表示MySQL服务创建的临时文件文件数；

比较理想的配置是：`created_tmp_disk_tables / created_tmp_tables * 100% <= 25%`

```sql
show global status like '%tmp%';
+-------------------------+---------+
| Variable_name           | Value   |
+-------------------------+---------+
| Created_tmp_disk_tables | 1262422 |
| Created_tmp_files       | 213     |
| Created_tmp_tables      | 5518097 |
+-------------------------+---------+

# 查看tmp_table_size
# tmp_table_size 临时表的内存缓存大小( 临时表是指sql执行时生成临时数据表 )
# 默认值 16777216
# 最小值 1
# 最大值 18446744073709551615
# 单位字节 默认值也就是16M多
show global variables like 'tmp_table_size';

# 设置tmp_table_size（立即生效重启后失效）
set global tmp_table_size = 1024 * 1024 * 64;
```

**修改MySQL配置文件my.cnf**

```sql
[mysqld]
tmp_table_size = 100000000
```

首先在优化sql的时候就应该尽量避免临时表，如果必须使用临时表且同时执行大量sql生成大量临时表时适当增加tmp_table_size；如果生成的临时表数据量大于tmp_table_size则会将临时表存储于磁盘而不是内存；

**注意**
MySQL中的`max_heap_table_size`参数也会影响到临时表的内存缓存大小，max_heap_table_size 是MEMORY内存引擎的表大小，因为临时表也是属于内存表，所以也会受此参数的限制，所以如果要增加tmp_table_size的大小，也需要同时增加 max_heap_table_size 的大小；

可以通过`Created_tmp_disk_tables`和`Created_tmp_tables`状态来分析是否需要增加tmp_table_size；

```sql
# 查看状态
show global status like 'Created_tmp_disk_tables';
show global status like 'Created_tmp_tables';
show global status like 'Created_tmp_files';

# Created_tmp_disk_tables : 磁盘临时表的数量
# Created_tmp_tables      : 内存临时表的数量
# Created_tmp_files
```

###MySQL 查询
数据库是很容易产生瓶颈的地方，其中最影响速度的就是那些查询非常慢的语句，这些慢查询语句，可能是写的不够合理或者是大数据下多表的联合查询等等，所以要重点监控找出这些语句并进行优化。

```sql
show status like '%quer%';
show status like '%questions%';
+-------------------------+-----------+
| Variable_name           | Value     |
+-------------------------+-----------+
| Queries                 | 31903     |
| Questions               | 10        |
| Slow_queries            | 0         |
+-------------------------+-----------+

show variables like '%quer%';
+-------------------------------+-------------------------------------------+
| Variable_name                 | Value                                     |
+-------------------------------+-------------------------------------------+
| ft_query_expansion_limit      | 20                                        |
| have_query_cache              | YES                                       |
| log_queries_not_using_indexes | OFF                                       |
| log_slow_queries              | OFF                                       |
| long_query_time               | 10.000000                                 |
| query_alloc_block_size        | 8192                                      |
| query_cache_limit             | 1048576                                   |
| query_cache_min_res_unit      | 4096                                      |
| query_cache_size              | 134217728                                 |
| query_cache_type              | ON                                        |
| query_cache_wlock_invalidate  | OFF                                       |
| query_prealloc_size           | 8192                                      |
| slow_query_log                | OFF                                       |
| slow_query_log_file           | /usr/local/mysql/var/mysql-2-205-slow.log |
+-------------------------------+-------------------------------------------+
```

`Queries`由服务器执行的语句的数目，该变量包括存储程序中执行的语句，不像`Questions`变量。它不包含 COM_PING 和 COM_STATISTICS 命令。这个变量添加近 MySQL 5.0.76版本中。

`Questions`由服务器执行的语句的数目，如在 MySQL 5.0.72版本，它包括由客户机发送到服务器的执行状态，不再包含存储程序中执行语句，不同于queries变量。这个变量不包含COM_PING，COM_STATISTICS，COM_STMT_PREPARE，COM_STMT_CLOSE和COM_STMT_RESET等命令。

`Queries`是一个全局性状态变量，而`Questions`是一个会话，可以用来看看有多少会话通过当前连接发到服务器。queries速度上升和下降都是正常的，它不是一个固定阈值的指标。但需要注意的是如果其数值发生急剧下降等突然变化，那就可能出现了严重问题。

`Slow_queries`：查询时间超过long_query_time时间的数量，如何定义一个慢查询取决于数据库的使用情况和性能要求。但总之如果慢查询的数量很高，那你需要记录慢查询来定位数据库中的问题并进行调试。

通过在MySQL 配置文件中添加以下值来启用：

```sql
# 启用慢查询日志
slow-query-log = 1

# 指定MySQL实际的日志文件存储位置
slow-query-log-file = /var/lib/mysql/mysql-slow.log

# 完成MySQL查询多少用时算慢查询
long_query_time = 1
```

### MySQL 的查询缓存

如果你有很多重复的查询并且数据不经常改变那建议使用缓存查询。人们经常不理解 `query_cache_size`的实际含义而将它设置为 GB 级，但这样设置实际上会降低服务器的性能。

原因是在更新过程中线程需要锁定缓存。通常设置为 200-300 MB应该足够了。如果你的网站比较小的，你可以尝试给 64M 并在以后需要时及时增加。

```sql
# 查询缓存命中率
mysql.performance.qcache_hits

#键缓存利用率
mysql.performance.key_cache_utilization
```

### MySQL 主从复制

说了那么多，还有一个很重要的 MySQL 主从复制，主从复制的好处不用多说：采用主从服务器这种架构，稳定性得以提升。如果主服务器发生故障，可以使用从服务器来提供服务。在主从服务器上分开处理用户的请求，可以提升数据处理效率。将主服务器上的数据复制到从服务器上，保护数据免受意外的损失等等。

```sql
# 查看连接状态
mysql>SHOW SLAVE STATUS\G；
```

在 Master 与 Slave 之间完成一个异步的复制过程需要由三个线程来完成，其中两个线程（ Sql 线程和 IO 线程）在 Slave 端，另外一个线程（ IO 线程）在 Master 端。`Slave_IO_Running` 和 `Slave_SQL_Running` 两列的值都为 "Yes"，这表明 Slave 的 I/O 和 SQL 线程都在正常运行，主从同步功能也就是正常的。

```sql
mysql.replication.slave_io_running
mysql.replication.slave_sql_running
mysql.replication.last_io_errno
mysql.replication.last_sql_errno
```



[参考]

- https://www.jianshu.com/p/1b5bdfdfad3a

