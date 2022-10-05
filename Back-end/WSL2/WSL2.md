# 使用WSL2中一些笔记

WSL2和宿主机windows之间的网络通信:

[Accessing network applications with WSL | Microsoft Docs](https://docs.microsoft.com/en-us/windows/wsl/networking)


##### 外部设备访问本机上WSL2内部的端口

外部设备访问本机的WSL2内的端口，需要在宿主机上配置一个端口转发

[ruby on rails - Connecting to WSL2 server via local network - Stack Overflow](https://stackoverflow.com/questions/61002681/connecting-to-wsl2-server-via-local-network)

```bash

netsh interface portproxy add v4tov4 listenport=<port-to-listen> listenaddress=0.0.0.0 connectport=<port-to-forward> connectaddress=<forward-to-this-IP-address>

netsh interface portproxy add v4tov4 listenport=1935 listenaddress=0.0.0.0 connectport=1935 connectaddress=172.27.146.74

# windows宿主机上配置防火墙，允许1935端口流入数据
netsh advfirewall firewall add rule name= "Open Port 1935" dir=in action=allow protocol=TCP localport  
=1935


netsh interface portproxy show all
netsh interface portproxy delete v4tov4 listenport=1935 listenaddress=172.27.146.74

```

注意connectaddress一般为WSL2中的port，由`ip addr`命令查看 inet。 listenaddress就是宿主机windows的地址了，这里的0.0.0.0是监听在any addr上。

#### WSL2上开启systemd

[WSL 2 - Enabling systemd (github.com)](https://gist.github.com/djfdyuruiry/6720faa3f9fc59bfdf6284ee1f41f950)

[arkane-systems/genie: A quick way into a systemd "bottle" for WSL (github.com)](https://github.com/arkane-systems/genie)

[DamionGans/ubuntu-wsl2-systemd-script: Script to enable systemd support on current Ubuntu WSL2 images [Unsupported, no longer updated] (github.com)](https://github.com/DamionGans/ubuntu-wsl2-systemd-script)

#### WSL2上运行snapd

最初打算是通过snap安装v2ray-core准备翻墙的，但是snapd这个Deamon又启动不起来。

```bash
sudo apt install snapd
sudo snap install v2ray-core v2ray
```

因为WSL2默认是systemd的，而snapd貌似只支持systemd，`sudo service snapd start` 这样的命令用不了。

直接运行这个bash就可以work了:

```bash
sudo apt-get update && sudo apt-get install -yqq daemonize dbus-user-session fontconfig
sudo daemonize /usr/bin/unshare --fork --pid --mount-proc /lib/systemd/systemd --system-unit=basic.target
exec sudo nsenter -t $(pidof systemd) -a su - $LOGNAME
```

 https://github.com/microsoft/WSL/issues/5126

#### 限制WSL2内存使用量

发现用WSL2内存占用太大，Vmmem进程用久了就占用百分之90+%。最后命令行都打不出来。

解决方案：

- 用wsl命令行重启WSL2 distro
- 限制内存使用

在`%UserProfile%\.wslconfig`创建一个文件，里面配置上:

```txt
[wsl2]
memory=5GB
```

配置规范：

```txt
[wsl2]
kernel=<path>              # An absolute Windows path to a custom Linux kernel.
memory=<size>              # How much memory to assign to the WSL2 VM.
processors=<number>        # How many processors to assign to the WSL2 VM.
swap=<size>                # How much swap space to add to the WSL2 VM. 0 for no swap file.
swapFile=<path>            # An absolute Windows path to the swap vhd.
localhostForwarding=<bool> # Boolean specifying if ports bound to wildcard or localhost in the WSL2 VM should be connectable from the host via localhost:port (default true).

# <path> entries must be absolute Windows paths with escaped backslashes, for example C:\\Users\\Ben\\kernel
# <size> entries must be size followed by unit, for example 8GB or 512MB
```

注意： .wslconfig文件一定是UTF8的，不要带BOM，还有不要带空格什么的，不然配置文件不会起作用。

- https://blog.simonpeterdebbarma.com/2020-04-memory-and-wsl/
- https://github.com/microsoft/WSL/issues/4166
- https://github.com/microsoft/WSL/issues/5389


#### WSL用了export再import导入到一个特定目录无法指定Linux默认用户的问题

如果没有经过标题的操作，运行以下命令就可以了:

```bash
<DistroName> config --set-default-user [username] 
```
但是，经过导出导入后，默认登陆进去是root用户，我想换回来我自己的用户。发现以上方法不行了。

还好找到了WSL官方的这个[issue](https://github.com/Microsoft/WSL/issues/3974)。

步骤：

- 写一个PowerShell函数：

```powershell
 Function WSL-SetDefaultUser ($distro, $user) { Get-ItemProperty Registry::HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Lxss\*\ DistributionName | Where-Object -Property DistributionName -eq $distro | Set-ItemProperty -Name DefaultUid -Value ((wsl -d $distro -u $user -e id -u) | Out-String); };
```

- 再通过PowerShell调用这个函数:

```powershell
 WSL-SetDefaultUser <DistroName> <username>
```

这个方法比较麻烦，发现还操作注册表了，对用户暴露了过多细节，这只是临时方案，希望微软以后改进。比如这样:

```powershell
wsl <DistroName> --set-default-user <username>
```

#### 升级到windows11后，WSL2无网络连接问题

发现开了v2ray代理翻不了墙了，apt update测试了下，我去，没网络连接了，所以DNS用不了，以前好好的。Orz。

通过这个issue 解决了 https://github.com/microsoft/WSL/issues/6404#issuecomment-963960301

修改`/etc/wsl.conf` 文件的配置:

```conf
[network]
generateResolvConf = true
```

然后运行 `wsl --shutdown` 重启WSL2就可以了。windows 11真是小问题不断啊。