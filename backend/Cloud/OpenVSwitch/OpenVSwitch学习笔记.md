
## 简介

OVS是一个高性能的L2软交换机（VLAN标签，MAC地址学习，），当然，它也可以工作在L3层，作为一个软路由。跨平台，跨系统。并且也支持各种网络设备协议。比如:

- NetFlow, 思科开发的收集IP流量信息，监控网络流量的协议。可以监控流量模式和流量大小。
- sFlow，也是一种监控网络流量的协议，主要是捕获网络包和网卡counter等。
- IPFIX，是一种IP流量导出协议，可以把路由器，交换机和其他网路设备的流量信息导出。基于NetFlow，由于是IETF标准支持更加灵活的数据结构
- RSPAN，思科的功能，允许网络流量从一个源端口被镜像到另一个交换机上的一个目标端口。用作远程网络监控
- LACP，链路聚合控制协议，把多个物理链路聚合成一个逻辑链路，提高带宽和冗余。
- 802.1ag，连接错误管理。一个IEEE标准，包括检测，校验，隔离连接错误（以太网）。网络监控和诊断。

> 以上主要是支持一些标准的网络监控和诊断协议。

OVS也提供以下丰富的网络功能：

- VLAN支持，802.1Q VLAN model。
- L2上的STP等，也就是避免L2层的网络环路。
- 网络上的QoS机制
- 流量策略，比如控制虚拟机之间的流量速率
- 支持Open'Flow协议，这也是在软件定义网络（SDN）中的一个主流协议，也就是控制面和数据面分离。
- IPv6的网络支持
- 隧道协议，支持GRE，VxLAN等协议。 也就是说你可以利用OVS建立云中的网络，也就是VPC这个概念的技术基石。这里的隧道协议，在容器网络中，比如k8s的网络解决方案（Calico，Flannel）中也有用到。
- 数据包转发。
- 多表转发流水线

![[Pasted image 20240927105112.png]]

## 为什么需要OpenVSwitch

Hypervisior程序需要能够在虚拟机之间以及与外部世界（宿主机，Internet）之间建立流量桥梁。在基于Linux的管理程序上，这过去意味着使用内置的L2交换机（Linux bridge），它快速可靠。因此，有理由问为什么使用Open vSwitch。

答案是Open vSwitch针对的是多服务器虚拟化部署（可以简单理解成云厂商的数据中心，成千上万的虚拟机在物理机上运行），而之前的堆栈并不适合这种环境。这些环境通常以高度动态的网络端点、逻辑抽象的维护以及（有时）与专用交换硬件的集成或卸载为特征。

以下特性和设计考虑有助于Open vSwitch满足上述要求。

#### 状态的流动性

与网络实体（例如虚拟机）关联的所有网络状态都应该易于识别，并且可以在不同主机之间迁移。这可能包括传统的“软状态”（如L2学习表中的条目）、L3转发状态、策略路由状态、ACL、QoS策略、监控配置（如NetFlow、IPFIX、sFlow）等。

Open vSwitch支持在实例之间配置和迁移慢速（配置）和快速网络状态。例如，如果VM在宿主机之间迁移（漂移，其实也类似k8s中的容器漂移），则不仅可以迁移相关配置（SPAN规则、ACL、QoS），还可以迁移任何实时网络状态（例如，包括可能难以重建的现有状态）。此外，Open vSwitch状态由真实数据模型进行类型化和支持，允许开发结构化自动化系统。


#### 对动态网络的响应

虚拟环境的特点通常是变化率很高。虚拟机来来往往，虚拟机在时间上前后移动，逻辑网络环境的变化等等。

Open vSwitch支持许多功能，允许网络控制系统在环境变化时做出响应和适应。这包括简单的计数和可见性支持，如NetFlow、IPFIX和sFlow。但也许更有用的是，Open vSwitch支持支持远程触发器的网络状态数据库（OVSDB）。因此，编排软件可以“监视”网络的各个方面，并在它们发生变化时做出响应。例如，这在今天被广泛用于响应和跟踪VM迁移。

Open vSwitch还支持OpenFlow作为导出远程访问以控制流量的方法。这有很多用途，包括通过检查发现或链路状态流量（如LLDP、CDP、OSPF等）进行全局网络发现。

#### 逻辑标签的维护

