# SSH学习笔记

## 基本用法

### 最基本远程登录

```bash
ssh user@node  -p [port]
```
默认端口是22，不输入port就是22。

然后输入密码就可以了，只要对端机器安装了SSH server，类似于:

```bash
sudo apt install openssh-server
```

### 免密登录

用到SSH的公私钥，用`ssh-keygen`生成公私钥对，把公钥添加到对端机器上：

```bash
ssh-copy-id user@node -p [port]
```

还可以这样：

```bash
ssh user@node -p [port] 'mkdir -p .ssh && cat >> .ssh/authorized_keys' < ~/.ssh/id_rsa.pub
```

### 配置别名

每次都输入 ssh user@node -p [port]，时间久了也会觉得很麻烦，特别是当 user, remote 和 port 都得输入，而且还不好记忆的时候。配置别名可以让我们进一步偷懒。

比如我想用 ssh myserver 来替代上面这么一长串，那么在 ~/.ssh/config 里面追加以下内容：

```txt
Host myserver
    HostName [remote-node]
    User [user]
    Port [port]
```

### scp 远程拷贝文件

在两台机之间传输文件可以用 scp，它的地址格式与 ssh 基本相同，都是可以省略用户名和端口，稍微的差别在与指定端口时用的是大写的 -P 而不是小写的。不过，如果你已经配置了别名，那么这都不重要，因为 scp 也支持直接用别名。scp 用起来很简单，看看下面的例子就明白了：

```bash
# 把本地的 /path/to/local/file 文件传输到远程的 /path/to/remote/file
scp -P port /path/to/local/file user@node:/path/to/remote/file

# 也可以使用别名
scp /path/to/local/file myserver:/path/to/remote/file

# 把远程的 /path/to/remote/file 下载到本地的 /path/to/local/file
scp myserver:/path/to/remote/file /path/to/local/file

# 远程的默认路径是家目录
# 下面命令把当前目录下的 file 传到远程的 ~/dir/file
scp file myserver:dir/file

# 加上 -r 命令可以传送文件夹
# 下面命令可以把当前目录下的 dir 文件夹传到远程的家目录下
scp -r dir myserver:

# 别忘了 . 可以用来指代当前目录
# 下面命令可以把远程的 ~/dir 目录下载到当前目录里面
scp -r myserver:dir/ .
```

## 代理

### 反向代理

相当于 frp 或者 ngrok

1. 场景1

公网访问内网机器，假设有个公网机器HostB，有个公司的内网机器HostC， 我现在有个家里的机器HostA，我HostA想访问HostC，有这样的需求吧？这样我就需要在HostC上运行`ssh -R [HostB-PORT]:HostC:[HostC-PORT] user@HostB` 这个命令就是把HostC的指定 port转发到HostB上的指定端口上。HostB这时候称为跳板机，我HostA 通过SSH的普通登录命令登录到HostB上，就可以在HostB上运行`ssh HostC-user@HostC -p [HostB-Port]` 登录内网的HostC机器了。

当然SSH会经常断线，所以有个工具叫autossh可以代替SSH，`sudo apt install autossh`。

```bash
autossh -NfR [HostB-PORT]:HostC:[HostC-PORT]  user@HostB
```

-N 表示非不执行命令，只做端口转发；-f 表示在后台运行

在HostC这台内网机器上，如果你要设为开机启动，你懂得。就不细说了。注意！HostB跳板机一定要是公网IP。


2. 场景2

一个非登录的例子，通过反向代理，让别人通过跳板机看到你内网搭建的Web网站。原理基本一致。

```bash
 ssh -NR 0.0.0.0:[HostB-port]:HostC-localhost:[HostC-Web-port]  user@HostB
```

这时候，你的远方朋友就可以通过在自己的家里的机器访问 HostB:[HostB-prot] 这个URL访问到你的Web站点了

3. 场景3

HostB使用HostC的http代理,比如为了翻墙什么的。

