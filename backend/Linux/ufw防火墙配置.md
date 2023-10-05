#### 前言

之前写了 ![[iptables命令详解]]
但是iptables的配置规则比较复杂，提供了最大灵活性，但是对于小白用户不太方便。所以有一个ufw(uncomplicated firewall)命令来简化这些防火墙配置，这个工具是构建在iptables之上的，底层调用了iptables。

- 检查当前UFW的状态:

```bash
sudo ufw status
sudo ufw status verbose
```

- 打开和关闭UFW:

```bash
sudo ufw enbale
sudo ufw disable
```

- 阻止一个特定的IP向任何地址发起的任何请求

```bash
sudo ufw deny from 203.0.113.100
```

```txt
OutputStatus: active

To                         Action      From
--                         ------      ----
Anywhere                   DENY        203.0.113.100     

```

- 阻止一个子网发向任何地址的任何请求

```bash
sudo ufw deny from 203.0.113.0/24
```

```txt
Status: active  
  
To Action From  
-- ------ ----  
Anywhere DENY 203.0.113.0/24
```

- 阻止特定IP发起到某个特定网卡流入连接。在多网卡机器上有用。

```bash
udo ufw deny in on eth0 from 203.0.113.100
```

```txt
To Action From  
-- ------ ----  
Anywhere on eth0 DENY IN 203.0.113.100
```


- 允许某个IP过来的任何请求数据

```bash
sudo ufw allow from 203.0.113.101
```

```txt
Status: active  
  
To Action From  
-- ------ ----  
Anywhere ALLOW 203.0.113.101
```

- 允许特定IP发起到某个特定网卡的流入连接。在多网卡机器上有用。

```bash
sudo ufw allow in on eth0 from 203.0.113.102
```

- 删除某个特定规则

```bash
sudo ufw delete <rule>
sudo ufw delete allow in on eth0 from 203.0.113.102
sudo delete <rule_num>
# check rule num
sudo ufw status
```

- 列出当前的app profile。app的概念其实就是一系列规则的封装，打包，相当于规则集合，这样就方便以人类可读的方式。比如我们对OpenSSH，或者Nginx的网络端口或者IP做限制做规则，就可以封装成一个app名为Nginx的Profile

```bash
sudo ufw app list
```

- 打开允许一个app profile/ 删除一个App

```bash
sudo ufw allow "AppName"
sudo ufw delete allow "AppName"
```

- 允许一个特定IP或子网的SSH协议流入（就是开放22端口，供SSH使用）

```bash
sudo ufw allow from 203.0.113.103 proto tcp to any port 22
sudo ufw allow from 203.0.113.0/24 proto tcp to any port 22
```

- 允许任何一个地址的请求发向22端口 ( 也是开放SSH端口，如果是http就80，https就是443) 

```bash
sudo ufw allow 22
sudo ufw allow proto tcp from any to any port 80,443,22
```

- 允许一个特定IP或子网的resync协议流入（开放873端口）

```bash
sudo ufw allow from 203.0.113.103 to any port 873
sudo ufw allow from 203.0.113.0/24 to any port 873
```

- 开放一个特定地址或子网发起的MySQL连接服务

```bash
sudo ufw allow from 203.0.113.103 to any port 3306
sudo ufw allow from 203.0.113.0/24 to any port 3306
```

- 开放一个特定地址或子网发起的PostgreSQL连接服务

```bash
sudo ufw allow from 203.0.113.103 to any port 5432
sudo ufw allow from 203.0.113.0/24 to any port 5432
```

- 阻止Email发出去，其实也就是限制STMP请求，STMP默认的端口是用25,阻止25端口的出口流量

```bash
sudo ufw deny out 25
```



## Trouble Shooting

- [Can I use WSL's ufw as a firewall for Windows 10? - Super User](https://superuser.com/questions/1626459/can-i-use-wsls-ufw-as-a-firewall-for-windows-10/1626513#1626513)
- [ubuntu - ufw iptables jump rule errors with "Could not load logging rules" - Unix & Linux Stack Exchange](https://unix.stackexchange.com/questions/617240/ufw-iptables-jump-rule-errors-with-could-not-load-logging-rules)
- [ubuntu - UFW is active but not enabled why? - Super User](https://superuser.com/questions/590600/ufw-is-active-but-not-enabled-why)
