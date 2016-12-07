#### EAM系统架构三：MySQL数据库高可用方案

Heartbeat 项目是 Linux-HA 工程的一个组成部分，它实现了一个高可用集群系统。心跳服务和集群通信是高可用集群的两个关键组件，在 Heartbeat 项目里，由 heartbeat 模块实现了这两个功能。下面描述了 heartbeat 模块的可靠消息通信机制，并对其实现原理做了一些介绍。

heartbeat （Linux-HA）的工作原理：heartbeat最核心的包括两个部分，心跳监测部分和资源接管部分，心跳监测可以通过网络链路和串口进行，而且支持冗 余链路，它们之间相互发送报文来告诉对方自己当前的状态，如果在指定的时间内未收到对方发送的报文，那么就认为对方失效，这时需启动资源接管模块来接管运 行在对方主机上的资源或者服务。

##### 1. 在MySQL主从服务器上都安装Heartbeat套件

###### 1.1 首先安装epel-release源，采用yum安装方式
```language
[root@MySQL_Master_Node01 ~]# yum install heartbeat* -y
```

###### 1.2 养成查看 README.config 文件的习惯
通过README文件可知：
在 /usr/share/doc/heartbeat 下存在Heartbeat的必要配置文件
```language
[root@MySQL_Master_Node01 ~]# cat /etc/ha.d/README.config 
You need three configuration files to make heartbeat happy,
and they all go in this directory.

They are:
	ha.cf		Main configuration file
	haresources	Resource configuration file
	authkeys	Authentication information

These first two may be readable by everyone, but the authkeys file
must not be.

The good news is that sample versions of these files may be found in
the documentation directory (providing you installed the documentation).

If you installed heartbeat using rpm packages then
this command will show you where they are on your system:
		rpm -q heartbeat -d

If you installed heartbeat using Debian packages then
the documentation should be located in /usr/share/doc/heartbeat
```

##### 2. 配置Heartbeat

###### 2.1 复制 /usr/share/doc/heartbeat 下的 ha.cf , haresources , authkeys 文件到 /etc/ha.d/ 文件夹下
```language
[root@MySQL_Master_Node01 ~]# cd /usr/share/doc/heartbeat-3.0.4/
[root@MySQL_Master_Node01 heartbeat-3.0.4]# cp authkeys ha.cf haresources /etc/ha.d/
```

###### 2.1 编辑 authkeys	Authentication information 文件
编辑完成后修改权限为 600
```language
[root@MySQL_Master_Node01 ha.d]# vim authkeys

#
#       Authentication file.  Must be mode 600
#
#
#       Must have exactly one auth directive at the front.
#       auth    send authentication using this method-id
#
#       Then, list the method and key that go with that method-id
#
#       Available methods: crc sha1, md5.  Crc doesn't need/want a key.
#
#       You normally only have one authentication method-id listed in this file
#
#       Put more than one to make a smooth transition when changing auth
#       methods and/or keys.
#
#
#       sha1 is believed to be the "best", md5 next best.
#
#       crc adds no security, except from packet corruption.
#               Use only on physically secure networks.
#
auth 3
#1 crc
#2 sha1 HI!
3 md5 Hello!

[root@MySQL_Master_Node01 ha.d]# chmod 600 authkeys
```