分布式虚拟交换机（如VMware vDS和思科的Nexus 1000V）通常通过在网络数据包中附加或操纵标签来维护网络中的逻辑上下文。这可用于唯一标识VM（以一种抵抗硬件欺骗的方式），或保存仅在逻辑域中相关的其他上下文。构建分布式虚拟交换机的大部分问题是高效和正确地管理这些标签。

Open vSwitch包括多种用于指定和维护标记规则的方法，所有这些方法都可以由远程进程访问以进行编排。此外，在许多情况下，这些标记规则以优化的形式存储，因此它们不必与重量级的网络设备耦合。例如，这允许配置、更改和迁移数千个标记或地址重映射规则。

同样，Open vSwitch支持GRE实现，可以同时处理数千个GRE隧道，并支持隧道创建、配置和拆除的远程配置。例如，这可用于连接不同数据中心中的专用VM网络。

#### 硬件集成

Open vSwitch的转发路径（内核内数据路径）旨在将数据包处理“卸载”到硬件芯片组，无论是安装在经典硬件交换机机箱中还是安装在宿主机NIC中。这允许Open vSwitch控制路径既能控制纯软件实现，也能控制硬件交换机。

有许多正在进行的努力将Open vSwitch移植到硬件芯片组。其中包括多个商用硅芯片组（博通和Marvell），以及一些特定于供应商的平台。文档中的“移植”部分讨论了如何进行这样的移植。

硬件集成的优势不仅在于虚拟化环境中的性能。如果物理交换机也公开了Open vSwitch控制抽象，那么裸机和虚拟化托管环境都可以使用相同的自动网络控制机制进行管理。

#### 总结

在许多方面，Open vSwitch针对的是设计空间中与以前的虚拟机监控程序网络堆栈不同的点，专注于大规模基于Linux的虚拟化环境中对自动化和动态网络控制的需求。

Open vSwitch的目标是使内核内代码尽可能小（这是性能所必需的），并在适用的情况下重用现有的子系统（例如Open vSwitch使用现有的QoS堆栈）。从Linux 3.3开始，Open vSwitch作为内核的一部分包含在内，大多数流行的发行版都提供了用户空间实用程序的打包。


## 安装

```bash
# Debian Ubuntu，核心的OVS用户态组件，还有文档，其他的OVS内核数据路径已经在上游的OS内核里面了
sudo apt install openvswitch-switch openvswitch-common
# 如果需要高速的用户态交换，需要DPDK的支持
sudo apt install openvswitch-switch-dpdk
```

```bash
# Red Hat,这个包就支持内核数据路径
sudo yum install openvswitch 
# DPDK的用户态支持
sudo yum install openvswitch-dpdk
sudo yum install openvswitch-ipsec
```


## 教程

### OVS Faucet教程

本教程演示了Open vSwitch如何与通用的OpenFlow控制器配合使用，使用Faucet控制器作为简单的入门方法。它使用Open vSwitch的“main”分支和Faucet的1.6.15版本进行了测试。它不使用OVS或Faucet中的高级或最近添加的功能，因此这两款软件的其他版本可能同样适用。

本教程的目标是以端到端的方式演示Open vSwitch和Faucet，即展示它是如何从顶部的Faucet控制器配置，通过OpenFlow流表(Flow Table)，到数据路径处理工作的。在此过程中，除了帮助理解每个级别的架构外，我们还讨论了性能和故障排除问题。我们希望这个演示能让用户和潜在用户更容易理解Open vSwitch的工作原理以及如何对其进行调试和故障排除。

> Faucet是一个OpenFlow控制器，提供高级的网络控制和自动化，也就是通过OpenFlow协议来控制OVS。可以通过yaml来配置网络，来执行交换，路由，管理ACLs的功能。也继承了监控等功能。

#### 设置OVS

本节解释了如何设置Open vSwitch，以便在本教程中将其与Faucet一起使用。

您可能已经在一台或多台计算机或虚拟机上安装了Open vSwitch，可能设置为控制一组虚拟机或物理网络。这令人钦佩，但我们将以不同的方式使用Open vSwitch来设置一个名为OVS“沙盒”的模拟环境。沙盒不使用虚拟机或容器，这使得它更加有限，但另一方面（在本文作者看来）它更容易设置。

