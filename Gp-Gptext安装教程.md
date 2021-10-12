

### 1.1、集群介绍

5台虚拟机，1个master节点，4个segment的集群，示例：

```shell
wuxiang-test-1（master）
wuxiang-test-2
wuxiang-test-3
wuxiang-test-4
wuxiang-test-5
```

#### 1.2、修改主机名称

由于虚拟机重启后主机名称变为localhost，所以要永久性地修改为wuxiang-test-1这种形式，进行如下操作：

### 2、安装准备

#### 2.1、修改各节点hosts（所有节点）

```shell
[root@wuxiang-test-2 ~]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.40.218 wuxiang-test-1
192.168.40.238 wuxiang-test-2
192.168.40.239 wuxiang-test-3
192.168.40.240 wuxiang-test-4
192.168.40.241 wuxiang-test-5x
```

注：标注了所有节点的配置项可以在安装greenplum并配置免密后用gpssh统一操作3.3。

#### 2.2、修改network文件（所有节点，名称有差异）

```shell
[root@wuxiang-test-2 ~]# cat /etc/sysconfig/network
NISDOMAIN=QI
HOSTNAME=wuxiang-test-2
```

#### 2.3修改内核文件（所有节点）

```shell
[root@wuxiang-test-2 ~]# cat /etc/sysctl.conf
vm.swappiness = 10
kernel.shmmax = 500000000
kernel.shmmni = 4096
kernel.shmall = 4000000000
kernel.sem = 250 512000 100 2048
kernel.sysrq = 1
kernel.core_uses_pid = 1
kernel.msgmnb = 65536
kernel.msgmax = 65536
kernel.msgmni = 2048
net.ipv4.tcp_syncookies = 1
net.ipv4.ip_forward = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.tcp_tw_recycle = 1
net.ipv4.tcp_max_syn_backlog = 4096
net.ipv4.conf.all.arp_filter = 1
net.ipv4.ip_local_port_range = 1025 65535
net.core.netdev_max_backlog = 10000
vm.overcommit_memory = 2　　
最后使配置生效：
[root@wuxiang-test-2 ~]# sysctl -p
```

#### 2.4、修改进程数文件（所有节点）

[root@wuxiang-test-2 ~]# cat /etc/security/limits.d/20-nproc.conf 

```shell
# Default limit for number of user's processes to prevent

# accidental fork bombs.

# See rhbz #432903 for reasoning.

*          soft    nproc     4096
           root       soft    nproc     unlimited
```

#### 2.5、关闭防火墙（所有节点）

```shell
查看防火墙状态：firewall-cmd --state
关闭防火墙：systemctl stop firewalld.service
禁止防火墙开机启动：systemctl disable firewalld.service
修改配置（所有节点）：
[root@wuxiang-test-2 ~]# cat /etc/selinux/confin
SELINUX=disabled
SELINUXTYPE=targeted 
```

#### 2.6、创建用户（各节点共享）

```shell
groupadd -g 530 gpadmin
useradd -g 530 -u 530 -m -d /home/gpadmin -s /bin/bash gpadmin
chown -R gpadmin:gpadmin /home/gpadmin
echo "gpadmin" | passwd --stdin gpadmin
```

### 3、安装Greenplum DB

#### 3.1、在Master节点上安装Greenplum 

安装包下载地址：https://network.pivotal.io/products/pivotal-gpdb/#/releases/413133/file_groups/1866
安装包是rpm格式的执行rpm安装命令：

```shell
[root@wuxiang-test-1 ~]# rpm -ivh greenplum-db-5.21.0-rhel6-x86_64.rpm 
```

默认的安装路径是/usr/local。
将/usr/local/greenplum-db-5.21.0文件拷贝至所有节点（可以压缩再解压，也可以使用gpssh方式）
然后需要修改该路径gpadmin操作权限（所有节点）：

```shell
chown -R gpadmin:gpadmin /usr/local chown -R gpadmin:gpadmin /opt
```

建立软连接（所有节点）:

```shell
ln -s /usr/local/greenplum-db-5.21.0 greenplum-db
```

#### 3.2、创建hostlist、seg_hosts文件

切换gpadmin用户，创建conf文件夹，

