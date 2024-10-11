# 使用WSL2中一些笔记

WSL2和宿主机windows之间的网络通信:

[Accessing network applications with WSL | Microsoft Docs](https://docs.microsoft.com/en-us/windows/wsl/networking)


##### 外部设备访问本机上WSL2内部的端口

外部设备访问本机的WSL2内的端口，需要在宿主机上配置一个端口转发

[ruby on rails - Connecting to WSL2 server via local network - Stack Overflow](https://stackoverflow.com/questions/61002681/connecting-to-wsl2-server-via-local-network)

```powershell

netsh interface portproxy add v4tov4 listenport=<port-to-listen> listenaddress=0.0.0.0 connectport=<port-to-forward> connectaddress=<forward-to-this-IP-address>

netsh interface portproxy add v4tov4 listenport=1935 listenaddress=0.0.0.0 connectport=1935 connectaddress=172.27.146.74

# windows宿主机上配置防火墙，允许1935端口流入数据
netsh advfirewall firewall add rule name= "Open Port 1935" dir=in action=allow protocol=TCP localport  
=1935


netsh interface portproxy show all
netsh interface portproxy delete v4tov4 listenport=1935 listenaddress=172.27.146.74

```

注意connectaddress一般为WSL2中的port，由`ip addr`命令查看 inet。 listenaddress就是宿主机windows的地址了，这里的0.0.0.0是监听在any addr上。


还有一个无关于WSL2的，也是在在宿主机上配置一个端口转发，你的宿主机上运行着一个VPN Client，这个Client连接着一个VPN网络。而跟你宿主机所在的一个局域网的机器没有VPN Client，我需要让这个机器访问到VPN网络中（比如云IDC机房的某台机器的服务端口），也是可以这样做:



```powershell
# 添加端口转发
netsh interface portproxy add v4tov4 listenport=<local-port> listenaddress=<local-ip> connectport=<vpn-port> connectaddress=<vpn-ip>
# 查看端口转发
netsh interface portproxy show all
# 删除端口转发
netsh interface portproxy delete v4tov4 listenport=<local-port> listenaddress=<local-ip>
```
        
- **`<local-port>`**: 你本机监听的一个端口，局域网中的机器就是连接到这个端口上
- **`<local-ip>`**: 你本机的IP (通常使用 `0.0.0.0` 来监听所有网卡).
- **`<vpn-port>`**: 你需要转发到VPN网络的某个服务的端口.
-  **`<vpn-ip>`**: VPN网络中那台服务的IP地址

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


#### WSL2突然在windows 11上无法打开GUI应用

用管理员权限运行Powershell, 在PS中输入:

```ps
wsl --update
wsl --shutdown
wsl --shutdown <linux-distribution>
```

重新进入WSL2的Linux系统，就可以工作了 。[Linux GUI was not working in wsl2 · Issue #7250 · microsoft/WSL (github.com)](https://github.com/microsoft/WSL/issues/7250)


### WSL2 新的特性---支持mirror网络，与host共享一个网络

这个是实验阶级的特性，只有windows 11 23H才有，并且WSL要升级 `wsl --update --pre-release`

[Windows Subsystem for Linux September 2023 update - Windows Command Line (microsoft.com)](https://devblogs.microsoft.com/commandline/windows-subsystem-for-linux-september-2023-update/)

- autoMemoryReclaim    让WSL虚拟机自动回收多余的内存，而不是一直增长
- 稀疏VHD      自动回收WSL的VHD虚拟硬盘空间。
- 镜像网络模式   让WSL的网络与宿主机windows 共享一个网络，也就是WSL启动的服务的端口，在局域网可以直接通过宿主机的IP地址+端口进行直接访问。
- dns隧道     修改了WSL解析DNS的原理，提高网路的兼容性
- 防火墙  可以通过windows的防火墙的规则，应用到WSL上，对WSL虚拟机进行更高级的防火墙控制
- 自动代理  让WSL自动使用windows宿主机上的代理信息，提高网络兼容性

#### autoMemoryReclaim 

默认是关闭的，

|Setting name|Value|Default|Notes|
|:--|:--|:--|:--|
|`autoMemoryReclaim`|string|disabled|Automatically releases cached memory after detecting idle CPU usage. Set to `gradual` for slow release, and `dropcache` for instant release of cached memory.|

当设置为 "gradual "时，在闲置 5 分钟后，WSL 将开始缓慢释放 Linux 中的缓存内存，并将其作为可用内存释放回 Windows 主机。这意味着，当你不使用 WSL 虚拟机时，它的内存大小会自动缩小！

这是通过 WSL 通过查看 CPU 使用率是否连续 5 分钟持续较低来检测您处于空闲状态，然后我们开始使用 [cgroup memory.reclaim](https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html#:~:text=like%20disk%20readahead.-,memory.reclaim,-A%20write%2Donly) 功能来回收缓存内存。我们回收您的虚拟机内存大小的固定部分，该大小是经过计算的，以便如果您的虚拟机已满缓存内存，则 30 分钟后缓存内存将为零（例如：如果您有 3000MB 内存，我们每分钟回收 100MB） 。 `memory.reclaim` cgroup 功能允许我们随着时间的推移智能地回收一部分内存，在性能和内存使用之间取得平衡。

但是，此功能确实需要在 WSL 中禁用 cgroups v1，这可能会导致一些问题。在早期测试中，我们注意到，当在 WSL 中将 docker 守护程序作为服务运行时，这会破坏 docker 守护进程，因此，如果您使用此功能，我们建议您使用 Docker Desktop 来满足您的 docker 需求。我们正在与 Docker 团队合作，以在未来解决这个问题。

如果您想自定义空闲检测阈值等，我们建议您不启用此功能并创建 [bash 脚本](https://gist.github.com/craigloewen-msft/496078591e0bbbfdec9f144c6b50a8cc)

您也可以将其设置为 `dropcache`，这样在检测到空闲后会完全放弃缓存，并且不需要更改任何 cgroup。

#### sparseVHD

|Setting name|Value|Default|Notes|
|:--|:--|:--|:--|
|`sparseVhd`|bool|false|When set to true, any newly created VHD will be set to sparse automatically.|

WSL 虚拟硬盘 (VHD) 的大小会随着您的使用而增加，现在启用此功能后，它们的大小也会自动缩小！此新设置会自动将任何新 VHD 设置为稀疏 VHD，这可以自动减小其大小。此外，我们添加了命令，允许您根据需要将现有发行版设置为稀疏或不稀疏。`wsl --manage <distro> --set-sparse <true/false>`

#### Mirrored Network

|Setting name|Value|Default|Notes|
|:--|:--|:--|:--|
|`networkingMode`|string|NAT|If the value is `mirrored` then this turns on mirrored networking mode. Default or unrecognized strings result in NAT networking.|

网络连接的改进一直是 WSL 的最高要求，此功能旨在改进 WSL 中的网络连接体验！这是对 WSL 传统 NAT 网络架构的彻底改革，将其转变为一种名为 "镜像 "的全新网络模式。该模式的目的是将 Windows 上的网络接口镜像到 Linux 上，以增加新的网络功能并提高兼容性。

以下是启用该模式的当前优势：
- 支持 IPv6
- 使用本地主机地址 127.0.0.1 从 Linux 连接到 Windows 服务器
- 直接从局域网（LAN）连接到 WSL
- 提高 VPN 的网络兼容性
- 支持组播

#### DNS tunnel

|Setting name|Value|Default|Notes|
|:--|:--|:--|:--|
|`dnsTunneling`|bool|false|Changes how DNS requests are proxied from WSL to Windows|

WSL 无法连接到互联网的影响因素之一是对 Windows 主机的 DNS 调用被阻止。这是因为 WSL VM 发送到 Windows 主机的 DNS 网络数据包被现有网络配置阻止。 DNS 隧道通过使用虚拟化功能直接与 Windows 通信来修复此问题。

这使我们能够在不发送网络数据包的情况下解析 DNS 名称请求，这样即使您有 VPN、特定防火墙设置或其他网络配置，您也可以获得更好的互联网连接。此功能应该可以提高网络兼容性，从而降低 WSL 内部没有网络连接的可能性。


#### Hyper-V 防火墙

|Setting name|Value|Default|Notes|
|:--|:--|:--|:--|
|`firewall`|bool|false|Setting this to true allows the Windows Firewall rules, as well as rules specific to Hyper-V traffic, to filter WSL network traffic.|

Hyper-V 防火墙允许您指定将应用于 WSL 的防火墙设置和规则。此外，默认情况下，Windows 上的所有现有防火墙设置和规则都将自动应用于您的 WSL 发行版。

启用此功能后，您可以通过在 Windows 防火墙设置中创建新的防火墙规则并查看它们立即应用于 WSL 来测试它，或者您可以通过在 PowerShell 中运行以下命令来创建仅直接应用于 WSL 的新规则：`New-NetFirewallHyperVRule`

如果您想了解有关防火墙规则和设置的更多信息，请运行以下命令查看该命令的帮助消息：`New-NetFirewallHyperVRule -?`。


#### autoProxy

|Setting name|Value|Default|Notes|
|:--|:--|:--|:--|
|`autoProxy`|bool|false|Enforces WSL to use Windows’ HTTP proxy information|

该功能旨在提高使用 HTTP 代理时的网络兼容性。目前，如果您在 Windows 上使用 HTTP 代理，它不会直接适用于您的 WSL 发行版。通常情况下，如果你想在 WSL 上设置 HTTP 代理，你需要使用与在 Linux 机器上相同的方式进行设置，否则你可能会遇到连接问题。这项功能就是为了解决这个问题，它会自动使用 Windows 上的 HTTP 代理信息来设置 Linux 中的 HTTP 代理。

