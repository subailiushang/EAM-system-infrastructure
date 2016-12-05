#### EAM系统架构一：主从MySQL数据库安装

##### 1. 服务器时间同步：

```language
[root@MySQL_Master_Node01 ~]# ntpdate 202.193.158.37
[root@MySQL_Master_Node01 ~]# hwclock -w #系统硬件时间和系统时间进行同步
```

##### 2. RPM包方式安装MySQL服务

MySQL服务有多种安装方式：
* RPM包安装和官方YUM源安装方式最为简单
* 二进制文件方式安装（[适合多实例安装](http://www.jianshu.com/p/f7fbcb6909ef)）

本文主要采用RPM包的简单安装方式

###### 2.1 MySQL官方站点下载RPM包
[MySQL-5.7.16下载地址](http://101.44.1.3/files/2085000006113738/cdn.mysql.com//Downloads/MySQL-5.7/mysql-5.7.16-1.el6.x86_64.rpm-bundle.tar)

###### 2.2 Md5校验解压安装
```language
[root@MySQL_Master_Node01 ~]# wget http://101.44.1.3/files/2085000006113738/cdn.mysql.com//Downloads/MySQL-5.7/mysql-5.7.16-1.el6.x86_64.rpm-bundle.tar
[root@MySQL_Master_Node01 ~]# md5sum mysql-5.7.16-1.el6.x86_64.rpm-bundle.tar
[root@MySQL_Master_Node01 ~]# tar xvf mysql-5.7.16-1.el6.x86_64.rpm-bundle.tar
```
![2.2 MySQL校验安装.PNG](https://github.com/subailiushang/EAM-system-infrastructure/blob/master/Pictures/2.2MySQL%E6%A0%A1%E9%AA%8C%E5%AE%89%E8%A3%85.PNG)
```language
[root@MySQL_Master_Node01 ~]# yum install mysql-community-* -y
```
###### 2.3 查看my.cnf配置文件，了解mysql文件的目录
```language
[root@MySQL_Master_Node01 ~]# cat /etc/my.cnf
# For advice on how to change settings please see
# http://dev.mysql.com/doc/refman/5.7/en/server-configuration-defaults.html

[mysqld]
#
# Remove leading # and set to the amount of RAM for the most important data
# cache in MySQL. Start at 70% of total RAM for dedicated server, else 10%.
# innodb_buffer_pool_size = 128M
#
# Remove leading # to turn on a very important data integrity option: logging
# changes to the binary log between backups.
# log_bin
#
# Remove leading # to set options mainly useful for reporting servers.
# The server defaults are faster for transactions and fast SELECTs.
# Adjust sizes as needed, experiment to find the optimal values.
# join_buffer_size = 128M
# sort_buffer_size = 2M
# read_rnd_buffer_size = 2M
datadir=/var/lib/mysql #数据库存放目录
socket=/var/lib/mysql/mysql.sock #socket通信文件存放目录

# Disabling symbolic-links is recommended to prevent assorted security risks
symbolic-links=0

log-error=/var/log/mysqld.log #错误日志目录
pid-file=/var/run/mysqld/mysqld.pid #进程号目录
```

##### 3. 启动验证MySQL数据库

```language
[root@MySQL_Master_Node01 ~]# /etc/init.d/mysqld start
Starting mysqld:                                           [  OK  ]
[root@MySQL_Master_Node01 ~]# netstat -lntup | grep 3306
tcp        0      0 :::3306                     :::*                        LISTEN      2115/mysqld
[root@MySQL_Master_Node01 ~]# ps -ef | grep mysqld
root       1915      1  0 23:29 pts/0    00:00:00 /bin/sh /usr/bin/mysqld_safe --datadir=/var/lib/mysql --socket=/var/lib/mysql/mysql.sock --pid-file=/var/run/mysqld/mysqld.pid --basedir=/usr --user=mysql
mysql      2115   1915  0 23:29 pts/0    00:00:00 /usr/sbin/mysqld --basedir=/usr --datadir=/var/lib/mysql --plugin-dir=/usr/lib64/mysql/plugin --user=mysql --log-error=/var/log/mysqld.log --pid-file=/var/run/mysqld/mysqld.pid --socket=/var/lib/mysql/mysql.sock
root       2158   1362  0 23:32 pts/0    00:00:00 grep mysqld
```

##### 4. 设置数据库管理员密码

由于rpm包安装完mysql后会出现mysqladm无法修改密码的情况，可以使用--skip-grant-tables跳过密码验证后重新设置密码：
*此处应该注意在mysql 5.7版本中修改密码由原来的password字段修改为 authentication_string
具体不同版本可以通过 select * from mysql.user \G 进行查看
```language
[root@MySQL_Master_Node01 ~]# /etc/init.d/mysqld stop
[root@MySQL_Master_Node01 ~]# mysqld_safe --user=mysql --skip-grant-tables --skip-networking &
[root@MySQL_Master_Node01 ~]# mysql -u root mysql
mysql> use mysql;
mysql> update user set authentication_string=password('P@ssw0rd') where user='root' and host='localhost';
mysql> flush privileges;
```