比方说在本地HostC的127.0.0.1:1080运行了HTTP代理服务，现在我想让另一台机子HostB 也能够使用这个HTTP代理。

HostC$ ssh -NRf 11080:localhost:1080 user@HostB
HostC$ ssh remote
HostB$ export http_proxy=http://127.0.0.1:11080/
HostB$ export https_proxy=http://127.0.0.1:11080/
HostB$ curl http://www.google.com


下面就总结一下，最本质的反向代理用法。

假设三台机器，HostA和HostB共同处于一个可互相访问的内网局域网，有个外网HostC作为跳板机，那么：

HostA 就可以将自己可以访问的 HostB:PortB 的服务暴露给外网服务器 HostC:PortC，在 HostA 上运行：

HostA$ ssh -RfN HostC:PortC:HostB:PortB  user@HostC

那么链接 HostC:PortC 就相当于链接 HostB:PortB。使用时需修改 HostC 的 /etc/ssh/sshd_config，添加：

GatewayPorts yes

相当于内网穿透，比如 HostA 和 HostB 是同一个内网下的两台可以互相访问的机器，HostC是外网跳板机，HostC不能访问 HostA，但是 HostA 可以访问 HostC。

那么通过在内网 HostA 上运行 ssh -R 告诉 HostC，创建 PortC 端口监听，把该端口所有数据转发给我（HostA），我会再转发给同一个内网下的 HostB:PortB。

同内网下的 HostA/HostB 也可以是同一台机器，所以就变成了以上提到的三个场景，换句话说就是内网 HostA 把自己可以访问的端口暴露给了外网 HostC。


### 正向代理

相当于 iptable 的 port forwarding，但是iptable的性能更好，只是不是很方便。

正向代理的场景，我做远程办公的时候是用到的，因为公司的区块链Validator和explorer这些节点，是有访问限制的，为了安全，但是开发人员有些时候就需要登录上去，就有专门的DevOps机器来作为入口来访问这些节点，所以管理人员需要把我的IP加入到访问DevOps机器的白名单里面，但是我的家里出口IP由于运营商的机制会经常变动，所以需要一台固定的外网机器IP来访问DevOps机器。

所以需要在我的外网机器上运行一个正向代理，登录到DevOps机器上，当然是为了图方便，不然要先登录到外网机器，再从外网机器ssh登录到DevOps机器，做了两次登录，比较麻烦。

首先在外网机器上运行正向代理：

```bash
 ssh -CfNg -L [public-ip-node]:222:devop.t.hmny.io:22  sh@devop.t.hmny.io
```

然后就可以从我家里的本地机器登录DevOps机器了：

```bash
ssh sh@[public-ip-node] -p 222
```

注意，这里的sh用户名是DevOps机器上给我创建的用户名，相当于把我的外网机器的222端口数据转发到DevOps机器上的22端口了。

1. 场景1：远程端口映射到其他机器

HostB 上启动一个 PortB 端口，映射到 HostC:PortC 上，在 HostB 上运行：

HostB$ ssh -L 0.0.0.0:PortB:HostC:PortC user@HostC

这时访问 HostB:PortB 相当于访问 HostC:PortC（和 iptable 的 port-forwarding 类似）。

2. 场景2：本地端口通过跳板映射到其他机器

HostA 上启动一个 PortA 端口，通过 HostB 转发到 HostC:PortC上，在 HostA 上运行：

HostA$ ssh -L 0.0.0.0:PortA:HostC:PortC  user@HostB

这时访问 HostA:PortA 相当于访问 HostC:PortC。

两种用法的区别是，第一种用法本地到跳板机 HostB 的数据是明文的，而第二种用法一般本地就是 HostA，访问本地的 PortA，数据被 ssh 加密传输给 HostB 又转发给 HostC:PortC。


## References:

- https://zhuanlan.zhihu.com/p/57630633
- https://zhuanlan.zhihu.com/p/21999778


