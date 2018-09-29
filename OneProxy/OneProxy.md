

OneProxy的主要功能有：
1. 垂直分库
2. 水平分表
3. Proxy集群
4. 读高可用
5. 读写分离（master不参与读）
6. 读写分离（master参与读）
7. 写高可用
8. 读写随机

一、重要概念
   Server Group
   在OneProxy中，一组主从复制的MySQL集群被称为Server Group。如图. 所示，有Server Group A和Server Group B。
   ![image](https://github.com/luoyan321/Mysql/blob/master/OneProxy/图/图片1.png) 
   
   在OneProxy中，垂直分库和水平分表的实现思路都是建立在Server Group的概念上。为了更好地说明，我们假设以下场景。
  
     A）Server Group A中有三张表table X, table Y, table Z，其中应用对table X操作非常频繁，占用大量I/O带宽，严重影响了应用对tableY, tableZ的操作效率。
   ![image](https://github.com/luoyan321/Mysql/blob/master/OneProxy/图/图片2.png) 
   
    解决方案1.0：把table X移到另一组数据库，即Server Group B中（如图所示），然后通过修改OneProxy的配置来改变table X的路由规则，无须改动应用。
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
	export ONEPROXY_HOME=/usr/local/oneproxy/   #根据自己环境配置，修改为oneproxy解压后的目录路径
	#valgrind --leak-check=full \
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
 
三、应用场景与配置范例
下面给出在以下几种场景下，如何正确的配置OneProxy

  1. 垂直分库
  ![image](https://github.com/luoyan321/Mysql/blob/master/OneProxy/图/图片5.png) 

 
OneProxy的配置文件conf/proxy.conf：
###############################
[oneproxy]
keepalive = 1
event-threads = 4
log-file = log/oneproxy.log           				#指定日志文件路径
pid-file = log/oneproxy.pid          			    #指定PID文件路径
lck-file = log/oneproxy.lck          				#指定LCK文件路径
mysql-version = 5.6.27               				#版本
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
 
 

