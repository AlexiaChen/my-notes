# Linux学习笔记

## Linux有用的命令

![[Pasted image 20220706211521.png]]

[15 Tips On How to Use 'Curl' Command in Linux (tecmint.com)](https://www.tecmint.com/linux-curl-command-examples/)


对于自签证书，用curl访问https自签网址接口, 需要加入一个`-k`参数

```bash
curl --ssl -X 'GET' -k 'https://localhost:5000/api/invitecode' -H 'accept: application/json' -H '  
X-Api-Key-Id: LocalGabby' -H 'X-Api-Key: LocalGabbyAPIKey'  
{"code":201,"message":"Generate captcha Success","data":{"invitecode":"bF8hAH"}}
```

自签证书一般不靠谱，生产环境推荐千万不要用。仅仅可以本地测试，其实现在大多数浏览器都默认禁用了自签网址的访问，需要费好大劲才可以调整，Edge浏览器访问https网址，它会自动安装自签证书。


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

### 删除整个硬盘分区

比如时硬盘`/dev/sdb`

`sudo fdisk /dev/sdb`

进入交互界面，然后键入`d` 就可以选择分区了，一个一个删除。最后键入`w` 保存。

然后格式化为`ext4` 

```bash
sudo mkfs -t ext4 -c /dev/sdb
```

### 扩展整个逻辑卷

在ubuntu中，LVM的逻辑卷很有用。但是默认安装ubuntu server这个的硬盘的逻辑卷不会全部使用完整个分区。比如

```bash
mathxh@mathxh-server:~/Hi-mina/models$ df -h  
Filesystem Size Used Avail Use% Mounted on  
tmpfs 3.2G 1.8M 3.2G 1% /run  
/dev/mapper/ubuntu--vg-ubuntu--lv 98G 66G 27G 72% /  
tmpfs 16G 0 16G 0% /dev/shm  
tmpfs 5.0M 0 5.0M 0% /run/lock  
/dev/sda2 2.0G 340M 1.5G 19% /boot  
/dev/sda1 1.1G 6.1M 1.1G 1% /boot/efi  
tmpfs 3.2G 8.0K 3.2G 1% /run/user/1000
mathxh@mathxh-server:~$ sudo lsblk  
[sudo] password for mathxh:  
NAME MAJ:MIN RM SIZE RO TYPE MOUNTPOINTS  
loop0 7:0 0 63.3M 1 loop /snap/core20/1879  
loop2 7:2 0 111.9M 1 loop /snap/lxd/24322  
loop3 7:3 0 49.8M 1 loop /snap/snapd/18357  
loop4 7:4 0 53.2M 1 loop /snap/snapd/19122  
loop5 7:5 0 63.5M 1 loop /snap/core20/1891  
sda 8:0 0 931.5G 0 disk  
├─sda1 8:1 0 1G 0 part /boot/efi  
├─sda2 8:2 0 2G 0 part /boot  
└─sda3 8:3 0 928.5G 0 part  
   └─ubuntu--vg-ubuntu--lv 253:0 0 100G 0 lvm /  
sdb 8:16 0 465.8G 0 disk
```

以上就是sda3的分区，对于逻辑卷只使用了100G，还剩900多G没用。

```bash
mathxh@mathxh-server:~$ sudo vgdisplay  
--- Volume group ---  
VG Name ubuntu-vg  
System ID  
Format lvm2  
Metadata Areas 1  
Metadata Sequence No 2  
VG Access read/write  
VG Status resizable  
MAX LV 0  
Cur LV 1  
Open LV 1  
Max PV 0  
Cur PV 1  
Act PV 1  
VG Size <928.46 GiB  
PE Size 4.00 MiB  
Total PE 237685  
Alloc PE / Size 25600 / 100.00 GiB  
Free PE / Size 212085 / <828.46 GiB  
VG UUID H760ZR-dqT2-nAV2-6sDa-kDEt-dE0L-XCA0Up
```

以上就可以看到逻辑卷LVM这种格式，只分配了100G，就是Alloc PE那里。而还剩800多G空闲，Free PE那里。

然后再来查看逻辑卷的路径:

```bash
mathxh@mathxh-server:~$ sudo lvdisplay  
--- Logical volume ---  
LV Path /dev/ubuntu-vg/ubuntu-lv  
LV Name ubuntu-lv  
VG Name ubuntu-vg  
LV UUID tghkoK-lPh6-k8CZ-abQY-op4D-uWqN-DqhdYz  
LV Write Access read/write  
LV Creation host, time ubuntu-server, 2023-05-14 05:04:42 +0000  
LV Status available  
# open 1  
LV Size 100.00 GiB  
Current LE 25600  
Segments 1  
Allocation inherit  
Read ahead sectors auto  
- currently set to 256  
Block device 253:0
```

注意LV Path那项。然后我们通过这个Path扩展逻辑卷，直接分配完剩下的空闲区域

```bash
mathxh@mathxh-server:~$ sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv  
Size of logical volume ubuntu-vg/ubuntu-lv changed from 100.00 GiB (25600 extents) to <928.46 GiB (237685 extents).  
Logical volume ubuntu-vg/ubuntu-lv successfully resized.
mathxh@mathxh-server:~$ sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv  
resize2fs 1.46.5 (30-Dec-2021)  
Filesystem at /dev/mapper/ubuntu--vg-ubuntu--lv is mounted on /; on-line resizing required  
old_desc_blocks = 13, new_desc_blocks = 117  
The filesystem on /dev/mapper/ubuntu--vg-ubuntu--lv is now 243389440 (4k) blocks long.
```

以上第二个命令resize2fs的参数是通过df -h看到的。与第一个命令不一样。

最后你就可以通过`dh -h` 看到，你想要的结果了

```bash
mathxh@mathxh-server:~$ df -h  
Filesystem Size Used Avail Use% Mounted on  
tmpfs 3.2G 1.8M 3.2G 1% /run  
/dev/mapper/ubuntu--vg-ubuntu--lv 914G 66G 810G 8% /  
tmpfs 16G 0 16G 0% /dev/shm  
tmpfs 5.0M 0 5.0M 0% /run/lock  
/dev/sda2 2.0G 340M 1.5G 19% /boot  
/dev/sda1 1.1G 6.1M 1.1G 1% /boot/efi  
tmpfs 3.2G 8.0K 3.2G 1% /run/user/1000
```

可以看到，/dev/mapper/ubuntu--vg-ubuntu--lv 变大了。


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

下面这个是既保留控制台的输出，同时也定向到日志文件里。而上面那个是只定向到文件里

```bash
cmd_line 2>&1 | tee node.log &
```

References:

- https://unix.stackexchange.com/questions/74520/can-i-redirect-output-to-a-log-file-and-background-a-process-at-the-same-time

#### 找到特定进程中的环境变量

```bash
ps -ef | grep <name>

cat /proc/<pid>/environ | tr '\0' '\n'
# or
strings /proc/<pid>/environ

```

#### 查看CPU和GPU信息

```bash
# cpu
cat /proc/cpuinfo 
# gpu
sudo lshw -C display
```

#### 安装ubuntu server

用这个https://rufus.ie/en/  把iso镜像烧录进USB stick中，选择MBR。然后启动的时候，就一直默认选择下一步就行，server的安装都是控制台安装，没有GUI，

#### 增加锁定内存物理页大小的限制

一般来说我们通过`ulimit -l` 查看到的系统限制了锁定物理页的大小，也就是内存的页最大允许多大被锁定在物理内存里面，不给换出到磁盘。我们要把其调整到`unlimited` 这样，一些需要大内存的程序，或者安全相关的程序才可以比较好的工作。

[Use prlimit set RLIMIT_MEMLOCK to 1G or bigger size but it did not work · Issue #902 · jedisct1/libsodium (github.com)](https://github.com/jedisct1/libsodium/issues/902)


```bash
sudo sh -c "ulimit -l unlimited && exec su $USER"
```

以上命令好像不行，即使重启。

打开`/etc/security/limits.conf` , 然后新增以下

```txt
<user-name>               hard    memlock         unlimited
<user-name>               soft    memlock         unlimited
```

打开 `/etc/pam.d/common-session` ，然后增加:

```txt
session required pam_limits.so
```

然后保存这些修改，重启机器，就可以生效了。然后就可以看到`ulimit -l` 变成没有限制了。


## 安装Nvidia的GPU Driver

一般用默认的就行:

```bash
# 会显示推荐的驱动
ubuntu-drivers devices
# 按照推荐的驱动安装，注意别安装-open后缀的
sudo ubuntu-drivers autoinstall
# or
sudo apt install nvidia-driver-510
sudo reboot
```

然后，不出意外，以下命令就可以显示NVidia信息了:

```bash
nvidia-smi
```

当然，你也可以手动安装，[Linux AMD64 Display Driver Archive | NVIDIA](https://www.nvidia.com/en-us/drivers/unix/linux-amd64-display-archive/)  下载匹配的驱动，sudo bash .run的文件就行了


当然，也有可能遇到[nvidia-smi no devices were found_Nightmare004的博客-CSDN博客](https://blog.csdn.net/qq_39942341/article/details/128346642)  其实就是驱动安装错误了，这个就需要不停地折腾了。 [nvidia-smi 输出“No devices were found_缄默0603的博客-CSDN博客](https://blog.csdn.net/weixin_58045467/article/details/129402251)

## 安装CUDA Toolkit

[Ubuntu系统nvidia-cuda-toolkit安装 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/625219352)