有两种方法可以启动沙箱：一种使用已经安装在系统上的Open vSwitch，另一种使用已构建但尚未安装的Open vSwitch副本。后者更常用，因此经过更好的测试，但两者都应该有效。下面的说明解释了这两种方法（下面的步骤是后者），但是建立沙盒环境，我就不举例了，看文档 https://docs.openvswitch.org/en/latest/tutorials/faucet/#setting-up-ovs

我自己用第一种，就是直接通过包管理安装的OVS。

#### 设置Faucet

https://docs.openvswitch.org/en/latest/tutorials/faucet/#setting-up-faucet

#### 概览

现在Open vSwitch和Faucet已经准备好了，下面是我们在本教程剩余部分要做的事情的概述：

- 交换：使用Faucet建立L2网络。
- 路由：使用Faucet在多个L3网络之间路由。
- ACL：添加和修改访问控制规则。

在每一步中，我们将看看所讨论的功能是如何从顶部的Faucet controller（控制面）到底部的数据面层工作的。从最高到最低级别，这些层和连接它们的软件组件是：

- Faucet

作为系统的顶层，这是网络配置的权威来源。

Faucet连接到各种监控和性能工具，但我们在本教程中不会使用它们。我们对系统的主要见解将是通过faucet.yaml进行配置，并通过faucet_log观察状态，如MAC学习和ARP解析，并判断我们何时搞砸了配置语法或语义。

- OVS的OpenFlow子系统

OpenFlow是由开放网络基金会标准化的协议，像Faucet这样的控制器使用它来控制Open vSwitch和其他交换机如何处理网络中的数据包。

我们将使用Open vSwitch附带的ovs-ofctl实用程序来观察并偶尔修改Open vSwitch的OpenFlow行为。我们还将使用ovs appctl，一个用于与ovs vSwitch和其他Open vSwitch守护进程通信的实用程序，来询问“如果……会怎样？”类型的问题。

此外，默认情况下，OVS沙箱将OpenFlow的Open vSwitch日志记录级别提高到足够高的水平，这样我们就可以通过读取其日志文件来了解OpenFlow的行为。

- OVS的数据路径

这本质上是一个旨在加速数据包处理的缓存。Open vSwitch包括一些不同的数据路径，例如基于Linux内核的数据路径和仅限用户空间的数据路径（有时称为“DPDK”数据路径）。OVS沙箱使用后者，但其背后的原理同样适用于其他数据路径。

在每一步中，我们讨论每一层的设计如何影响性能。我们演示了如何使用Open vSwitch功能对整个系统进行调试、故障排除和理解。
#### 交换

第二层（L2）交换是现代网络的基础。它也非常简单，是一个很好的起点，所以让我们在Faucet中设置一个带有一些VLAN的交换机，看看它在每一层是如何工作的。首先将以下内容放入inst/fouset.yaml中：

```yaml
dps:
    switch-1:
        dp_id: 0x1
        timeout: 3600
        arp_neighbor_timeout: 3600
        interfaces:
            1:
                native_vlan: 100
            2:
                native_vlan: 100
            3:
                native_vlan: 100
            4:
                native_vlan: 200
            5:
                native_vlan: 200
vlans:
    100:
    200:
```

此配置文件定义了一个名为switch-1的交换机（“datapath”或“dp”）。交换机有5个端口，编号为1到5。端口1、2和3位于VLAN 100中，端口4和5位于VLAN 2中。Faucet可以通过其数据路径ID（定义为0x1）识别交换机。

> 这也设置了较高的MAC学习和ARP超时。默认值为5分钟和约8分钟，这在生产中很好，但有时对于手动实验来说太快了。

#### 路由

https://docs.openvswitch.org/en/latest/tutorials/faucet/#routing

#### 访问控制列表-ACLs

https://docs.openvswitch.org/en/latest/tutorials/faucet/#acls
## How-to指南

