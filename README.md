# Redis 教程


## 运维

### 注意事项
1. redis实际上线，需要设置配置timeout，即设置连接过期时间，不然容易造成连接超过maxclients数量，一般timeout=300（秒）
   并且添加空闲检测和验证，设置`tcp-keepalive=60`即每隔60秒进行连接活性检测，防止大量死链接占用资源
2. tcp-backlog，默认511，一般不需要更改，但是需要查看linux系统的`/proc/sys/net/core/somaxconn`的值，如果该值过小
    可以将值改为511
3. 设置慢查询监控
4. 设置client_biggest_input_buf，设置最大缓冲区，最大为10M
5. 查看ulimit -n的值，表示进程可以打开最大文件数，现在默认一般都很大了，如果值不大，可以调整大一些，如：`ulimit -n 100001`

### flushall
说明：如果不小心flushall操作了，并且开启了AOF，那么可以进行修复
```
config set auto-aof-rewrite-percentage 1000
config set auto-aof-rewrite-min-size 1000000000 # 主要是防止redis进行aof文件重写
# 去掉appendonly.aof末尾的flushall命令
# 检查修复后的文件
redis-check-aof
# 进行重启，在redis客户端的 `debug reload`并不会加载appendonly.aof；这里有个尴尬，就是从服务器的处理
# 感觉上从服务器只能
systemctl restart redis.service
```

### 哨兵
说明：由于暂时不清楚是哪个配置文件影响，哨兵启动起来后自己下线，所以采用简单一些的配置
sentinel.conf；当然每个哨兵都有自己的配置文件，其中的例如端口等配置文件需要相应更改
```
port 26379
daemonize yes
logfile "/usr/src/redis/sentinel01/redis.log"
dir "/usr/src/redis/sentinel01/data"
sentinel monitor mymaster 127.0.0.1 6379 2      # 最后的2，表示需要2个哨兵同一才能通过投票
sentinel down-after-milliseconds mymaster 30000
sentinel parallel-syncs mymaster 1
sentinel failover-timeout mymaster 15000
sentinel auth-pass mymaster foobared013310      # 密码
bind 127.0.0.1
```
备注：主节点的配置文件需要特别注意，需要配置`masterauth "foobared013310"`，应该有可能它会转化为从节点，那么需要用到这个
参考：<https://www.cnblogs.com/guolianyu/p/10249687.html>



## 监控命令
- `info clients` 查看客户端信息
- `info stats`   查看整体信息
- `client list`  返回客户端列表信息

## 注意事项
- 禁止在实际项目中使用`monitor`命令，会积累输出缓冲，造成内存飙升

## 集群

### 报错

#### 1. 报错
问题：`MASTER aborted replication with an error: NOAUTH Authentication re`
解决：在各个节点的配置文件中，添加`masterauth foobared013310`；备注：后面的字符串为密码

#### 2. 报错
问题：ip，端口和密码都设置对了，但依然无法连接，报`Uncaught RedisClusterException: Couldn't map cluster keyspace using any provided seed`
解决：redis节点可能绑定的ip为`bind 127.0.0.1`，改为`bind 0.0.0.0`即可

#### 3. 主节点宕机后启动报错
现象：复制从节点的appendonly.aof，主节点启动时报权限不足
```
redis Can't open the append-only file: Permission denied
```
原因：复制时用的root用户，所以粘贴的`appendonly.aof`属于root用户
措施：更改文件所有者所属组即可，`chown redis:redis appendonly.aof`

### 4. predis获取redis集群返回的`MOVED 10439 127.0.0.1:6380`
现象：这样返回127.0.0.1，predis没法跳转
原因：是因为设置集群meet时，采用的是`127.0.0.1`
措施：要么开始时设置注意采用公网ip，要么就更改节点配置文件（不是redis.conf哦，而是自动生成的），如：将`data/redis-6384/nodes-6384.conf`
     中的127.0.0.1更改为具体公网ip
yii2中的mojifan/yii2-predis配置，当然实际是配置predis
```
'redis' => [
            'class' => 'mojifan\redis\Connection',
            // 尽量全部写完，虽然可以通过一个节点获取另外一个节点信息，但是没法设置密码
            'servers' => [
                ['host' => '132.232.70.99', 'port' => 6379, 'password' => 'foobared013310', 'database' => 0, 'alias' => 'master-first'],
                ['host' => '132.232.70.99', 'port' => 6380, 'password' => 'foobared013310', 'database' => 0, 'alias' => 'master-second'],
                ['host' => '132.232.70.99', 'port' => 6381, 'password' => 'foobared013310', 'database' => 0, 'alias' => 'master-third'],
                ['host' => '132.232.70.99', 'port' => 6382, 'password' => 'foobared013310', 'database' => 0, 'alias' => 'slave-first'],
                ['host' => '132.232.70.99', 'port' => 6383, 'password' => 'foobared013310', 'database' => 0, 'alias' => 'slave-second'],
                ['host' => '132.232.70.99', 'port' => 6384, 'password' => 'foobared013310', 'database' => 0, 'alias' => 'slave-third'],
            ],
            'options' => [
                'cluster' => 'redis'
            ],
        ],

```



### 遗忘特性

#### 集群中的具体节点
1. 对具体节点进行命令操作，一般命令就只是在当前节点操作，不会扩展到其他节点，如`keys *`，获取当前节点的key
2. 从节点在不升为主节点之前，只提供备份作用，可以通过`keys *`查看key，但是无法获取具体的键对应的值，除非使用`readonly`

## 安装redis
参考资料：<http://www.redis.net.cn/tutorial/3503.html>

A 说明：windows环境
下载地址：<https://github.com/dmajkic/redis/downloads>。(已有)
下载到的Redis支持32bit和64bit。根据自己实际情况选择，将64bit的内容cp到自定义盘符安装目录取名redis。 如 `C:\redis`

打开一个cmd窗口 使用cd命令切换目录到 `C:\redis` 运行 `redis-server.exe redis.windows.conf` 。
备注：可以在配置文件中
```
requirepass xxxx    #设置密码
port        xxxx    #设置端口
```
如果想方便的话，可以把redis的路径加到系统的环境变量里，这样就省得再输路径了，后面的那个redis.conf可以省略，如果省略，会启用默认的。输入之后，会显示如下界面：省略...

这时候另启一个cmd窗口，原来的不要关闭，不然就无法访问服务端了。

切换到redis目录下运行 `redis-cli.exe -h 127.0.0.1 -p 6379` 。
如果有设置密码，则输入：auth 密码       

设置键值对 `set myKey abc`

取出键值对 `get myKey`

B 说明：linux环境

本教程使用的版本为 2.8.17，下载并安装：
```
$ wget http://download.redis.io/releases/redis-2.8.17.tar.gz
$ tar xzf redis-2.8.17.tar.gz
$ cd redis-2.8.17
$ make && make install
```
make完后 `/usr/local/bin`目录下会出现编译后的redis服务程序`redis-server`,还有用于测试的客户端程序`redis-cli`
备注：好像`redis-server`和`redis-cli`都不在`/usr/local/bin`下面，而是在解压目录的`src/`目录下

