

OneProxy的主要功能有：
1. 垂直分库
2. 水平分表
3. Proxy集群
4. 读高可用
5. 读写分离（master不参与读）
6. 读写分离（master参与读）
7. 写高可用
8. 读写随机
  
重要概念
     Server Group
     在OneProxy中，一组主从复制的MySQL集群被称为Server Group。如图. A所示，有Server Group A和Server Group B。
      ![image](https://github.com/luoyan321/Mysql/blob/master/OneProxy/图/图片1.png) 
      
     在OneProxy中，垂直分库和水平分表的实现思路都是建立在Server Group的概念上。为了更好地说明，我们假设以下场景。
      
     A）Server Group A中有三张表table X, table Y, table Z，其中应用对table X操作非常频繁，占用大量I/O带宽，严重影响了应用对tableY, tableZ的操作效率。
      ![image](https://github.com/luoyan321/Mysql/blob/master/OneProxy/图/图片2.png) 
                                                                 
     解决方案1.0：把table X移到另一组数据库，即Server Group B中（如图C所示），然后通过修改OneProxy的配置来改变table X的路由规则，无须改动应用。
      ![image](https://github.com/luoyan321/Mysql/blob/master/OneProxy/图/图片3.png) 
 
                                                                 
     B）在使用了解决方案1.0后，系统的I/O压力得到缓解。由于后期业务越来越多，Server Group B的写入压力越来越大，响应时间变慢。
     解决方案2.0 : 把Server Group B中的table X水平拆分，将X_00, X_01留在Server Group B中，把X_02，X_03留在Server Group C中，如图D所示
       ![image](https://github.com/luoyan321/Mysql/blob/master/OneProxy/图/图片4.png) 
 
                                                                                
 
 
二、安装步骤
　　 1）下载
     wget http://www.onexsoft.com/software/oneproxy-rhel6-linux64-v6.2.0-ga.tar.gz  
     2）上传到目标主机的目录：/usr/local 
     3）cd /usr/local/
　　　 tar zxvf oneproxy-rhel6-linux64-v6.2.0-ga.tar.gz
     4）cd oneproxy/
     5）修改demo.sh
###############################
#/bin/bash
#
export ONEPROXY_HOME=/usr/local/oneproxy/   #根据自己环境配置，修改为oneproxy解压后的目录路径
# valgrind --leak-check=full \
${ONEPROXY_HOME}/bin/oneproxy --defaults-file=${ONEPROXY_HOME}/conf/proxy.conf
#####################################

 
 　 6）创建相关数据库，用户名和密码
        已经安装配置好MySQL
　　　mysql -uroot
　　　mysql> create database if not exists test character set utf8 ;
　　　mysql> grant insert, update, delete, select on test.* to test@'10.0.0.%' identified by 'test';　
     7）chmod +x ./demo.sh
　　　 ./demo.sh
     8）检查是否成功启动。
          ps aux | grep mysql-proxy | grep -v grep
          如有输出，则启动成功。
          若无输出，请检查运行日志/usr/local/oneproxy/log/oneproxy.log
　　　  注：目前OneProxy有个限制，如果/etc/hosts文件有IPv6地址，则无法启动，因此需要注释掉
          [root@oneproxy oneproxy]# vim /etc/hosts 
　　　　   127.0.0.1 localhost localhost.localdomain localhost4 localhost4.localdomain4
　　　   　#::1 localhost localhost.localdomain localhost6 localhost6.localdomain6
    9）通过mysql client连接OneProxy
        mysql -u test -h 10.0.0.9 -P3307 -p
　　  注:-h 后加上IP（最好不要是 localhost或者127.0.0.1，这种写法可能导致其使用unix socket连接而无法连接上）
 
应用场景与配置范例
     下面给出在以下几种场景下，如何正确的配置OneProxy
     1. 垂直分库
      ![image](https://github.com/luoyan321/Mysql/blob/master/OneProxy/图/图片5.png) 

 
OneProxy的配置文件conf/proxy.conf：
###############################
[oneproxy]
keepalive = 1
event-threads = 4
log-file = log/oneproxy.log           #指定日志文件路径
pid-file = log/oneproxy.pid           #指定PID文件路径
lck-file = log/oneproxy.lck           #指定LCK文件路径
mysql-version = 5.6.27                #版本
proxy-address = 0.0.0.0:3307                        #指定自身监听端口  
proxy-master-addresses.1 = 10.0.0.10:3306@A         #指定主服务器的IP地址  格式：IP地址:端口@oneproxy组
proxy-slave-addresses.1 = 10.0.0.11:3306@A          #指定从服务器的IP地址  格式：IP地址:端口@oneproxy组
proxy-master-addresses.2 = 10.0.0.12:3306@B         
proxy-slave-addresses.2 = 10.0.0.13:3306@B
proxy-user-list = test/1378F6CC3A8E8A43CA388193FBED5405982FBBD3@test         #用户列表   格式：用户名/密文密码@数据库名称
proxy-part-template = conf/template.txt
proxy-part-tables.1 = conf/part.txt                                          #指定分表分库的配置文件
proxy-part-tables.2 = conf/part2.txt                                         #指定分表分库的配置文件
proxy-charset = utf8_general_ci                                                #指定数据库字符集
proxy-group-policy.1  = A:master-only 
proxy-group-policy.2 = B:master-only
proxy-table-map.1=X:B
proxy-table-map.2=Y:A
proxy-table-map.3=Z:A
proxy-secure-client = 127.0.0.1
proxy-sequence.1 = default
#remote-address = 192.168.1.119:4041
#vip-address = 192.168.1.120/eth0:0
 
 
#####################################
注：具体参数含义参考附录
 
 2. 水平分表
   
![image](https://github.com/luoyan321/Mysql/blob/master/OneProxy/图/图片6.png) 
                                                                      
OneProxy的配置文件conf/proxy.conf：
###############################
[oneproxy]
keepalive = 1
event-threads = 4
log-file = log/oneproxy.log
pid-file = log/oneproxy.pid
lck-file = log/oneproxy.lck
mysql-version = 5.6.27
proxy-address = :3307
proxy-master-addresses.1 = 10.0.0.10:3306@A
proxy-slave-addresses.1 = 10.0.0.11:3306@A
proxy-master-addresses.2 = 10.0.0.12:3306@B
proxy-slave-addresses.2 = 10.0.0.13:3306@B
proxy-master-addresses.3 = 10.0.0.14:3306@C
proxy-slave-addresses.4 = 10.0.0.15:3306@C
proxy-user-list = test/1378F6CC3A8E8A43CA388193FBED5405982FBBD3@test
proxy-part-tables.1 = conf/part.txt

proxy-charset = gbk_chinese_ci
proxy-group-policy.1       = A:master-only
proxy-group-policy.2 = B:master-only
proxy-group-policy.3 = C:master-only
proxy-table-map.2=Y:A
proxy-table-map.3=Z:A
proxy-secure-client = 127.0.0.1
proxy-sequence.1 = default
#remote-address = 192.168.1.119:4041
#vip-address = 192.168.1.120/eth0:0
#####################################
 
OneProxy分库分表配置文件conf/part.txt
####################################
[
　　{
　　　　"table" : "X",
　　　　"pkey" : "id",
　　　　"type" : "char",
　　　　"method" : "crc32",
　　　　"partitions" :　　　　　
　　　　　　[
　　　　　　　　{ "suffix" : "_00", "group": "B" },
　　　　　　　　{ "suffix" : "_01", "group": "B" },
　　　　　　　　{ "suffix" : "_02", "group": "C" },
　　　　　　　　{ "suffix" : "_03", "group": "C"}
　　　　　　]

　　}
]
####################################
 
 

 
3. Proxy集群
 
4. 读高可用
     该方案是为了解决重要配置库的单点问题。在master不可用时，OneProxy会自动读取slave。
OneProxy的配置文件conf/proxy.conf：
###############################
[oneproxy]
keepalive = 1
event-threads = 4
log-file = log/oneproxy.log
pid-file = log/oneproxy.pid
lck-file = log/oneproxy.lck
mysql-version = 5.6.27
proxy-address = :3307
proxy-master-addresses.1 = 10.0.0.10:3306@A
proxy-slave-addresses.1 = 10.0.0.11:3306@A
proxy-user-list = test/1378F6CC3A8E8A43CA388193FBED5405982FBBD3@test

proxy-charset = gbk_chinese_ci
proxy-group-policy.1       = A:read_failover
proxy-secure-client = 127.0.0.1
proxy-sequence.1 = default
#remote-address = 192.168.1.119:4041
#vip-address = 192.168.1.120/eth0:0
 
#####################################
注：10.0.0.10为只读主库，10.0.0.11为只读丛库
 
 5. 读写分离（master不参与读）
  读写分离能有效的解决应用读负载较重且能忍受一定延迟的场景。此种模式下，读负载只能由slave承担，写与事务负载只能由master承担。
OneProxy的配置文件conf/proxy.conf：
###############################
[oneproxy]
keepalive = 1
event-threads = 4
log-file = log/oneproxy.log
pid-file = log/oneproxy.pid
lck-file = log/oneproxy.lck
mysql-version = 5.6.27
proxy-address = :3307
proxy-master-addresses.1 = 10.0.0.10:3306@A
proxy-slave-addresses.1 = 10.0.0.11:3306@A
proxy-user-list = test/1378F6CC3A8E8A43CA388193FBED5405982FBBD3@test
proxy-charset = gbk_chinese_ci
proxy-group-policy.1       = A:read_slave
proxy-secure-client = 127.0.0.1
proxy-sequence.1 = default
#remote-address = 192.168.1.119:4041
#vip-address = 192.168.1.120/eth0:0
 
#####################################
 
注：10.0.0.10为主库，10.0.0.11为丛库
 
 6. 读写分离（master参与读）
 这是另一种读写分离模式，所有类型的负载（读、写、事务）都有可能由master承担。
OneProxy的配置文件conf/proxy.conf：
###############################
[oneproxy]
keepalive = 1
event-threads = 4
log-file = log/oneproxy.log
pid-file = log/oneproxy.pid
lck-file = log/oneproxy.lck
mysql-version = 5.6.27
proxy-address = :3307
proxy-master-addresses.1 = 10.0.0.10:3306@A
proxy-slave-addresses.1 = 10.0.0.11:3306@A
proxy-user-list = test/1378F6CC3A8E8A43CA388193FBED5405982FBBD3@test

proxy-charset = gbk_chinese_ci
proxy-group-policy.1       = A:read_balance
proxy-secure-client = 127.0.0.1
proxy-sequence.1 = default
#remote-address = 192.168.1.119:4041
#vip-address = 192.168.1.120/eth0:0
 
#####################################
 
注：10.0.0.10为主库，10.0.0.11为丛库
 
 7. 写高可用
   这是专门针对XtraDB Cluster集群设计的一种模式。这种模式，只允许将一个节点作为写，而所有节点平均的承担所有的读负载。如图所示。

 ![image](https://github.com/luoyan321/Mysql/blob/master/OneProxy/图/图片7.png) 
                                                                     
以图.为例，若Node 1节点不可用，则任意选择另一台机器作为新的节点。如下图所示。
![image](https://github.com/luoyan321/Mysql/blob/master/OneProxy/图/图片8.png) 
                                                                      
 OneProxy在切换时，没有考虑数据的一致性，需要XtraDB Cluster本身来保证。其它类型的集群慎用。
OneProxy的配置文件conf/proxy.conf：
###############################
[oneproxy]
keepalive = 1
event-threads = 4
log-file = log/oneproxy.log
pid-file = log/oneproxy.pid
lck-file = log/oneproxy.lck
mysql-version = 5.6.27
proxy-address = :3307
proxy-master-addresses.1 = 10.0.0.10:3306@A
proxy-master-addresses.2 = 10.0.0.11:3306@A
proxy-master-addresses.3 = 10.0.0.12:3306@A
 
proxy-user-list = test/1378F6CC3A8E8A43CA388193FBED5405982FBBD3@test

proxy-charset = gbk_chinese_ci
proxy-group-policy.1       = A:write_other
proxy-secure-client = 127.0.0.1
proxy-sequence.1 = default
#remote-address = 192.168.1.119:4041
#vip-address = 192.168.1.120/eth0:0
 
#####################################
 
注：目前写入节点是由OneProxy自动选择的，无法手动指定。
 
     8. 读写随机
     这是专门针对XtraDB Cluster集群设计的一种模式。这种模式，所有的节点都平均的承担读写负载。
OneProxy的配置文件conf/proxy.conf：
###############################
[oneproxy]
keepalive = 1
event-threads = 4
log-file = log/oneproxy.log
pid-file = log/oneproxy.pid
lck-file = log/oneproxy.lck
mysql-version = 5.6.27
proxy-address = :3307
proxy-master-addresses.1 = 10.0.0.10:3306@A
proxy-master-addresses.2 = 10.0.0.11:3306@A
proxy-master-addresses.3 = 10.0.0.12:3306@A
 
proxy-user-list = test/1378F6CC3A8E8A43CA388193FBED5405982FBBD3@test

proxy-charset = gbk_chinese_ci
proxy-group-policy.1       = A:write_balance
proxy-secure-client = 127.0.0.1
proxy-sequence.1 = default
#remote-address = 192.168.1.119:4041
#vip-address = 192.168.1.120/eth0:0
 
#####################################
 

1、口令加密
　　此时可以启动oneproxy
　　cd /usr/local/oneproxy
      sh ./demo.sh
　　进入管理端口,然后键入passwd <string>。
mysql -uadmin -pOneProxy -P4041 --protocol=TCP
passwd test
　　输出为：
 　　1378F6CC3A8E8A43CA388193FBED5405982FBBD3
 
 
OneProxy管理
　　介绍  	
    OneProxy提供了目前两种管理功能。
第一，提供了查看配置信息（例如后端 DB情况等）与系统的动态变化信息（例如OneProxy与后端数据库建立的连接变化等）。
第二，提供了更改配置的功能。例如控制后端DB的上线或者下线。
   
　　登录 
　　OneProxy的默认管理端口是4041，用户名为admin，密码为OneProxy，可以使用如下命令登录：
mysql -u admin -h 10.128.130.237 -P4041 -pOneProxy
输入list help查看管理接口提供的命令
命令	描述	例子
LIST HELP	列出所有命令	list help
LIST BACKEND	列出所有后端数据库	list backend
LIST GROUP	列出所有的server group，具体含义请参考重要概念	list group
LIST POOL	列出OneProxy与每个后端数据库建立的连接池大小与连接池配置	list pool
LIST QUEUE	列出每个队列里到达的请求数量与已处理完成的请求	list queue
LIST THREADS	列出每个线程处理过的请求数	list threads
LIST TABLEMAP	列出table与server group的对应关系	list tablemap
LIST USERS 	列出用户	list users
LIST SQLSTATS 	列出执行过的SQL统计	list sqlstats [hash]
LIST SQLTEXT	列出执行过的SQL与hashcode的对应关系	list sqltext [hash]
SET  MASTER	指定某后端DB为写库	set master '192.168.1.119:3306'
SET  SLAVE	指定某后端DB为读库	set slave '192.168.1.119:3306'
SET  OFFLINE	下线指定的后端数据库	set offline '192.168.1.119:3306'
SET  ONLINE	上线指定的后端数据库	set online '192.168.1.119:3306'
SET  GPOLICY	指定server group的策略。 
预定义策略，0 代表由 Lua Script 来决定，默认为 Master Only；
1 代表 Read Failover；
2 代表Read/Write Split（Master 节点不参与读操作）；
3 代表双 Master 结构，或者是 XtraDB Cluster结构，即多主对等的方式；
4 代表 Read/WriteSplit（Master 节点共同参与读操作）;
5 代表读写随机。	set gpolicy default 1
SET  GMASTER	针对XtraDB Cluster，指定某个编号的数据库为写库	set gmaster default 1
SET  GACCESS	指定server group允许的sql类型。0：无任何限制，缺省值；
1：禁止 DDL 操作；
2：禁止不带 Where 条件的 Select、 Update 或 Delete。	set gaccess default 1
SET  POOLMIN	设置OneProxy与后端数据库连接池的最少连接数	set poolmin '192.168.1.119:3306' 5
SET  POOLMAX	设置OneProxy与后端数据库连接池的最大连接数，
实际的连接数可以超过这个数值，	set poolmax '192.168.1.119:3306' 300 
SET  SQLSTATS	打开、关闭或者清空SQL统计	set sqlstats {on|off|clear}
MAP	把某个表归属给某个server group	map my_test1_0 default
UNMAP	删除表与server group的映射关系	unmap my_test1_0
SUSPEND	让event process停止指定秒数	suspend 10
SHUTDOWN	关闭proxy	shutdown force

解释
A)　　--proxy-master-addresses
重命名了MySQL Proxy里的参数（proxy-backend-addresses），觉得这个名字更容易记,所有允许写操作的Master节点。格式如下：
格式：ip:port@groupname
其中“groupname”指的是“Server Group”的名字，如果不指定，则默认为“default”。
 
B)　　--proxy-slave-addresses
重命名了MySQL Proxy里的参数（proxy-read-only-backend-addresses），觉得这个名字更容易记，所有允许只读操作的Slave节点。格式如下：
格式：ip:port@groupname
其中“groupname”指的是“Server Group”的名字，如果不指定，则默认为“default”。
   
C)　　--proxy-user-list
OneProxy止前接管了客户端的登录验证，即客户端在登录验证时不再需要和后端的MySQL数据库通信了，这就要求OneProxy必须有一个完整的用户列表，用来验证客户端登录，可以通过这个参数来指定允许访问的用户名和口令列表，所有后端的数据库都必须存在同样用户名和口令的登录账号。
可以用“bin/mysqlhash”来生成加密后的口令，例如： 
[root@ANYSQLSRV1 oneproxy]# bin/mysqlhash test
A94A8FE5CCB19BA61C4C0873D391E987982FBBD3
然后在命令行参数里指定这个用户名和口令：
--proxy-user-list=test:A94A8FE5CCB19BA61C4C0873D391E987982FBBD3
如果有多个用户需要指定，多次指定此选项即可。也可以指定用户连接后端MySQL时的默认数据库，如果不指定，则连接到“proxy-database”指定的默认数据库。指定的格式如下：
--proxy-user-list=username:password@default_db
要求OneProxy后端所有的数据库都有同名用户、同名的数据库，及相应的访问权限，在OneProxy端并不支持改变一个会话的默认后端数据库，即传统的“USE”命令在OneProxy里有其他的含义。
 
D)　　--proxy-database
OneProxy基于现有连接池机制的考虑，并不支持切换数据库的功能，通过OneProxy连接的会话，只能对应到此参数指定的数据库里，“use”命令在OneProxy中也是被禁用的，原因是现有的代码里每一次获得连接池中的连接后，都会多发一个“use”命令，在高并发环境中会影响性能（后续版本将会对此作出改进），这是在使用OneProxy时需要注意的地方。
此选项默认值是“test”，这个库在默认安装时都会创建好的。
 
E)　　--proxy-charset
通过OneProxy来管理多个数据库时，要求所有的数据库字符集是一致的，同样是基于现有连接池机制的考虑，可以选的值在README文件里有，默认值是“utf8_general_ci”,为了防止连接池中不同的连接出现不同的设置，“set”命令在OneProxy里也是不生效的。
 
F)     --proxy-group-policy
使用OneProxy时可以透时地对下层的数据架构做改造，可以通过Lua脚本来实现的，出于对性能及便捷性（许多人不是很懂Lua）的考虑，将简单的Failover及读写分离的方案固化到OneProxy里，使用C语言来实现，即不需要写任何Lua代码也可以透明地使用Failover和读写分离方案了。指定格式：
--proxy-group-policy=<servergroup>:<policy_value>
对于复杂的分库分表，则还需要编写Lua脚本来实现，在“lua”子目录下有一个示例脚本“oneproxy.lua”就是用来做分库分表的。其中“servergroup”表示针对哪个“Server Group”进行设置。 
此选项可以设置的“policy_value”值有：
0
默认值，什么也不做，依赖于Lua脚本来实现。
1
Read Failover功能，对于读操作，首先从Master读取，如果Master不可用，则从Slave端读取。
2
Read/Write Split功能，对于读操作，首先从Slave读取，如果Slave端不可用，则从Master端读取。除非所有的Slave都不可用，否则Master不参与读操作。
3
针对XtraDB Cluster群集环境的Read/Write Split功能，从集群中固定地选择一台作为写入节点，其他的节点作为读节点；如果选中的写入节点不可用，则重新选一台作为写入节点，其他可用的节点继续提供读，这个策略提供了写入节点的自动漂移，在双Master节点上也可以设置成这种策略。
4
Read/Write Split功能，对于读操作，从Master和Slave中随机选一台进行查询操作，写入操作则从在Master上进行。
5
随机读写功能，对于读操作，从Master和Slave中随机选一台进行查询操作，写入操作则是随机选一台Master进行操作。 
后续会继续内置更多的策略进去，以及不断优化已有的策略，为大家透明地做MySQL架构改造而努力。
 
G)　　--proxy-security-level
安全级别，提升安全性，默认值为0，即没有任何设置。设成1禁止通过OneProxy来做DDL操作；设置为2则必须要有Where条件；设置为3只允许只读的操作。
 
H)　　--proxy-group-security
可针对某一个“Server Group”来指定安全级别，指定格式：
--proxy-group-policy=<servergroup>:<policy_value>
安全级别，提升安全性，默认值为0，即没有任何设置。设成1禁止通过OneProxy来做DDL操作；设置为2则必须要有Where条件；设置为3只允许只读的操作。
 
I)　　--event-threads
这个参数本身的意义没有变化，但内部的实现经过大幅优化，使得单个OneProxy实例可以支持20万以上的QPS转发。在已知的数据库Proxy中，OneProxy的转发能力一直是第一，此选项允许设置的最大值是48。
数十万的转发能力表示由OneProxy本身引起的时延增加基本可以忽略，这一点是OneProxy相对于MySQL Proxy而言的一个重大进步，解决了MySQL Proxy的并发处理能力。
通常这个值可以设为CPU Core数量的两倍，用8C或16C配备万兆网卡的机器来跑OneProxy可以达到最好的效果。