```shell
[gpadmin@wuxiang-test-1 ~]# cd conf/  [gpadmin@wuxiang-test-1 conf]# cat hostlist  wuxiang-test-1  wuxiang-test-2  wuxiang-test-3  wuxiang-test-4  wuxiang-test-5  [gpadmin@wuxiang-test-1 conf]# cat seg_hosts  wuxiang-test-2  wuxiang-test-3  wuxiang-test-4  wuxiang-test-5
```

#### 3.3、配置免密连接

```shell
[root@ wuxiang-test-1 ~]# su gpadmin
[gpadmin@ wuxiang-test-1 ~]# source /usr/local/greenplum-db/greenplum_path.sh  
[gpadmin@ wuxiang-test-1 ~]# gpssh-exkeys -f /home/gpadmin/conf/hostlist

[STEP 1 of 5] create local ID and authorize on local host
  ... /home/gpadmin/.ssh/id_rsa file exists ... key generation skipped

[STEP 2 of 5] keyscan all hosts and update known_hosts file

[STEP 3 of 5] authorize current user on remote hosts
  ... send to wuxiang-test-1
  ... send to wuxiang-test-2
  ... send to wuxiang-test-3
  ... send to wuxiang-test-4
  ... send to wuxiang-test-5
```

#提示：这里提示输入各个子节点gpadmin用户密码

```shell
[STEP 4 of 5] determine common authentication file content

[STEP 5 of 5] copy authentication files to all remote hosts
  ... finished key exchange with wuxiang-test-1
  ... finished key exchange with wuxiang-test-2
  ... finished key exchange with wuxiang-test-3
  ... finished key exchange with wuxiang-test-4
  ... finished key exchange with wuxiang-test-5
[INFO] completed successfully

```

测试免密是否成功：

```shell
[gpadmin@wuxiang-test-1 ~]# ssh wuxiang-test-4
```

或者用gpssh：

```shell
[gpadmin@wuxiang-test-1 ~]$ gpssh -f /home/gpadmin/conf/hostlist
=> pwd
[wuxiang-test-1] /home/gpadmin
[wuxiang-test-4] /home/gpadmin
[wuxiang-test-5] /home/gpadmin
[wuxiang-test-3] /home/gpadmin
[wuxiang-test-2] /home/gpadmin
=> exit
```

显示上面内容，即为成功。
回到顶部

###  4、初始化数据库

####  4.1、创建资源目录

```shell
source /usr/local/ greenplum-db/greenplum_path.sh
gpssh -f /home/gpadmin/conf/hostlist #统一处理所有节点
```

```shell
#创建资源目录 /opt/greenplum/data下一系列目录（生产目录个数可根据需求生成）
=> mkdir -p /opt/greenplum/data/master
=> mkdir -p /opt/greenplum/data/primary
=> mkdir -p /opt/greenplum/data/mirror
=> mkdir -p /opt/greenplum/data2/primary
=> mkdir -p /opt/greenplum/data2/mirror
```

#### 4.2、环境变量配置（所有节点）

```shell
[gpadmin@wuxiang-test-1 ~]$ cat /home/gpadmin/.bash_profile

# .bash_profile

# Get the aliases and functions

if [ -f ~/.bashrc ]; then
    . ~/.bashrc
fi

# User specific environment and startup programs

PATH=$PATH:$HOME/.local/bin:$HOME/bin

export PATH

source /usr/local/greenplum-db/greenplum_path.sh
export MASTER_DATA_DIRECTORY=/opt/greenplum/data/master/gpseg-1
export GPPORT=5432
export PGDATABASE=gp_sydb
注：不能用gpssh编辑文件
让环境变量生效：
source /home/gpadmin/.bash_profile
```

#### 4.3、NTP配置

启用master节点上的ntp，并在Segment节点上配置和启动NTP：
#master 节点

```shell
[root@wuxiang-test-1 ~]# echo "server 127.127.1.0" >>/etc/ntp.conf 
```

#Segment节点

```shell
[root@wuxiang-test-2 ~]# echo "server wuxiang-test-1 perfer" >>/etc/ntp.conf
```

#master节点

```shell
[root@wuxiang-test-1 ~]# systemctl start  ntpd
[root@wuxiang-test-1 ~]# systemctl enable  ntpd
```

#### 4.4、检查各节点的连通性

