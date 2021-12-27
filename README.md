# ldirectord介绍
ldirectord原始链接地址：http://www.vergenet.net/linux/ldirectord/
ldirectord 是一个守护程序，用于监视和管理集群中的real server的负载平衡。pcs是用来使ldirectord出现故障之后进行转移。
ldirectord在启动的时候读取/etc/ha.d/ldirectord.cf配置文件进行解析；解析文件后，将在 LVS 上创建虚拟服务器的条目；
启动之后将定期监视指定的真实服务器，如果它们被视为活动服务器，则将其添加到每个虚拟服务器的列表中；如果实际服务器出现故障，则会将其从该列表中删除。
对于每个配置，只能启动一个 ldirectord 实例，但对于不同的配置，可以启动更多的 ldirectord 实例。
ldirectord在启动的时候还会检查ipvsadm模块是否存在。

# ldirectord本身状态检测介绍
PCS中的ldirectord资源的脚本目录：/usr/lib/ocf/resource.d/heartbeat/ldirectord
此脚本是shell脚本，执行的启动命令是：/usr/sbin/ldirectord /etc/ha.d/ldirectord.cf start
执行的检测状态命令是：/usr/sbin/ldirectord /etc/ha.d/ldirectord.cf status 2>&1
在检测状态时调用的命令：
    /bin/sh /usr/lib/ocf/resource.d/heartbeat/ldirectord monitor
    /usr/bin/perl -w /usr/sbin/ldirectord /etc/ha.d/ldirectord.cf status
在/usr/sbin/ldirectord status检测状态时，它只会检测ldirectord进程本身是否存在，不会检查lvs的real server状态；

# real server状态检测介绍
lvs的real server状态是在ldirectord启动之后，其变成一个守护进程，定时去检测real server的状态，并进行维护ipvsadm的删除、添加real server。
守护进程：ldirectord tcp:172.16.130.161:19000
    检测real server时进程：ldirectord tcp:172.16.130.161:19000 checking 172.16.130.21
在启动之后对real server检查的日志可以使用monitorfile="/var/log/ldirectord_monitor.log"指定，在ldirectord代码中其写日志逻辑是：
    service_set($v, $r, "Failed to bind to $$r{server} with DBI->errstr:".DBI->errstr, {do_log => 1});



# 为什么数据库连不上但是不会导致ldirectord切换？
1.当lvs监控的数据库出现问题，那么ldirectord本身没有出现问题的话，那么pcs不会切换；
只有当"ldirectord进程本身出现问题"或者"ipvsadm模块没有安装"时，那么pcs才会切换到其他节点。
比如修改了监控用户的密码，ldirectord进程本身是正常的，只是由于real server检查无法连接到数据库了，导致没有可用的real servere
2.因为原始的ldirectord代码中没有记录当real server出现问题时具体的错误信息，所以当连不上数据库时，只能从monitorfile指定的日志文件看到最终的状态，具体的描述信息无法看到。
3.故而这里修改了ldirectord增加了检查real server状态时的日志输出，日志输出位置在monitorfile指定的日志文件中
4.其他注意事项：
如果配置了monitorfile，那么需要配置此日志文件的logrotate：
cat >/etc/logrotate.d/ldirectord_monitor<<EOF
/var/log/ldirectord_monitor.log {
    missingok
    compress
    copytruncate
    daily
    rotate 10
    minsize 2048
    size 500M
    notifempty
}
EOF

