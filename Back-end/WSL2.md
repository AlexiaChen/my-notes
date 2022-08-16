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