```shell
[gpadmin@wuxiang-test-1 bin]$ cd /usr/local/greenplum-db/bin
[gpadmin@wuxiang-test-1 bin]$ gpcheckperf -f /home/gpadmin/conf/hostlist -r N -d /tmp
/usr/local/greenplum-db/./bin/gpcheckperf -f /home/gpadmin/conf/hostlist -r N -d /tmp

-------------------

--  NETPERF TEST
-------------------

[Warning] retrying with port 23012
[Warning] retrying with port 23024
[Warning] retrying with port 23036
[Warning] retrying with port 23048
[Warning] retrying with port 23060

====================

==  RESULT
====================

Netperf bisection bandwidth test
wuxiang-test-1 -> wuxiang-test-2 = 110.490000
wuxiang-test-3 -> wuxiang-test-4 = 112.120000
wuxiang-test-5 -> wuxiang-test-1 = 108.990000
wuxiang-test-2 -> wuxiang-test-1 = 102.830000
wuxiang-test-4 -> wuxiang-test-3 = 112.010000
wuxiang-test-1 -> wuxiang-test-5 = 108.930000

Summary:
sum = 655.37 MB/sec
min = 102.83 MB/sec
max = 112.12 MB/sec
avg = 109.23 MB/sec
median = 110.49 MB/sec
我在安装过程中由于反复尝试了多次，出现了如下错误：
[gpadmin@wuxiang-test-1 bin]$ gpcheckperf -f /home/gpadmin/conf/hostlist -r N -d /tmp

/usr/local/greenplum-db/./bin/gpcheckperf -f /home/gpadmin/conf/hostlist -r N -d /tmp
-------------------

--  NETPERF TEST
-------------------

[Warning] retrying with port 23012
[Warning] retrying with port 23024
[Warning] retrying with port 23036
[Warning] retrying with port 23048
[Error] unable to start netserver ... abort netperf test

====================

==  RESULT
====================
```

经尝试是由于端口占用过多导致，gpcheckperf文件中默认是尝试5次，如果5次都没连通，则会报这个错误，由于未找到删除端口办法，所以修改了gpcheckperf文件中xrange为10次

#### 4.5、执行初始化

```shell
[gpadmin@wuxiang-test-1 bin]$ cd /usr/local/greenplum-db/docs/cli_help/gpconfigs
[gpadmin@wuxiang-test-1 gpconfigs]$ cp gpinitsystem_config initgp_config
[gpadmin@wuxiang-test-1 gpconfigs]$ vim initgp_config
```

修改内容：

```shell
#目录与4.1创建的目录一致 declare -a DATA_DIRECTORY=(/opt/greenplum//data/primary /opt/greenplum//data/primary /opt/greenplum//data/primary /opt/greenplum//data2/primary /opt/greenplum//data2/primary /opt/greenplum//data2/primary)
declare -a MIRROR_DATA_DIRECTORY=(/opt/greenplum/data/mirror /opt/greenplum/data/mirror /opt/greenplum/data/mirror /opt/greenplum/data2/mirror /opt/greenplum/data2/mirror /opt/greenplum/data2/mirror)

ARRAY_NAME="gp_sydb"　　　　　　　　　　　　　　　　　　　　　　　　#初始化数据库名称
MASTER_HOSTNAME=wuxiang-test-1 　　　　　　　　　　　　　　　　　 #主节点名称
MASTER_DIRECTORY=/opt/greenplum/data/master
MASTER_DATA_DIRECTORY=/opt/greenplum/data/master/gpseg-1
DATABASE_NAME=gp_sydb
MACHINE_LIST_FILE=/home/gpadmin/conf/seg_hosts
```

Ip最好是一万以
执行初始化：

