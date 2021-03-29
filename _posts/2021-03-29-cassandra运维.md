---
layout: post
title: cassandra运维
date: 2021-03-29
tags: cassandra    
---
参考
http://docs.datastax.com/en/landing_page/doc/landing_page/planning/planningHardware.html
https://docs.datastax.com/en/developer/java-driver/3.4/manual/
http://docs.datastax.com/en/archived/cassandra/2.2/cassandra/install/installRecommendSettings.html
http://tobert.github.io/pages/als-cassandra-21-tuning-guide.html


内存
max(min(1/2RAM, 1024M), min(1/4RAM, 32G))

修改jvm参数 jvm.options  
cassandra-env.sh  
MAX_HEAP_SIZE   -- Xms Xmx  
HEAP_NEWSIZE    -- Xmn

注意:  
内存小于12GB 使用ParNew/ConcurrentMarkSweep垃圾回收
内存大于12GB 使用G1

必须要修改的配置  
vim /etc/sysctl.conf  
vm.max_map_count=1048575  
net.core.rmem_max=16777216  
net.core.wmem_max=16777216  
net.core.rmem_default=16777216  
net.core.wmem_default=16777216  
net.core.optmem_max=40960  
net.ipv4.tcp_rmem=4096 87380 16777216  
net.ipv4.tcp_wmem=4096 65536 16777216

生效  
sysctl -p

修改服务器进程数  
vi /etc/security/limits.conf  添加用户级别句柄和进程  
*   soft noproc   65535  
*   hard noproc   65535  
*   soft nofile   1000000  
*   hard nofile   1000000  
*   soft memlock unlimited  
*   soft as unlimited  
*   soft core unlimited  
*   soft stack 10240  
说明：* 代表针对所有用户  
noproc 是代表最大进程数  
nofile 是代表最大文件打开数  

系统级别句柄  
sysctl -w fs.file-max =65536  

设置用户最大进程数  
vi /etc/profile  
ulimit -u 10000  (添加这行)  
source  /etc/profile	（说明：使修改免重启并生效）  

swap禁止  
swapoff -a  

修改配置  
cluster_name: (集群名字)
rpc_address: 162.16.6.42       （说明：本机ip）  
listen_address: 162.16.6.42    （说明：本机ip）  
- seeds: “162.16.6.42,127.0.0.1”      #集群ip 单机只留第一个就行了  
#注释下面两行  
#commitlog_sync: periodic  
#commitlog_sync_period_in_ms: 10000  
#去掉注释  
commitlog_sync: batch  
commitlog_sync_batch_window_in_ms: 2  
memtable_offheap_space_in_mb: 2048  
memtable_allocation_type: offheap_objects  
authenticator: PasswordAuthenticator  
native_transport_port: 9042  

创建目录  
mkdir logs  
mkdir data  

启动  
nohup ./cassandra &  
./nodetool status  查看状态  

关闭  
nodetool drain  
nodetool stodaemon  
kill xxx  

登录
./cqlsh -u cassandra -p cassandra ip port  
创建用户
CREATE USER admin WITH PASSWORD '123456' SUPERUSER;  

问题排查(https://docs.datastax.com/en/dse/5.1/dse-dev/datastax_enterprise/tools/nodetool/toolsGetTraceProbability.html)  
使用nodetool  
nodetool help  
nodetool help command  

read-repair工具  
cassandra reaper  

  
转载请注明：[这不是一只猫的博客](http://1024.notacat.cn) » [点击阅读原文](http://1024.notacat.cn/2021/03/cassandra%E8%BF%90%E7%BB%B4/)


