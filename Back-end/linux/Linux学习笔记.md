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




