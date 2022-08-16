## iptables防火墙

iptables这个命令是Linux网络防火墙的接口命令，可以管理，建立防火墙规则，主要是IPv4。

```bash
# systemctl start iptables
# systemctl stop iptables
# systemctl restart iptables
```

- Table: 是一系列Chain的集合
- Chain: 一个Chain表示一堆Rules
- Rule：一个匹配条件，用于匹配数据Packet
- Target: 当匹配正在发生时，需要触发的动作
- Policy: 当没有匹配发生时，默认的触发动作

语法

```bash
iptables --table <TABLE> -A/-C/-D... <CHAIN> <rule> --jump <Target>
```


##### Table

- filter: packet过滤默认使用的Table。包含的Chain有INPUT, OUTPUT和FORWARD
- nat: 关于Network Address Translation的Table。包含的Chain有PREROUTING 和POSTROUTING
- mangle: 用于专门的数据包的修改。内置的Chain包括PREROUTING和OUTPUT。
- raw: 配置免于连接跟踪的情况。内置链是PREROUTING和OUTPUT。
- security: 用于强制访问控制(Mandatory Access Control)

##### Chain

- INPUT: 一组Rule，用于指向localhost Socket的Packet。
- FORWARD: 为通过设备路由的Packet的一组Rule
- OUTPUT： 用于本地生成的Packet，旨在向外传输的一组Rule
- PREROUTING： 修改已到达的Packet
- POSTROUTING： 修改正离开的Packet

以上是内建的Chain，用户当然可以自己建立自己的Chain。

### Options

- `-A, -append`: 给特定的Chain加入一些参数

语法:

```bash
iptables [-t table] --append [chain] [parameters]
```

所有流经任何端口的数据流量都被舍去

```bash
iptables -t filter --append INPUT -j DROP
```

![[Pasted image 20220816155737.png]]

- `-D,-delete`： 删除特定Chain下的Rule

语法:

```bash
iptables [-t table] --delete [chain] [rule_number]
```

删除INPUT Chain下的Rule 2:

```bash
iptables -t filter --delete INPUT 2
```

![[Pasted image 20220816160401.png]]

- `-C,-check`: 检查一个rule是否在某个Chain中存在，如果存在返回0，如果不存在返回1

语法:

```bash
iptables [-t table] --check [chain] [parameters]
```

```bash
iptables -t filter --check INPUT -s 192.168.1.123 -j DROP
```

![[Pasted image 20220816160626.png]]

### Parameters

参数提供了匹配packet和执行特定动作的功能

- `-p, -proto` 是Packet的遵守的协议，可能的值是tcp，udp，icmp，ssh等

```bash
iptables [-t table] -A [chain] -p {protocol_name} [target]

# 所有流经出入本机的UDP Packet全部舍去
iptables -t filter -A INPUT -p udp -j DROP
```

![[Pasted image 20220816161216.png]]

- `-s, -source` 数据包的源地址

```bash
iptables [-t table] -A [chain] -s {source_address} [target]

# 在INPUT Chain中允许所有从指定IP发送出去的Packet，没有指定协议就表示所有协议
iptables -t filter -A INPUT -s 192.168.1.230 -j ACCEPT
```

![[Pasted image 20220816161509.png]]

- `-d, -destination` 数据包的发送地址

```bash
iptables [-t table] -A [chain] -d {destination_address} [target]

# 在OUTPUT Chain中加入一个Rule，这个Rule是流经本机的所有发送到指定IP的数据包都被舍弃
iptables -t filter -A OUTPUT -d 192.168.1.123 -j DROP
```

![[Pasted image 20220816161946.png]]

- `-i, -in-interface` 匹配指定网卡的数据包

```bash
iptables [-t table] -A [chain] -i {interface} [target]

# 所有流入wlan0无线网卡的数据包都被丢弃，
iptables -t filter -A INPUT -i wlan0 -j DROP
```

![[Pasted image 20220816162342.png]]

- `-o, -out-inerface` 与以上相反的方向
- `-j, -jump` 这个参数指定了匹配的动作

