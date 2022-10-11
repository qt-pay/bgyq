### Ansible: 入门

3.28 kolla和kube-proxy

emm，还是通`yaml`文件来定义执行步骤吧。

命令行有局限性，而且看起来不够直观

#### 先看再学

看到别人的大量的ansible时，先确定它们使用的module实在哪个版本支持的。

eg：2.5+ 才支持`loop`，在低版本的ansible中要使用`with_items`来代替。

#### 幂等性

幂等性是某些操作的属性，可以多次执行而不改变初始应用程序的结果。 这个概念出现在大多数Ansible模块中：您指定了所需的最终状态，Ansible决定是否应该运行任务。 默认情况下，此原则不适用于command模块。 

幂等性，使得ansible以结果为导向的，指定一个目标状态，ansible会自动判断，当前状态是否与目标状态一致，如果一致，则不进行任何操作，否则执行。

这个和kubernetes的desire status 和 current status多像。

#### ansible-doc

**ansible-doc**：显示模块帮助

```shell
ansible-doc [options] [module...]
-a				显示所有模块的文档
-l,--list		列出可用模块
-s，--snippet	显示指定模块的playbook片段
示例:
ansible-doc -l 		列出所有模块
ansible-doc ping	查看指定模块帮助用法
## ansible-doc -s 参数好用
ansible-doc -s ping	查看指定模块帮助用法,以片段方式，简单的了解
```

end

#### ansible常用参数

```bash
# 查看test_hosts组主机的主机名
# test_hosts：/etc/ansible/hosts 中定义的主机组
$ ansible -i /etc/ansible/hosts test_hosts -u root -m command -a 'hostname' -k
SSH password: 
192.168.31.110 | SUCCESS | rc=0 >>
host1

 -i：specify inventory host path
 -u：Remote User
 -m：Module name
 -a：Module Args
 -k：ask for privilege escalation password。


```

在Inventory(/etc/ansible/hosts)中配置`ansible_ssh_use`r和`ansible_ssh_pass`后可实现Ansible免密连接或者使用证书实现免密连接，即不使用`-k`参数。

end

#### ansible 并发

依赖Python的**multiprocessing** 库

#### ansible常用模块

好多模块都依赖state进行判断操作了，but 默认的comman模块除外。

##### 在远端机器上执行后台脚本

如果不用nohub, 脚本中的`&`任务会随着`/tmp/start_log.sh` 退出而消失...

```bash
## 挂在后端执行
$ ansible  -i /tmp/dodo/logtail-hosts  canary -m shell -a "nohup /tmp/start_log.sh &"

$ /tmp/start_log.sh
#!/bin/bash

python3  /root/test_log.py &

python3 /root/test_log_2.py &

python3 /root/test_log_3.py &


```

end

##### command

command的是默认模块，因此可以省略`-m`参数。

- The `command` module takes the command name followed by a list of space-delimited arguments.

- The given command will be executed on all selected nodes.

