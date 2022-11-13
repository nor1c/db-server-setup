## Table of Content
- [Basic Commands](https://github.com/nor1c/db-server-setup#-basic-commands)
- [Links](https://github.com/nor1c/db-server-setup#-links)
- [Database Optimization](https://github.com/nor1c/db-server-setup#-database-optimization)


## # Basic Commands
- Find where MySQL `my.cnf` is located
```
# find / -name my.cnf
```


## # Links

- Database replication: https://www.digitalocean.com/community/tutorials/how-to-set-up-master-slave-replication-in-mysql
- Database backup: http://www.ducea.com/2006/05/27/backup-your-mysql-databases-automatically-with-automysqlbackup/ or https://www.backuphowto.info/how-backup-mysql-database-automatically-linux-users
- Database optimization: https://www.tecmint.com/mysql-mariadb-performance-tuning-and-optimization/ and http://www.monitis.com/blog/101-tips-to-mysql-tuning-and-optimization/
- And for the networking, this tutorial may be helpful http://www.yolinux.com/TUTORIALS/LinuxTutorialNetworking.html
- Releem docs https://releem.com/docs/getstarted
- MySQL Performance Parameters https://releem.com/docs/mysql-performance-parameters


## # Database Optimization

### # Avoid Swappiness in MySQL

Swapping is process that occurs when system moves part of memory to a special disk space called “swap”. The event usually appears when your system runs out of physical memory and instead of freeing up some RAM, the system pushed the information into disk. As you might have guess the disk is much slower than your RAM.

By default the option is enabled:
```
# sysctl vm.swappiness 

vm.swappiness = 60
```

To disable swappiness, run the following command:
```
# sysctl -w vm.swappiness=0
```


### # Optimizing InnoDB buffer pool Usage

The InnoDB engine has a buffer pool used for caching data and indexes in memory. This of course will help your MySQL/MariaDB queries be executed significantly faster. Choosing the proper size here requires some very important decisions and good knowledge on your system’s memory consumption.

Here is what you need to consider:
- How much memory you need for other processes. This includes your system processes, page tables, socket buffers.
- Is your server dedicated for MySQL or you will be running other memory hungry services.

On a dedicated box, you would probably want to give about 60-70% of the memory to the innodb_buffer_pool_size. If you plan on running more services on a single box, you should re-consider the amount of memory you dedicate for your innodb_buffer_pool_size.

The value that you should edit in my.cnf is:
```
innodb_buffer_pool_size
```


### # Set MySQL Max Connections

The `max_connections` directive tells your server how many concurrent connections are permitted. The MySQL/MariaDB server allows the value given in max_connections + 1 for user with SUPER privileges. The connection is opened only for the time MySQL query is executed – after that it is closed and new connection can take its place.

Keep in mind that too many connections can cause high RAM usage and lock up your MySQL server. Usually small websites will require between **100-200** connections while larger may require **500-800** or even more. The value you apply here strongly depends on your particular MySQL/MariaDB usage.

You can dynamically change the value of `max_connections`, without having to restart the MySQL service by running:
```
# mysql -u root -p
mysql> set global max_connections := 300;
```

or add `max_connections` to `my.cnf`
```
...
max_connections=2000
```

### # Configure MySQL thread_cache_size

The `thread_cache_size` directive sets the amount of threads that your server should cache. As the client disconnects, his threads are put in the cache if they are less than the `thread_cache_size`. Further requests are completed by using the threads stored in the cache.

The default value of thread_cache_size is based on this formula:
```
8 + (max_connections / 100)
```

To improve your performance you can set the `thread_cache_size` to a relatively high number. To find the thread cache hit rate, you can use the following technique:
```
mysql> show status like 'Threads_created';
mysql> show status like 'Connections';
```

Now use the following formula to calculate the thread cache hit rate percentage:
```
100 - ((Threads_created / Connections) * 100)
```

If you get a low number, it means that most of the new mysql connections are starting new thread instead of loading from cache. You will surely want to increase the thread_cache_size in such cases.

The good thing here is that the `thread_cache_size` can be dynamically changed without having to restart the MySQL service. You can achieve this by running:

```
mysql> set global thread_cache_size = 16;
```

or add `thread_cache_size` to `my.cnf`

```
...
thread_cache_size=16
```

<br>

Other articles that talks about **thread_cache_size**:
- https://releem.com/docs/mysql-performance-tuning/thread_cache_size

### # Disable MySQL Reverse DNS Lookups

By default MySQL/MariaDB perform a DNS lookup of the user’s IP address/Hostname from which the connection is coming. For each client connection, the IP address is checked by resolving it to a host name. After that the host name is resolved back to an IP to verify that both match.

This unfortunately may cause delays in case of badly configured DNS or problems with DNS server. This is why you can disable the reverse DNS lookup by adding the following in your configuration file:

```
[mysqld]
...
skip-name-resolve
```

You will have to restart the MySQL service after applying these changes.

### # Configure MySQL query_cache_size

If you have many repetitive queries and your data does not change often – use query cache. People often do not understand the concept behind the `query_cache_size` and set this value to gigabytes, which can actually cause degradation in the performance.

The reason behind that is the fact that threads need to lock the cache during updates. Usually value of 200-300 MB should be more than enough. If your website is relatively small, you can try giving the value of 64M and increase in time.

You will have to add the following settings in the MySQL configuration file:
```
query_cache_type = 1
query_cache_limit = 256K
query_cache_min_res_unit = 2k
query_cache_size = 80M
```

### # Configure tmp_table_size and max_heap_table_size

Both directives should have the same size and will help you prevent disk writes. The tmp_table_size is the maximum amount of size of internal in-memory tables. In case the limit in question is exceeded the table will be converted to on-disk MyISAM table.

This will affect the database performance. Administrators usually recommend giving 64M for both values for every GB of RAM on the server.

```
[mysqld]
tmp_table_size= 64M
max_heap_table_size= 64M
```

### # Enable MySQL Slow query Logs

Logging slow queries can help you determine issues with your database and help you debug them. This can be easily enabled by adding the following values in your MySQL configuration file:

```
slow-query-log = 1
slow-query-log-file = /var/lib/mysql/mysql-slow.log
long_query_time = 1
```

The first directive enables the logging of slow queries, while the second one tells MySQL where to store the actual log file. Use `long_query_time` to define the amount of time that is considered long for MySQL query to be completed.

### # Check for MySQL idle Connections

Idle connections consume resources and should be interrupted or refreshed when possible. Such connections are in "sleep" state and usually stay that way for long period of time. To look for idled connections you can run the following command:

```
# mysqladmin processlist -u root -p | grep "Sleep"
```

This will show you list of processes that are in sleep state. The event appears when the code is using persistent connection to the database. When using PHP this event can appear when using `mysql_pconnect` which opens the connection, after that executes queries, removes the authentication and leaves the connection open. This will cause any per-thread buffers to be kept in memory until the thread dies.

The first thing you would do here is to check the code and fix it. If you don’t have access to the code that is being ran, you can change the `wait_timeout` directive. The default value is **28800** seconds, while you can safely decrease it to something like **60**:

```
wait_timeout=60
```

### # Set MySQL max_allowed_packet

MySQL splits data into packets. Usually a single packet is considered a row that is sent to a client. The `max_allowed_packet` directive defines the maximum size of packet that can be sent.

Setting this value too low can cause a query to stall and you will receive an error in your MySQL error log. It is recommended to set the value to the size of your largest packet.

### # Check MySQL Performance Tuning

Measuring your MySQL/MariaDB performance is something that you should do on regular basis. This will help you see if something in the resource usage changes or needs to be improved.

There are plenty of tools available for benchmarking, but I would like to suggest you one that is simple and easy to use. The tool is called **mysqltuner**.

To download and run it, use the following set of commands:

```
# wget https://github.com/major/MySQLTuner-perl/tarball/master
# tar xf master
# cd major-MySQLTuner-perl-993bc18/
# ./mysqltuner.pl 
```

You will receive a detailed report about your MySQL usage and recommendation tips. Here is a sample output of default MariaDB installation:
![image](https://user-images.githubusercontent.com/7555972/201525859-464d5496-7852-4a3b-bdcc-82f480d6ebb8.png)

### # Optimize and Repair MySQL Databases

Sometimes MySQL/MariaDB database tables get crashed quite easily, especially when unexpected server shut down, sudden file system corruption or during copy operation, when database is still accessed. Surprisingly, there is a free open source tool called **mysqlcheck**, which automatically check, repair and optimize databases of all tables in Linux.

```
# mysqlcheck -u root -p --auto-repair --check --optimize --all-databases
# mysqlcheck -u root -p --auto-repair --check --optimize databasename
```