###### 2.2 编辑 ha.cf		Main configuration file 文件
文件修改如下：
```
# There are lots of options in this file.  All you have to have is a set
# of nodes listed {"node ...} one of {serial, bcast, mcast, or ucast},
# and a value for "auto_failback".
# 这文件下面有很多的选 项，你必须设置的有节点列表集{node ...}，{serial,bcast,mcast,或ucast}中的一个，auto_failback的值
#
# ATTENTION: As the configuration file is read line by line,
#     THE ORDER OF DIRECTIVE MATTERS!
# 注意：配置文件是逐行读取的，并且选项的顺序是会影响最终结果的。
#
# In particular, make sure that the udpport, serial baud rate
# etc. are set before the heartbeat media are defined!
# debug and log file directives Go into effect when they
# are encountered.
# 特别注意，确保 udpport,serial baud rate等配置在心跳检测媒体（heartbeat media）前！他们将影响debug和log file指令。
# 也就是是在定义网卡，串口等心跳检测接口前先要定义端口号。
#
# All will be fine if you keep them ordered as in this example.
# 如果你保持他们在此例子中的顺序的话一切都不会有问 题。
#
#       Note on logging:
#       If all of debugfile, logfile and logfacility are not defined, 
#       logging is the same as use_logd yes. In other case, they are
#       respectively effective. if detering the logging to syslog,
#       logfacility must be "none".
# 记录日志方面的注意事项：
# 如果debugfile,logfile和logfacility都没 有定义，日志记录就相当于use_logd yes。否则，他们将分别生效。如果要阻止记录日志到syslog，那么logfacility必须设置为“none”
#
# File to write debug messages to
# 写入debug消息的文件
#debugfile /var/log/ha-debug
#
#
#  File to write other messages to
# 写 入其他消息的文件
#logfile /var/log/ha-log
#
#
# Facility to use for syslog()/logger 
# 用于syslog()/logger的设备 
logfacility local0
#
#
# A note on specifying "how long" times below...
# 在下面指定多长时间时应该注意
# The default time unit is seconds
# 缺省的时间单位是秒
#  10 means ten seconds
#  10 就代表10秒
#
# You can also specify them in milliseconds
#  1500ms means 1.5 seconds
# 你也可以指定他们以毫秒为单位
#  1500ms表示 1.5秒
#
# keepalive: how long between heartbeats?
# keepalive: 在heartbeat之间连接保持多久
#keepalive 2
#
# deadtime: how long-to-declare-host-dead?
# deadtime：
#  If you set this too low you will get the problematic
#  split-brain (or cluster partition) problem.
#  See the FAQ for how to use warntime to tune deadtime.
#  如果这个时间值设置得太低可能会导致出现很难判断的问题，如何使用warntime来调节 deadtime请查看FAQ。
#
#deadtime 30
#
# warntime: how long before issuing "late heartbeat" warning?
# See the FAQ for how to use warntime to tune deadtime.
# 
#warntime 10
#
#
# Very first dead time (initdead)
#
# On some machines/OSes, etc. the network takes a while to come up
# and start working right after you've been rebooted.  As a result
# we have a separate dead time for when things first come up.
# It should be at least twice the normal dead time.
# 在某些机器/操作系统等中，网络在机器重启后需要花一定的时间启动并正常工作。因此我们必须分开他们初次起来的dead time，这个值应该最少设置为两倍的正常dead time。
#
#initdead 120
#
#
# What UDP port to use for bcast/ucast communication?
# 用于bacst/ucast通讯的UDP 端口
#
#udpport 694
#
# Baud rate for serial ports...
# 串口的 波特率
#baud 19200
# 
# serial serialportname ...
# serial 串口名称
#serial /dev/ttyS0 # Linux
#serial /dev/cuaa0 # FreeBSD
#serial /dev/cuad0      # FreeBSD 6.x
#serial /dev/cua/a # Solaris
#
#
# What interfaces to broadcast heartbeats over?
# 广播heartbeats的接口
#
#bcast eth0  # Linux
#bcast eth1 eth2 # Linux
#bcast le0  # Solaris
#bcast le1 le2  # Solaris
#
# Set up a multicast heartbeat medium
# 设置一个多 播心跳介质
# mcast [dev] [mcast group] [port] [ttl] [loop]
# 
# [dev]  device to send/rcv heartbeats on 发送/接收heartbeats的设备
# [mcast group] multicast group to join (class D multicast address 224.0.0.0 - 239.255.255.255) 加入到的多播组（D类多播地址224.0.0.0 - 239.255.255.255）
# [port]  udp port to sendto/rcvfrom udp(set this value to the same value as "udpport" above) 端口用于发送/接收udp（设置这个值跟上面的udpport为相同值）
# [ttl]  the ttl value for outbound heartbeats. this effects how far the multicast packet will propagate.  (0-255) Must be greater than zero.
#   外流的 heartbeats的ttl值。这个影响多播包能传播多远。（0-255）必须要大于0 。
# [loop]  toggles loopback for outbound multicast heartbeats.if enabled, an outbound packet will be looped back and received by the interface it was sent #   on. (0 or 1) Set this value to zero.
#   为多播heartbeat开关loopback。如 果enabled，一个外流的包将被回环到原处并由发送它的接口接收。（0或者1）设置这个值为0。
#
#mcast eth0 225.0.0.1 694 1 0
#
# Set up a unicast / udp heartbeat medium
# 配 置一个unicast / udp heartbeat 介质
# ucast [dev] [peer-ip-addr]
#
# [dev]  device to send/rcv heartbeats on 用于发送/接收heartbeat的设备
# [peer-ip-addr] IP address of peer to send packets to 包被发送到的对等的IP地址
#
#ucast eth0 192.168.1.2
#
#
# About boolean values...
# 关于boolean值
# Any of the following case-insensitive values will work for true:
# 下面的非大 小写敏感的值将认为是true：
#  true, on, yes, y, 1
# Any of the following case-insensitive values will work for false:
# 下面的非大小写敏感的值将认为是false：
#  false, off, no, n, 0
#
#
#
# auto_failback:  determines whether a resource will
# automatically fail back to its "primary" node, or remain
# on whatever node is serving it until that node fails, or
# an administrator intervenes.
# auto_failback:  决定一个resource是否自动恢复到它的primary节点，或者不管什么节点，都继续运行在上面直到节点出现故障或管# 理员进行干预。
#
#
# The possible values for auto_failback are:
# auto_failback 的可能值有：
#  on - enable automatic failbacks
#  on   - 允许自动failbacks
#  off - disable automatic failbacks
#  off - 禁止自动failbacks
#  legacy - enable automatic failbacks in systems where all nodes do not yet support the auto_failback option.
#  legacy - 在所有节点都还不支持auto_failback的选项中允许自动failbacks
# auto_failback "on" and "off" are backwards compatible with the old "nice_failback on" setting.
# auto_failback "on"和"off"向后兼容旧的"nice_failback on"设置。
#
# See the FAQ for information on how to convert from "legacy" to "on" without a flash cut.
#  (i.e., using a "rolling upgrade" process)
# 查看FAQ获取如何从"legacy"转为到"on"并不会闪断的 信息。
#
#
# The default value for auto_failback is "legacy", which
# will issue a warning at startup.  So, make sure you put
# an auto_failback directive in your ha.cf file.
# (note: auto_failback can be any boolean or "legacy")
# 缺省的auto_failback值是“legacy”，它在启动的时候会 发送一个警告。因此，确保你在ha.cf文件中配置了auto_failback指令。
#
auto_failback on
#
#
#       Basic STONITH support
#       Using this directive assumes that there is one stonith 
#       device in the cluster.  Parameters to this device are 
#       read from a configuration file. The format of this line is:
# 基本上STONITH支持
# 使用这个指令假设有一个stonith设备在集群中。这个设备的参数 从一个配置文件中读取，这行的格式是：
#
#         stonith 
#
#       NOTE: it is up to you to maintain this file on each node in the
#       cluster!
# 注意：在集群中的每个节点上的这个文 件都靠你去维护。
#
#stonith baytech /etc/ha.d/conf/stonith.baytech
#
#       STONITH support
#       You can configure multiple stonith devices using this directive.
# 你可以使用这个指令配置多个stonith设备：
#       The format of the line is:
# 这行的格式是：
#         stonith_host 
#
#         is the machine the stonith device is attached to or * to mean it is accessible from any host.
#   表示stonith设备联结到的机器或者用*来表示从任何主机都可以访问。
#         is the type of stonith device (a list of supported drives is in /usr/lib/stonith.)
#   是stonith设备的类型（支持的设备的列表在/usr/lib/stonith中）
#         are driver specific parameters.  To see the format for a particular device, run:
#   是驱动指定的参数，要查看特定设备的格式，运行：
#           stonith -l -t 
#
#
# Note that if you put your stonith device access information in
# here, and you make this file publically readable, you're asking
# for a denial of service attack ;-)
# 需要注意如果你将你的stonith设备的访问信息放在这里，并且你让这个文件开放读权限，那么你是在召唤一个DoS攻 击。
#
# To get a list of supported stonith devices, run
# 要得到支持的 stonith设备的列表，运行
#  stonith -L
#
# For detailed information on which stonith devices are supported
# and their detailed configuration options, run this command:
# 要哪个stonith设备是支持的详细信息和它们详细的 配置选项，运行这个命令：
#  stonith -h
#
#stonith_host *     baytech 10.0.0.3 mylogin mysecretpassword
#stonith_host ken3  rps10 /dev/ttyS1 kathy 0 
#stonith_host kathy rps10 /dev/ttyS1 ken3 0 
#
# Watchdog is the watchdog timer.  If our own heart doesn't beat for
# a minute, then our machine will reboot.
# Watchdog是一个watchdog计时器，如果我们的心 超过一分钟不跳，我们的机器将会reboot。
#
# NOTE: If you are using the software watchdog, you very likely
# wish to load the module with the parameter "nowayout=0" or
# compile it without CONFIG_WATCHDOG_NOWAYOUT set. Otherwise even
# an orderly shutdown of heartbeat will trigger a reboot, which is
# very likely NOT what you want.
# 注意：如果你使用软件watchdog，你很可能希望用参数“nowayout=0”来加载这个模块或编译它的时候去掉
# CONFIG_WATCHDOG_NOWAYOUT 设置。否则，即使一个有序的关闭heartbeat也会触发重启，这很可能不是你想要的。
#
#watchdog /dev/watchdog
#       
# Tell what machines are in the cluster
# 说 明说明机器在这个集群里面
# node nodename ... -- must match uname -n
# node nodename ... --必须要匹配uname -n
#node ken3
#node kathy
#
# Less common options...
# 非常用的选项
# Treats 10.10.10.254 as a psuedo-cluster-member
# Used together with ipfail below...
# note: don't use a cluster node as ping node 
# 将10.10.10.254看成一个伪集群成员，与下面的 ipfail一起使用。
# 注意：不要使用一个集群节点作为ping节点
# 
#ping 10.10.10.254
#
# Treats 10.10.10.254 and 10.10.10.253 as a psuedo-cluster-member
#       called group1. If either 10.10.10.254 or 10.10.10.253 are up
#       then group1 is up
# Used together with ipfail below...
# 将 10.10.10.254和10.10.10.254看成一个叫group1的伪集群成员。如果10.10.10.254或10.10.10.253是 up的，那么group1为up
# 与下面的ipfail一起使用。
#
#ping_group group1 10.10.10.254 10.10.10.253
#
# HBA ping derective for Fiber Channel
# Treats fc-card-name as psudo-cluster-member
# used with ipfail below ...
# 用 于Fiber Channel的HBA ping指令，将fc-card-name看成是伪集群成员，与下面的ipfail一起使用。
#
# You can obtain HBAAPI from http://hbaapi.sourceforge.net.  You need 
# to get the library specific to your HBA directly from the vender
# To install HBAAPI stuff, all You need to do is to compile the common
# part you obtained from the sourceforge. This will produce libHBAAPI.so 
# which you need to copy to /usr/lib. You need also copy hbaapi.h to 
# /usr/include.
# 你可以从http://hbaapi.sourceforge.net获 取HBAAPI，你需要从vender获得用于你的HBA指令的特定的库来安装HBAAPI。
# 你所需要做的是编译你从sourceforge 获得的通用部分，它会生成libHBAAPI.so，然后你要将它拷贝到/usr/lib目录。同时
# 你也要吧hbaapi.h拷贝到/usr /include 。
# 
# The fc-card-name is the name obtained from the hbaapitest program 
# that is part of the hbaapi package. Running hbaapitest will produce
# a verbose output. One of the first line is similar to:
#  Apapter number 0 is named: qlogic-qla2200-0
# Here fc-card-name is qlogic-qla2200-0.
# fc-card-name是从hbaapitest程序获取的名字，它 是hbaapi包的一部分。运行hbaapitest将生成一个冗长的输出，其中第一行类似：
#   Apapter number 0 is named: qlogic-qla2200-0
# 在这里fc-card-name是qlogic-qla2200-0
#
#hbaping fc-card-name
#
#
# Processes started and stopped with heartbeat.  Restarted unless
#  they exit with rc=100
# 与heartbeat 一起启动和停止的进程。重启，除非它们的以rc=100退出。
#
#respawn userid /path/name/to/run
#respawn hacluster /usr/lib/heartbeat/ipfail
#
# Access control for client api
#        default is no access
# 用于客户端api的访问控制，缺省为不可访问。
#
#apiauth client-name gid=gidlist uid=uidlist
#apiauth ipfail gid=haclient uid=hacluster
###########################
#
# Unusual options.
# 非常选项
###########################
#
# hopfudge maximum hop count minus number of nodes in config  
#hopfudge 1
#
# deadping - dead time for ping nodes 上面设置的用来ping的节点的死亡时间
#deadping 30
#
# hbgenmethod - Heartbeat generation number creation method，Normally these are stored on disk and incremented as needed.
# hbgenmethod - Heartbeat产生数字的生产方法。通常执行存储在磁盘上并在需要时进行增量。
#
#hbgenmethod time
#
# realtime - enable/disable realtime execution (high priority, etc.) defaults to on
# realtime - 允许/禁止实时执行（高优先级）缺省为on
#realtime off
#
# debug - set debug level .defaults to zero
# debug - 设置debug等级，缺省为0
#debug 1
#
# API Authentication - replaces the fifo-permissions-based system of the past
# APT认证 - 代替以前的fifo-permission-base系统
#
# You can put a uid list and/or a gid list.If you put both, then a process is authorized if it qualifies under either the uid list, or under the gid list.
# 可以放上一个uid列表和/或gid列表。如果两个都放，那么符合uid列表或gid列表中的进程都将通过验证
#
#
# The groupname "default" has special meaning.  If it is specified, then
# this will be used for authorizing groupless clients, and any client groups
# not otherwise specified.
# 组名“default”有特定的意思。如果它被指定，那么它将用于验证无组的客户端和任何没有另 外指定的客户组
#
# There is a subtle exception to this.  "default" will never be used in the 
# following cases (actual default auth directives noted in brackets)
# 这是一个复杂的表达式，“default”将从不用于下面的情况（现实中缺省的 验证指令记录在括号中）
#    ipfail  (uid=HA_CCMUSER)
#    ccm    (uid=HA_CCMUSER)
#    ping  (gid=HA_APIGROUP)
#    cl_status (gid=HA_APIGROUP)
#
# This is done to avoid creating a gaping security hole and matches the most likely desired configuration.
# 它 避免生成一个安全漏洞缺口并匹配到了可能很多人最渴望的配置。
#
#apiauth ipfail uid=hacluster
#apiauth ccm uid=hacluster
#apiauth cms uid=hacluster
#apiauth ping gid=haclient uid=alanr,root
#apiauth default gid=haclient
#  message format in the wire, it can be classic or netstring, 
# default: classic
# 网线中的信息格式，可以是classic或netstring
#
#msgfmt  classic/netstring
#
# Do we use logging daemon?
# If logging daemon is used, logfile/debugfile/logfacility in this file
# are not meaningful any longer. You should check the config file for logging
# daemon (the default is /etc/logd.cf)
# more infomartion can be fould in http://www.linux-ha.org/ha_2ecf_2fUseLogdDirective
# Setting use_logd to "yes" is recommended
# 我们是否使用记录监控？
# 如果使用了记录监控，此文件里面的 logfile/debugfile/logfacility将不再有意义。你应该检查在配置文件中是否有记录监控（缺省为/etc/logd.cf）
# 更 多的信息可以在http://www.linux-ha.org/ha_2ecf_2fUseLogdDirective中 找到。推荐配置use_logd为yes。
# 
# use_logd yes/no
#
# the interval we  reconnect to logging daemon if the previous connection failed
# default: 60 seconds
# 如果前一个连接失败了，我们再次连接到记录监控器的间隔。
#conn_logd_time 60
#
#
# Configure compression module
# It could be zlib or bz2, depending on whether u have the corresponding 
# library in the system.
# 配置压缩模块
# 它可 以为zlib或bz2，基于我们的系统中是否有相应的库。
#
#compression bz2
#
# Confiugre compression threshold
# This value determines the threshold to compress a message,
# e.g. if the threshold is 1, then any message with size greater than 1 KB
# will be compressed, the default is 2 (KB)
# 配置压缩的限度
# 这个值决定压缩一个信息的限度，例如：如果限度为1，那么任何大于1KB的消息都会被压缩，缺省为 2（KB）
#compression_threshold 2
```
实际配置如下：
```language
[root@MySQL_Master_Node01 ha.d]# grep -v "#" ha.cf 
debugfile /var/log/ha-debug  #debug日志模式
logfile	/var/log/ha-log
logfacility	local0
keepalive 2
deadtime 30
warntime 10
initdead 120
udpport	694
ucast eth0 192.168.42.131 #采用单播的方式传递给slave服务器
auto_failback on
node	MySQL_Master_Node01
node	MySQL_Slaver_Node01

ping 192.168.42.2 #一般设置为网关地址
```

