## Tomcat

### timezone

当Tomcat记录日志时间和容器/vm不相同时，需要修改Tomcat的时区来保障.



#### JAVA_OPTS

在java的启动参数中添加` -Duser.timezone=GMT+08`

#### TZ env

就是修改/etc/profile文件，在文件的末尾添加 `export TZ='Asia/Shanghai'`，然后使用命令
`source /etc/profile`使其生效即可。

If the `TZ` environment variable does not have a value, the operation chooses a time zone by default. 

很奇怪，容器里时区是对的呢...

The TZ environment variable is **used to establish the local time zone**. The value of the variable is used by various time functions to compute times relative to Coordinated Universal Time (UTC) (formerly known as Greenwich Mean Time (GMT)). The time on the computer should be set to UTC.