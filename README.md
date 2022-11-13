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

To improve your performance you can set the `thread_cache_size` to a relatively high number. To find the thread cache hit rate, you can use the following technique:
```
mysql> show status like 'Threads_created';
mysql> show status like 'Connections';
```

Now use the following formula to calculate the thread cache hit rate percentage:
```
100 - ((Threads_created / Connections) * 100)
```
or (based on [this article](https://releem.com/docs/mysql-performance-tuning/thread_cache_size))
```
8 + (max_connections / 100)
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
