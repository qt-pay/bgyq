### Centos System Unit

### service type

Type : 启动类型simple、forking、oneshot、notify、dbus

- Type=simple（默认值）：systemd认为该服务将立即启动，服务进程不会fork。如果该服务要启动其他服务，不要使用此类型启动，除非该服务是socket激活型
- Type=forking：systemd认为当该服务进程fork，且父进程退出后服务启动成功。对于常规的守护进程（daemon），除非你确定此启动方式无法满足需求， 使用此类型启动即可。使用此启动类型应同时指定`PIDFile=`，==以便systemd能够跟踪服务的主进程==。
- Type=oneshot：这一选项适用于只执行一项任务、随后立即退出的服务。可能需要同时设置 `RemainAfterExit=yes` 使得 `systemd` 在服务进程退出之后仍然认为服务处于激活状态。
- Type=notify：与 `Type=simple` 相同，但约定服务会在就绪后向 `systemd` 发送一个信号，这一通知的实现由 `libsystemd-daemon.so` 提供
- Type=dbus：若以此方式启动，当指定的 BusName 出现在DBus系统总线上时，systemd认为服务就绪。

- PIDFile ： pid文件路径

#### 不使用forking的情况

今天启动 Redis 时阻塞很长时间，之后显示启动失败，启动状态如下。

```bash
systemd[1]: redis.service start operation timed out. Terminating.

systemd[1]: Failed to start A persistent key-value database.

systemd[1]: Unit redis.service entered failed state.
```

看了下 service 文件，发现 Systemd 启动命令如下

`ExecStart=/usr/sbin/redis-server /etc/redis.conf`

手动运行这条命令，发现是正常的，所以猜想是 service 文件的问题，后来发现只需要把 [Service] 部分的 Type=forking 注释掉就行了。

If set to forking, it is expected that the process configured with ExecStart= will call fork() as part of its start-up. The parent process is expected to exit when start-up is complete and all communication channels are set up. The child continues to run as the main daemon process. This is the behavior of traditional UNIX daemons. If this setting is used, it is recommended to also use the PIDFile= option, so that systemd can identify the main process of the daemon. systemd will proceed with starting follow-up units as soon as the parent process exits.

因为 Redis 配置文件里配置的是`daemonize off`

相当于已经`nginx -g`（daemonize off）时不能使用`forking`类型。

同理，kibana也不能使用forking类型使用notify可以

### service unit examp

```bash
[Unit]
Description=Redis
After=network.target

[Service]
Type=forking
PIDFile=/var/run/redis_6379.pid
ExecStart=/root/redis-5.0.4/src/redis-server /etc/redis/redis.conf
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s QUIT $MAINPID
PrivateTmp=true

[Install]
WantedBy=multi-user.target


## nginx
[Unit]
Description=nginx - high performance web server
Documentation=http://nginx.org/en/docs/
After=network.target remote-fs.target nss-lookup.target

[Service]
Type=forking
PIDFile=/usr/local/nginx/logs/nginx.pid
ExecStartPre=/usr/local/nginx/sbin/nginx -t -c /usr/local/nginx/conf/nginx.conf
ExecStart=/usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s QUIT $MAINPID
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

#### MAINPID

`$MAINPID` is a systemd variable for your service that points to the PID of the main application.

`$MAINPID`是服务的systemd变量，它指向主应用程序的PID。



### 引用

1. https://developer.aliyun.com/article/387495