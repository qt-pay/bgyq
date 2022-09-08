## Linux rsyslog and /var/log/messages

### Linux Log Files that are Located under /var/log Directory

日志很关键，重启后怎么追查日志就靠他们了- a message addict.

需要安装和配置好rsyslog service才能在/var/log/记录和轮替日志

If you spend lot of time in Linux environment, it is essential that you know where the log files are located, and what is contained in each and every log file.

When your systems are running smoothly, take some time to learn and understand the content of various log files, which will help you when there is a crisis and you have to look though the log files to identify the issue.

`/etc/rsyslog.conf` **controls what goes inside some of the log files.** For example, following is the entry in rsyslog.conf for /var/log/messages.

```bash
$ grep "/var/log/messages" /etc/rsyslog.conf
*.info;mail.none;authpriv.none;cron.none                /var/log/messages
```

In the above output,

- *.info indicates that all logs with type INFO will be logged.
- mail.none,authpriv.none,cron.none indicates that those error messages should not be logged into the /var/log/messages file.
- You can also specify *.none, which indicates that none of the log messages will be logged.

The following are the 20 different log files that are located under /var/log/ directory. Some of these log files are distribution specific. For example, you’ll see dpkg.log on Debian based systems (for example, on Ubuntu).

1. **/var/log/messages** – Contains global system messages, including the messages that are logged during system startup. There are several things that are logged in /var/log/messages including mail, cron, daemon, kern, auth, etc.
2. **/var/log/dmesg** – Contains kernel ring buffer information. When the system boots up, it prints number of messages on the screen that displays information about the hardware devices that the kernel detects during boot process. These messages are available in kernel ring buffer and whenever the new message comes the old message gets overwritten. You can also view the content of this file using the [dmesg command](https://www.thegeekstuff.com/2010/10/dmesg-command-examples/).
3. **/var/log/auth.log** – Contains system authorization information, including user logins and authentication machinsm that were used.
4. **/var/log/boot.log** – Contains information that are logged when the system boots
5. **/var/log/daemon.log** – Contains information logged by the various background daemons that runs on the system
6. **/var/log/dpkg.log** – Contains information that are logged when a package is installed or removed using [dpkg command](https://www.thegeekstuff.com/2010/06/install-remove-deb-package/)
7. **/var/log/kern.log** – Contains information logged by the kernel. Helpful for you to troubleshoot a custom-built kernel.
8. **/var/log/lastlog** – Displays the recent login information for all the users. This is not an ascii file. You should use lastlog command to view the content of this file.
9. **/var/log/maillog /var/log/mail.log** – Contains the log information from the mail server that is running on the system. For example, sendmail logs information about all the sent items to this file
10. **/var/log/user.log** – Contains information about all user level logs
11. **/var/log/Xorg.x.log** – Log messages from the X
12. **/var/log/alternatives.log** – Information by the update-alternatives are logged into this log file. On Ubuntu, update-alternatives maintains symbolic links determining default commands.
13. **/var/log/btmp** – This file contains information about failed login attemps. Use the last command to view the btmp file. For example, “last -f /var/log/btmp | more”
14. **/var/log/cups** – All printer and printing related log messages
15. **/var/log/anaconda.log** – When you install Linux, all installation related messages are stored in this log file
16. **/var/log/yum.log** – Contains information that are logged when a package is installed using yum
17. **/var/log/cron** – Whenever [cron daemon](https://www.thegeekstuff.com/2009/06/15-practical-crontab-examples/) (or [anacron](https://www.thegeekstuff.com/2011/05/anacron-examples/)) starts a cron job, it logs the information about the cron job in this file
18. **/var/log/secure** – Contains information related to authentication and authorization privileges. For example, sshd logs all the messages here, including unsuccessful login.
19. **/var/log/wtmp** or **/var/log/utmp** – Contains login records. Using wtmp you can find out who is logged into the system. who command uses this file to display the information.
20. **/var/log/faillog** – Contains user failed login attemps. Use faillog command to display the content of this file.

### What is Syslog: Daemons, Message Formats and Protocols

Pretty much everyone’s heard about syslog: with its roots in the 80s, it’s still used for a lot of the logging done today. Mostly because of its long history, syslog is quite a vague concept, referring to many things. Which is why you’ve probably heard:

- *Check syslog, maybe it says something about the problem* – referring to /var/log/messages.
- *Syslog doesn’t support messages longer than 1K* – about message format restrictions.
- *Syslog is unreliable* – referring to the UDP protocol.

In this post, we’ll explain the different facets by being specific: instead of saying “syslog”, you’ll read about **syslog daemons**, about **syslog message formats** and about **syslog protocols**.

#### Syslog daemons

A syslog daemon is a program that:

- **can receive local syslog messages**. Traditionally */dev/log* UNIX socket and kernel logs.
- **can write them to a file**. Traditionally */var/log/messages* or */var/log/syslog* will receive everything, while some categories of messages go to specific files, like */var/log/mail*.
- **can forward them to the network or other destinations**. Traditionally, via UDP. Usually, the daemon also implements equivalent network listeners (UDP in this case).

This is where *syslog* is often referring to [syslogd or sysklogd](http://www.infodrom.org/projects/sysklogd/index.php), the original BSD syslog daemon. Development for it stopped for Linux since 2007, but continued for BSDs and OSX. There are alternatives, most notably:

- [**rsyslog**](http://www.rsyslog.com/). Originally a fork of syslogd, it still can be used as a drop in replacement for it. Over the years, it evolved into a performance-oriented, multipurpose logging tool, that can [read data from multiple sources, parse and enrich logs in various ways, and ship to various destinations](https://sematext.com/blog/2015/10/05/recipe-apache-logs-rsyslog-parsing-elasticsearch/)


- [**syslog-ng**](https://syslog-ng.org/). Unlike rsyslog, it used a different configuration format from the start (rsyslog eventually got to the same conclusion, but still supports the BSD syslog config syntax as well – which can be confusing at times). You’d see a similar feature set to rsyslog, like [parsing unstructured data and shipping it to Elasticsearch](https://www.balabit.com/blog/how-to-parse-data-with-syslog-ng-store-in-elasticsearch-and-analyze-with-kibana/) or [Kafka](https://www.balabit.com/documents/syslog-ng-ose-latest-guides/en/syslog-ng-ose-guide-admin/html/configuring-destinations-kafka.html). It’s still fast and light, and while it may not have the ultimate performance of rsyslog (depends on the use-case, see the comments section), it has better documentation and it’s more portable
- [**nxlog**](http://nxlog.co/). Yet another syslog daemon which evolved into a multi-purpose log shipper, it sets itself apart by [working well on Windows](https://sematext.com/blog/2016/02/01/sending-windows-event-logs-to-logsene-using-nxlog-and-logstash/)

In essence, a modern syslog daemon is a log shipper that works with various syslog message formats and protocols. If you want to learn more about log shippers in general, we wrote a [side-by-side comparison of Logstash and 5 other popular shippers, including rsyslog and syslog-ng](https://sematext.com/blog/2016/09/13/logstash-alternatives/).

> 技术以前常用的daemon是`syslogd or sysklogd` 现在呢，用更好的`rsyslog` 、`syslog-ng` 或`nxlog`

#### Myths about syslog daemons

The one we come across most often is that syslog daemons are no good if you **log to files or if you want to parse unstructured data**. This used to be true years ago, but then so was [Y2K](https://en.wikipedia.org/wiki/Year_2000_problem). Things changed in the meantime. In the myth’s defense, some distributions ship with old versions of rsyslog and syslog-ng. Plus, the default configuration often only listens for */dev/log* and kernel messages (it doesn’t need more), so it’s easy to generalize.

#### Syslog message formats

You’ll normally find syslog messages in two major formats:

- the original BSD format ([RFC3164](https://tools.ietf.org/html/rfc3164))
- the “new” format ([RFC5424](https://tools.ietf.org/html/rfc5424))

#### RFC3164 a.k.a “the old format”

Although RFC suggests it’s a standard, [RFC3164](https://tools.ietf.org/html/rfc3164) was more of a collection of what was found in the wild at the time (2001), rather than a spec that implementations will adhere to. As a result, you’ll find slight variations of it. That said, most messages will look like the [RFC3164](https://tools.ietf.org/html/rfc3164) example:

```
<34>Oct 11 22:14:15 mymachine su: 'su root' failed for lonvick on /dev/pts/8
```

This is how the application should log to */dev/log*, and you can see some structure:

- *<34>* is a **priority number**. It represents the **f****acility** number multiplied by 8, to which **s****everity** is added. In this case, facility=4 (*Auth*) and severity=2 (*Critical*).
- *Oct 11 22:14:15* is commonly known as **syslog timestamp**. It misses the year, the time-zone and doesn’t have sub-second information. For those reasons, rsyslog also parses RFC3164-formatted messages with an [ISO-8601](https://en.wikipedia.org/wiki/ISO_8601) timestamp instead
- *mymachine* is a **host name** where the message was written.
- *su:* is a **tag**. Typically this is the process name – sometimes having a PID (like *su[1234]:*). The tag typically ends in a colon, but it may end up just with the square brackets or with a space.
- the **message (MSG)** is everything after the tag. In this example, since we have the colon to separate the tag and the message, the message actually starts with a space. This tiny detail often gives a lot of headache when parsing.

In */var/log/messages*, you’ll often see something like this:

```
Oct 11 22:14:15 su: 'su root' failed for lonvick on /dev/pts/8
```

This isn’t a syslog message format, it’s just how most syslog deamons write messages to files by default. Usually, **you can choose how the output data looks like**, for example [rsyslog has templates](http://www.rsyslog.com/doc/master/configuration/templates.html).

#### RFC5424 [a.k.a.][also know as] “the new format”

[RFC5424](https://tools.ietf.org/html/rfc5424) came up in 2009 to deal with the problems of [RFC3164](https://tools.ietf.org/html/rfc3164). First of all, it’s an actual standard, that daemons and libraries chose to implement. Here’s an example message:

```
<34>1 2003-10-11T22:14:15.003Z mymachine.example.com su - - - 'su root' failed for lonvick on /dev/pts/8
```

Now we get an [ISO-8601](https://en.wikipedia.org/wiki/ISO_8601) timestamp, amongst other improvements. We also get more structure: the dashes you can see there are places for PID, message ID and other structured data you may have. That said, [RFC5424](https://tools.ietf.org/html/rfc5424) structured data never really took off, as people preferred to put [JSON in the syslog message](https://sematext.com/blog/2013/05/28/structured-logging-with-rsyslog-and-elasticsearch/) (whether it’s the old or the new format). Finally, the new format supports UTF8 and other encodings, not only ASCII, and it’s easier to extend because it has a version number (in this example, the *1* after the priority number).

#### Myths around the syslog message formats

The ones we see more often are:

- **you can’t send syslog messages over 1K**. It is true that [RFC3164 stated that messages shouldn’t go over 1K,](https://tools.ietf.org/html/rfc3164#section-4.1) because it was around the maximum size of a UDP packet (now it can be higher with jumbo packets). Modern daemons don’t respect that constraint – the message limit is configurable in both rsyslog and syslog-ng. [RFC5424 made this official](https://tools.ietf.org/html/rfc5424#section-6.1), not only because of UDP changes but also because many other protocols are supported (see below).
- **timestamps aren’t exact**. That is true for [RFC3164 timestamps](https://tools.ietf.org/html/rfc3164#section-4.1.2), but not for the [RFC5424 ones](https://tools.ietf.org/html/rfc5424#section-6.2.3).
- **there’s no structure beyond predefined fields**. In fact, any modern syslog will happily parse a JSON from the message field. They can parse any kind of message format (structured or not).

#### Syslog protocols

Originally, syslog messages were sent over the wire via UDP – which was also [mentioned in RFC3164](https://tools.ietf.org/html/rfc3164#section-2). It was later standardized in [RFC5426](https://tools.ietf.org/html/rfc5426), after the new message format ([RFC5424](https://tools.ietf.org/html/rfc5424)) was published.

Modern syslog daemons support other protocols as well. Most notably:

- TCP

  . Just like the UDP, it was first used in the wild and then documented. That documentation finally came with

   RFC6587, which describes two flavors:

  - messages are **delimited by a trailer character**, typically a newline
  - messages are **framed based on an octet count**

- **TLS**. Standardized in [RFC5425](https://tools.ietf.org/html/rfc5425), which allows for encryption and certificate-based authorization

- [**RELP**](https://en.wikipedia.org/wiki/Reliable_Event_Logging_Protocol). [Unlike plain TCP, RELP adds application-level acknowledgements](http://blog.gerhards.net/2008/04/on-unreliability-of-plain-tcp-syslog.html), which provides at-least-once guarantees on delivering messages. You can also get RELP with TLS if you need encryption and authorization

Besides writing to files and communicating to each other, modern syslog daemons can also write to other destinations. For example, datastores like [MySQL](https://www.balabit.com/documents/syslog-ng-ose-latest-guides/en/syslog-ng-ose-guide-admin/html/configuring-destinations-sql.html) or [Elasticsearch](https://sematext.com/blog/2015/10/05/recipe-apache-logs-rsyslog-parsing-elasticsearch/) or queue systems such as [Kafka](https://sematext.com/blog/2015/10/06/recipe-rsyslog-apache-kafka-logstash/) and [RabbitMQ](https://www.balabit.com/sites/default/files/documents/syslog-ng-ose-latest-guides/en/syslog-ng-ose-guide-admin/html/configuring-destinations-amqp.html). Each such destination often comes with its own protocol and message format. For example, Elasticsearch uses JSON over HTTP (though you can also [secure it](https://sematext.com/blog/2017/01/18/elasticsearch-security-authentication-encryption-backup/) and [send syslog messages over HTTPS](https://sematext.com/blog/2014/03/18/ecrypting-logs-on-their-way-to-elasticsearch/)).

#### syslog daemon 配置示例：

http://kumu-linux.github.io/blog/2013/08/28/rsyslog-remote/