[How-to Guides — Open vSwitch 3.4.90 documentation](https://docs.openvswitch.org/en/latest/howto/)
### OVS与KVM

本文档描述了如何将Open vSwitch与基于内核的虚拟机（KVM）一起使用。

#### 设置

KVM使用tunctl来处理各种bridge模式，您可以使用Debian/Ubuntu软件包uml实用程序安装这些模式

```bash
apt-get install uml-utilities
```

接下来，您需要修改或创建qemu-ifup和qemu-ifdown脚本的自定义版本。在本指南中，我们将使用本指南中描述的Open vSwitch bridges示例创建自定义版本。

创建以下两个文件并将其存储在已知位置。例如：

```bash
$ cat << 'EOF' > /etc/ovs-ifup
#!/bin/sh

switch='br0'
ip link set $1 up
ovs-vsctl add-port ${switch} $1
EOF
```

```bash
$ cat << 'EOF' > /etc/ovs-ifdown
#!/bin/sh

switch='br0'
ip addr flush dev $1
ip link set $1 down
ovs-vsctl del-port ${switch} $1
EOF
```

```bash
# 创建一个名叫br0的网桥
ovs-vsctl add-br br0
# 然后，为您希望虚拟机通过其进行通信的NIC在网桥中添加一个端口（例如eth0）
ovs-vsctl add-port br0 eth0
# 接下来，我们将启动一个使用我们的ifup和ifdown脚本的虚拟机：
 kvm -m 512 -net nic,macaddr=00:11:22:EE:EE:EE -net \
    tap,script=/etc/ovs-ifup,downscript=/etc/ovs-ifdown -drive \
    file=/path/to/disk-image,boot=on
```

这将启动虚拟机并将一个tap设备与虚拟机关联。ovs-ifup脚本将在br0网桥上添加一个端口，以便虚拟机能够通过该网桥进行通信。

要获取更多信息并进行调试，您可以使用Open vSwitch实用程序，如ovs dpctl和ovs ofctl，例如：

```bash
ovs-dpctl show
ovs-ofctl show br0
```

您应该看到每个KVM虚拟机的tap设备都作为端口添加到网桥中（例如tap0）

> tap设备一般就是指虚拟机里面的虚拟网卡在宿主机上的表示设备，把虚拟网卡桥接到网桥上，也就是把这个tap桥接到网桥上，而宿主机上的eth0物理网卡也被桥接到网桥上，那么虚拟机的网络就与外部的宿主机打通了。


### OVS与SELinux

[Open vSwitch with SELinux — Open vSwitch 3.4.90 documentation](https://docs.openvswitch.org/en/latest/howto/selinux/)

### OVS与Libvirt

支持libvirt 0.9.11及其以上

```bash
virsh --version
libvirtd --version
```

#### 限制

目前OVS只支持libvirt的bridge网络模式，而不支持libvirt默认创建的NAT模式。使用bridge模式必须让用户手动创建一个bridge网桥设备
#### 设置

首先，使用ovs-vsctl实用程序创建Open vSwitch网桥（这必须使用管理权限完成）：

```bash
ovs-vsctl add-br ovsbr
```

```bash
# 编辑虚拟机的配置
virsh edit <vm-name>
```

找到XML配置中的interface标签，虚拟机中的每一个网卡都有一个类似以下的配置:

```xml
<interface type='network'>
 <mac address='52:54:00:71:b1:b6'/>
 <source network='default'/>
 <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
</interface>
```

修改成如下:

```xml
<interface type='bridge'>
 <mac address='52:54:00:71:b1:b6'/>
 <source bridge='ovsbr'/>
 <virtualport type='openvswitch'/>
 <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
</interface>
```

接口类型必须设置为bridge。<source>XML元素指定此接口将连接到哪个网桥设备。\<virtualport\>元素表示<source>元素中的网桥是Open vSwitch网桥。

然后（重新）启动VM并验证虚拟机的vnet接口是否连接到ovsbr网桥：

```bash
ovs-vsctl show
```

#### TroubleShooting

如果VM不想启动，请尝试从终端运行libvirtd进程，以便在控制台中打印所有错误，或者检查Libvirt/Open vSwitch日志文件以查找可能的根本原因。

### OVS与SSL

[Open vSwitch with SSL — Open vSwitch 3.4.90 documentation](https://docs.openvswitch.org/en/latest/howto/ssl/)

### 用GRE隧道连接虚拟机

本文档描述了如何使用Open vSwitch允许两个不同宿主机上的VM通过基于端口的GRE隧道进行通信。

> 本指南涵盖了配置GRE隧道所需的步骤。同样的方法可用于Open vSwitch支持的任何其他隧道协议。

![[Pasted image 20240927153534.png]]

#### 设置

假设环境是以下环境

##### 两个物理网络

也就是一台宿主机上的两个物理网卡分别接入两个不同的网络，一个传输网络，一个管理网络。

- 传输网络

以太网，用于运行OVS的主机之间的隧道流量。根据所使用的隧道协议（使用GRE），可能需要对物理交换机进行一些配置（例如，可能需要调整MTU）。物理交换硬件的配置不在本教程条目的范围内。

- 管理网络

严格来说，这个网络不是必需的，但这是一种为物理宿主机提供远程访问IP地址的简单方法，因为IP地址不能直接分配给作为OVS网桥一部分的物理接口。

##### 两个物理宿主机

环境假定使用两个宿主机，分别命名为host1和host2。这两个宿主机都是运行Open vSwitch的管理程序。每个主机都有两个物理网卡(NIC)，eth0和eth1，配置如下：

- eth0已连接到传输网络。eth0有一个IP地址，用于通过传输网络与Host2通信。
- eth1连接到管理网络。eth1有一个IP地址，用于访问物理主机进行管理。

##### 四个虚拟机

每个主机将运行两个虚拟机（VM）。vm1和vm2在主机1上运行，而vm3和vm4在主机2上运行。

每个VM都有一个在物理主机上显示为Linux设备的单一接口（例如tap0）。

> VM的虚拟网卡接口可能显示为Linux设备，名称如vnet0、vnet1等。vnet0对应tap0，vnet1对应tap1

#### 配置步骤

在开始之前，您需要确保知道在宿主机1和宿主机2上分配给eth0的IP地址（也就是传输网络上宿主机的IP地址），因为在配置过程中需要这些地址。

在宿主机1上执行以下配置。

```bash
# 创建一个OVS网桥，此时你还没有把eth0桥接到这个OVS网桥上
ovs-vsctl add-br br0
# 启动宿主机上的两台虚拟机vm1 vm2,注意，要把虚拟机的interface设备为桥接模式，指向网桥
virsh start <vm1>
virsh start <vm2>
# 如果没有在配置中桥接到网桥，那么需要手动桥接 tap0对应vm1，tap1对应vm2
ovs-vsctl add-port br0 tap0
ovs-vsctl add-port br0 tap1
# 把gre0这个port桥接到OVS网桥上，gre0这个接口其实就是GRE隧道的端点，这个隧道连接到host2的eth0的IP上
ovs-vsctl add-port br0 gre0 \
    -- set interface gre0 type=gre options:remote_ip=<IP of eth0 on host2>
# 如果你还想让你在宿主机1上的VM访问Internet，比如eth0的链路可以连接互联网
ovs-vsctl add-port br0 eth0
```

在宿主机上如法炮制以上步骤，只是GRE的remote_ip那里需要修改成\<IP of eth0 on host1\>

#### 测试

无论虚拟机是在同一台主机上运行还是在不同的主机上运行，任何虚拟机之间的Ping都应该正常工作。

使用ip route show（或等效命令），VM内运行的操作系统的路由表不应显示宿主机使用的ip子网，（也就是与宿主机不在同一个网络内）而应显示VM操作系统内配置的ip子网络。为了帮助说明这一点，最好在客户机VM中使用与宿主机上使用的IP子网分配非常不同的IP子网络分配。

#### TroubleShooting

如果不同宿主机上的VM之间的连接不起作用，请检查以下项目：

- 确保宿主机1和宿主机2通过eth0（连接到传输网络的NIC）具有完全的网络连接。这可能需要使用额外的IP路由或IP路由规则。
- 确保宿主机1上的gre0指向宿主机2上的eth0，宿主机2上得gre0指向宿主机1上得eth0。
- 确保所有虚拟机都分配了同一子网上的IP地址；此配置中没有IP路由功能。

> 以上其实就是把两台宿主机的VM之间的内部网络变成了一个大二层网络

### 用VxLAN隧道连接虚拟机-用户态空间

本文档描述了如何使用Open vSwitch允许两个不同宿主机上的VM通过VXLAN隧道进行通信。与[[#用GRE隧道连接虚拟机]] 不同，此配置完全在用户空间中工作。

> 本指南涵盖了配置VXLAN隧道所需的步骤。同样的方法可用于Open vSwitch支持的任何其他隧道协议。

![[Pasted image 20240927161909.png]]

#### 设置

本指南假设你的环境配置如下
##### 两个物理宿主机

环境假定使用两个主机，分别命名为host1和host2。我们只详细介绍了host1的配置，但host2也可以使用类似的配置。两台主机都应配置Open vSwitch（带或不带DPDK）、QEMU/KVM和合适的VM镜像。在继续之前，Open vSwitch应该正在运行。

#### 配置步骤

```bash
# 创建一个br-int网桥，netdev的dp type就表明使用用户态空间，而不是内核。主要是用于内核不支持OVS模块的情况，然后就是给网桥设置一个外部ID，这个ID的值也是叫br-int。外部ID主要是用来识别和集成外部系统和控制器的。设置网桥的失败模式是standalone，这个模式下，网桥转发数据包，即使没有控制器连接，如果设置为secure，那么必须要有控制器，才可以转发数据包
ovs-vsctl --may-exist add-br br-int \
  -- set Bridge br-int datapath_type=netdev \
  -- br-set-external-id br-int bridge-id br-int \
  -- set bridge br-int fail-mode=standalone
# 启动虚拟机，并把虚拟机对应的tap挂到网桥上
ovs-vsctl add-port br-int tap0
# 配置虚拟机里面的IP地址，要进入虚拟机里面
ip addr add 192.168.1.1/24 dev eth0
ip link set eth0 up
# 在宿主机1里面，把vxlan0这个端点挂到网桥上。172.168.1.2 is the remote tunnel end point address. On the remote host this will be 172.168.1.1
ovs-vsctl add-port br-int vxlan0 \
  -- set interface vxlan0 type=vxlan options:remote_ip=172.168.1.2
# 创建一个br-phy的网桥。在用户空间而不是基于内核的Open vSwitch中运行Open vSwitch时，需要这个额外的网桥。此网桥的目的是允许使用内核网络堆栈进行路由和ARP解析。数据路径需要查找路由表和ARP表，以准备隧道Header并将数据传输到输出端口。使用eth1而不是eth0。这是为了确保保持网络连接。
ovs-vsctl --may-exist add-br br-phy \
    -- set Bridge br-phy datapath_type=netdev \
    -- br-set-external-id br-phy bridge-id br-phy \
    -- set bridge br-phy fail-mode=standalone \
         other_config:hwaddr=<mac address of eth1 interface>

# 把eth1/dpdk0挂载到br-phy网桥上，如果物理端口eth1作为内核网络接口运行，请运行
ovs-vsctl --timeout 10 add-port br-phy eth1
ip addr add 172.168.1.1/24 dev br-phy
ip link set br-phy up
ip addr flush dev eth1 2>/dev/null
ip link set eth1 up
iptables -F

# 如果接口是DPDK接口并绑定到igb_uio或vfio驱动程序，请运行：
ovs-vsctl --timeout 10 add-port br-phy dpdk0 \
  -- set Interface dpdk0 type=dpdk options:dpdk-devargs=0000:06:00.0
ip addr add 172.168.1.1/24 dev br-phy
ip link set br-phy up
iptables -F
```

命令是不同的，因为DPDK接口不由内核管理，因此端口详细信息对任何ip命令都不可见。

> 尝试对DPDK接口使用内核网络命令将导致通过eth1的连接丢失。有关更多详细信息，请参阅 [Basic Configuration — Open vSwitch 3.4.90 documentation](https://docs.openvswitch.org/en/latest/faq/configuration/)

一旦完成以上，检查缓存的路由:

```bash
ovs-appctl ovs/route/show
```

如果隧道路由缺失，请立即添加：

```bash
ovs-appctl ovs/route/add 172.168.1.1/24 br-phy
```

在宿主机2上如法炮制以上步骤，只是remote_ip那里要换一下, 还有一些本机IP
#### 测试

使用此设置，ping到VXLAN目标设备（192.168.1.2）应该可以工作。流量将被VXLAN封装并通过eth1/dpdk0接口发送。

#### 隧道相关的命令

##### 隧道路由表

```bash
# 加入一个路由
ovs-appctl ovs/route/add <IP address>/<prefix length> <output-bridge-name> <gw>
# 查看所有路由配置
ovs-appctl ovs/route/show
# 删除路由
ovs-appctl ovs/route/del <IP address>/<prefix length>
# 查找和显示一个目标的路由
ovs-appctl ovs/route/lookup <IP address>
```

##### ARP

```bash
# 查看ARP缓存内容
ovs-appctl tnl/arp/show
# 刷新ARP缓存
 ovs-appctl tnl/arp/flush
# 设置一个指定的ARP条目
ovs-appctl tnl/arp/set <bridge> <IP address> <MAC address>
```

##### Ports

```bash
# 检查隧道端口在ovs-vswitchd的监听情况
ovs-appctl tnl/ports/show
# 设置VxLAN的UDP source port的范围
ovs-appctl tnl/egress_port_range <num1> <num2>
# 显示当前范围
ovs-appctl tnl/egress_port_range
```
##### Datapath

```bash
# 检查datapath的port
ovs-appctl dpif/show
# 检查datapath流向
ovs-appctl dpif/dump-flows
```

### 用VLAN隔离虚拟机的流量

![[Pasted image 20240927171034.png]]

#### 设置

假设你的环境配置如下：
##### 两个物理网络

- 数据网络

用于VM数据流量的以太网，它将在VM之间承载VLAN tag的流量。您的物理交换机必须能够转发VLAN标记的流量，并且物理交换机端口应作为VLAN中继运行。（通常这是默认行为。配置物理交换硬件超出了本文的范围。）

- 管理网络

该网络不是严格要求的，但它是一种为物理宿主机提供远程访问IP地址的简单方法，因为IP地址不能直接分配给eth0（稍后将详细介绍）。

##### 两个物理宿主机

环境假定使用两个宿主机：host1和host2。两台宿主机都在运行Open vSwitch。每个主机都有两个NIC，eth0和eth1，配置如下：

- eth0已连接到数据网络。没有为eth0分配IP地址。
- eth1连接到管理网络（如果需要）。eth1有一个IP地址，用于访问物理宿主机进行管理。

##### 四个虚拟机

每个宿主机将运行两个虚拟机（VM）。vm1和vm2在主机1上运行，而vm3和vm4在主机2上运行。

每个VM都有一个在物理宿主机上显示为Linux设备的单一接口（例如tap0）。

> VM接口可能显示为Linux设备，名称如vnet0、vnet1等。

#### 配置步骤

```bash
# 创建一个OVS网桥
ovs-vsctl add-br br0
# 把eth0挂到网桥上，默认情况下，所有OVS端口都是VLAN trunk口，因此eth0将通过所有VLAN。当您将eth0添加到OVS网桥时，可能分配给eth0的任何IP地址都会停止工作。在将eth0添加到OVS网桥之前，应将分配给eth0的IP地址迁移到其他接口。这就是通过eth1进行单独管理连接的原因。
ovs-vsctl add-port br0 eth0
# 将vm1添加为VLAN 100上的“访问端口”。这意味着从VM1进入OVS的流量将被取消标记，并被视为VLAN 100的一部分：
ovs-vsctl add-port br0 tap0 tag=100
# 在VLAN 200上添加VM2：
ovs-vsctl add-port br0 tap1 tag=200
```

在宿主机2上如法炮制，把vm3标记成100，vm4标记成200
#### 检验测试

- 从vm1到vm3的Ping应该成功，因为这两个VM在同一个VLAN上。
- 从vm2到vm4的Ping也应该成功，因为这些VM也在彼此相同的VLAN上。
- 从vm1/vm3到vm2/vm4的Ping不应成功，因为这些VM位于不同的VLAN上。如果你有一个配置为在VLAN之间转发的路由器，那么ping将起作用，但到达vm3的数据包应该具有路由器的源MAC地址，而不是vm1。

### 如何使用VTEP模拟器

[How to Use the VTEP Emulator — Open vSwitch 3.4.90 documentation](https://docs.openvswitch.org/en/latest/howto/vtep/)

### 使用sFlow监控虚拟机的流量

[Monitoring VM Traffic Using sFlow — Open vSwitch 3.4.90 documentation](https://docs.openvswitch.org/en/latest/howto/sflow/)

### 使用DPDK加强的OVS

[Using Open vSwitch with DPDK — Open vSwitch 3.4.90 documentation](https://docs.openvswitch.org/en/latest/howto/dpdk/)