错误：
```
make[1]: *** [persist-settings] Error 1
make[1]: Leaving directory `/usr/src/redis-2.8.17/src'
make: *** [all] Error 2
```
解决方法：根据错误提示，执行 `cd ./src && make all`

下面启动redis服务.
```
$ ./redis-server
```
注意这种方式启动redis 使用的是默认配置。也可以通过启动参数告诉redis使用指定配置文件使用下面命令启动。
```
$ ./redis-server redis.conf
```
redis.conf是一个默认的配置文件。我们可以根据需要使用自己的配置文件。
**备注：更改配置文件中 `daemonize` 选项为yes 开启守护进程**

启动redis服务进程后，就可以使用测试客户端程序redis-cli和redis服务交互了。 比如：
```
$ ./redis-cli
redis> set foo bar
OK
redis> get foo
"bar"
```

- linux的yum安装

```
yum list | grep redis;//查看
yum install -y redis;
service redis start;//启动服务端
redis-cli;//启动客户端
```

**设置密码**
yum方式安装的redis配置文件通常在`/etc/redis.conf`中，打开配置文件找到
```
#requirepass foobared  
```

去掉行前的注释，并修改密码为所需的密码,保存文件
```
[plain] view plain copy
requirepass myRedis  
```
重启redis
```
[plain] view plain copy
sudo service redis restart  
```
或者  
```
sudo service redis stop  
sudo redis-server /etc/redis.conf  
```
这个时候尝试登录redis，发现可以登上，但是执行具体命令是提示操作不允许
```
[plain] view plain copy
redis-cli -h 127.0.0.1 -p 6379  
redis 127.0.0.1:6379>  
redis 127.0.0.1:6379> keys *  
(error) ERR operation not permitted  
redis 127.0.0.1:6379> select 1  
(error) ERR operation not permitted  
redis 127.0.0.1:6379[1]>  
```
尝试用密码登录并执行具体的命令看到可以成功执行
```
[plain] view plain copy
redis-cli -h 127.0.0.1 -p 6379 -a myRedis  
redis 127.0.0.1:6379> keys *  
1) "myset"  
2) "mysortset"  
redis 127.0.0.1:6379> select 1  
OK  
redis 127.0.0.1:6379[1]> config get requirepass  
1) "requirepass"  
2) "myRedis"  
```
2.通过命令行进行配置
```
[plain] view plain copy
redis 127.0.0.1:6379[1]> config set requirepass my_redis    #暂时性操作
OK  
redis 127.0.0.1:6379[1]> config get requirepass  
1) "requirepass"  
2) "my_redis"  
```
无需重启redis
使用第一步中配置文件中配置的老密码登录redis，会发现原来的密码已不可用，操作被拒绝
```
[plain] view plain copy
redis-cli -h 127.0.0.1 -p 6379 -a myRedis  
redis 127.0.0.1:6379> config get requirepass  
(error) ERR operation not permitted  
```
使用修改后的密码登录redis，可以执行相应操作
```
[plain] view plain copy
redis-cli -h 127.0.0.1 -p 6379 -a my_redis  
redis 127.0.0.1:6379> config get requirepass  
1) "requirepass"  
2) "my_redis  
```
尝试重启一下redis，用新配置的密码登录redis执行操作，发现新的密码失效，redis重新使用了配置文件中的密码
```
[plain] view plain copy
sudo service redis restart  
Stopping redis-server:                                     [  OK  ]  
Starting redis-server:                                     [  OK  ]  
redis-cli -h 127.0.0.1 -p 6379 -a my_redis  
redis 127.0.0.1:6379> config get requirepass  
(error) ERR operation not permitted  
redis-cli -h 127.0.0.1 -p 6379 -a myRedis  
redis 127.0.0.1:6379> config get requirepass  
1) "requirepass"  
2) "myRedis"  
```
除了在登录时通过 `-a `参数制定密码外，还可以登录时不指定密码，而在执行操作前进行认证。
```
[plain] view plain copy
redis-cli -h 127.0.0.1 -p 6379  
redis 127.0.0.1:6379> config get requirepass  
(error) ERR operation not permitted  
redis 127.0.0.1:6379> auth myRedis  
OK  
redis 127.0.0.1:6379> config get requirepass  
1) "requirepass"  
2) "myRedis"  
```
3.master配置了密码，slave如何配置
若master配置了密码则slave也要配置相应的密码参数否则无法进行正常复制的。
slave中配置文件内找到如下行，移除注释，修改密码即可
```
[plain] view plain copy
#masterauth  mstpassword  
```


二 安装php的redis扩展

A 说明：windows环境下
a 下载php_redis.dll，下载地址：<http://windows.php.net/downloads/pecl/releases/redis/>
b 将php_redis.dll复制到E:\wamp\bin\php\php5.5.12\ext下
c 在php配置文件中增加 extension=php_redis.dll；(已有)
d 重启服务
备注：无论系统是32位还是64位，都用32位的php_redis.dll(巨坑)

B 说明：linux环境下
开始在 PHP 中使用 Redis 前， 我们需要确保已经安装了 redis 服务及 PHP redis 驱动，且你的机器上能正常使用 PHP。 接下来让我们安装 PHP redis 驱动：
下载地址为:<https://github.com/nicolasff/phpredis>。
或 <https://github.com/phpredis/phpredis>   (推荐)
或者 wget <http://pecl.php.net/get/redis-2.2.7.tgz>

PHP安装redis扩展
```
[jhj@localhost ~]$ tar xzf redis-2.2.7.tgz
[jhj@localhost ~]$ cd redis-2.2.7

/usr/local/php/bin/phpize              #php安装后的路径
 
./configure --with-php-config=/usr/local/php/bin/php-config
 
make && make install
```
完成显示：`extension_dir = "/usr/local/php/lib/php/extensions/no-debug-zts-20131226/"`

修改php.ini文件
```
vi /usr/local/php/lib/php.ini
```
增加如下内容:
`extension=redis.so` (注意这儿是so，不是io)
安装完成后重启php-fpm 或 `apache(/usr/local/apache2/bin/apachectl restart
)`。查看phpinfo信息，就能看到redis扩展。

参考资料：<http://www.redis.net.cn/tutorial/3526.html>



三 Redis 配置
参考资料：<http://www.redis.net.cn/tutorial/3504.html>
Redis 的配置文件位于 Redis 安装目录下，文件名为 redis.conf。
你可以通过 CONFIG 命令查看或设置配置项。

语法
Redis CONFIG 命令格式如下：
```
[root@localhost bin]# /usr/local/bin/redis-cli       备注：先执行客户端
redis 127.0.0.1:6379> CONFIG GET CONFIG_SETTING_NAME
```
实例
```
redis 127.0.0.1:6379> CONFIG GET loglevel
 
1) "loglevel"
2) "notice"
```
使用 * 号获取所有配置项：

实例
```
redis 127.0.0.1:6379> CONFIG GET *
 
  1) "dbfilename"
  2) "dump.rdb"
  3) "requirepass"
  4) ""
  5) "masterauth"
  6) ""
  7) "unixsocket"
  8) ""
  9) "logfile"
  ....
```
  编辑配置
你可以通过修改 `/root/redis-2.8.17/redis.conf` 文件或使用 `CONFIG set` 命令来修改配置。
注意：修改配置文件后，重启redis时，需要带着配置文件才行，不然配置不会生效；
`/usr/local/bin/redis-server /root/redis-2.8.17/redis.conf`
备注：`redis.conf`放在`/etc/redis.conf` 也可以，位置任意；

语法
`CONFIG SET 命令基本语法`：
```
redis 127.0.0.1:6379> CONFIG SET CONFIG_SETTING_NAME NEW_CONFIG_VALUE
```
实例
```
redis 127.0.0.1:6379> CONFIG SET loglevel "notice"
OK
redis 127.0.0.1:6379> CONFIG GET loglevel
 
1) "loglevel"
2) "notice"
```
参数说明
`/root/redis-2.8.17/redis.conf`(安装目录中) 配置项说明如下：

1. Redis默认不是以守护进程的方式运行，可以通过该配置项修改，使用yes启用守护进程
```
daemonize no
```
2. 当Redis以守护进程方式运行时，Redis默认会把pid写入`/var/run/redis.pid`文件，可以通过pidfile指定
```
pidfile /var/run/redis.pid
```
3. 指定Redis监听端口，默认端口为6379，作者在自己的一篇博文中解释了为什么选用6379作为默认端口，因为6379在手机按键上MERZ对应的号码，而MERZ取自意大利歌女Alessia Merz的名字
```
port 6379
```
4. 绑定的主机地址
```
bind 127.0.0.1
```
5.当 客户端闲置多长时间后关闭连接，如果指定为0，表示关闭该功能
```
timeout 300
```
6. 指定日志记录级别，Redis总共支持四个级别：debug、verbose、notice、warning，默认为verbose
```
loglevel verbose
```
7. 日志记录方式，默认为标准输出，如果配置Redis为守护进程方式运行，而这里又配置为日志记录方式为标准输出，则日志将会发送给`/dev/null`
```
#这个就是设置日志文件，
#logfile /var/log/redis.log    #日志路径

logfile stdout
```
8. 设置数据库的数量，默认数据库为0，可以使用`SELECT <dbid>`命令在连接上指定数据库id
```
databases 16
```
9. 指定在多长时间内，有多少次更新操作，就将数据同步到数据文件(注意是同步，不是转移)，可以多个条件配合
```
save <seconds> <changes>
```
Redis默认配置文件中提供了三个条件：
```
save 900 1

save 300 10