- The command(s) will not be processed through the shell, so variables like `$HOME` and operations like `"<"`, `">"`, `"|"`, `";"` and `"&"` will not work. Use the [shell](https://docs.ansible.com/ansible/latest/modules/shell_module.html#shell-module) module if you need these features.

  这里比较关键，不是通过目的主机的shell来处理的，所以没法引用操作主机的变量。

  `command`模块参数中出现变量时，是先替换本地的变量然后再去远端执行，而不是将变量名直接传到目标主机再替换。

```bash
## 文件不存在时，行ls命令
$ ansible '*' -a 'creates=/tmp/a.txta ls /tmp'
21.48.5.82 | CHANGED | rc=0 >>
a.ll

## -a ： --arguments 
## 执行touch命令
$ ansible '*' -m command -a 'touch /tmp/a.txta'
 [WARNING]: Consider using the file module with state=touch rather than running 'touch'.  If you need to use command
because file is insufficient you can add 'warn: false' to this command task or set 'command_warnings=False' in
ansible.cfg to get rid of this message.

21.48.5.82 | CHANGED | rc=0 >>

## 文件已存在，则跳过ls命令执行
$ ansible '*' -a 'creates=/tmp/a.txta ls /tmp'
21.48.5.82 | SUCCESS | rc=0 >>
skipped, since /tmp/a.txta exists



## 不存在文件，跳过ls 命令；文件存在则，执行
$ ansible '*' -a 'removes=/tmp/a.txta ls /tmp'
21.48.5.83 | SUCCESS | rc=0 >>
skipped, since /tmp/a.txta does not exist

21.48.5.82 | CHANGED | rc=0 >>
a.ll
a.txta



## 不支持管道
$ ansible '*'  -a "ls /tmp/|ls -l"
21.48.5.82 | FAILED | rc=2 >>
ls: cannot access /tmp/|ls: No such file or directorynon-zero return code



## 不能同时指定多个命令
$ ansible '*' -a "chdir=/home pwd ls ./"
21.48.5.83 | CHANGED | rc=0 >>
/homepwd: ignoring non-option arguments

21.48.5.82 | CHANGED | rc=0 >>
/homepwd: ignoring non-option arguments


## chdir 切换目录
$ ansible '*' -a "chdir=/home pwd "
21.48.5.83 | CHANGED | rc=0 >>
/home

21.48.5.82 | CHANGED | rc=0 >>
/home

```

###### ssh无法远程传输变量

所以需要shell模块。

```bash
## 这里再本机之间将$lol转换成a了，所以相当: ssh -p 22 21.48.5.84  "echo a" 
$ ssh -p 22 21.48.5.84  "echo $lol" 
a
$ ssh -p 22 21.48.5.84
Last login: Thu Mar 26 10:19:11 2020 from 21.48.5.83
$ echo  $lol

$ ansible '*'  -a "echo $HOME"
21.48.5.82 | CHANGED | rc=0 >>
/root

$ ansible '*' -a "echo $lol"
21.48.5.83 | CHANGED | rc=0 >>
a

21.48.5.82 | CHANGED | rc=0 >>
a
```







##### shell

shell模块可以使用command模块所有选项，但功能比command模块更强大，因为其支持"|"道符。

```bash
$ ansible "*" -m shell -a "ss -aon|wc -l" 
21.48.5.83 | CHANGED | rc=0 >>
518

21.48.5.82 | CHANGED | rc=0 >>
275

$ ansible "*" -m shell -a "executable=/bin/bash  ss -aon|wc -l" 
21.48.5.83 | CHANGED | rc=0 >>
519

21.48.5.82 | CHANGED | rc=0 >>
273

## 调用目标机器的shell去执行命令。
$ echo $lol
a
$ ansible '*' -a "echo $lol"
21.48.5.83 | CHANGED | rc=0 >>


21.48.5.82 | CHANGED | rc=0 >>


#### 
- name: Run a command that uses non-posix shell-isms (in this example /bin/sh doesn't handle redirection and wildcards together but bash does)
  shell: cat < /tmp/*txt
  args:
    executable: /bin/bash
```

###### shell使用多行命令

使用`|`和`;`都可以

```bash
- name: Exec multi line command with |
  shell: | # 注意这里
    echo "1"
    ls
    wrong_command
    umask
- name: Exec multi line command with ; 
  shell:  
    echo "1";  # 使用分号间隔多条命令
    ls;
    wrong_command;
    umask
```



##### raw

Executes a low-down and dirty SSH command, not going through the module subsystem. 不依赖ansible的任何子模块。

 `raw`模块只适用于下列两种场景：

* 第一种情况是在较老的（Python 2.4和之前的版本）主机上，
* 另一种情况是对任何没有安装Python的设备（如路由器）。 在任何其他情况下，使用`shell`或`command`模块更为合适。

就像`script`模块一样，`raw`模块不需要远程系统上的python

如果从playbook中使用raw，则可能需要使用`gather_facts: no`禁用事实收集。

``` bash
$ ansible '*' -m raw -a "executable=/bin/bash  ss -aon|wc -l"
21.48.5.83 | CHANGED | rc=0 >>
518
Shared connection to 21.48.5.83 closed.


21.48.5.82 | CHANGED | rc=0 >>
272
Shared connection to 21.48.5.82 closed.
```



##### script

`ansible pts -m script -a kafka_disable.py`

script模块可以帮助我们在远程主机上执行ansible主机上的脚本，也就是说，脚本一直存在于ansible主机本地，不需要手动拷贝到远程主机后再执行.

```bash
$ cat /tmp/a.sh 
#!/bin/bash
echo "test module script

$ ansible '*' -m script  -a /tmp/a.sh
21.48.5.83 | CHANGED => {
    "changed": true, 
    "rc": 0, 
    "stderr": "Shared connection to 21.48.5.83 closed.\r\n", 
    "stderr_lines": [
        "Shared connection to 21.48.5.83 closed."
    ], 
    "stdout": "test module script\r\n", 
    "stdout_lines": [
        "test module script"
    ]
}
21.48.5.82 | CHANGED => {
    "changed": true, 
    "rc": 0, 
    "stderr": "Shared connection to 21.48.5.82 closed.\r\n", 
    "stderr_lines": [
        "Shared connection to 21.48.5.82 closed."
    ], 
    "stdout": "test module script\r\n", 
    "stdout_lines": [
        "test module script"
    ]
}

## ansible -f 20 -i /tmp/t.ip all -m script -a 'dirs.py' > /tmp/source_data_dirs.sh
#!/usr/bin/python
import os, sys, socket

hostname = socket.gethostname()
ip = socket.gethostbyname(hostname)
d = {}
dir_name = '/home/data/logs/takumi/'
if os.path.exists(dir_name):
    r = [ name for name in os.listdir(dir_name) if os.path.isdir(os.path.join(dir_name, name)) ]
else:
    r = []
d[ip] = r
print d,

```



##### file

Manage files and file properties

Consider using the file module with state=touch rather than running "touch".就是说不要直接使用command模块去touch文件，而要有file模块来处理。

**force**：在两种情况下强制创建软链接。1、源文件不存在但之后会建立的情况；2、目标软件已存在，需要先取消之前的软链接，然后创建新的软链接。选项：yes|no
**group**：定义文件/目录的属组
**mode**：定义文件/目录的权限
**path**：必选项，定义文件/目录的路径
**recurse**：递归的设置文件的属性，只对目录有效
**src**：要被链接到的路径，只应用于state=link的情况
**dest**：被链接到的路径，只应用于state=link的情况
**state**：

- directory：如果目录不存在，创建目录
- file：文件存在返回SUCCESS，不存在返回FAILED
- link：创建软链接；hard：创建硬链接
- touch：如果文件不存在，则会创建一个新的文件，如果已存在，则更新其最后修改时间
- absent：删除目录/文件或者取消链接文件

文件操作：

```bash
$ ansible '*' -a "removes=/tmp/lo.txt echo 'file exists'"
21.48.5.83 | SUCCESS | rc=0 >>
skipped, since /tmp/lo.txt does not exist

21.48.5.82 | SUCCESS | rc=0 >>
skipped, since /tmp/lo.txt does not exist

You have new mail in /var/spool/mail/root
$ ansible '*' -m file -a "path=/tmp/lo.txt state=touch "
21.48.5.83 | CHANGED => {
    "changed": true, 
    "dest": "/tmp/lo.txt", 
    "gid": 0, 
    "group": "root", 
    "mode": "0644", 
    "owner": "root", 
    "size": 0, 
    "state": "file", 
    "uid": 0
}
21.48.5.82 | CHANGED => {
    "changed": true, 
    "dest": "/tmp/lo.txt", 
    "gid": 0, 
    "group": "root", 
    "mode": "0644", 
    "owner": "root", 
    "size": 0, 
    "state": "file", 
    "uid": 0
}
$ ansible '*' -a "removes=/tmp/lo.txt echo 'file exists'"
21.48.5.82 | CHANGED | rc=0 >>
file exists

21.48.5.83 | CHANGED | rc=0 >>
file exists

## 若存在则删除
$ ansible '*' -m file -a 'path=/tmp/a.ll state=absent'
21.48.5.83 | SUCCESS => {
    "changed": false, 
    "path": "/tmp/a.ll", 
    "state": "absent"
}
21.48.5.82 | CHANGED => {
    "changed": true, 
    "path": "/tmp/a.ll", 
    "state": "absent"
}
```



下面是官网给的例子，真棒。

```yaml
- name: Change file ownership, group and permissions
  file:
    path: /etc/foo.conf
    owner: foo
    group: foo
    mode: '0644'

- name: Give insecure permissions to an existing file
  file:
    path: /work
    owner: root
    group: root
    mode: '1777'

- name: Create a symbolic link
  file:
    src: /file/to/link/to
    dest: /path/to/symlink
    owner: foo
    group: foo
    state: link

- name: Create two hard links
  file:
    src: '/tmp/{{ item.src }}'
    dest: '{{ item.dest }}'
    state: hard
  loop:
    - { src: x, dest: y }
    - { src: z, dest: k }

- name: Touch a file, using symbolic modes to set the permissions (equivalent to 0644)
  file:
    path: /etc/foo.conf
    state: touch
    mode: u=rw,g=r,o=r

- name: Touch the same file, but add/remove some permissions
  file:
    path: /etc/foo.conf
    state: touch
    mode: u+rw,g-wx,o-rwx

- name: Touch again the same file, but dont change times this makes the task idempotent
  file:
    path: /etc/foo.conf
    state: touch
    mode: u+rw,g-wx,o-rwx
    modification_time: preserve
    access_time: preserve

- name: Create a directory if it does not exist
  file:
    path: /etc/some_directory
    state: directory
    mode: '0755'

- name: Update modification and access time of given file
  file:
    path: /etc/some_file
    state: file
    modification_time: now
    access_time: now

- name: Set access time based on seconds from epoch value
  file:
    path: /etc/another_file
    state: file
    access_time: '{{ "%Y%m%d%H%M.%S" | strftime(stat_var.stat.atime) }}'

- name: Recursively change ownership of a directory
  file:
    path: /etc/foo
    state: directory
    recurse: yes
    owner: foo
    group: foo

- name: Remove file (delete file)
  file:
    path: /etc/foo.txt
    state: absent

- name: Recursively remove directory
  file:
    path: /etc/foo
    state: absent
```

##### copy

复制文件到远程主机，copy模块包含如下选项：

- backup：在覆盖之前将原文件备份，备份文件包含时间信息。有两个选项：yes|no 
- content：用于替代"src",可以直接设定指定文件的值 
- dest：必选项。要将源文件复制到的远程主机的绝对路径，如果源文件是一个目录，那么该路径也必须是个目录 
- directory_mode：递归的设定目录的权限，默认为系统默认权限
- force：如果目标主机包含该文件，但内容不同，如果设置为yes，则强制覆盖，如果为no，则只有当目标主机的目标位置不存在该文件时，才复制。默认为yes
- others：所有的file模块里的选项都可以在这里使用
- src：要复制到远程主机的文件在本地的地址，可以是绝对路径，也可以是相对路径。如果路径是一个目录，它将递归复制。在这种情况下，如果路径使用"/"来结尾，则只复制目录里的内容，如果没有使用"/"来结尾，则包含目录在内的整个内容全部复制，类似于rsync。 
- validate ：The validation command to run before copying into place. The path to the file to validate is passed in via '%s' which must be present as in the visudo example below.

```bash
$ ansible "*" -m copy -a "src=/tmp/a.txt dest=/home/l.k"
192.168.2.13 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": true,
    "checksum": "3eea65eb24d01e6db7615b930d9536d3bed4e491",
    "dest": "/home/l.k",
    "gid": 0,
    "group": "root",
    "md5sum": "39dbc49ac266cd2de8d852ff6ad13898",
    "mode": "0644",
    "owner": "root",
    "secontext": "unconfined_u:object_r:user_home_dir_t:s0",
    "size": 838,
    "src": "/root/.ansible/tmp/ansible-tmp-1585229584.47-184045450866056/source",
    "state": "file",
    "uid": 0
}
$ ls /home/l.k  -l
-rw-r--r--. 1 root root 838 Mar 26 09:33 /home/l.k

```



##### filesystem

This module creates a filesystem.

Requirements

The below requirements are needed on the host that executes this module.

- Uses tools related to the *fstype* (`mkfs`) and `blkid` command. When *resizefs* is enabled, `blockdev` command is required too.

块设备上创建文件系统
选项：
**dev**：目标块设备
**force**：在一个已有文件系统的设备上强制创建
**fstype**：文件系统的类型
**opts**：传递给mkfs命令的选项

```bash
#Filesystem模块只能对整个块设备操作，不能分区
$ ansible test_hosts -m filesystem -a 'dev=/dev/sdb fstype=ext4'
192.168.31.110 | SUCCESS => {
    "changed": true
}
[root@Ansible ~]# ansible test_hosts -a 'fdisk -l /dev/sdb'
192.168.31.110 | SUCCESS | rc=0 >>
Disk /dev/sdb: 107.4 GB, 107374182400 bytes
255 heads, 63 sectors/track, 13054 cylinders
Units = cylinders of 16065 * 512 = 8225280 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x00000000
[root@Ansible ~]#
```

如果需要像平时样对块设备进行分区操作，则需要编写脚本让Ansible执行。这里用到了copy和command模块.

```bash
#如果需要像平时样对块设备进行分区操作，则需要编写脚本让Ansible执行。这里用到了copy和command模块
[root@Ansible ~]# cat fdisk.sh 
#!/bin/bash
#make partition
fdisk /dev/sdb <<EOF
n
p
1
+10G
p
w
EOF
[root@Ansible ~]#
[root@Ansible ~]# ansible test_hosts -m copy -a 'src=/root/fdisk.sh dest=/tmp/'
192.168.31.110 | SUCCESS => {
    "changed": true, 
    "checksum": "f265923f0a4578e36c82418df1a068a55867c1b1", 
    "dest": "/tmp/fdisk.sh", 
    "gid": 0, 
    "group": "root", 
    "md5sum": "87c9e0b074e4828b37d6e334d1cf7ce1", 
    "mode": "0644", 
    "owner": "root", 
    "size": 71, 
    "src": "/root/.ansible/tmp/ansible-tmp-1486437971.22-237961786906240/source", 
    "state": "file", 
    "uid": 0
}
[root@Ansible ~]# ansible test_hosts -m command -a 'sh /tmp/fdisk.sh'
192.168.31.110 | SUCCESS | rc=0 >>
...
Disk /dev/sdb: 107.4 GB, 107374182400 bytes
255 heads, 63 sectors/track, 13054 cylinders
Units = cylinders of 16065 * 512 = 8225280 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x166637ee
   Device Boot      Start         End      Blocks   Id  System
/dev/sdb1               1        1306    10490413+  83  Linux
[root@Ansible ~]# ansible test_hosts -a 'mkfs.ext4 /dev/sdb1'
#格式化分区
```

end

##### mount

[https://www.yfshare.vip/2017/04/05/Ansible%E5%B8%B8%E7%94%A8%E6%A8%A1%E5%9D%97/#FileSystem](https://www.yfshare.vip/2017/04/05/Ansible常用模块/#FileSystem)

##### yum

使用yum包管理器来管理软件包
选项：
**conf_file**：yum的配置文件
**disable_gpg_check**：关闭gpg_check
**disablerepo**：不启用某个源
**enablerepo**：启用某个源
**list**：查看yum列表
**name**：要进行操作的软件包名字，也可以传递一个url或者一个本地的rpm包的路径
**state**：状态(present/installed/absent/removed/latest)

>  `removed==absent`, `installed==present`, these are accepted as aliases

好用

```yaml
## absent/removed
- name: Remove the Apache package
  yum:
    name: httpd
    state: absent
- name: Remove docker
  yum:
    name: ["docker-ce", "containerd.io", "runc"]
    state: absent
    
##
- name: List ansible packages and register result to print with debug later.
  yum:
    list: ansible
  register: result

- name: Install package with multiple repos enabled
  yum:
    name: sos
    enablerepo: "epel,ol7_latest"

- name: Install package with multiple repos disabled
  yum:
    name: sos
    disablerepo: "epel,ol7_latest"

- name: Install a list of packages
  yum:
    name:
      - nginx
      - postgresql
      - postgresql-server
    state: present

- name: Download the nginx package but do not install it
  yum:
    name:
      - nginx
    state: latest
    download_only: true
    
# 源代码
if state in ['installed', 'present']:
    if disable_gpg_check:
        yum_basecmd.append('--nogpgcheck')
    res = install(module, pkgs, repoq, yum_basecmd, conf_file, en_repos, dis_repos)
elif state in ['removed', 'absent']:
    res = remove(module, pkgs, repoq, yum_basecmd, conf_file, en_repos, dis_repos)
elif state == 'latest':
    if disable_gpg_check:
        yum_basecmd.append('--nogpgcheck')
    res = latest(module, pkgs, repoq, yum_basecmd, conf_file, en_repos, dis_repos)
else:
    # should be caught by AnsibleModule argument_spec
    module.fail_json(msg="we should never get here unless this all"
            " failed", changed=False, results='', errors='unexpected state')

return res
```

end

##### services

用于管理服务
常用选项：
**arguments**：为命令提供一些附加参数
**enabled**：是否开机启动，选项 yes|no
**name**：必选项，服务名称
**pattern**：定义一个模式，如果通过status指令来查看服务状态时，没有响应，它会通过ps命令在进程中根据该模式进行查找，如果匹配到，则认为该服务依然运行
**runlevel**：运行级别
**sleep**：如果执行了restarted，则在stop和start之间等待几秒钟
**state**：对当前服务执行启动/停止/重启/重新加载等操作(started/stopped/restarted/reloaded)

```yaml
$ ansible "*.82" -m service -a "name=sshd state=started"
21.48.5.82 | SUCCESS => {
    "changed": false, 
    "name": "sshd", 
    "state": "started", 
    "status": {
        ...
    }
}

##
- name: Enable service httpd, and not touch the state
  service:
    name: httpd
    enabled: yes

## pattern
- name: Start service foo, based on running process /usr/bin/foo
  service:
    name: foo
    pattern: /usr/bin/foo
    state: started

- name: Restart network service for interface eth0
  service:
    name: network
    state: restarted
    args: eth0
```

end

##### 略，见文章

[https://www.yfshare.vip/2017/04/05/Ansible%E5%B8%B8%E7%94%A8%E6%A8%A1%E5%9D%97/#script%E6%A8%A1%E5%9D%97](https://www.yfshare.vip/2017/04/05/Ansible常用模块/#script模块)

##### setup

其实类似saltstack的grains静态信息收集，收集一些主机硬件信息或者以及其他如fqdn等等

啧啧，一下就说到点子上了。

```bash
$ ansible '*' -m setup  -a 'filter=ansible_memory_mb'
192.168.2.13 | SUCCESS => {
    "ansible_facts": {
        "ansible_memory_mb": {
            "nocache": {
                "free": 4101,
                "used": 1674
            },
            "real": {
                "free": 2143,
                "total": 5775,
                "used": 3632
            },
            "swap": {
                "cached": 0,
                "free": 0,
                "total": 0,
                "used": 0
            }
        },
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false
}

```

end

##### loop: need ansible 2.5.

This error usually comes in ***Ansible\*** versions below 2.5. The ***loop\*** command is not available in ***Ansible\*** versions less than 2.5.

```bash
$ cat t2.yaml 
- hosts: localhost
  tasks:
  - name: add several users
    command: "echo {{ item }}"
    loop:
      - testuser1
      - testuser2
      
$ ansible-playbook -i hosts.test  ./t2.yaml 
ERROR! The field 'loop' is supposed to be a string type, however the incoming data structure is a <class 'ansible.parsing.yaml.objects.AnsibleSequence'>

The error appears to have been in '/tmp/lol/t2.yaml': line 3, column 5, but may
be elsewhere in the file depending on the exact syntax problem.

The offending line appears to be:

  tasks:
  - name: add several users
    ^ here

```

啧啧

If you are facing this error, you have two options to solve this error

- Upgrade Ansible version to 2.5+

- Use ***with_items\*** instead of ***loop\*** command in the script. The same code should work.

  ```bash
  $ cat ./t2.yaml
  - hosts: localhost
    tasks:
    - name: add several users
      command: "echo {{ item }}"
      with_items:
        - testuser1
        - testuser2
  
  $ ansible-playbook -i hosts.test  ./t2.yaml 
  
  PLAY ***************************************************************************
  
  TASK [setup] *******************************************************************
  ok: [localhost]
  
  TASK [add several users] *******************************************************
  changed: [localhost] => (item=testuser1)
  changed: [localhost] => (item=testuser2)
  
  PLAY RECAP *********************************************************************
  localhost                  : ok=2    changed=1    unreachable=0    failed=0   
  
  ```

  end



##### template

template模块用法和copy模块用法基本一致，它主要用于复制配置文件。可以按需求修改配置文件内容来复制模板到被控主机上。与`copy`模块类似，若目标文件的父目录不存在时，也会报错、

template 也是使用jinjia2模板语言。

{{    }}  ：用来装载表达式，比如变量、运算表达式、比较表达式等。

{%  %}  ：用来装载控制语句，比如 if 控制结构，for循环控制结构。

{#   #}  ：用来装载注释，模板文件被渲染后，注释不会包含在最终生成的文件中。

Ansible 使用 “{{ var }}” 来引用变量. 如果一个值以 “{” 开头, YAML 将认为它是一个字典, 所以我们必须引用它

总结下：利用jinjia2每次将变量和template模板，组合替换然后生成配置文件。

比如，写一个模板yaml来部署pod，就可以利用jinjia2来替换image name.

由于Ansible是使用`Jinja2`来编写`template`模版的，所以需要使用`*.j2`为文件后缀

```bash
$ cat pod.j2
apiVersion: v1
kind: Pod
metadata:
  name: {{POD_NAME}}
spec:
  containers:
  - name: {{ CONTAINER_NAME }}
    image: {{ IMAGE_NAME }}
    
## 怎么再只能点，我想让CONTAINER_NAME未设置时，默认等于POD_NAME
$ cat  pod_pb.yml
- name: Test Template
  hosts: test
  vars:
    POD_NAME: "test_pod"
    IMAGE_NAME: "nginx:1.12"
    CONTAINER_NAME: "test"
  tasks:
    - name: pod yaml
      template:
        src: pod.j2
        dest: /tmp/pod_pb.yaml
    - fail: msg="Container Name Undefined"
      when: CONTAINER_NAME is undefined

$ ansible-playbook  pod_pb.yml

PLAY [Test Template] **********************************************************************

TASK [Gathering Facts] ********************************************************************
ok: [192.168.2.13]

TASK [pod yaml] ***************************************************************************
changed: [192.168.2.13]

TASK [fail] *******************************************************************************
skipping: [192.168.2.13]

PLAY RECAP ********************************************************************************
192.168.2.13               : ok=2    changed=1    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0

$ cat /tmp/pod_pb.yaml
apiVersion: v1
kind: Pod
metadata:
  name: test_pod
spec:
  containers:
  - name: test
    image: nginx:1.12

```



#### playbook

最简单的例子：

```bash

$ cat test.yml
- hosts: test
  remote_user: root
  # 这下面可以接好多个task
  tasks:
  - name: test playbook
    ping:
$ ansible-playbook --check test.yml

PLAY [test] *******************************************************************************

TASK [Gathering Facts] ********************************************************************
ok: [192.168.2.13]

TASK [test playbook] **********************************************************************
ok: [192.168.2.13]

PLAY RECAP ********************************************************************************
192.168.2.13               : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

```

##### register

在ansible的playbook中task之间的相互传递变量. 挺有用的结合条件判断做控制。

```yaml
## 官网例子
tasks:
  - command: /bin/false
    register: result
    ignore_errors: True

  - command: /bin/something
    when: result is failed

  # In older versions of ansible use ``success``, now both are valid but succeeded uses the correct tense.
  - command: /bin/something_else
    when: result is succeeded

  - command: /bin/still/something_else
    when: result is skipped

## 文件不存在时，创建文件
tasks:
     - name: Check that the somefile.conf exists
       stat:
         path: /etc/file.txt
       register: stat_result

     - name: Create the file, if it doesnt exist already
       file:
         path: /etc/file.txt
         state: touch
       when: stat_result.stat.exists == False 
```

end

##### handlers

handlers即当它的调用者执行成功时，才会触发handlers。

```yaml
### 避免nginx.conf没有修改时就触发nginx -s reload
---
- hosts: test70
  remote_user: root
  tasks:
  - name: Modify the configuration
    lineinfile:
      path=/etc/nginx/conf.d/test.zsythink.net.conf
      regexp="listen(.*) 8080(.*)"
      line="listen\1 8088\2"
      backrefs=yes
      backup=yes
    notify:
      restart nginx
 
  handlers:
  - name: restart nginx
    service:
      name=nginx
      state=restarted
      
      
 ## 检测文件不存在则创建

$ cat p1.yml 
- name: test handlers
  hosts: test
  tasks:
  - name: file state
    file: path=/tmp/a.ll
          state=file
    ignore_errors: True
    register: file_state
  - name: handler_file
    file: path=/tmp/a.ll
          state=touch
    ## failed 则说明不存在
    when: file_state is failed     
```

综上所述，上例中的play表示，如果"Modify the configuration"真正的修改了配置文件（实际的操作），那么则执行"restart nginx"任务，如果"Modify the configuration"并没有进行任何实际的改动，则不执行"restart nginx" ，通常来说，任务执行后如果做出了实际的操作，任务执行后的状态为changed（前文中解释过changed状态，此处不再赘述），所以，任务执行后的状态为changed则会执行对应的handlers，这就是handlers的作用，

默认情况下，所有task执行完毕后，才会执行各个handler，并不是执行完某个task后，立即执行对应的handler，如果你想要在执行完某些task以后立即执行对应的handler，则需要使用meta模块，示例如下

```yaml
---
- hosts: test70
  remote_user: root
  tasks:
  - name: task1
    file: path=/testdir/testfile
          state=touch
    notify: handler1
  - name: task2
    file: path=/testdir/testfile2
          state=touch
    notify: handler2
 
  - meta: flush_handlers
 
  - name: task3
    file: path=/testdir/testfile3
          state=touch
    notify: handler3
 
  handlers:
  - name: handler1
    file: path=/testdir/ht1
          state=touch
  - name: handler2
    file: path=/testdir/ht2
          state=touch
  - name: handler3
    file: path=/testdir/ht3
          state=touch
```

如上例所示，我在task1与task2之后写入了一个任务，我并没有为这个任务指定name属性，这个任务使用meta模块，meta任务是一种特殊的任务，meta任务可以影响ansible的内部运行方式，上例中，meta任务的参数值为flush_handlers，"meta: flush_handlers"表示立即执行之前的task所对应handler，什么意思呢？意思就是，在当前meta任务之前，一共有两个任务，task1与task2，它们都有对应的handler，当执行完task1与task2以后，立即执行对应的handler，而不是像默认情况那样在所有任务都执行完毕以后才能执行各个handler，那么我们来实际运行一下上述剧本，运行结果如下。

#### role

Roles 的概念来自于这样的想法：通过 include 包含文件并将它们组合在一起，组织成一个简洁、可重用的抽象对象。这种方式可使你将注意力更多地放在大局上，只有在需要时才去深入了解细节。

Ad-Hoc 适用于临时命令的执行，Playbook 适合中小项目，而大项目一定要是有 Roles。Roles 不仅支持 Tasks 的集合，同时包括 var_files、tasks、handlers、meta、templates等。

> role下面还有加一个yaml，记录该roles对应的执行机器。
>
> --

```bash
$ tree ansible_playbooks/
ansible_playbooks/
└── roles  必须叫roles
    ├── dbsrvs -------------role1_name
    │   ├── defaults  ---------必须存在的目录，存放默认的变量，模板文件中的变量就是引用自这里。defaults中的变量优先级最低，通常我们可以临时指定变量来进行覆盖
    │   │   └── main.yml
    │   ├── files -------------ansible中unarchive、copy等模块会自动来这里找文件，从而我们不必写绝对路径，只需写文件名
    │   │   ├── mysql.tar.gz
    │   │   └── nginx.tar.gz
    │   ├── handlers -----------存放tasks中的notify指定的内容
    │   │   └── main.yml
    │   ├── meta
    │   ├── tasks --------------存放playbook的目录，其中main.yml是主入口文件，在main.yml中导入其他yml文件，要采用import_tasks关键字，include要弃用了
    │   │   ├── install.yml
    │   │   └── main.yml -------主入口文件
    │   ├── templates ----------存放模板文件。template模块会将模板文件中的变量替换为实际值，然后覆盖到客户机指定路径上
    │   │   └── nginx.conf.j2
    │   └── vars
    └── websrvs -------------role2名称
        ├── defaults
        │   └── main.yml
        ├── files
        │   ├── mysql.tar.gz
        │   └── nginx.tar.gz
        ├── handlers
        │   └── main.yml
        
        ├── meta
        ├── tasks
        │   ├── install.yml
        │   └── main.yml
        ├── templates
        │   └── nginx.conf.j2
        └── vars
```



##### 变量

vars and set_fact 的对比

https://zhuanlan.zhihu.com/p/149608103

##### 条件选择

```bash
## 这个是错的
- name: checking the file exists
     command: touch file.txt
     when: $(! -s /etc/file.txt)
```



##### 循环



#### 常用命令

```bash
  -f FORKS, --forks=FORKS
                        specify number of parallel processes to use
                        (default=5)
                        
   -C, --check           don't make any changes; instead, try to predict some
                        of the changes that may occur


ansible newbbs -f 20 -m shell -a "npm install pm2 -g"
ansible newbbs -m copy -a "src=/etc/profile.d/lib.sh dest=/etc/profile.d/lib.sh"

ansible-playbook -i /home/data/mihoyo-ansible-roles/hosts play.yml -e "hosts=plat-aliyun-sh-dev" --list-hosts


### 执行playbook
 ansible-playbook -i /tmp/dodo/hosts  /tmp/dodo/roles/logtail-agent/play.yml

$ cat /tmp/dodo/roles/logtail-agent/play.yml
 - hosts: all
  roles:
   - logtail-agent
```



#### 读取本地文件作为变量

```bash
## 没用... 我想读client端的文件
- name: Include vars of stuff.yaml into the 'stuff' variable (2.2).
  include_vars:
    file: stuff.yaml
    name: stuff
```



#### 自己写的第一个playboook

https://www.ityoudao.cn/posts/ansible-modules-replace/

https://blog.csdn.net/VIP099/article/details/105780270

有个小坑：

基础，执行复杂playbook时，需要在roles目录下，调用playbook_name.单文件的playbook，直接运行就行了

* 简单playbook执行就行了：`ansible-playbook /srv/playbook/kafka.yml`

  这个类似，使用ansible在每个机器上执行脚本

  `ansible pts -m script -a kafka_disable.py`

* 复杂playbook执行

  ```bash
  $ ansible-playbook -i /home/data/hosts play.yml -e "hosts=plat-aliyun-sh-dev" 
  ## 关键是这个play.yml
  $ cat /etc/ansible/roles/plat-jaeger-agent/play.yml
  - hosts: "{{ hosts }}"
    roles:
     - plat-jaeger-agent
  
  ```

  

ansible自带的replace写正则比shell的sed写舒服多了

```bash
$ cat  logtail-agent/tasks/config.yml
---
- name: config ilogtaild
  shell: sed -i  s"/cpu_usage_limit\(.*\):\(.*\),/cpu_usage_limit\1:\ {{cpu_usage_limit}},/" /tmp/ilogtail_config.json
  when: cpu_usage_limit
  ignore_errors: true

## ansible中的变量引用时还有格式 前面不能用空格...
## so， 需要转义一下前面的空格... 我承认我有堵的成分...
- name: config ilogtaild
  shell: sed -i  s"/cpu_usage_limit\(.*\):\(.*\),/cpu_usage_limit\1: {{cpu_usage_limit}},/" /tmp/ilogtail_config.json
                                                                   ^ here
We could be wrong, but this one looks like it might be an issue with
missing quotes.  Always quote template expression brackets when they
start a value. For instance:

    with_items:
      - {{ foo }}

Should be written as:

    with_items:
      - "{{ foo }}"

## ansible shell replace model
  replace:
    path: "{{ ilogtail_config }}"
    regexp: 'buffer_file_size(.*):(.*)'
    replace: 'buffer_file_size\1: {{buffer_file_size}},'
```

end



```bash
[root@plat-op-01 dodo]# tree
.
├── hosts
└── roles
    └── logtail-agent
        ├── defaults
        │   └── main.yml
        ├── handlers
        │   └── main.yml
        ├── play.retry
        ├── play.yml
        └── tasks
            ├── config.yml
            ├── install.yml
            └── main.yml

5 directories, 8 files

## 看目录
$ pwd
/tmp/dodo/roles
## 执行
$ ansible-playbook  -i /tmp/dodo/hosts  /tmp/dodo/roles/logtail-agent/play.yml

## 调用playbook的总入口
$  cat roles/logtail-agent/play.yml
- hosts: all
  roles:
   - logtail-agent

## 任务总入口
$ cat roles/logtail-agent/tasks/main.yml
---
- import_tasks: install.yml
- import_tasks: config.yml

## install sub mission
$ cat roles/logtail-agent/tasks/install.yml
---
- name: download install script
  get_url:
    url: "{{ install_script_url }}"
    dest: /tmp/{{ script_name }}
    #dest: /tmp/logtail.sh
    mode: 0755
    force: yes
  register: get_script

# install ilogtaild and add service for management by chkconfig.
- name: install ilogtaild
  shell: /tmp/{{ script_name }} install auto
  register: result
  failed_when: false

- name: verify ilogtaild
  fail:
    msg: "{{ result.stdout_lines }}"
  failed_when: result.rc != 0

# config
$ cat roles/logtail-agent/tasks/config.yml
---
## 用自带模块replace ，谢谢~~
- name: config ilogtaild
  shell: shell: sed -i  s"/cpu_usage_limit\(.*\):\(.*\),/cpu_usage_limit\1:\ {{cpu_usage_limit}},/"  {{ logtail_config }}
  when: cpu_usage_limit
  ignore_errors: true
  register: r
  notify:
    restart i
## 上面写的不行，因为使用notify会导致没改一个配置项就会重启一次服务...
$ cat roles/logtail-agent/tasks/config.yml
---
- name: config cpu usage limit
  replace:
    path: "{{ ilogtail_config }}"
    ## 比shell的sed写正则舒服多了
    regexp: 'cpu_usage_limit(.*):(.*)'
    replace: 'cpu_usage_limit\1: {{cpu_usage_limit}},'
  when: cpu_usage_limit

- name: config memory usage limit
  replace:
    path: "{{ ilogtail_config }}"
    regexp: 'mem_usage_limit(.*):(.*)'
    replace: 'mem_usage_limit\1: {{mem_usage_limit}},'
  when: mem_usage_limit

- name: config max_bytes_per_sec
  replace:
    path: "{{ ilogtail_config }}"
    regexp: 'max_bytes_per_sec(.*):(.*)'
    replace: 'max_bytes_per_sec\1: {{max_bytes_per_sec}},'
  when: max_bytes_per_sec

- name: config buffer_file_num
  replace:
    path: "{{ ilogtail_config }}"
    regexp: 'buffer_file_num(.*):(.*)'
    replace: 'buffer_file_num\1: {{buffer_file_num}},'
  when: buffer_file_num

- name: config buffer_file_size
  replace:
    path: "{{ ilogtail_config }}"
    regexp: 'buffer_file_size(.*):(.*)'
    replace: 'buffer_file_size\1: {{buffer_file_size}},'
  when: buffer_file_size

- name: restart ilogtaild
  shell: /etc/init.d/ilogtaild restart
  when: (cpu_usage_limit) or (mem_usage_limit) or
        (max_bytes_per_sec) or (buffer_file_num) or
        (buffer_file_size)


## handlers
## 任务执行后的状态为changed则会执行对应的handlers。
$ cat roles/logtail-agent/handlers/main.yml
---
- name: restart i
  shell: echo 'ok' >> /tmp/a.txt

## 默认变量
$ cat logtail-agent/defaults/main.yml
install_script_url: http://logtail-release-cn-hangzhou.oss-cn-hangzhou.aliyuncs.com/linux64/logtail.sh
script_name: logtail.sh
cpu_usage_limit:
```

end



#### 第二个playbook

变量命名规则

我不知道...这个规则--

变量名包含字母、数字以及下划线，具体示例如下：

合法的命名：foo_port、foo5等

非法的命名：foo-port、foo port、foo.port、12等

而且学到了使用debug模块，可以打印输出信息

```bash
## 会输出整个stat_usr_sbin_dhclient_script对象的全部属性00
- name: debug
      debug:
        msg: "{{stat_usr_sbin_dhclient_script}}"
```

下面是错误实例...

```bash
$ cat ./etc_resolve.yml
---
- hosts: dns
  tasks:
    - name: read /etc/resolv.conf
      shell: cat /etc/resolv.conf
      register: etc_resolv

    - name: replace search
      when: etc_resolv.stdout.find('search') != -1
      replace:
        path: /etc/resolv.conf
        regexp: 'search (.*)'
        replace: 'search \1 mhy.link'

    - name: replace search
      when: etc_resolv.stdout.find('search') == -1
      lineinfile:
        dest: /etc/resolv.conf
        line: 'search mhy.link'


    - name: check usr_sbin_dhclient-script
      stat:
        path: /usr/sbin/dhclient-script
      ## 用短横线报错--
      register: stat_usr_sbin_dhclient_script


    - name: check sbin_dhclient-script
      stat:
        path: /sbin/dhclient-script
      register: stat_sbin_dhclient_script

    - name: debug
      debug:
        msg: "{{stat_usr_sbin_dhclient_script}}"
   # - name: edit usr_sbin_dhclient-script
   #   lineinfile:
   #     dest: /usr/sbin/dhclient-script
   #     insertafter: 'SAVEDIR=(.*)'
   #     line: 'SEARCH="mhy.link"'
   #   when: stat_usr_sbin_dhclient-script.stat.exists

   # - name: edit sbin_dhclient-script
   #   lineinfile:
   #     dest: /sbin/dhclient-script
   #     insertafter: 'SAVEDIR=(.*)'
   #     line: 'SEARCH="mhy.link"'
   #   when: stat_sbin_dhclient-script.stat.exists

##
短横线声明报错
  register: stat_sbin_dhclient-script
##
fatal: [10.14.251.134]: FAILED! => {"msg": "The task includes an option with an undefined variable. The error was: Unable to look up a name or access an attribute in template string ({{stat_sbin_dhclient-script}}).\nMake sure your variable name does not contain invalid characters like '-': unsupported operand type(s) for -: 'StrictUndefined' and 'StrictUndefined'\n\nThe error appears to have been in '/tmp/dodo/etc_resolve.yml': line 36, column 7, but may\nbe elsewhere in the file depending on the exact syntax problem.\n\nThe offending line appears to be:\n\n        msg: \"{{stat_usr_sbin_dhclient_script}}\"\n    - name: debug2\n      ^ here\n"}
        to retry, use: --limit @/tmp/dodo/etc_resolve.retry

PLAY RECAP *******************************************************************************************************************************************************************
10.14.251.134              : ok=6    changed=2    unreachable=0    failed=1


###完整版
---
- hosts: dns
  tasks:
    - name: read /etc/resolv.conf
      shell: cat /etc/resolv.conf
      register: etc_resolv

    - name: replace search
      when: etc_resolv.stdout.find('search') != -1
      replace:
        path: /etc/resolv.conf
        regexp: 'search (.*)'
        replace: 'search \1 mhy.link'

    - name: add search
      when: etc_resolv.stdout.find('search') == -1
      lineinfile:
        dest: /etc/resolv.conf
        line: 'search mhy.link'


    - name: check usr_sbin_dhclient-script
      stat:
        path: /usr/sbin/dhclient-script
      register: stat_usr_sbin_dhclient_script


    - name: check sbin_dhclient-script
      stat:
        path: /sbin/dhclient-script
      register: stat_sbin_dhclient_script

    - name: edit usr_sbin_dhclient_script
      lineinfile:
        dest: /usr/sbin/dhclient-script
        insertafter: 'SAVEDIR=(.*)'
        line: 'SEARCH="mhy.link"'
      when: stat_usr_sbin_dhclient_script.stat.exists

    - name: edit sbin_dhclient_script
      lineinfile:
        dest: /sbin/dhclient-script
        insertafter: 'SAVEDIR=(.*)'
        line: 'SEARCH="mhy.link"'

```

#### 1000个机器运行ansible

优化加速

##### 关闭gather

默认情况下`ansible`会使用`facts`来检查目标服务器的环境变量，如果没有必要使用目标服务器的环境（如：网卡IP，主机名等），可以关闭这个功能，即设置

```bash
避免获取Gathering Facts
---
- hosts: all
  gather_facts: False
```

##### 并发和异步

Ansible默认是**同步阻塞模式**，它会等待所有的机器都执行完毕才会在前台返回。Ansible可以采取异步执行模式。异步模式下，Ansible会将节点的任务丢在后台，每台被控制的机器都有一个job_id，Ansible会根据这个job_id去轮训该机器上任务的执行情况，例如某机器上此任务中的某一个阶段是否完成，是否进入下一个阶段等。即使任务早就结束了，但只有轮训检查到任务结束后才认为该job结束。Ansible可以指定任务检查的时间间隔，默认是10秒。除非指定任务检查的间隔为0，否则会等待所有任务都完成后，Ansible端才会释放占用的shell。如果指定时间间隔为0，则Ansible会立即返回(至少得连接上目标主机，任务发布成功之后立即返回)，并不会去检查它的任务进度。

有以下的一些场景你是需要使用到ansible的polling特性的：

- 你有一个task需要运行很长的时间,这个task很可能会达到timeout.
- 你有一个任务需要在大量的机器上面运行
- 你有一个任务是不需要等待它完成的

```bash
---
- hosts: all
  gather_facts: False
  serial: 100
  tasks:
    - name: ping
      ping:
      async: 300
      ## 不等待校验，只显示changed...
      ## 这个设成0不太好啊，看不到执行结果--
      poll: 0


TASK [ping] ******************************************************************************************************************************************************************
changed: [10.14.41.212]
changed: [10.14.41.160]
changed: [10.14.41.166]
changed: [10.14.41.161]
changed: [10.14.41.167]
changed: [10.14.41.164]
changed: [10.14.41.165]
changed: [10.12.0.229]
changed: [10.12.0.228]
changed: [10.12.0.221]

PLAY [all] *********************
```



##### serial

控制滚动执行个数，即每次在多少个机器上执行，也可以是写百分比。

比如：`serial 10%`

```bash

---
- hosts: all
  gather_facts: False
  serial: 10
  tasks:
    - name: ping
      ping:
      async: 300
      poll: 3

---
PLAY [all] *******************************************************************************************************************************************************************

TASK [ping] ******************************************************************************************************************************************************************
ok: [10.10.54.184]
ok: [10.14.28.140]
ok: [10.14.30.158]
ok: [10.10.54.183]
ok: [10.14.30.159]
ok: [10.10.54.182]
ok: [10.14.28.147]
ok: [10.14.33.253]
ok: [10.14.33.252]
ok: [10.17.7.169]

PLAY [all] *****************************
```