###### 2.3 编辑资源配置文件 haresources	Resource configuration file 
添加主节点 VIP地址 服务
此处的服务脚本必须满足 /etc/init.d/ 中服务脚本的要求
```language
MySQL_Master_Node01 192.168.42.253/24/eth0:1 mysqld
```

##### 3. 启动服务

主节点测试：

* 关闭主备节点的mysql服务
* 启动主节点的 heartbeat服务
* 稍等片刻，查看 vip地址和mysql服务是否启动
* 观察 /var/log/ha-debug.log日志文件

```language
[root@MySQL_Master_Node01 ~]# /etc/init.d/mysqld stop
[root@MySQL_Master_Node01 ~]# /etc/init.d/heartbeat start
```
![MySQL_Master_Node01_ifconfig.png](D:\EAM系统基础架构\EAM-system-infrastructure\Pictures\MySQL_Master_Node01_ifconfig.png)

![snipaste20161207_131725.png](D:\EAM系统基础架构\EAM-system-infrastructure\Pictures\snipaste20161207_131725.png)

备节点测试：
* 关闭主节点heartbeat 服务
* 观察备节点服务器 vip地址和mysqld服务以及日志文件

```language
[root@MySQL_Master_Node01 ~]# /etc/init.d/heartbeat stop
Stopping High-Availability services: Done.
```

![snipaste20161207_132107.png](D:\EAM系统基础架构\EAM-system-infrastructure\Pictures\snipaste20161207_132107.png)

![snipaste20161207_132240.png](D:\EAM系统基础架构\EAM-system-infrastructure\Pictures\snipaste20161207_132240.png)

##### 4. 注意事项
heartbeat软件不是基于服务的。也就是单纯的mysql服务出现故障是不会切换到备节点的


