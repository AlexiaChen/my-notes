### 最简单的DDoS检测和阻止攻击

DDoS的原理和DoS一样，只是用了大量的机器分布式地发起攻击。大量发起的无用的网络包，比如大量无用的ICMP，UDP，Http,  DNS,TCP-SYN包，反正各种各样的协议包可能都用。我提到的都是常见的。

业内牛逼的可能都是用大数据一些AI模型来评估，检测DDoS, 一整套自动监控自动开启防火墙策略的自动化软件 。我们这里要讲解一个简单的方法，只是起到一个引导作用，全用手工来做。

##### 检测评估是否被DDoS

首先检测当前时刻某些共同的子网(一般是/16，/24最常见)对本服务器发起的连接数

```bash
# subnet /16
netstat -ntu|awk '{print $5}'|cut -d: -f1 -s |cut -f1,2 -d'.'|sed 's/$/.0.0/'|sort|uniq -c|sort -nk1 -r
# subnet /24
netstat -ntu|awk '{print $5}'|cut -d: -f1 -s |cut -f1,2,3 -d'.'|sed 's/$/.0/'|sort|uniq -c|sort -nk1 -r
```

如果正在发生，那么以上命令会显示类似的结果:

![[Pasted image 20220817103736.png]]

你看到了，以上图片，当前有来自于192.168.0.0/16的子网的incoming连接有13个。

这里的13个并不多，如果你发现有当前有大量连接访问那么就可以猜测是被DDoS了。

还有一个命令是列出连接到本服务器的所有IP地址:

```bash
netstat -anp |grep 'tcp|udp' | awk '{print $5}' | cut -d: -f1 | sort | uniq -c
```

用netstat计算统计出每个IP地址对本服务器发起的连接数量:

```bash
sudo netstat -ntu | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -n
```

通过这几个命令你自己就可以根据经验做出是否被DDoS的评估，如果攻击方没有用什么特殊方法来隐藏的话。

##### 怎样阻止攻击

最简单粗暴的就是经过上一步的评估和分析后，直接用iptables阻止某些希望下的incoming的数据，把它丢弃。

```bash
sudo iptables -A INPUT -s IPAddress/subbet -j DROP
```

