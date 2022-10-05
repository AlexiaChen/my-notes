### 简介

[Nmap](https://so.csdn.net/so/search?q=Nmap&spm=1001.2101.3001.7020)也就是Network Mapper，网络发现（Network Discovery）和安全审计，是一款网络连接端扫描软件，用来扫描网上电脑开放的网络连接端。确定哪些服务运行在哪些连接端，并且推断计算机运行哪个操作系统。它是网络管理员比用的软件之一，以及用以评估网络系统保安，nmap的核心功能有：主机发现、端口扫描、版本侦测、操作系统侦测、防火墙/IDS规避、NSE脚本引擎

**Nmap包含四项基本功能：**

-   主机发现 (Host Discovery)
-   端口扫描 (Port Scanning)
-   版本侦测 (Version Detection)
-   操作系统侦测 (Operating System Detection)

### 基本命令行参数解析

##### Nmap支持的扫描类型

###### TCP扫描

TCP扫描一般用于检查和完成您和选定的目标系统之间的三方握手。TCP扫描通常是非常嘈杂的，几乎不费吹灰之力就可以检测到。之所以说 "嘈杂"，是因为这些服务可以记录发送者的IP地址，并可能触发入侵检测系统。

###### UDP扫描

UDP扫描是用来检查目标机器上是否有任何UDP端口启动并监听传入的请求。与TCP不同的是，UDP没有机制来回应肯定的Ack，所以在扫描结果中总是有几率出现假阳性(false-positive，也就是误判)。然而，UDP扫描被用来揭示可能在UDP端口上运行的特洛伊木马，甚至揭示隐藏的RPC服务。这种类型的扫描往往是相当慢的，因为一般来说，作为一种预防措施，机器倾向于放慢对这种流量的回应(response)。

###### SYN扫描

这是另一种形式的TCP扫描。不同的是，与正常的TCP扫描不同，Nmap自己制作一个SYN数据包，这是用来建立TCP连接的第一个数据包。这里需要注意的是，TCP连接从来没有完成（因为没有三次握手），而是对这些特别制作的SYN数据包的response被Nmap用来分析以产生扫描结果。

###### ACK扫描

ACK扫描用于确定一个特定的端口是否被过滤。事实证明，在试图探测防火墙及其现有的规则集时，这非常有帮助。简单的数据包过滤将允许已建立的连接（设置了ACK位的数据包），并不允许建立完连接后续的数据包通信。而更复杂的有状态防火墙可能直接不允许建立连接。

###### FIN扫描

也是一种隐蔽的扫描（就是不太会让对方发现你去扫描），像SYN扫描一样，但会发送一个TCP FIN数据包。如果收到这个数据包，大多数(不是所有)计算机都会发送一个RST包（复位包）回来，所以FIN扫描可以显示假阳性和阴性(误判)，但它可能会被一些IDS程序和其他对抗措施所忽视。

###### NULL扫描

NULL扫描是非常隐蔽的扫描，它们所做的正如其名 - 它们将所有的头字段设置为null。一般来说，这不是一个有效的数据包，少数目标(系统)不知道如何处理这样的数据包。这样的目标一般都是某个版本的WINDOWS（所以windows做服务器少），用NULL数据包扫描它们最终可能产生不可靠的结果。另一方面，当一个系统不运行windows时，这可以作为一种有效的方式来通过。

###### XMAS扫描

就像NULL扫描一样，这些扫描在本质上也是隐蔽的。由于TCP协议栈的实现方式，运行Windows的计算机不会对Xmas扫描做出反应。这种扫描的名称来自于被送出扫描的数据包内开启的一组标志。XMAS扫描是用来操纵TCP头中的PSH、URG和FIN标志的。

###### RPC扫描

RPC扫描是用来发现响应远程过程调用服务（RPC）的机器。RPC允许在一定的连接下，在某台机器上远程运行命令。RPC服务可以在一系列不同的端口上运行，因此，很难从正常的扫描中推断出RPC服务是否在运行。一般来说，不时地运行RPC扫描是一个好主意，以找出你有这些服务在运行。

###### IDLE扫描

IDLE扫描是本nmap文章中讨论的所有扫描中最隐蔽的一种，因为数据包是从外部主机上反弹下来的。对主机的控制一般是不需要的，但主机需要满足一组特定的条件。这是Nmap中比较有争议的选项之一，因为它只用于恶意攻击。

##### Nmap的命令

语法:

```bash
nmap [Scan Type(s)] [Options] {target specification}
```

常用参数:

```txt
1.  -sn: Ping Scan 只进行主机发现，不进行端口扫描。
    

4.  -PE/PP/PM: 使用ICMP echo、 ICMP timestamp、ICMP netmask 请求包发现主机。
    
5.  -PS/PA/PU/PY[portlist]: 使用TCP SYN/TCP ACK或SCTP INIT/ECHO方式进行发现。
    

7.  -sL: List Scan 列表扫描，仅将指定的目标的IP列举出来，不进行主机发现。
    
8.  -Pn: 将所有指定的主机视作开启的，跳过主机发现的过程。
    

10.  -PO[protocollist]: 使用IP协议包探测对方主机是否开启。
    
11.  -n/-R: -n表示不进行DNS解析；-R表示总是进行DNS解析。
    
12.  --dns-servers <serv1[,serv2],...>: 指定DNS服务器。
    
13.  --system-dns: 指定使用系统的DNS服务器
    
14.  --traceroute: 追踪每个路由节点
```

![[cdf7cb862814487a763ff607d1db74a.png]]

![[adb0dee3a5ec2b712780fb4a04e1eb3.png]]

![[a86380027d31411b09efc6dbfd9d566.png]]

![[7007004079f442b7d1cba4caed2f4b6.png]]

![[9b21dbdce8433b6c74c08d5123c29e3.png]]

![[9252ea8cc3933330d64eb63fa0ddcb4.png]]

![[c732b628d8b5c67e7497283001f0ce2.png]]



### 例子

- 直接对一个地址进行扫描:

```bash
nmap [host_name | ip_address]
```

- 列出扫描的更详细情况:

```bash
nmap -v [host_name | ip_address]
```

- 扫描多个地址:

```bash
nmap [host_name | ip_address] [...] ...
```

- 扫描整个子网

```bash
nmap 129.168.3.*
nmap 129.168.1.1/24
```

- 扫描哦多台服务器

```bash
nmap 192.168.0.101,102,103
```

- 扫描由文件提供的IP和hostname, 文件格式是每一行一个host或ip

```bash
nmap -iL [file_path]
```

- 扫描IP段

```bash
nmap 192.168.0.101-110
```

- 扫描顺便检测对端服务器的OS Version和开启的服务

```bash
nmap -A [hostname | ipaddress]

# 也可以用以下-O来开启探测OS Info
nmap -O [hostname | ip]
```

- 扫描顺带探测防火墙

```bash
nmap -sA 192.168.0.101
```

- 扫描host是否存活开启

```bash
nmap -sP [ip address]
```

- 快速扫描

```
nmap -F [ip addr]
```


- 打印本机的接口和路由信息

```nmap
nmap --iflist
```

- 扫描某个特定地址的特定端口

```bash
nmap -p 80 [hostname | ipaddr]
```

- 扫描指定TCP端口

```bash
nmap -p T:[port1],[port2]... [hostname | ip]
```

- 扫描一个UDP端口

```bash
nmap -sU 53 [hostname | ipaddr]
```

- 扫描多端口

```bash
nmap -p [port1],[port2]... [ip addr ]
```

- 扫描端口范围

```bash
nmap -p 80-180 [ip addr]
```

- 扫描远程主机的开启服务，并报告其服务版本号

```bash
nmap -sV [ip addr]
```

- 使用TCP Ack和TCP Sync的方式扫描

```bash
# TCP ACK
nmap -sA [ip addr]
# TCP Sync
nmap -sS [ip addr]
```

- 使用NULL 扫描

```bash
nmap -sN [ip addr]
```