save 60 10000
```
分别表示900秒（15分钟）内有1个更改，300秒（5分钟）内有10个更改以及60秒内有10000个更改。

 

10. 指定存储至本地数据库时是否压缩数据，默认为yes，Redis采用LZF压缩，如果为了节省CPU时间，可以关闭该选项，但会导致数据库文件变的巨大
```
rdbcompression yes
```
11. 指定本地数据库文件名，默认值为dump.rdb
```
dbfilename dump.rdb
```
12. 指定本地数据库存放目录
```
dir ./
```
13. 设置当本机为slav服务时，设置master服务的IP地址及端口，在Redis启动时，它会自动从master进行数据同步
```
slaveof <masterip> <masterport>
```
14. 当master服务设置了密码保护时，slav服务连接master的密码
```
masterauth <master-password>
```
15. 设置Redis连接密码，如果配置了连接密码，客户端在连接Redis时需要通过`AUTH <password>`命令提供密码，默认关闭
    
`#requirepass 123456`   #即设置连接密码为123456，注意，设置连接密码后，客户端不要密码仍然可以连接，但是无法操作；设置密码后，客户端启动命令：`redis-cli -h 127.0.0.1 -p 6379 -a 123456`

`requirepass foobared`

16. 设置同一时间最大客户端连接数，默认无限制，Redis可以同时打开的客户端连接数为Redis进程可以打开的最大文件描述符数，如果设置 maxclients 0，表示不作限制。当客户端连接数到达限制时，Redis会关闭新的连接并向客户端返回max number of clients reached错误信息
```
maxclients 128
```
17. 指定Redis最大内存限制，Redis在启动时会把数据加载到内存中，达到最大内存后，Redis会先尝试清除已到期或即将到期的Key，当此方法处理 后，仍然到达最大内存设置，将无法再进行写入操作，但仍然可以进行读取操作。Redis新的vm机制，会把Key存放内存，Value会存放在swap区
```
maxmemory <bytes>
```

18. 指定是否在每次更新操作后进行日志记录，Redis在默认情况下是异步的把数据写入磁盘，如果不开启，可能会在断电时导致一段时间内的数据丢失。因为 redis本身同步数据文件是按上面save条件来同步的，所以有的数据会在一段时间内只存在于内存中。默认为no
```
appendonly no
```

19. 指定更新日志文件名，默认为`appendonly.aof`
```
appendfilename appendonly.aof
```
20. 指定更新日志条件，共有3个可选值：     no：表示等操作系统进行数据缓存同步到磁盘（快）     always：表示每次更新操作后手动调用fsync()将数据写到磁盘（慢，安全）     everysec：表示每秒同步一次（折衷，默认值）
```
appendfsync everysec
```
 

21. 指定是否启用虚拟内存机制，默认值为no，简单的介绍一下，VM机制将数据分页存放，由Redis将访问量较少的页即冷数据swap到磁盘上，访问多的页面由磁盘自动换出到内存中（在后面的文章我会仔细分析Redis的VM机制）
```
vm-enabled no
```
22. 虚拟内存文件路径，默认值为`/tmp/redis.swap`，不可多个Redis实例共享
```
vm-swap-file /tmp/redis.swap
```
23. 将所有大于`vm-max-memory`的数据存入虚拟内存,无论`vm-max-memory`设置多小,所有索引数据都是内存存储的(Redis的索引数据 就是keys),也就是说,当`vm-max-memory`设置为0的时候,其实是所有value都存在于磁盘。默认值为0
```
vm-max-memory 0
```
24. Redis swap文件分成了很多的page，一个对象可以保存在多个page上面，但一个page上不能被多个对象共享，`vm-page-size`是要根据存储的 数据大小来设定的，作者建议如果存储很多小对象，page大小最好设置为32或者64bytes；如果存储很大大对象，则可以使用更大的page，如果不 确定，就使用默认值
```
vm-page-size 32
```
25. 设置swap文件中的page数量，由于页表（一种表示页面空闲或使用的bitmap）是在放在内存中的，，在磁盘上每8个pages将消耗1byte的内存。
```
vm-pages 134217728
```
26. 设置访问swap文件的线程数,最好不要超过机器的核数,如果设置为0,那么所有对swap文件的操作都是串行的，可能会造成比较长时间的延迟。默认值为4
```
vm-max-threads 4
```
27. 设置在向客户端应答时，是否把较小的包合并为一个包发送，默认为开启
```
glueoutputbuf yes
```
28. 指定在超过一定的数量或者最大的元素超过某一临界值时，采用一种特殊的哈希算法
```
hash-max-zipmap-entries 64

hash-max-zipmap-value 512
```
29. 指定是否激活重置哈希，默认为开启（后面在介绍Redis的哈希算法时具体介绍）
```
activerehashing yes
```
30. 指定包含其它的配置文件，可以在同一主机上多个Redis实例之间使用同一份配置文件，而同时各个实例又拥有自己的特定配置文件
```
include /path/to/local.conf
```

四 Redis 数据备份与恢复

`Redis SAVE` 命令用于创建当前数据库的备份。(一般采用`bgsave`，php扩展也有该方法)

语法
`redis Save` 命令基本语法如下：
备注：这是客户端执行的命令
```
redis 127.0.0.1:6379> SAVE 
实例
redis 127.0.0.1:6379> SAVE 
OK
```
该命令将在 redis 安装目录中创建`dump.rdb`文件。

恢复数据

如果需要恢复数据，只需将备份文件 (dump.rdb) 移动到 redis 安装目录并启动服务即可。获取 redis 目录可以使用 CONFIG 命令，如下所示：
 
redis 127.0.0.1:6379> CONFIG GET dir
1) "dir"
2) "/usr/local/redis/bin"
 
以上命令 CONFIG GET dir 输出的 redis 安装目录为 /usr/local/redis/bin。

`Bgsave`
创建 redis 备份文件也可以使用命令 BGSAVE，该命令在后台执行。

实例
```
127.0.0.1:6379> BGSAVE
 
Background saving started
```

五 Redis 安全
我们可以通过 redis 的配置文件设置密码参数，这样客户端连接到 redis 服务就需要密码验证，这样可以让你的 redis 服务更安全。

实例
我们可以通过以下命令查看是否设置了密码验证：
```
127.0.0.1:6379> CONFIG get requirepass
1) "requirepass"
2) ""
```
默认情况下 requirepass 参数是空的，这就意味着你无需通过密码验证就可以连接到 redis 服务。

你可以通过以下命令来修改该参数：
```
127.0.0.1:6379> CONFIG set requirepass "w3cschool.cc"
OK
127.0.0.1:6379> CONFIG get requirepass
1) "requirepass"
2) "w3cschool.cc"
```
设置密码后，客户端连接 redis 服务就需要密码验证，否则无法执行命令。

语法
AUTH 命令基本语法格式如下：
```
127.0.0.1:6379> AUTH password
```
实例
```
127.0.0.1:6379> AUTH "w3cschool.cc"
OK
127.0.0.1:6379> SET mykey "Test value"
OK
127.0.0.1:6379> GET mykey
"Test value"
```
六 Redis 性能测试
Redis 性能测试是通过同时执行多个命令实现的。

语法
redis 性能测试的基本命令如下：
```
redis-benchmark [option] [option value]
```
实例
以下实例同时执行 10000 个请求来检测性能：
```
redis-benchmark -n 100000
 
PING_INLINE: 141043.72 requests per second
PING_BULK: 142857.14 requests per second
SET: 141442.72 requests per second
GET: 145348.83 requests per second
INCR: 137362.64 requests per second
LPUSH: 145348.83 requests per second
LPOP: 146198.83 requests per second
SADD: 146198.83 requests per second
SPOP: 149253.73 requests per second
LPUSH (needed to benchmark LRANGE): 148588.42 requests per second
LRANGE_100 (first 100 elements): 58411.21 requests per second
LRANGE_300 (first 300 elements): 21195.42 requests per second
LRANGE_500 (first 450 elements): 14539.11 requests per second
LRANGE_600 (first 600 elements): 10504.20 requests per second
MSET (10 keys): 93283.58 requests per second
```
redis 性能测试工具可选参数如下所示：