```bash
iptables [-t table] -A [chain] [parameters] -j {target}

# 在FORWARD Chain里面添加舍弃所有数据包的Rule
iptables -t filter -A FORWARD -j DROP
```

![[Pasted image 20220816162644.png]]

#### 注意

```bash
# 删除用户自定义的Chain和Filter Table中的 rule
sudo iptables --flush
# 保存之前设置的配置
sudo iptables-save
# 恢复之前的配置
sudo iptables-restore
```

#### 多个有用的例子

- 查看所有的iptables上的防火墙rule

```bash
 iptables -L -n -v

iptables -t TABLE -L -v -n
```


- 阻止某个IP的数据包

```bash
iptables -A INPUT -s xxx.xxx.xxx.xxx -j DROP
iptables -A INPUT -p tcp -s xxx.xxx.xxx.xxx -j DROP

```

- 解除某个IP的数据包限制

```bash
iptables -D INPUT -s xxx.xxx.xxx.xxx -j DROP
```

- 阻止某个特定的端口

```bash
# 从这个端口流出的TCP数据都舍弃
iptables -A OUTPUT -p tcp --dport xxx -j DROP
# 从这个端口流入的TCP数据都允许通过
iptables -A INPUT -p tcp --dport xxx -j ACCEPT
```

- 多个端口允许

```bash
iptables -A INPUT  -p tcp -m multiport --dports 22,80,443 -j ACCEPT
iptables -A OUTPUT -p tcp -m multiport --sports 22,80,443 -j ACCEPT
```

- 允许某个网段特定端口的流出的TCP数据

也就是说，在本机，允许本机流出数据包，发送到特定网段机器的 22端口

```bash
iptables -A OUTPUT -p tcp -d 192.168.100.0/24 --dport 22 -j ACCEPT
```

- 阻止访问FaceBook网站

```bash
> host facebook.com 
facebook.com has address 66.220.156.68
> whois 66.220.156.68 | grep CIDR
CIDR: 66.220.144.0/20
> iptables -A OUTPUT -p tcp -d 66.220.144.0/20 -j DROP
```

- 配置端口转发

例子是把25端口的数据转发到2525端口上

```bash
iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 25 -j REDIRECT --to-port 2525
```


- 阻止进来的ping数据包

```bash
iptables -A INPUT -p icmp -i eth0 -j DROP
```

- 允许回环地址可以访问，这个应该始终需要保持这样

```bash
iptables -A INPUT -i lo -j ACCEPT
iptables -A OUTPUT -o lo -j ACCEPT
```

- 对丢弃的网络数据包做一个日志记录

日志默认写到 `/var/log/messages` 里，可以用grep查看

```bash
iptables -A INPUT -i eth0 -j LOG --log-prefix "IPtables dropped packets:"
```

- 阻止某个MAC地址的访问

```bash
iptables -A INPUT -m mac --mac-source 00:00:00:00:00:00 -j DROP
```


- 限制某个IP地址端口下的并发连接数量

```bash
iptables -A INPUT -p tcp --syn --dport 22 -m connlimit --connlimit-above 3 -j REJECT
```

- 搜索某个Table下的Rule

```bash
 iptables -L <Table> -v -n | grep <rule_text>
 iptables -L INPUT -v -n | grep 192.168.0.100
```

- 自定义一个Chain

```bash
iptables -N custom-filter
iptables -L
```

- Flush Chain或Rule

```bash
iptables -F
iptables -t nat -F
```

- 保存Rules到文件

```bash
iptables-save > ~/iptables.rules
```

- 从一个文件中恢复Rules

```bash
iptables-restore < ~/iptables.rules
```

- 允许 Established 和 Related 的连接

```bash
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A OUTPUT -m conntrack --ctstate ESTABLISHED -j ACCEPT
```

- 舍弃非法数据包

```bash
iptables -A INPUT -m conntrack --ctstate INVALID -j DROP
```

- 阻止某个IP地址向本机发起的连接

```bash
iptables -A INPUT -i eth0 -s xxx.xxx.xxx.xxx -j DROP
```

- 关闭邮件的流出

```bash
 iptables -A OUTPUT -p tcp --dports 25,465,587 -j REJECT
```

