# Database Optimizations
This is likely to be the most difficult chapter in the whole documentation.
There are so many little parameters which can greatly affect your DB performance, and it depends on each setup, which of those parameters need tuning and which can be left well alone.

Luckily the remote pollers are much easier to handle, once you found a good balance on your main poller.
They are usually much less loaded than the central DB and therefore more forgiving if you miscalculated or mis-estimated a parameter.

I am talking just from our experiences here, and there might be many more things which we did not consider.
We went thru a whole series of troubleshooting until I was able to understand the whole setup properly in enough detail to come up with the current settings.
They are probably still not optimal, but I learned a lot about the internal workings of MySQL and collected also a few hints that help tuning those parameters.

## How much RAM is my DB using?
This is usually the first question that comes into the mind of a server operator, as RAM implies a significant amount of costs.
THe answer mainly depends on some static parameters as well as some that are influenced by the number of connections to the DB.
Here is a good resource to do a quick check, how much your DB will need based on your tuning considerations: https://www.mysqlcalculator.com/

Static parameters:
 * key_buffer_size
 * query_cache_size
 * tmp_table_size
 * innodb_buffer_pool_size
 * innodb_additional_mem_pool_size
 * innodb_log_buffer_size	

Connection based parameters (multiply by max_connections):
  * sort_buffer_size
  * read_buffer_size
  * read_rnd_buffer_size
  * join_buffer_size
  * thread_stack
  * binlog_cache_size

Total usage is roughly the sum of all static parameters plus the number of connections times the sum of all connection based parameters.

## Tuning considerations

This part describes different parameters and how they affect your database system.

I am not a DB expert, but in the past i read a lot of articles covering DB tuning, troubleshooting and analysis and used this information to tune our environment with some visible success.
Please take this document not as given facts, but as a guideline and a source of condensed information, so that you either don't need to search all the bits on your own, or to find a useful set of keywords to use for deeper researches.

### InnoDB buffers
```innodb_buffer_pool =	20G ... sky is the limit```

This is probably one of the parameters with the most impact.
It tells your DB, how much RAM it can use to cache all the data that it keeps.
The more space you offer to the DB, the more data can be kept in memory, which means significantly faster data-processing.
General advice is to try and keep the whole DB content in the RAM, as this significantly reduces disk accesses which are much slower than RAM accesses.
In larger setups, this can mean 20..30..40GB of buffer space to begin with.

Monitoring hints:
  * Check the buffer-usage by looking at the free buffer pages, try to keep at least around 10-20% of free pages

### Heap size and temp tables
```max_heap_table_size = 4G ... 6G```

This is the space limit for the creation of in-memory tables.
Cacti's boost-mechanism uses a memory-based table, which can benefit largel from this parameter.
Increasing the space would help to keep more elements in the table during re-sync, which can increase the processing speed from memory to disk.

```max_tmp_tables = 64```

Increasing this value allows the DB to create more temporary tables.
Those are used for caching the results when queries contain sub-select statements within other statements.

### join buffer
```join_buffer_size = 128k ... 256k```

This is used to cache results when a join is used in an SQL query.
Be careful with this setting, as tempting as it can be to try and increase it to speed up joins.
We are talking about a “per-connection” value, which means with many connections the RAM would be exhausted pretty fast!
With many connections (specially in the main DB) you can easily run into a congestion situation.
Decreasing this can actually help improving performance in setups with many parallel connections!

### query cache
```query_cache_size = 64M ... 256M```

The query cache is discussed on many webpages, and most of them come to the same conclusion:

Yes, it will help to speed up your queries, if you choose the right value - but this is something you need to do yourself, as every DB is different.
If you choose it too high, then the overhead for managing the cached items will be higher than actually repeating a query on the DB from scratch.
Unfortunately, this is a trial-and-error setting, as every setup is completely different and the effects of the cache are also unpredictable.
As a result, nobody can give any real life experiences that would match other situations (and specifically yours).
 
### SSD considerations
```
innodb_write_io_threads 	= 16 ... 64
innodb_read_io_threads = 16 ..64
innodb_io_capacity = 10000
```