序号	选项	描述	默认值
```
1	-h	指定服务器主机名	127.0.0.1
2	-p	指定服务器端口	6379
3	-s	指定服务器 socket	
4	-c	指定并发连接数	50
5	-n	指定请求数	10000
6	-d	以字节的形式指定 SET/GET 值的数据大小	2
7	-k	1=keep alive 0=reconnect	1
8	-r	SET/GET/INCR 使用随机 key, SADD 使用随机值	
9	-P	通过管道传输 <numreq> 请求	1
10	-q	强制退出 redis。仅显示 query/sec 值	
11	--csv	以 CSV 格式输出	
12	-l	生成循环，永久执行测试	
13	-t	仅运行以逗号分隔的测试命令列表。	
14	-I	Idle 模式。仅打开 N 个 idle 连接并等待。	
```
实例
以下实例我们使用了多个参数来测试 redis 性能：
```
redis-benchmark -h 127.0.0.1 -p 6379 -t set,lpush -n 100000 -q
 
SET: 146198.83 requests per second
LPUSH: 145560.41 requests per second
```
以上实例中主机为 127.0.0.1，端口号为 6379，执行的命令为 set,lpush，请求数为 10000，通过 -q 参数让结果只显示每秒执行的请求数。

七 Redis 客户端连接
Redis 通过监听一个 TCP 端口或者 Unix socket 的方式来接收来自客户端的连接，当一个连接建立后，Redis 内部会进行以下一些操作：

首先，客户端 socket 会被设置为非阻塞模式，因为 Redis 在网络事件处理上采用的是非阻塞多路复用模型。
然后为这个 socket 设置 TCP_NODELAY 属性，禁用 Nagle 算法
然后创建一个可读的文件事件用于监听这个客户端 socket 的数据发送
最大连接数
在 Redis2.4 中，最大连接数是被直接硬编码在代码里面的，而在2.6版本中这个值变成可配置的。

maxclients 的默认值是 10000，你也可以在 redis.conf 中对这个值进行修改。
```
config get maxclients
 
1) "maxclients"
2) "10000"
```
实例
以下实例我们在服务启动时设置最大连接数为 100000：
```
redis-server --maxclients 100000
```
客户端命令(在客户端输入的命令)
S.N.	命令	描述
1	CLIENT LIST	返回连接到 redis 服务的客户端列表
2	CLIENT SETNAME	设置当前连接的名称
3	CLIENT GETNAME	获取通过 CLIENT SETNAME 命令设置的服务名称
4	CLIENT PAUSE	挂起客户端连接，指定挂起的时间以毫秒计
5	CLIENT KILL	关闭客户端连接

八 Redis 管道技术
Redis是一种基于客户端-服务端模型以及请求/响应协议的TCP服务。这意味着通常情况下一个请求会遵循以下步骤：

客户端向服务端发送一个查询请求，并监听Socket返回，通常是以阻塞模式，等待服务端响应。
服务端处理命令，并将结果返回给客户端。
Redis 管道技术
Redis 管道技术可以在服务端未响应时，客户端可以继续向服务端发送请求，并最终一次性读取所有服务端的响应。

优势：管道技术最显著的优势是提高了 redis 服务的性能

实例
查看 redis 管道，只需要启动 redis 实例并输入以下命令：
```
$(echo -en "PING\r\n SET w3ckey redis\r\nGET w3ckey\r\nINCR visitor\r\nINCR visitor\r\nINCR visitor\r\n"; sleep 10) | nc localhost 6379
 
+PONG
+OK
redis
:1
:2
:3
```
以上实例中我们通过使用 PING 命令查看redis服务是否可用， 之后我们们设置了 w3ckey 的值为 redis，然后我们获取 w3ckey 的值并使得 visitor 自增 3 次。

在返回的结果中我们可以看到这些命令一次性向 redis 服务提交，并最终一次性读取所有服务端的响应

九 Redis 分区
分区是分割数据到多个Redis实例的处理过程，因此每个实例只保存key的一个子集。

分区的优势
通过利用多台计算机内存的和值，允许我们构造更大的数据库。
通过多核和多台计算机，允许我们扩展计算能力；通过多台计算机和网络适配器，允许我们扩展网络带宽。
分区的不足
redis的一些特性在分区方面表现的不是很好：

涉及多个key的操作通常是不被支持的。举例来说，当两个set映射到不同的redis实例上时，你就不能对这两个set执行交集操作。
涉及多个key的redis事务不能使用。
当使用分区时，数据处理较为复杂，比如你需要处理多个rdb/aof文件，并且从多个实例和主机备份持久化文件。
增加或删除容量也比较复杂。redis集群大多数支持在运行时增加、删除节点的透明数据平衡的能力，但是类似于客户端分区、代理等其他系统则不支持这项特性。然而，一种叫做presharding的技术对此是有帮助的。
分区类型
Redis 有两种类型分区。 假设有4个Redis实例 R0，R1，R2，R3，和类似user:1，user:2这样的表示用户的多个key，对既定的key有多种不同方式来选择这个key存放在哪个实例中。也就是说，有不同的系统来映射某个key到某个Redis服务。

范围分区
最简单的分区方式是按范围分区，就是映射一定范围的对象到特定的Redis实例。

比如，ID从0到10000的用户会保存到实例R0，ID从10001到 20000的用户会保存到R1，以此类推。

这种方式是可行的，并且在实际中使用，不足就是要有一个区间范围到实例的映射表。这个表要被管理，同时还需要各 种对象的映射表，通常对Redis来说并非是好的方法。

哈希分区
另外一种分区方法是hash分区。这对任何key都适用，也无需是object_name:这种形式，像下面描述的一样简单：

用一个hash函数将key转换为一个数字，比如使用crc32 hash函数。对key foobar执行crc32(foobar)会输出类似93024922的整数。
对这个整数取模，将其转化为0-3之间的数字，就可以将这个整数映射到4个Redis实例中的一个了。93024922 % 4 = 2，就是说key foobar应该被存到R2实例中。注意：取模操作是取除的余数，通常在多种编程语言中用%操作符实现。

## Redis开发与运维

### 遗忘特性

#### 1. 复杂度较高的命令
说明：这些命令时间复杂度较高，如果命令操作的元素过多（如集合比较大，里面的元素比较多），容易阻塞redis
- smembers：可以尝试用sscan代替
- lrange：这个由于采用了quicklist，暂时待定
- hgetall ：可以尝试hscan
- zrange：可以尝试zscan
备注：直接采用pipeline，当然采用pipeline执行的命令，各个命令之间就不是原子性的咯

