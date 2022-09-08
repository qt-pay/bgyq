## Golang log

Only use log.Fatal from main.main or int functions.

因为Fatal会导致defer 无法执行--

### time format

日志输出，必然会包含时间戳，但是golang很离谱，它的时间格式化是固定的 `2006-01-02 15:04:05 -0700` 。

| 数字 |           含义           |
| :--: | :----------------------: |
| 2006 |     Year(four-digit)     |
|  06  |     Year(two-digit)      |
|  01  |    Month(zero-padded)    |
|  02  | DD of month(zero-padded) |
|  15  |   24-hour(zero-padded)   |
|  04  |   Minute(zero-padded)    |
|  05  |   Second(zero-padded)    |

一般的困扰主要有：

- 不知道只能固定要这个时间，换其他的，出来的结果莫名其妙，然后一脸懵逼；
- 为什么没有像其他语言一样，yyyy-mm-dd 这样的形式？
- 这个时间有什么特殊意义吗？为什么挑这么个时间，完全记不住；

如下示例代码：

```go
func main() {
	fmt.Println("Now time:", time.Now().Format("2006-01-02 15:04:05"))
}
// output
Now time: 2022-06-17 12:29:53
```

为什么选择这个时间？不少人有这样的疑问。有人猜测是 Go 项目启动的时间等。但仔细研究，发现 Go Team 还是用心良苦，目的是解决大家记忆问题。

比如常规的 ymd 格式，以 PHP 为例，一般这样 `Y-m-d H:i:s`，输出类似：2021-08-03 09:30:00，但如果我想输出：`21-8-4 9:30:00`，你不查手册，能写出来吗？你看看 PHP 文档中关于 date 格式化的说明，头有点大，竟然那么多，虽然常用的形式，大部分人都记得，但遇到不怎么常用的，就得查手册了。

反观 Go 语言，它直接使用一个具体的时间来当做格式化字符串，需要什么格式，改这个时间格式即可。比如上面的例子，常规方式：2006-01-02 15:04:05，而 21-8-4 9:30:00 这种格式，只需要对应的改变值即可：06-1-2 3:04:05。而且，我查了下，PHP 没法表示没有前导零的分钟数和秒数，而 Go 很容易实现。很显然，Go 的方式是更合理、更易用的，对于各种变化，也能够更自如的应对。

#### 设计原理

只不过，很多人对这个具体的时间觉得记不住。这一点，Go 官方也考虑到了。毕竟采用特殊的时间，目的就是为了解决大家记忆问题，因此要确保这个特殊时间也好记。Go 是这么设计的：

```bash
1: month (January, Jan, 01, etc)
2: day
3: hour (15 is 3pm on a 24 hour clock)
4: minute
5: second
6: year (2006)
7: timezone (GMT-7 is MST)
```

刚好是 1 2 3 4 5 6 7，据此进行变化即可。

### log demo

#### error log要全面

如下，一个服务启动报错，根据service找到它的启动命令，然后手动执行可以看到详细的报错

`Failed to parse configuration.Error msg 1 error(s) decoding:`错误提示很明显，该服务的配置文件启动上解析失败、

> Linux message文件也有记录

```bash
$ systemctl status test-agent.service 
● test-agent.service - UNI test Agent Daemon
   Loaded: loaded (/usr/lib/systemd/system/test-agent.service; enabled; vendor preset: disabled)
   Active: failed (Result: start-limit) since Wed 2022-09-07 15:52:31 CST; 7min ago
  Process: 16252 ExecStart=/bin/bash -c -l /usr/local/bin/test-agent (code=exited, status=1/FAILURE)
....


$ /bin/bash -c -l /usr/local/bin/test-agent
Failed to parse configuration.Error msg 1 error(s) decoding:

* 'ScsiIps[scsi2][0]' expected type 'string', got unconvertible type '[]interface {}'
test-agent/common/config.InitConfig
        /root/rpmbuild/BUILD/test-agent-release/common/config/config.go:114
main.main
        /root/rpmbuild/BUILD/test-agent-release/cmd/main.go:501
runtime.main
        /usr/local/go/src/runtime/proc.go:203
runtime.goexit
        /usr/local/go/src/runtime/asm_amd64.s:1357


$ cat /var/lib/message 
...

Sep  7 15:52:29 testnode systemd: Configuration file /usr/lib/systemd/system/network-audit-agent.service is marked executable. Please remove executable permission bits. Proceeding anyway.
Sep  7 15:52:29 testnode bash: Failed to parse configuration.Error msg 1 error(s) decoding:
Sep  7 15:52:29 testnode bash: * 'ScsiIps[scsi2][0]' expected type 'string', got unconvertible type '[]interface {}'
Sep  7 15:52:29 testnode bash: test-agent/common/config.InitConfig
Sep  7 15:52:29 testnode bash: /root/rpmbuild/BUILD/test-agent-release/common/config/config.go:114
Sep  7 15:52:29 testnode bash: main.main
Sep  7 15:52:29 testnode bash: /root/rpmbuild/BUILD/test-agent-release/cmd/main.go:501
Sep  7 15:52:29 testnode bash: runtime.main
Sep  7 15:52:29 testnode bash: /usr/local/go/src/runtime/proc.go:203
Sep  7 15:52:29 testnode bash: runtime.goexit
Sep  7 15:52:29 testnode bash: /usr/local/go/src/runtime/asm_amd64.s:1357
Sep  7 15:52:29 testnode systemd: test-agent.service: main process exited, code=exited, status=1/FAILURE
Sep  7 15:52:29 testnode systemd: Unit test-agent.service entered failed state.
Sep  7 15:52:29 testnode systemd: test-agent.service failed.
Sep  7 15:52:30 testnode systemd: test-agent.service holdoff time over, scheduling restart.
Sep  7 15:52:30 testnode systemd: Stopped UNI test Agent Daemon.
Sep  7 15:52:30 testnode systemd: Started UNI test Agent Daemon
```

end