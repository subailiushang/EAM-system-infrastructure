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
![2.2 MySQL校验安装.PNG](D:\EAM系统基础架构\EAM-system-infrastructure\Pictures\2.2MySQL校验安装.PNG)
