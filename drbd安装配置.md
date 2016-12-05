# DRBD（分布式同步块设备）配置安装

[TOC]

## 准备工作
1. 两台服务器时间同步，关闭iptables和selinux，分别设置主机名为server1和server2
```language
yum install ntpdate -y
ntpdate cn.pool.ntp.org
/etc/init.d/iptables stop
sed -i "s#SELINUX=enforcing#SELINUX=disabled#g" /etc/selinux/conf
hostname server1 server2
vim /etc/sysconfig/network修改主机名
```
2. 升级系统内核模块
```language
yum install kernel kernel-devel
```
3. 内核安装完成后关机，分别添加一个2GB的硬盘sdb
```language
shutdown -h now
```
![drbd.png](C:\Users\zhangzihao\Pictures\drbd.png)

## 磁盘分区

1. 对添加的磁盘进行分区
| 分区 | |分区大小|作用 |
|--------|--------|
|  sdb1      ||1GB|     数据区块   |
|sdb2||1GB|元数据交换区块|
sdb1作为数据区块可以进行格式化
sdb2作为数据交换区，只需要进行分区，不需要格式化文件系统
```language
[root@server2 ~]# fdisk /dev/sdb
```
2. 格式化文件系统】
```languag
[root@server2 ~]# mkfs.ext4 /dev/sdb1
[root@server2 ~]# tune2fs -c -1 /dev/sdb1
```

## 安装DRBD软件
1. 安装drbd的yum源，下载elrepo包
```language
[root@server1 ~]# yum install elrepo-release-6-6.el6.elrepo.noarch.rpm
[root@server1 ~]# yum install drbd kmod-drbd84 -y #安装时间比较长
```
2. 加载drbd内核模块
```language
[root@server1 ~]# modprobe drbd
[root@server1 ~]# lsmod | grep drbd
drbd                  372759  0
libcrc32c               1246  1 drbd
```
3. 查看默认的配置样例
```language
[root@server1 ~]# cd /usr/share/doc/drbd84-utils-8.9.5/
[root@server1 drbd84-utils-8.9.5]# vim drbd.conf.example
[root@server1 drbd84-utils-8.9.5]# sed -n "83,115p" drbd.conf.example > 									/etc/drbd.d/test.res
[root@server1 drbd84-utils-8.9.5]# cd /etc/drbd.d/
```
4. 修改配置文件
```language
vim test.res
resource data {    							#data为资源名称，可以随意
		protocol C;	#实时同步协议C
        net {
                cram-hmac-alg "sha1";
                shared-secret "Gei6mahcui4Ai0Oh";
        }

        on server1 {
                        device /dev/drbd0;#虚拟磁盘设备
                        disk /dev/sdb1;#实际磁盘设备
                        meta-disk /dev/sdb2;#元数据
                address 192.168.10.7:7780;
        }
        on server2 {
                        device /dev/drbd0;
                        disk /dev/sdb1;
                        meta-disk /dev/sdb2;
                address 192.168.10.8:7780;
        }
}
```
5. 主从服务器配置文件一致
```language
[root@server1 drbd.d]# scp test.res 192.168.10.8:/etc/drbd.d/
```
6. 初始化元数据区
```language
[root@server1 drbd.d]# drbdadm create-md data
The server's response is:
you are the 12625th user to install this version
initializing activity log
NOT initializing bitmap
Writing meta data...
New drbd meta data block successfully created.
```
7. 启动两台服务器的drbd服务
```language
[root@server1 drbd.d]# /etc/init.d/drbd start
Starting DRBD resources: [
     create res: data
   prepare disk: data
    adjust disk: data
     adjust net: data
]
.......
```
8. 查看设备情况
```language
[root@server1 drbd.d]# cat /proc/drbd 
version: 8.4.7-1 (api:1/proto:86-101)
GIT-hash: 3a6a769340ef93b1ba2792c6461250790795db49 build by mockbuild@Build64R6, 2016-01-12 13:27:11
 0: cs:Connected ro:Secondary/Secondary ds:Inconsistent/Inconsistent C r-----
    ns:0 nr:0 dw:0 dr:0 al:8 bm:0 lo:0 pe:0 ua:0 ap:0 ep:1 wo:f oos:8004
```
9. 设置主节点
```language
[root@server1 drbd.d]# drbdadm -- --overwrite-data-of-peer primary all
```
10. 观察设备变化
```language
[root@server1 drbd.d]# cat /proc/drbd 
version: 8.4.7-1 (api:1/proto:86-101)
GIT-hash: 3a6a769340ef93b1ba2792c6461250790795db49 build by mockbuild@Build64R6, 2016-01-12 13:27:11
 0: cs:Connected ro:Primary/Secondary ds:UpToDate/UpToDate C r-----
    ns:8001 nr:0 dw:0 dr:8147 al:8 bm:0 lo:0 pe:0 ua:0 ap:0 ep:1 wo:f oos:0
```
11. 创建文件夹挂载drbd0设备进行数据同步
```language
mkdir /data
mount /dev/drbd0 /data
```
12. 注意事项
```language
在主节点存入的数据在备节点是无法观察到的，因为是基于内核的块设备进行同步的
想要观察数据最好stop备节点，在进行备节点/dev/drbd0的挂载即可
```

# 




![QQ二维码.JPG](C:\Users\zhangzihao\Pictures\Saved Pictures\QQ二维码.JPG)