If you followed my suggestions on storage, the you are most likely th on a local SSD or SSD-backed storage array.
Since those provides much higher IOPS performance, you can increase the number of parallel read and write threads from the default to a much higher value.
Same goes also for the actual number of disk-operations per second.

If you use a newer version of Mysql, there is a last value which you can tune:

```innodb_io_capacity_max =	20000```

General rule of thumb seems to be 2x capacity or 2000, which ever is higher.

### Network related parameters
```wait_timeout =	600```

If connections are unused, the SQL server waits by default a too long time until they are closed.
This can keep a lot of RAM allocated but not used, therefore you should decrease this parameter to a reasonable value.
Since Cacti poll cycles are usually up to 5 minutes long, it should be sufficient to keep the connection open for max 2 poll cycles or 10 minutes.

```net_read_timeout = 60```

The server waits this long for more data on the current connection before closing it.
Increasing this value might help to lower the DB-Load because on the same connection, several queries might be still in the caches and can be re-used.

```open_files_limit = 65535```

You need to increase this value before you can actually increase the max_connections value below.

**Attention! This value is connected with the max open files in the OS!**
(implemented via SystemD setting, as this needs a modification in the ulimit settings)

```max_connections = 8000``` (for main poller)
```max_connections = 1000``` (for remote systems)

This is important to balance according to your maximum parallel connections.
Depending on the number of logins and use of spine data collector processes and threads setting, MariaDB will need to handle a lot of  parallel connections.
The rough calculation for spine is:
```total_connections = total_processes * (total_threads + script_servers + 1)```
Also you should add a few connections to leave headroom for user connections, which will change depending on the number of concurrent login accounts. 

If you set this value too low, pollers can be permanently locked out of the DB by MySQL internal mechanisms, if they try to open more connections than the server offers.
In this case you will see messages like this in the SQL logs:

```Host 'IP' is blocked because of many connection errors; unblock with 'mysqladmin flush-hosts'```


**Attention! This value is connected with the open_files_limit parameter!**

We increased the max_connections in the past to 10000 to prevent locking the remote pollers out from the main DB due to too many parallel connections.
This can happen when the connection breaks and a new connection is opened by the poller before the old one is closed.
Our long-term statistics show, that this value is reached only in rare conditions, so I decreased the value again to create a bit more headroom in the RAM for more important data.
Probably we can lower the value even more, as 4000 is the current absolute maximum seen in our connections diagram - and even this was only due to a fault-situation.

Monitoring hint:
 * check the maximum parallel connections value - it is an all-time high that can be reset if needed (flush-status)

```max_allowed_packet =	32M```

This name is a bit misleading, it is actually the maximum allowed size of an assembled (from many network packets) SQL query that the server can store in its receive buffer.
The value should be kept high enough to allow faster processing of SQL data that is sent from the remote pollers to the central server (Poller recovery, boost, data transfer during poll-cycles)


### TODO: some values from our old environment
These need to be reviewed and checked.
```
key_buffer_size = 3G
sort_buffer_size = 256M
read_buffer_size = 8M
read_rnd_buffer_size = 8M
innodb_log_file_size = 256M
innodb_flush_log_at_trx_commit =	2
innodb_flush_method = O_DIRECT
tmp_table_size =	256M
thread_cache_size = 256 ? 	[-1 = auto]
query_cache_limit =	512M
table_definition_cache = 4096
table_open_cache = 5120
symbolic-links = 0
sql_mode =	NO_ENGINE_SUBSTITUTION, NO_AUTO_CREATE_USER   [, NO_ZERO_IN_DATE, ERROR_FOR_DIVISION_BY_ZERO]
```


### Databases in NUNMA setups
When you are using a multi-CPU based system, the RAM is not shared between those CPUs as you might think.
Each physical bar of RAM is merely assigned to one physical CPU.
If the other CPU requests some content on a RAM section which the other CPU owns, it needs to be fetched by that CPU and copied over to the other one.
So both CPUs are blocked at the time of access and this can significantly affect your system's performance.

There are two nice articles by Jeremy Cole which i have found on the net which talk about exactly this problem:

 * https://blog.jcole.us/2010/09/28/mysql-swap-insanity-and-the-numa-architecture/
 * https://blog.jcole.us/2012/04/16/a-brief-update-on-numa-and-mysql/