```shell
[gpadmin@wuxiang-test-1 bin]$ gpinitsystem -h /home/gpadmin/conf/seg_hosts -c initgp_config
20190805:14:58:19:028221 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-Checking configuration parameters, please wait...
20190805:14:58:19:gpinitsystem:wuxiang-test-1:gpadmin-[FATAL]:-Configuration file initgp_config does not exist. Script Exiting!
[gpadmin@wuxiang-test-1 bin]$ cd /usr/local/greenplum-db/docs/cli_help/gpconfigs
[gpadmin@wuxiang-test-1 gpconfigs]$ gpinitsystem -h /home/gpadmin/conf/seg_hosts -c initgp_config -S
20190805:14:59:36:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-Checking configuration parameters, please wait...
20190805:14:59:36:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-Reading Greenplum configuration file initgp_config
20190805:14:59:36:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-Locale has not been set in initgp_config, will set to default value
20190805:14:59:36:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-Locale set to en_US.utf8
20190805:14:59:37:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-MASTER_MAX_CONNECT not set, will set to default value 250
20190805:14:59:37:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-Checking configuration parameters, Completed
20190805:14:59:37:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-Commencing multi-home checks, please wait...
....
20190805:14:59:38:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-Configuring build for standard array
20190805:14:59:38:028451 gpinitsystem:wuxiang-test-1:gpadmin-[WARN]:-Option -S supplied, but no mirrors have been defined, ignoring -S option
20190805:14:59:38:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-Commencing multi-home checks, Completed
20190805:14:59:38:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-Building primary segment instance array, please wait...
........................
20190805:14:59:47:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-Checking Master host
20190805:14:59:47:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-Checking new segment hosts, please wait...
........................
20190805:15:00:09:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-Checking new segment hosts, Completed
20190805:15:00:09:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-Greenplum Database Creation Parameters
20190805:15:00:09:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:---------------------------------------
20190805:15:00:09:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-Master Configuration
20190805:15:00:09:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:---------------------------------------
20190805:15:00:09:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-Master instance name       = gp_sydb
20190805:15:00:09:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-Master hostname            = wuxiang-test-1
20190805:15:00:09:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-Master port                = 5432
20190805:15:00:09:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-Master instance dir        = /opt/greenplum/data/master/gpseg-1
20190805:15:00:09:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-Master LOCALE              = en_US.utf8
20190805:15:00:09:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-Greenplum segment prefix   = gpseg
20190805:15:00:09:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-Master Database            = gp_sydb
20190805:15:00:09:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-Master connections         = 250
20190805:15:00:09:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-Master buffers             = 128000kB
20190805:15:00:09:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-Segment connections        = 750
20190805:15:00:09:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-Segment buffers            = 128000kB
20190805:15:00:09:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-Checkpoint segments        = 8
20190805:15:00:09:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-Encoding                   = UNICODE
20190805:15:00:09:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-Postgres param file        = Off
20190805:15:00:09:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-Initdb to be used          = /usr/local/greenplum-db/./bin/initdb
20190805:15:00:09:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-GP_LIBRARY_PATH is         = /usr/local/greenplum-db/./lib
20190805:15:00:09:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-HEAP_CHECKSUM is           = on
20190805:15:00:09:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-HBA_HOSTNAMES is           = 0
20190805:15:00:09:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-Ulimit check               = Passed
20190805:15:00:09:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-Array host connect type    = Single hostname per node
20190805:15:00:09:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-Master IP address [1]      = 192.168.40.218
20190805:15:00:09:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-Master IP address [2]      = ::1
20190805:15:00:09:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-Master IP address [3]      = fe80::f816:3eff:fe97:2cc8
20190805:15:00:09:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-Standby Master             = Not Configured
20190805:15:00:09:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-Primary segment #          = 6
20190805:15:00:09:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-Total Database segments    = 24
20190805:15:00:09:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-Trusted shell              = ssh
20190805:15:00:09:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-Number segment hosts       = 4
20190805:15:00:09:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-Mirroring config           = OFF
20190805:15:00:09:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:----------------------------------------
20190805:15:00:09:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-Greenplum Primary Segment Configuration
20190805:15:00:09:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:----------------------------------------
20190805:15:00:09:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-wuxiang-test-2     /opt/greenplum//data/primary/gpseg0     6000     2     0
20190805:15:00:09:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-wuxiang-test-2     /opt/greenplum//data/primary/gpseg1     6001     3     1
20190805:15:00:09:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-wuxiang-test-2     /opt/greenplum//data/primary/gpseg2     6002     4     2
20190805:15:00:09:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-wuxiang-test-2     /opt/greenplum//data2/primary/gpseg3     6003     5     3
20190805:15:00:09:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-wuxiang-test-2     /opt/greenplum//data2/primary/gpseg4     6004     6     4
20190805:15:00:09:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-wuxiang-test-2     /opt/greenplum//data2/primary/gpseg5     6005     7     5
20190805:15:00:09:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-wuxiang-test-3     /opt/greenplum//data/primary/gpseg6     6000     8     6
20190805:15:00:09:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-wuxiang-test-3     /opt/greenplum//data/primary/gpseg7     6001     9     7
20190805:15:00:09:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-wuxiang-test-3     /opt/greenplum//data/primary/gpseg8     6002     10     8
20190805:15:00:09:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-wuxiang-test-3     /opt/greenplum//data2/primary/gpseg9     6003     11     9
20190805:15:00:09:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-wuxiang-test-3     /opt/greenplum//data2/primary/gpseg10     6004     12     10
20190805:15:00:09:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-wuxiang-test-3     /opt/greenplum//data2/primary/gpseg11     6005     13     11
20190805:15:00:09:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-wuxiang-test-4     /opt/greenplum//data/primary/gpseg12     6000     14     12
20190805:15:00:09:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-wuxiang-test-4     /opt/greenplum//data/primary/gpseg13     6001     15     13
20190805:15:00:09:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-wuxiang-test-4     /opt/greenplum//data/primary/gpseg14     6002     16     14
20190805:15:00:09:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-wuxiang-test-4     /opt/greenplum//data2/primary/gpseg15     6003     17     15
20190805:15:00:09:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-wuxiang-test-4     /opt/greenplum//data2/primary/gpseg16     6004     18     16
20190805:15:00:09:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-wuxiang-test-4     /opt/greenplum//data2/primary/gpseg17     6005     19     17
20190805:15:00:09:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-wuxiang-test-5     /opt/greenplum//data/primary/gpseg18     6000     20     18
20190805:15:00:09:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-wuxiang-test-5     /opt/greenplum//data/primary/gpseg19     6001     21     19
20190805:15:00:09:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-wuxiang-test-5     /opt/greenplum//data/primary/gpseg20     6002     22     20
20190805:15:00:09:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-wuxiang-test-5     /opt/greenplum//data2/primary/gpseg21     6003     23     21
20190805:15:00:09:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-wuxiang-test-5     /opt/greenplum//data2/primary/gpseg22     6004     24     22
20190805:15:00:09:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-wuxiang-test-5     /opt/greenplum//data2/primary/gpseg23     6005     25     23

Continue with Greenplum creation Yy|Nn (default=N):

> y
> 20190805:15:00:26:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-Building the Master instance database, please wait...
> 20190805:15:00:37:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-Starting the Master in admin mode
> 20190805:15:00:45:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-Commencing parallel build of primary segment instances
> 20190805:15:00:46:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-Spawning parallel processes    batch [1], please wait...
> ........................
> 20190805:15:00:46:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-Waiting for parallel processes batch [1], please wait...
> ......................................
> 20190805:15:01:24:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:------------------------------------------------
> 20190805:15:01:24:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-Parallel process exit status
> 20190805:15:01:24:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:------------------------------------------------
> 20190805:15:01:24:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-Total processes marked as completed           = 24
> 20190805:15:01:24:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-Total processes marked as killed              = 0
> 20190805:15:01:24:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-Total processes marked as failed              = 0
> 20190805:15:01:24:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:------------------------------------------------
> 20190805:15:01:24:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-Deleting distributed backout files
> 20190805:15:01:24:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-Removing back out file
> 20190805:15:01:24:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-No errors generated from parallel processes
> 20190805:15:01:24:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-Restarting the Greenplum instance in production mode
> 20190805:15:01:24:008439 gpstop:wuxiang-test-1:gpadmin-[INFO]:-Starting gpstop with args: -a -l /home/gpadmin/gpAdminLogs -i -m -d /opt/greenplum/data/master/gpseg-1
> 20190805:15:01:24:008439 gpstop:wuxiang-test-1:gpadmin-[INFO]:-Gathering information and validating the environment...
> 20190805:15:01:24:008439 gpstop:wuxiang-test-1:gpadmin-[INFO]:-Obtaining Greenplum Master catalog information
> 20190805:15:01:24:008439 gpstop:wuxiang-test-1:gpadmin-[INFO]:-Obtaining Segment details from master...
> 20190805:15:01:25:008439 gpstop:wuxiang-test-1:gpadmin-[INFO]:-Greenplum Version: 'postgres (Greenplum Database) 5.21.0 build commit:27db6bab4c909daa8d6699d94cabc48f87b07fab'
> 20190805:15:01:25:008439 gpstop:wuxiang-test-1:gpadmin-[INFO]:-There are 0 connections to the database
> 20190805:15:01:25:008439 gpstop:wuxiang-test-1:gpadmin-[INFO]:-Commencing Master instance shutdown with mode='immediate'
> 20190805:15:01:25:008439 gpstop:wuxiang-test-1:gpadmin-[INFO]:-Master host=wuxiang-test-1
> 20190805:15:01:25:008439 gpstop:wuxiang-test-1:gpadmin-[INFO]:-Commencing Master instance shutdown with mode=immediate
> 20190805:15:01:25:008439 gpstop:wuxiang-test-1:gpadmin-[INFO]:-Master segment instance directory=/opt/greenplum/data/master/gpseg-1
> 20190805:15:01:26:008439 gpstop:wuxiang-test-1:gpadmin-[INFO]:-Attempting forceful termination of any leftover master process
> 20190805:15:01:26:008439 gpstop:wuxiang-test-1:gpadmin-[INFO]:-Terminating processes for segment /opt/greenplum/data/master/gpseg-1
> 20190805:15:01:26:008466 gpstart:wuxiang-test-1:gpadmin-[INFO]:-Starting gpstart with args: -a -l /home/gpadmin/gpAdminLogs -d /opt/greenplum/data/master/gpseg-1
> 20190805:15:01:26:008466 gpstart:wuxiang-test-1:gpadmin-[INFO]:-Gathering information and validating the environment...
> 20190805:15:01:26:008466 gpstart:wuxiang-test-1:gpadmin-[INFO]:-Greenplum Binary Version: 'postgres (Greenplum Database) 5.21.0 build commit:27db6bab4c909daa8d6699d94cabc48f87b07fab'
> 20190805:15:01:26:008466 gpstart:wuxiang-test-1:gpadmin-[INFO]:-Greenplum Catalog Version: '301705051'
> 20190805:15:01:26:008466 gpstart:wuxiang-test-1:gpadmin-[INFO]:-Starting Master instance in admin mode
> 20190805:15:01:27:008466 gpstart:wuxiang-test-1:gpadmin-[INFO]:-Obtaining Greenplum Master catalog information
> 20190805:15:01:27:008466 gpstart:wuxiang-test-1:gpadmin-[INFO]:-Obtaining Segment details from master...
> 20190805:15:01:27:008466 gpstart:wuxiang-test-1:gpadmin-[INFO]:-Setting new master era
> 20190805:15:01:27:008466 gpstart:wuxiang-test-1:gpadmin-[INFO]:-Master Started...
> 20190805:15:01:27:008466 gpstart:wuxiang-test-1:gpadmin-[INFO]:-Shutting down master
> 20190805:15:01:29:008466 gpstart:wuxiang-test-1:gpadmin-[INFO]:-Commencing parallel segment instance startup, please wait...
> .... 
> 20190805:15:01:33:008466 gpstart:wuxiang-test-1:gpadmin-[INFO]:-Process results...
> 20190805:15:01:33:008466 gpstart:wuxiang-test-1:gpadmin-[INFO]:-----------------------------------------------------
> 20190805:15:01:33:008466 gpstart:wuxiang-test-1:gpadmin-[INFO]:-   Successful segment starts                                            = 24
> 20190805:15:01:33:008466 gpstart:wuxiang-test-1:gpadmin-[INFO]:-   Failed segment starts                                                = 0
> 20190805:15:01:33:008466 gpstart:wuxiang-test-1:gpadmin-[INFO]:-   Skipped segment starts (segments are marked down in configuration)   = 0
> 20190805:15:01:33:008466 gpstart:wuxiang-test-1:gpadmin-[INFO]:-----------------------------------------------------
> 20190805:15:01:33:008466 gpstart:wuxiang-test-1:gpadmin-[INFO]:-Successfully started 24 of 24 segment instances 
> 20190805:15:01:33:008466 gpstart:wuxiang-test-1:gpadmin-[INFO]:-----------------------------------------------------
> 20190805:15:01:33:008466 gpstart:wuxiang-test-1:gpadmin-[INFO]:-Starting Master instance wuxiang-test-1 directory /opt/greenplum/data/master/gpseg-1 
> 20190805:15:01:34:008466 gpstart:wuxiang-test-1:gpadmin-[INFO]:-Command pg_ctl reports Master wuxiang-test-1 instance active
> 20190805:15:01:34:008466 gpstart:wuxiang-test-1:gpadmin-[INFO]:-No standby master configured.  skipping...
> 20190805:15:01:34:008466 gpstart:wuxiang-test-1:gpadmin-[INFO]:-Database successfully started
> 20190805:15:01:34:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-Completed restart of Greenplum instance in production mode
> 20190805:15:02:04:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-Scanning utility log file for any warning messages
> 20190805:15:02:04:028451 gpinitsystem:wuxiang-test-1:gpadmin-[WARN]:-*******************************************************
> 20190805:15:02:04:028451 gpinitsystem:wuxiang-test-1:gpadmin-[WARN]:-Scan of log file indicates that some warnings or errors
> 20190805:15:02:04:028451 gpinitsystem:wuxiang-test-1:gpadmin-[WARN]:-were generated during the array creation
> 20190805:15:02:04:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-Please review contents of log file
> 20190805:15:02:04:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-/home/gpadmin/gpAdminLogs/gpinitsystem_20190805.log
> 20190805:15:02:04:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-To determine level of criticality
> 20190805:15:02:04:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-These messages could be from a previous run of the utility
> 20190805:15:02:04:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-that was called today!
> 20190805:15:02:04:028451 gpinitsystem:wuxiang-test-1:gpadmin-[WARN]:-*******************************************************
> 20190805:15:02:04:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-Greenplum Database instance successfully created
> 20190805:15:02:04:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-------------------------------------------------------
> 20190805:15:02:04:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-To complete the environment configuration, please 
> 20190805:15:02:04:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-update gpadmin .bashrc file with the following
> 20190805:15:02:04:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-1. Ensure that the greenplum_path.sh file is sourced
> 20190805:15:02:04:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-2. Add "export MASTER_DATA_DIRECTORY=/opt/greenplum/data/master/gpseg-1"
> 20190805:15:02:04:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-   to access the Greenplum scripts for this instance:
> 20190805:15:02:04:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-   or, use -d /opt/greenplum/data/master/gpseg-1 option for the Greenplum scripts
> 20190805:15:02:04:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-   Example gpstate -d /opt/greenplum/data/master/gpseg-1
> 20190805:15:02:04:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-Script log file = /home/gpadmin/gpAdminLogs/gpinitsystem_20190805.log
> 20190805:15:02:04:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-To remove instance, run gpdeletesystem utility
> 20190805:15:02:04:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-To initialize a Standby Master Segment for this Greenplum instance
> 20190805:15:02:04:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-Review options for gpinitstandby
> 20190805:15:02:04:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-------------------------------------------------------
> 20190805:15:02:04:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-The Master /opt/greenplum/data/master/gpseg-1/pg_hba.conf post gpinitsystem
> 20190805:15:02:05:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-has been configured to allow all hosts within this new
> 20190805:15:02:05:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-array to intercommunicate. Any hosts external to this
> 20190805:15:02:05:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-new array must be explicitly added to this file
> 20190805:15:02:05:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-Refer to the Greenplum Admin support guide which is
> 20190805:15:02:05:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-located in the /usr/local/greenplum-db/./docs directory
> 20190805:15:02:05:028451 gpinitsystem:wuxiang-test-1:gpadmin-[INFO]:-------------------------------------------------------
> 若初始化失败，则重新执行4.1，删除已初始化的数据。
> 执行psql -d postgres进入到数据库，则说明安装完成。
```

### 5、数据库操作

#### 5.1、停止和启动集群

```shell
gpstop -M fast
gpstart -a
```

#### 5.2、登陆数据库

```shell
$ psql -d postgres
```

#### 5.3、集群状态

```shell
gpstate -e #查看mirror的状态
gpstate -f #查看standby master的状态
gpstate -s #查看整个GP群集的状态
gpstate -i #查看GP的版本
gpstate --help #帮助文档，可以查看gpstate更多用法
目前为止数据库已经操作完毕。默认只有本地可以连数据库，如果需要别的I可以连，需要修改gp_hba.conf文件。 
```

https://www.cnblogs.com/grapelet520/p/11304543.html