## 运用
```php
<?php
    //连接本地的 Redis 服务
   $redis = new Redis();

   //连接redis服务端，也可以pconnect()长连接
   $redis->connect('127.0.0.1', 6379);

   header('content-type:text/html;charset=utf-8');


   /********************Redis 连接命令**************************************/
   //验证密码
   $redis->auth('123456');

   //打印字符串 $redis->echo('redis echo!');
   
   //查看服务是否运行,客户端向 Redis 服务器发送一个 PING ，如果服务器运作正常的话，会返回一个 PONG 。
   echo "Server is running: "+ $redis->ping().'<br/>';

   //关闭当前连接
   //$redis->quit();
   
   //切换到指定的数据库，默认使用0号数据库
   //$redis->select(1);
   
   /********************Redis 配置**************************************/
   //var_dump($redis->config('set','requirepass','123456'));//设置密码
   echo "获取密码: ";
   var_dump($redis->config('get','requirepass'));

   //将更改的配置写入配置文件
   //$redis->config('rewrite','requirepass');

   /********************Redis keys 命令 **************************************/
   //示例数据(字符串命令)
   $redis->set("keys_one", "data_one");//测试删除数据
   $redis->set("keys_two", "data_two");
   $redis->set("keys_two", "data_two");

   //删除key数据
   $redis->del('keys_one');

   // 获取存储的数据并输出(字符串命令)
   echo '-Redis keys 命令--测试数据是否删除：'.$redis->get("keys_one").'<br/>';

   //序列化给定 key ，并返回被序列化的值。
   echo '-Redis keys 命令--测试数据序列化的值：'.$redis->dump("keys_two").'<br/>';

   //判断key 的数据是否存在，返回1表示存在，0表示不存在
   echo "-Redis keys 命令--判断数据是否存在：".$redis->exists('keys_two').'<br/>';

   //为数据设置过期时间，并判断是否成功
   echo "-Redis keys 命令--设置过期时间：".$redis->expire('keys_two',30).'<br/>';

   //为数据设置过期时间，并判断是否成功,与上面不同的是这是设置的时间是时间戳；
   echo "-Redis keys 命令--设置过期时间：".$redis->expireat('keys_two',1493840000).'<br/>';

   //为数据设置过期毫秒数
   echo "-Redis keys 命令--设置过期时间：".$redis->pExpire('keys_two',1500).'<br/>';

   //为数据设置过期毫秒时间戳
   echo "-Redis keys 命令--设置过期时间：".$redis->pExpireat('keys_two',1555555555005).'<br/>';

   //查找所有符合给定模式 pattern 的 key
   echo "-Redis keys 命令--查找符合给定模式：".'<br/>';
   var_dump($redis->keys('keys*'));

   //将当前数据库的 key 移动到给定的数据库 db 当中；redis默认使用数据库 0,条件：只有在当前数据库有该key，并且目标数据库没有该key时，才能成功
   //语法：MOVE KEY_NAME DESTINATION_DATABASE
   //echo "-Redis keys 命令--移动key到给定数据库：".$redis->move('keys_two',1).'<br/>';

   //移除 key 的过期时间，key 将持久保持。
   echo "-Redis keys 命令--将key 将持久保持：".$redis->persist('keys_two').'<br/>';

   //以毫秒为单位返回 key 的剩余的过期时间,当 key 不存在时，返回 -2 。 当 key 存在但没有设置剩余生存时间时，返回 -1 。 否则，以毫秒为单位，返回 key 的剩余生存时间。
   echo "-Redis keys 命令--返回 key 的剩余的过期时间：".$redis->Pttl('keys_two').'<br/>';

   //以秒为单位，返回给定 key 的剩余生存时间
   echo "-Redis keys 命令--返回 key 的剩余的过期时间：".$redis->ttl('keys_two').'<br/>';

   //从当前数据库中随机返回一个 key
   echo "-Redis keys 命令--从当前数据库中随机返回一个 key：".$redis->randomKey().'<br/>';

   //修改 key 的名称，备注：新名存在，则旧名数据覆盖新名，当然旧名就删除了
   //echo "-Redis keys 命令--修改 key 的名称 ：".$redis->rename('keys_two','new_keys').'<br/>';

   //修改 key 的名称，与上面不同的是，这个仅当新名不存在时，才成功
   echo "-Redis keys 命令--修改 key 的名称 ：".$redis->renamenx('keys_two','new_key').'<br/>';

   //返回 key 所储存的值的类型
   echo "-Redis keys 命令--返回 key 所储存的值的类型：".$redis->type('new_keys').'<br/>';



   /********************Redis 字符串 命令 **************************************/
   //设置 redis 字符串数据
   $redis->set("tutorial-name", "Redis tutorial");

   //如果 key 已经存在并且是一个字符串(也包括数字)， APPEND 命令将 value 追加到 key 原来的值的末尾。
   $redis->append('tutorial-name', 'world'); 

   // 获取存储的数据并输出
   echo '-string 命令--获取字符串数据：'.$redis->get("tutorial-name").'<br/>';

   

   //获取key就旧值，并为其设置新值；
   echo '-string 命令--获取旧值，设置新值：'.$redis->getSet('tutorial-name','new_tutorial').'<br/>';

   //获取key 所储存的字符串值，获取指定偏移量上的位(bit),对不存在的 key 或者不存在的 offset 进行 GETBIT， 返回 0
   echo '-string 命令--获取指定字符串上特定偏移量上的位：'.$redis->getBit('tutorial-name',2).'<br/>';

   //设置key 所储存的字符串值，设置指定偏移量上的位(bit)
   echo '-string 命令--设置指定字符串上特定偏移量上的位：'.$redis->setBit('tutorial-name',2,1).'<br/>';

   //获取所有(一个或多个)给定 key 的值；
   echo '-string 命令--获取所有(一个或多个)给定 key 的值：';
   var_dump($redis->mGet(['tutorial-name','new_keys']));

   //语法：SETEX KEY_NAME TIMEOUT VALUE ；功能：将值 value 关联到 key(即覆盖其内容)，并将 key 的过期时间设为 seconds (以秒为单位)。猜测：就是set()+expire()
   $redis->setex('tutorial-name',30,'new_setext');

   // 这个命令和 SETEX 命令相似，但它以毫秒为单位设置 key 的生存时间，而不是像 SETEX 命令那样，以秒为单位。
   $redis->pSetex('pSetex',30000,'new_psetex');

   //在指定的 key 不存在时，为 key 设置指定的值。
   $redis->setnx('setnx','new_setnax');

   //语法：SETRANGE KEY_NAME OFFSET VALUE；功能：用 value 参数覆写给定 key 所储存的字符串值，从偏移量 offset 开始。
   $redis->setRange('setnx',2,'___new_append');

   // 获取字符串数据的子字符串并输出
   echo '-string 命令--获取字符串数据的子字符串：'.$redis->getRange("setnx",0,6).'<br/>';

   //获取 key 所储存的字符串值的长度
   echo '-string 命令--获取字符串数据的长度：'.$redis->strlen('setnx').'<br/>';

   //同时设置一个或多个 key-value 对。
   $redis->mSet(['mset_one'=>'mset_data','mset_two'=>'mset_data']);


   //同时设置一个或多个 key-value 对，当且仅当所有给定 key 都不存在
   $redis->mSetnx(['msetnx_one'=>'mset_data','msetnx_two'=>'mset_data']);

   //获取所有(一个或多个)给定 key 的值；
   echo '-string 命令--获取所有(一个或多个)给定 key 的值：';
   var_dump($redis->mGet(['mset_one','mset_two']));

   //将 key 中储存的数字值增一,如果 key 不存在，那么 key 的值会先被初始化为 0 ，然后再执行 INCR 操作。如果值包含错误的类型，或字符串类型的值不能表示为数字，那么返回一个错误。本操作的值限制在 64 位(bit)有符号数字表示之内。
   $redis->set('incr',2);
   $redis->incr('incr');
   echo '-string 命令--将 key 中储存的数字值增一: '.$redis->get('incr').'<br/>';

   //将 key 中储存的数字加上指定的增量值
   $redis->incrBy('incr',10);
   echo '-string 命令--将 key 中储存的数字值增加给定值: '.$redis->get('incr').'<br/>';

   //为 key 中所储存的值加上指定的浮点数增量值。
   $redis->incrByFloat('incr',10.22);
   echo '-string 命令--将 key 中储存的数字值增加给定浮点值: '.$redis->get('incr').'<br/>';

   //将 key 中储存的数字值减一。如果 key 不存在，那么 key 的值会先被初始化为 0 ，然后再执行 DECR 操作。如果值包含错误的类型，或字符串类型的值不能表示为数字，那么返回一个错误。本操作的值限制在 64 位(bit)有符号数字表示之内。不能操作浮点数
   $redis->decr('decr');
   echo '-string 命令--将 key 中储存的数字值减一: '.$redis->get('decr').'<br/>';

   //将 key 所储存的值减去指定的减量值。其他说明与decr()相同
   $redis->decrBy('decr',10);
   echo '-string 命令--将 key 中储存的数字值减给定值: '.$redis->get('decr').'<br/>';

   
   
   /********************Redis hash 命令 **************************************/

   //同时将多个 field-value (域-值)对设置到哈希表 key 中。此命令会覆盖哈希表中已存在的字段。user:1 为键值，备注：特别适合存储用户对象信息
   //"user:1"只是简单字符串而已
   $redis->hMset("user:1",[
   							'username'=>'猜测：只能为一层数组',
   							'password'=>'w3cschool.cc',
   							'points'=>200,
   							'number'=>3
   						  ]
   	);

   //获取所有给定字段的值
   echo '-Redis hash 命令--获取所有给定字段的值:';
   var_dump($redis->hMget('user:1',['username','password']));

   //为哈希表中的字段赋值 。如果哈希表不存在，一个新的哈希表被创建并进行 HSET 操作。如果字段已经存在于哈希表中，旧值将被覆盖。
   $redis->hSet('user:1','username','这是新值');

   //只有在字段 field 不存在时，设置哈希表字段的值。
   $redis->hSetnx('user:1','username','这是新值二');

   //获取存储在哈希表中指定字段的值
   echo "-Redis hash 命令--获取存储在哈希表中指定字段的值:".$redis->hGet('user:1','username').'<br/>';

   //获取在哈希表中指定 key 的所有字段和值,结果为关联数组(即包括键)
   echo "-Redis hash 命令--获取在哈希表中指定 key 的所有字段和值:";
   var_dump($redis->hGetAll("user:1"));

   //获取哈希表中所有值，结果为索引数组
   echo "-Redis hash 命令-获取哈希表中所有值:";
   var_dump($redis->hVals("user:1"));

   //删除一个或多个哈希表字段
   $redis->hDel('user:1','points');

   //查看哈希表 key 中，指定的字段是否存在。存在返回1，不存在返回0；
   echo "-Redis hash 命令--查看哈希表 key 中，指定的字段是否存在:".$redis->hExists('user:1','points').'<br/>';

   //为哈希表中的字段值加上指定增量值。增量也可以为负数，相当于对指定字段进行减法操作。如果哈希表的 key 不存在，一个新的哈希表被创建并执行 HINCRBY 命令。如果指定的字段不存在，那么在执行命令前，字段的值被初始化为 0 。对一个储存字符串值的字段执行 HINCRBY 命令将造成一个错误。本操作的值被限制在 64 位(bit)有符号数字表示之内。返回计算后的数字；
   echo '-Redis hash 命令--为哈希表中的字段值加上指定增量值并获取结果:'.$redis->hIncrBy("user:1",'number',20).'<br/>';

   //为哈希表 key 中的指定字段的浮点数值加上增量 increment；如果指定的字段不存在，那么在执行命令前，字段的值被初始化为 0 。
   echo '-Redis hash 命令--为哈希表中的字段值加上指定浮点增量值并获取结果:'.$redis->hIncrByFloat("user:1",'number',20.55).'<br/>';

   //获取哈希表中的所有字段名。 当 key 不存在时，返回一个空列表。
   echo '-Redis hash 命令--获取哈希表中的所有字段名:';
   var_dump($redis->hKeys("user:1"));

   //获取哈希表中字段的数量,当 key 不存在时，返回 0 。
   echo '-Redis hash 命令--获取哈希表中的所有字段名:'.$redis->hLen("user:1").'<br/>';



   /********************Redis 列表命令 **************************************/

   //将一个或多个值插入到列表头部，列表不存在，则创建列表后，再加入元素
   $redis->lpush("push_test", "left_push_one");//从列表头加入
   //将一个或多个值插入到已存在的列表头部，列表 不存在时操作无效。
   $redis->lPushx("push_test", "left_push_two");
   //将一个或多个值插入到列表的尾部(最右边)
   $redis->rpush("push_test", "right_push_one");//从列表尾加入
   //将一个或多个值插入到已存在的列表尾部(最右边)。如果列表不存在，操作无效。
   $redis->rPushx("push_test", "right_push_two");
   //测试阻塞数据
   $redis->rpush("choke", "one");
   $redis->rpush("choke", "two");
   $redis->rpush("choke", "three");

   //从列表头部弹出元素,弹出的元素将从列表中删除
   echo '-Redis 列表命令--lPop演示：';
   var_dump($redis->lPop("push_test"));

   //从列表尾部弹出元素,弹出的元素将从列表中删除
   var_dump($redis->rPop("push_test"));

   /*
   非阻塞行为
   当 BLPOP 被调用时，如果给定 list 内至少有一个非空列表，那么弹出遇到的第一个非空列表的头元素，并和被弹出元素所属的列表的名字 list 一起，组成结果返回给调用者。
   当存在多个给定 list 时， BLPOP 按给定 list 参数排列的先后顺序，依次检查各个列表。 我们假设 list list1 不存在，而 list2 和 list3 都是非空列表。考虑以下的命令：
   BLPOP list1 list2 list3 0
   BLPOP 保证返回一个存在于 list2 里的元素（因为它是从 list1 --> list2 --> list3 这个顺序查起的第一个非空列表）。
   阻塞行为
   如果所有给定 list 都不存在或包含空列表，那么 BLPOP 命令将阻塞连接， 直到有另一个客户端对给定的这些 list 的任意一个执行 LPUSH 或 RPUSH 命令为止。
   一旦有新的数据出现在其中一个列表里，那么这个命令会解除阻塞状态，并且返回 list 和弹出的元素值。
   当 BLPOP 命令引起客户端阻塞并且设置了一个非零的超时参数 timeout 的时候， 若经过了指定的 timeout 仍没有出现一个针对某一特定 list 的 push 操作，则客户端会解除阻塞状态并且返回一个 nil 的多组合值(multi-bulk value)。

   timeout 参数表示的是一个指定阻塞的最大秒数的整型值。当 timeout 为 0 是表示阻塞时间无限制。
    */
   // 当给定多个 列表 参数时，按参数 列表 的先后顺序依次检查各个列表，弹出第一个非空列表的头元素。并和被弹出元素所属的列表的名字 list 一起，组成结果返回给调用者。弹出的元素将从列表中删除
   echo '-Redis 列表命令--blPop演示：';
   var_dump($redis->blPop("push_test",'choke',0));//最后一个参数是阻塞时间，0代表永久阻塞

   // 当给定多个 列表 参数时，按参数 列表 的先后顺序依次检查各个列表，弹出第一个非空列表的尾元素。并和被弹出元素所属的列表的名字 list 一起，组成结果返回给调用者。弹出的元素将从列表中删除
   echo '-Redis 列表命令--brPop演示：';
   var_dump($redis->brPop("push_test",'choke',0));

   //移除列表的最后一个元素，并将该元素添加到另一个列表并返回
   $redis->rpoplpush('choke','another_list');

   //从列表的末尾弹出一个值，将弹出的元素插入到另外一个列表中并返回它； 如果列表没有元素会阻塞列表直到等待超时或发现可弹出元素为止。
   $redis->brpoplpush('choke','another_list',30);

   //通过索引获取列表中的元素。你也可以使用负数下标，以 -1 表示列表的最后一个元素， -2 表示列表的倒数第二个元素，以此类推。
   echo '-Redis 列表命令--lindex演示：';
   echo $redis->lindex('choke',1);

   //通过索引设置列表元素的值
   $redis->lSet('choke',0,'zero'); 
   
   //在列表的元素前或者后插入元素。 当指定元素不存在于列表中时，不执行任何操作。 当列表不存在时，被视为空列表，不执行任何操作。 如果 key 不是列表类型，返回一个错误。
   //语法：LINSERT KEY_NAME BEFORE EXISTING_VALUE NEW_VALUE 
   $redis->lInsert('choke','before','two','before_data');//第三个参数是元素值；
   $redis->lInsert('choke','after','two','after_data');
   
   //返回列表的长度。 如果列表 key 不存在，则 key 被解释为一个空列表，返回 0 。 如果 key 不是列表类型，返回一个错误。
   echo '-Redis 列表命令--lLen演示：'.$redis->lLen('choke').'<br/>';
   
   //根据参数 COUNT 的值，移除列表中与参数 VALUE 相等的元素。被移除元素的数量。 列表不存在时返回 0 。
   //COUNT 的值可以是以下几种：
   //count > 0 : 从表头开始向表尾搜索，移除与 VALUE 相等的元素，数量为 COUNT 。
   //count < 0 : 从表尾开始向表头搜索，移除与 VALUE 相等的元素，数量为 COUNT 的绝对值。
   //count = 0 : 移除表中所有与 VALUE 相等的值。
   echo '-Redis 列表命令--lrem演示：'.$redis->lrem('choke','before',0).'<br/>';

   //对一个列表进行修剪(trim)，就是说，让列表只保留指定区间内的元素，不在指定区间之内的元素都将被删除。下标 0 表示列表的第一个元素，以 1 表示列表的第二个元素，以此类推。 你也可以使用负数下标，以 -1 表示列表的最后一个元素， -2 表示列表的倒数第二个元素，以此类推。命令执行成功时，返回 ok 。
   echo '-Redis 列表命令--ltrim演示：'.$redis->ltrim('choke',0,5).'<br/>';

   // 获取存储的数据并输出(并不会删除元素)，输出元素下标为0到2的所有元素
   $arList = $redis->lrange("push_test", 0 ,3);
   var_dump($arList);


   /********************Redis 集合命令 **************************************/

   //设置set数据，备注：set是可以自动排出重复的数据，当你需要存储一个列表数据，又不希望出现重复数据时，set是一个很好的选择，并且set提供了判断某个成员是否在一个set集合内的重要接口，这个也是list所不能提供的
   $redis->sadd('data_one','set_one','set_two');
   $redis->sadd('data_two','set_two','set_three');
   $redis->sadd('data_three','set_two','set_four');

   //获取集合的成员数量
   echo '-Redis 集合命令--scard演示：';
   var_dump($redis->scard('data_one'));

   //取出列表所有值
   var_dump($redis->sMembers('data_one'));

   //判断 member 元素是否是集合 key 的成员,如果成员元素是集合的成员，返回 1 。 如果成员元素不是集合的成员，或 key 不存在，返回 0 。
   echo '-Redis 集合命令--sismember演示：'.$redis->sismember('data_one','set_two').'<br/>';

   //取出给定列表中值的交集
   echo '-Redis 集合命令--sInter演示：';
   var_dump($redis->sInter('data_one','data_two','data_three'));

   //返回给定所有集合的交集并存储在 destination 中
   $redis->sInterStore('destination_inner','data_one','data_two');//集合desination被附上差集；

   //取出在第一个列表中出现，在后面列表中没有的数据
   echo '-Redis 集合命令--sDiff演示：';
   var_dump($redis->sDiff('data_one','data_two','data_three'));

   //sDiffStore 返回给定所有集合的差集并存储在 destination 中
   $redis->sDiffStore('destination_diff','data_one','data_two');//集合desination被附上差集；

   //取出所有列表中数据(不重复的)
   echo '-Redis 集合命令--sUnion演示：';
   var_dump($redis->sUnion('data_one','data_two','data_three'));

   // 所有给定集合的并集存储在 destination 集合中
   $redis->sUnionStore('destination_union','data_one','data_two');

   //从列表中删除值(可以删除多个)，感觉有点别扭，因为都知道要删除的值了。
   $redis->sRem('data_one','set_one','set_two');

   //移除并返回集合中的一个随机元素
   echo '-Redis 集合命令--sPop演示：';
   var_dump($redis->sPop('data_one'));

   //返回集合中一个或多个随机数,从 Redis 2.6 版本开始， Srandmember 命令接受可选的 count 参数：如果 count 为正数，且小于集合基数，那么命令返回一个包含 count 个元素的数组，数组中的元素各不相同。如果 count 大于等于集合基数，那么返回整个集合。如果 count 为负数，那么命令返回一个数组，数组中的元素可能会重复出现多次，而数组的长度为 count 的绝对值。该操作和 SPOP 相似，但 SPOP 将随机元素从集合中移除并返回，而 Srandmember 则仅仅返回随机元素，而不对集合进行任何改动。
   echo "-Redis 集合命令--sRandMember演示：";
   var_dump($redis->sRandMember('data_three',-2));//随机返回2个元素，并且元素可以重复

   //将 member 元素从 source 集合移动到 destination 集合，语法：SMOVE SOURCE DESTINATION MEMBER,如果成员元素被成功移除，返回 1 。 如果成员元素不是 source 集合的成员，并且没有任何操作对 destination 集合执行，那么返回 0 。
   $redis->sMove('data_one','data_two','set_one');

   //迭代集合中的元素，必要性：当 KEYS 命令被用于处理一个大的数据库时， 又或者 SMEMBERS 命令被用于处理一个大的集合键时， 它们可能会阻塞服务器达数秒之久
   //参考资料：http://www.redis.net.cn/order/3608.html
   //http://doc.redisfans.com/key/scan.html#scan
   echo "-Redis 集合命令--sscan演示：".'<br/>';
   //var_dump($redis->sscan('暂时不知道杂用'));

   
   /********************Redis 有序集合命令 **************************************/

   //Sort Set数据，与set类似，区别是set不是自动有序的，而sorted set可以通过用户额外提供一个优先级(score)的参数来为成员排序，并且是插入有序的，即自动排序。 
   //比如:全班同学成绩的SortedSets，value可以是同学的学号，而score就可以是其考试得分，这样数据插入集合的，就已经进行了天然的排序。 

   //向有序集合添加一个或多个成员，或者更新已存在成员的分数，如果某个成员已经是有序集的成员，那么更新这个成员的分数值，并通过重新插入这个成员元素，来保证该成员在正确的位置上。分数值可以是整数值或双精度浮点数。如果有序集合 key 不存在，则创建一个空的有序集并执行 ZADD 操作。当 key 存在但不是有序集类型时，返回一个错误。
   $redis->zAdd('sort_data',1,'刘作超');
   $redis->zAdd('sort_data',4,'章台名');
   $redis->zAdd('sort_data',3,'呵呵');
   $redis->zAdd('sort_data',2,'哈哈');//自动根据score排序，score就是代表第二步参数
   $redis->zAdd('sort_data',5,'嘿嘿');
   $redis->zAdd('sort_two',5,'嘿嘿');
   $redis->zAdd('sort_two',15,'哈哈');

   //获取有序集合的成员数
   echo "-Redis 集合命令--zCard演示：".$redis->zCard('sort_data').'<br/>';

   //计算在有序集合中指定区间分数的成员数
   echo "-Redis 集合命令--zCount演示：".$redis->zCount('sort_data',2,5).'<br/>';

   //有序集合中对指定成员的分数加上增量 increment，返回成员的新分数值，以字符串形式表示。
   echo "-Redis 集合命令--zIncrBy演示：".$redis->zIncrBy('sort_data',100,'呵呵').'<br/>';

   //计算给定的一个或多个有序集的交集并将结果集存储在新的有序集合 key 中，可以进行多种运算，包括求和(默认)，最小值和最大值。但是现在不知道杂用？
   echo "-Redis 集合命令--zinterstore 演示：".'<br/>';
   $redis->zinterstore('sort_destination',['sort_data','sort_two']);
   
   //Zlexcount 命令 - 在有序集合中计算指定字典区间内成员数量
   
   //Zremrangebylex 命令 - 移除有序集合中给定的字典区间的所有成员
   
   //通过字典区间返回有序集合的成员
   echo "-Redis 集合命令--zRangeByLex演示：";
   $redis->zAdd('abc',1,'a');
   $redis->zAdd('abc',2,'b');
   $redis->zAdd('abc',3,'c');
   var_dump($redis->zRangeByLex('abc','[a','[b'));

   //通过分数返回有序集合指定区间内的成员(从小到大),通过分数返回有序集合指定区间内的成员,获取sort set数据,min和max可以是-inf和+inf 表示负无穷和正无穷
   echo "-Redis 集合命令--zRangeByScore演示：";
   var_dump($redis->zRangeByScore('sort_data',1,5,['limit'=>[0,3]]));

   //返回有序集中指定分数区间内的成员，分数从高到低排序
   echo "-Redis 集合命令--zRevRangeByScore演示：";
   var_dump($redis->zRevRangeByScore('sort_data',1,5,['limit'=>[0,3]]));

   //返回有序集合中指定成员的索引
   echo "-Redis 集合命令--Zrank演示：".$redis->Zrank('sort_data','嘿嘿').'<br/>';

   //返回有序集合中指定成员的排名，有序集成员按分数值递减(从大到小)排序 
   echo "-Redis 集合命令--zRevRank演示：".$redis->zRevRank('sort_data','嘿嘿').'<br/>';

   //返回有序集中，成员的分数值
   echo "-Redis 集合命令--zScore演示：".$redis->zScore('sort_data','嘿嘿').'<br/>';

   //计算给定的一个或多个有序集的并集，并存储在新的 key 中
   $redis->zunionstore('sort_set_destination',['sort_data','abc']);

   //zscan 迭代有序集合中的元素（包括元素成员和元素分值）

   //删除数据，数据必须和加入数据完全吻合才能被删除；
   $redis->zRem('sort_data',1,'刘作超');

   //移除有序集合中给定的排名区间的所有成员
   $redis->zRemRangeByRank('abc',1,2);

   //移除有序集合中给定的分数score区间的所有成员
   $redis->zRemRangeByScore('sort_data',1,3);

   //获取sort set数据,按score值递增(从小到大)来排序,下标参数start和stop都以0为底，也就是说，以0表示有序集第一个成员，以1表示有序集第二个成员，以此类推。 你也可以使用负数下标，以-1表示最后一个成员，-2表示倒数第二个成员，以此类推。
   var_dump($redis->zRange('sort_data',0,100));
   var_dump($redis->zRange('sort_data',1,2,true));//显示score

   //获取sort set数据，按score值递增(从大到小)来排序,返回有序集中指定区间内的成员，通过索引，分数从高到底
   echo "-Redis 集合命令--zRevRange演示：";
   var_dump($redis->zRevRange('sort_data',0,3));
   var_dump($redis->zRevRange('sort_data',0,3,true));

   //获取有序集元素个数
   var_dump($redis->zCard('sort_data'));

   /********************Redis HyperLogLog **************************************/

   //基数:猜测--集合中不算重复值的个数，如：数据集 {1, 3, 5, 7, 5, 7, 8}， 那么这个数据集的基数集为 {1, 3, 5 ,7, 8}, 基数(不重复元素)为5。 基数估计就是在误差可接受的范围内，快速计算基数。
   //添加指定元素到 HyperLogLog 中。
   $redis->pfadd('pf_data',['redis','mysql']);
   $redis->pfadd('pf_data2',['dd','d2d']);
   //返回给定 HyperLogLog 的基数估算值。
   echo "-Redis 集合命令--pfcount演示：".$redis->pfcount('pf_data').'<br/>';

   //将多个 HyperLogLog 合并为一个 HyperLogLog
   $redis->pfmerge('pf_destination',['pf_data','pf_data2']);

   /********************Redis 发布订阅 **************************************/
   //参考：http://www.redis.net.cn/tutorial/3514.html

   /********************Redis 事务 **************************************/
   /**Redis 事务可以一次执行多个命令， 并且带有以下两个重要的保证：
	事务是一个单独的隔离操作：事务中的所有命令都会序列化、按顺序地执行。事务在执行的过程中，不会被其他客户端发送来的命令请求所打断。
	事务是一个原子操作：事务中的命令要么全部被执行，要么全部都不执行。
	一个事务从开始到执行会经历以下三个阶段：

	开始事务。
	命令入队。
	执行事务。
	*/
	//监视一个(或多个) key ，如果在事务执行之前这个(或这些) key 被其他命令所改动，那么事务将被打断。注意：这步在最前面
	$redis->watch(['sort_data','abc']);

	//标记一个事务块的开始。
	$redis->multi();

	//事务操作的动作
	$redis->append('ab','a');

	//取消 WATCH 命令对所有 key 的监视。
	//$redis->unwatch();

	// 执行所有事务块内的命令,执行成功，返回事务操作依次返回的内容，执行失败，则返回'nil'
	$redis->exec();

	// 执行所有事务块内的命令,总是返回ok
	//$redis->discard();
	
	/********************Redis 脚本 **************************************/
	//http://www.redis.net.cn/tutorial/3516.html
	
	/********************Redis 服务器命令**************************************/

	//异步执行一个 AOF（AppendOnly File） 文件重写操作,重写会创建一个当前 AOF 文件的体积优化版本。即使 Bgrewriteaof 执行失败，也不会有任何数据丢失，因为旧的 AOF 文件在 Bgrewriteaof 成功之前不会被修改。注意：从 Redis 2.4 开始， AOF 重写由 Redis 自行触发， BGREWRITEAOF 仅仅用于手动触发重写操作。
	//$redis->bgrewriteaof();
	
	//在后台异步保存当前数据库的数据到磁盘
	$redis->bgSave();
	
	//关闭客户端连接
	echo "-Redis 服务器命令--client演示";
	var_dump($redis->client('list'));
	$redis->client('kill','127.0.0.1:59039');//猜测:好像是list中的addr;

	//设置当前连接的名称
	$redis->client('Setname','first_link');

	//返回 CLIENT SETNAME 命令为连接设置的名字。 因为新创建的连接默认是没有名字的， 对于没有名字的连接， CLIENT GETNAME 返回空白回复。
	echo "-Redis 服务器命令--client演示获取连接名字：".$redis->client('Getname').'<br/>';

	//阻塞客户端命令一段时间（以毫秒计）
	//$redis->client('Pause',30000);//可用版本>= 2.9.50
	
	// 获取集群节点的映射数组，可用版本 >= 3.0.0
	// redis 127.0.0.1:6379> CLUSTER SLOTS 
	
	//获取 Redis 命令详情数组
	//redis 127.0.0.1:6379> COMMAND
	
	//获取 Redis 命令总数
	//redis 127.0.0.1:6379> COMMAND COUNT
	
	//获取给定命令的所有键
	//redis 127.0.0.1:6379> COMMAND GETKEYS
	
	//返回当前服务器时间,一个包含两个字符串的列表： 第一个字符串是当前时间(以 UNIX 时间戳格式表示)，而第二个字符串是当前这一秒钟已经逝去的微秒数。
	echo "-Redis 服务器命令--time演示：";
	var_dump($redis->time());

	//获取指定 Redis 命令描述的数组
	//命令：COMMAND INFO command-name [command-name ...] 
	
	//重置 INFO 命令中的某些统计数据
	//CONFIG RESETSTAT
	
	//获取 Redis 服务器的各种信息和统计数值
	//INFO [section] 
	
	//实时打印出 Redis 服务器接收到的命令，调试用
	//redis 127.0.0.1:6379> MONITOR 
	
	//返回主从实例所属的角色
	//redis 127.0.0.1:6379> ROLE

	//返回当前数据库的 key 的数量
	echo "-Redis 服务器命令--dbSize演示：".$redis->dbSize().'<br/>';
	
	//获取 key 的调试信息
	//命令：DEBUG OBJECT key
	echo "-Redis 服务器命令--debug演示：";
	var_dump($redis->debug('choke'));

	//让 Redis 服务崩溃
	//命令：DEBUG SEGFAULT
	
	//删除当前数据库的所有key
	$redis->flushDB();
	
	// 删除所有数据库的所有key
	$redis->flushAll();

	//返回最近一次 Redis 成功将数据保存到磁盘上的时间，以 UNIX 时间戳格式表示
	echo "-Redis 服务器命令--lastSave演示：".$redis->lastSave().'<br/>';

	// 执行一个同步保存操作，将当前 Redis 实例的所有数据快照(snapshot)以 RDB 文件的形式保存到硬盘。很少在生产环境直接使用SAVE 命令，因为它会阻塞所有的客户端的请求，可以使用BGSAVE 命令代替. 如果在BGSAVE命令的保存数据的子进程发生错误的时,用 SAVE命令保存最新的数据是最后的手段
	//$redis->save();
	
	// 异步保存数据到硬盘，并关闭服务器
	//redis 127.0.0.1:6379> SHUTDOWN [NOSAVE] [SAVE]
	
	//将当前服务器转变为指定服务器的从属服务器(slave server)
	/**
	 * 将当前服务器转变为指定服务器的从属服务器(slave server)。
	*
	*	如果当前服务器已经是某个主服务器(master server)的从属服务器，那么执行 SLAVEOF host port 将使当前服务器停止对旧主服务器的同步，丢弃旧数据集，转而开始对新主服务器进行同步。
	*
	*	另外，对一个从属服务器执行命令 SLAVEOF NO ONE 将使得这个从属服务器关闭复制功能，并从从属服务器转变回主服务器，原来同步所得的数据集不会被丢弃。
	*
	*	利用『 SLAVEOF NO ONE 不会丢弃同步所得数据集』这个特性，可以在主服务器失败的时候，将从属服务器用作新的主服务器，从而实现无间断运行。
	 */
	// 命令：redis 127.0.0.1:6379> SLAVEOF host port 
	//$redis->slaveof('127.0.0.1','6379');//未验证写法是否正确
	
	// 管理 redis 的慢日志，Redis 用来记录查询执行时间的日志系统。查询执行时间指的是不包括像客户端响应(talking)、发送回复等 IO 操作，而单单是执行一个查询命令所耗费的时间。另外，slow log 保存在内存里面，读写速度非常快，因此你可以放心地使用它，不必担心因为开启 slow log 而损害 Redis 的速度。
	//命令：redis 127.0.0.1:6379> SLOWLOG subcommand [argument]
	//实例：
	//redis 127.0.0.1:6379> SLOWLOG LEN
	//redis 127.0.0.1:6379> SLOWLOG RESET
	//redis 127.0.0.1:6379> SLOWLOG LEN
	
	//用于复制功能(replication)的内部命令
	//redis 127.0.0.1:6379> SYNC


   print_r(get_class_methods($redis));
```
