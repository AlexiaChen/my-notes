# Linux学习笔记

## Linux有用的命令

![[Pasted image 20220706211521.png]]

[15 Tips On How to Use 'Curl' Command in Linux (tecmint.com)](https://www.tecmint.com/linux-curl-command-examples/)


## Linux的文件权限

![[Pasted image 20220706214815.png]]

𝐎𝐰𝐧𝐞𝐫𝐬𝐡𝐢𝐩 

- Owner: the owner is the user who created the file or directory 
- Group: a group can have multiple users. All users in the group have the same permissions to access the file or directory 
- Other: other means those who are not owners or members of the group

𝐏𝐞𝐫𝐦𝐢𝐬𝐬𝐢𝐨𝐧 There are three types of permissions.

- Read (r): the read permission allows the user to read a file.
- Write (w): the write permission allows the user to change the content of the file.
- Execute (x): the execute permission allows a file to be executed.

## Linux系统调用

[The Fascinating World of Linux System Calls – Sysdig](https://sysdig.com/blog/fascinating-world-linux-system-calls/)

[Efficient File Copying On Linux (eklitzke.org)](https://eklitzke.org/efficient-file-copying-on-linux)

[Using select(2) | Aivars Kalvāns](https://aivarsk.com/2017/04/06/select/)

## Linux ELF

[Generating executable files from scratch · julianneswinoga/yabfc Wiki (github.com)](https://github.com/julianneswinoga/yabfc/wiki/Generating-executable-files-from-scratch)

## Linux网络

[IPv4 route lookup on Linux (bernat.ch)](https://vincent.bernat.ch/en/blog/2017-ipv4-route-lookup-linux)

## Linux底层图形机制

主要是x server相关 [BetterOS.org : an attempt to make computer machines run better](http://betteros.org/tut/graphics1.php)


### 调整分区大小

先用sudo fdisk -l 查看当前某个硬盘有几个分区，比如分区有

- /dev/sda1
- /dev/sda2

比如只有一个sda硬盘。

为了调整sda2分区的大小，先用sudo fdisk 进入`/dev/sda`中:

```bash
sudo fdisk /dev/sda
```

然后会进入一个交互式cli界面，输入m查看命令帮助，

输入p查看当前分区，如果要扩展sda2分区，那么需要命令d，删除sda2分区。

然后输入n ，在输入一个p命令表示主分区，重建一个新的分区，新分区会要你输入新分区的大小，是通过sector的 start和end表示大小。比需要输入start 和 end的值，才可以确定新分区的大小，对于end的值一般可以 +/- N个G 或者MB等等，有提示。

然后输入保存命令w

最后退出fdisk的交互式界面。

校验新建的分区的数据一致性:

```bash
e2fsck -f /dev/sda2
```

然后你可以看到sda2变大了或者变小了，接下来就是用一个命令适配让linux文件系统本身适配到这个分区的大小。

```bash
sudo resize2fs /dev/sda2
```


#### Linux的管道性能

- [How fast are Linux pipes anyway? (mazzo.li)](https://mazzo.li/posts/fast-pipes.html)
- [cat - pipe and redirection speed, `pv` and UUOC - Unix & Linux Stack Exchange](https://unix.stackexchange.com/questions/607737/pipe-and-redirection-speed-pv-and-uuoc/607745#607745)
- [Optimizing Linux Pipes | Hackaday](https://hackaday.com/2022/06/03/optimizing-linux-pipes/)


### lsof命令详解和例子


- 列出所有被打开的文件描述符 lsof | more
- 列出某个路径下被打开的文件描述符 lsof  \<dir\>
- 列出某个用户下被打开的文件描述符 lsof -u \<username\> | more
- 列出除了某个用户以外的所有用户下被打开的FD lsof -u ^\<username\>
- 列出所有打开的网络和Unix Domain的FD lsof -i -U
- 列出所有的打开的IPv4的网络连接的FD lsof -i 4, 同理 IPv6就是 lsof -i 6

![[Pasted image 20220816115240.png]]

- 列出某个进程打开的IPv4网络连接的FD  lsof -i 4 -a -p \<pid\>  同上
- 列出在某个端口下的TCP或UDP进程  lsof -i \<TCP | UDP\>:\<port\>, 也可以是端口范围

![[Pasted image 20220816115639.png]]

- 列出打开某个设备的所有FD  lsof \<device_name\>

![[Pasted image 20220816115755.png]]

- 列出打开NFS挂载点的进程和FD lsof -b \<nfs_mount_point\>
- 列出与应用进程名相关的被打开的FD  lsof -c \<binary_name\>

![[Pasted image 20220816120002.png]]

- 列出在某个IP下的Socket FD lsof -i@\<ip\>

![[Pasted image 20220816120131.png]]

- 列出某个进程下的打开的FD lsof -p \<pid\>

- 列出某个目录下打开的文件FD lsof +D \<dir\>

- 找出哪些进程打开了某个文件 ps -fp "$(lsof -t \<\file_path> | xargs echo)"


#### wget命令实例

语法

```bash
wget [options] [url]
```

- 下载一个文件并保存在当前工作目录

```bash
wget https://cdn.kernel.org/pub/linux/kernel/v4.x/linux-4.17.2.tar.xz
```

- 下载一个文件并另存为一个文件

```bash
wget -O latest-hugo.zip https://github.com/gohugoio/hugo/archive/master.zip
```

- 下载一个文件并存放到特定目录

```bash
wget -P /mnt/iso http://mirrors.mit.edu/centos/7/isos/x86_64/CentOS-7-x86_64-Minimal-1804.iso
```

- 下载限速

```bash
wget --limit-rate=1<m | k |g> https://dl.google.com/go/go1.10.3.linux-amd64.tar.gz
```

- 断点下载，适用于大文件

```bash
wget -c http://releases.ubuntu.com/18.04/ubuntu-18.04-live-server-amd64.iso
```

- 后台下载

```bash
wget -b https://download.opensuse.org/tumbleweed/iso/openSUSE-Tumbleweed-DVD-x86_64-Current.iso
```

- 修改User-Agent

因为有些时候，服务器会限定Agent，所以需要模拟浏览器之类的

```bash
wget --user-agent="Mozilla/5.0 (X11; Linux x86_64; rv:60.0) Gecko/20100101 Firefox/60.0" http://wget-forbidden.com/
```

- 同时下载多个文件

```bash
# txt文件里面是回车换行的一个URLs清单
wget -i \<file_name\>.txt
```

- 下载FTP

```bash
wget --ftp-user=FTP_USERNAME --ftp-password=FTP_PASSWORD ftp://ftp.example.com/filename.tar.gz
```

- 备份一个网页镜像

```bash
wget -m -k -p https://example.com
```


- 下载https的文件

```bash
wget --no-check-certificate https://domain-with-invalid-ss.com
```

- 下载到stdout上，主要用于管道

```bash
wget -q -O - "http://wordpress.org/latest.tar.gz" | tar -xzf - -C /var/www
```


#### dmesg命令详解

dmesg命令可以查看kerne相关的消息日志（这些消息在内核的ring buffer中），这些消息包括硬件，设备驱动初始化，还有一些系统启动之初内核模块的消息，这个命令主要诊断硬件相关的错误，警告。

- `sudo dmesg -L` 显示全部彩色化的信息
- `sudo dmesg --follow` 实时监控
- `sudo dmesg | grep -i <memory | usb | tty | eth | sda>` 查看内存，USB，终端，网卡，硬盘等信息
- `sudo dmesg -H` 打开dmesg 的时间戳
- `sudo dmesg -T` 打开dmesg 的时间显示，可读性更高， 可以用`--time-format=<ctime | reltime | delta | notime | iso>`指定时间格式
- `sudo dmesg -f <kern | user | mail | daemon | auth | syslog | lpr | news>` 过滤显示，内核，用户消息，邮件消息，daemon消息。认证消息，打印机消息，网络消息
- `sudo dmesg -l <emerg | alert | crit | err | warn | notice | info | debug>` 过滤消息的级别


比如就可以用dmesg查看被kernel的OOM killer模块杀死的进程，一般内存占用过大，Linux内核会把它干掉，程序不知不觉消失了:

```bash
dmesg -T | grep -i "killed process"

grep -i "killed process" /var/log/messages
```


#### Linux命令行纠错和使用的项目

- [tldr-pages/tldr: 📚 Collaborative cheatsheets for console commands (github.com)](https://github.com/tldr-pages/tldr)
- [nvbn/thefuck: Magnificent app which corrects your previous console command. (github.com)](https://github.com/nvbn/thefuck)

#### cron任务调度

![[Pasted image 20220827214101.png]]


#### 标准错误和输出重定向

老是忘记，记录一下。其中2>&1的意思是把标准错误流重新定向到标准输出流的地址。   0是标准输入流，1是标准输出流，2是标准错误流 ，最后一个&符号，当然就是后台运行的意思了。

```bash
cmd_line > outputfile 2>&1 &
```

References:

- https://unix.stackexchange.com/questions/74520/can-i-redirect-output-to-a-log-file-and-background-a-process-at-the-same-time