## 前言

写这个的目的是为了平时没事的时候，用GNS3这样的网络仿真环境以弥补自己网络知识上的不足。所以可能会是一个长期的学习教程型系列文章。为什么用GNS3，不用ns3？因为GNS3是带图形界面的，容易展示网络拓扑。ns3大部分是命令行，配置网络是直接写C++代码，不是拖动图形界面上的节点控件。所以ns3入门门槛也更高，也支持更复杂各种各样的网络，比如卫星网络，5G网络等等你可能想到的都可以，但是，我的规划不是成为网络专家，所以GNS3也达到我目的了。据说用GNS3拿来学习CCNA， CCIE的考试也足够了？  请考过相关网络认证考试的大神指点一下。

好了，不多说了。

## 正文

因为GNS3推荐用MacOS或者windows，所以为了不遇到坑，我就不在我的WLS2上的ubuntu上安装了。直接安装在windows上。

首先我们先来认识一下VLAN是什么东西，VLAN是虚拟局域网，也就是不是物理上(路由器)分割的，是通过设备里面的软件来实现的，它是逻辑上的。VLAN可以在一个物理LAN内划分出多个子网络，这个些子网络是互相隔离的，即使在同一个网段内！如果要链接这些子网，必须通过路由机制, VLAN间的通信也靠三层交换机，或者三层路由器来实现路由。这样可以用来阻止广播风暴，隔离广播域，也对一些企业的组织架构更加灵活方便划分网络。


好了，我们来配置一个简单的用交换机链接的二层网络，用3台PC主机通过以太网线链接到一个以太网交换机(L2设备)上。并启动所有节点设备。是下图这样:

![image](https://user-images.githubusercontent.com/8574915/173171998-d9f08b96-4c0d-40d7-a25d-f35bbd1cc6a1.png)

![image](https://user-images.githubusercontent.com/8574915/173173302-f1b53315-2d68-44c6-a758-6b22ac8c68dd.png)


启动所有节点是点击那个绿色的三角形按钮。可以看到PC1是连接到交换机的port 0端口上， PC2是port 1，依次类推。

然后分别右键这三个VPCs节点，进入节点的console里面，如果不知道怎么用，打?号，看帮助。我们需要给每个PC主机配置一个自己的IP地址，子网掩码和网关。

```bash
PC1> ip 192.168.25.2 24 192.168.25.254
Checking for duplicate address...
PC1 : 192.168.25.2 255.255.255.0 gateway 192.168.25.254
PC2> ip 192.168.25.3 24 192.168.25.254
Checking for duplicate address...
PC1 : 192.168.25.3 255.255.255.0 gateway 192.168.25.254
PC3> ip 192.168.25.4 24 192.168.25.254
Checking for duplicate address...
PC1 : 192.168.25.4 255.255.255.0 gateway 192.168.25.254

```

PC1的IP是 192.168.25.2  子网掩码是24，网关都是一样的129.168.25.254
PC2的IP是192.168.25.3
PC3是192.168.25.4

这样测试一下，这三台主机就互通了，因为在一个网络（局域网）内了，我们可以用ping来测试一下:

```bash
PC3> ping 192.168.25.2
84 bytes from 192.168.25.2 icmp_seq=1 ttl=64 time=0.940 ms
84 bytes from 192.168.25.2 icmp_seq=2 ttl=64 time=1.070 ms
84 bytes from 192.168.25.2 icmp_seq=3 ttl=64 time=0.778 ms
PC1> ping 192.168.25.4
84 bytes from 192.168.25.4 icmp_seq=1 ttl=64 time=0.752 ms
84 bytes from 192.168.25.4 icmp_seq=2 ttl=64 time=0.611 ms
84 bytes from 192.168.25.4 icmp_seq=3 ttl=64 time=0.677 ms
PC2> ping 192.168.25.4
84 bytes from 192.168.25.4 icmp_seq=1 ttl=64 time=1.316 ms
84 bytes from 192.168.25.4 icmp_seq=2 ttl=64 time=1.198 ms
```

好了，到目前为止，这3台PC机器组成了一个物理上的LAN网络。我们现在需要进入交换机里，通过交换机提供的功能配置VLAN。我们目前打算划分2个VLAN，PC1和PC2属于VLAN 1， PC3 属于VLAN 2。  比如在一个公司中，PC1和2是研发部，PC3是财务部。为了部门之间的数据安全，简单用VLAN进行隔离这两个网络。


双击交换机节点，进入交换机的GUI配置:


![image](https://user-images.githubusercontent.com/8574915/173172514-c7cf2aca-68b9-4324-9254-e830ed94b7f3.png)

接下来我们要通过端口来划分VLAN，根据之前的拓扑图，为了实现需求，我们就需要把port 0 到 port 1都划分为VLAN 1，port 2是VLAN 2。配置修改成以下这样:

![image](https://user-images.githubusercontent.com/8574915/173172706-6edcaa7c-2821-486a-9624-a937bb84e24e.png)


好了，接下来，我们来测试下，我们的VLAN功能是否配置成功了:

```bash
PC1> ping 192.168.25.3
84 bytes from 192.168.25.3 icmp_seq=1 ttl=64 time=0.538 ms
84 bytes from 192.168.25.3 icmp_seq=2 ttl=64 time=0.892 ms

PC1> ping 192.168.25.4
host (192.168.25.4) not reachable

PC2>  ping 192.168.25.2
84 bytes from 192.168.25.2 icmp_seq=1 ttl=64 time=0.984 ms
84 bytes from 192.168.25.2 icmp_seq=2 ttl=64 time=0.944 ms
84 bytes from 192.168.25.2 icmp_seq=3 ttl=64 time=0.957 ms

PC2>  ping 192.168.25.4
host (192.168.25.4) not reachable

PC3>  ping 192.168.25.2
host (192.168.25.2) not reachable

PC3>  ping 192.168.25.3
host (192.168.25.3) not reachable

```

从以上ping的结果来看，PC1和PC2可以互通。PC3 和 PC1 - PC2之间网络不通。这样，我们的目的就达成了。

如果细心的人可能会发现，这个交换机界面上的port目前设置的都是access port，这个是什么意思呢？这个就涉及VLAN的另一个知识点了，因为一个交换机就那么多口，明显不够用嘛，不然这个划分VLAN很小，无法扩展啊。所以VLAN的port就有了另一种叫trunk port类型的端口，一般access port链接终端网络设备，trunk port链接交换机这个就可以扩展了。

EOF
