
## 概览

### 我们到底在构建什么

基础设施即服务（IaaS）云平台的构建可能是一项复杂的任务，根据定义，它们有很多选项，这常常会让即使是经验丰富的管理员在构建云平台时感到困惑。本操作手册旨在提供一套简明扼要的指南，以最小化麻烦地帮助您快速上手并运行CloudStack。

> 本指南仅用于构建CloudStack测试/演示云，因为已经做出了某些网络选择，以便在最短时间内让您快速上手。此指南不能用于生产环境设置。

> 如果您没有物理服务器可以“玩耍”，您可以使用例如Oracle VirtualBox 6.1+。要求是在实例的设置页面上将“启用嵌套VT-x/AMD-V”作为扩展功能启用。您需要创建一个“Red Hat（64位）”类型的实例和40GB以上的磁盘空间。您需要在实例中有1个网卡，桥接到笔记本电脑/台式机的网卡（无线或有线网卡都可以），并且最好将适配器类型设置为“虚拟化网络（virtio-net）”，以获得更好的网络性能（实例设置，网络部分，Adapter1，展开“高级”）。确保实例上的网卡配置为混杂模式（在VirtualBox中选择“允许全部”或仅选择“允许实例”作为混杂模式），以便它可以将流量从CloudStack系统VM传递到网关。此外，请确保您已经允许足够多的内存（6G+）和足够多的CPU核心数（3+）进行演示目的。

### 安装流程的高层次概览

本指南将重点介绍在CentOS 7.9上使用KVM构建CloudStack云，并使用NFS存储和VLAN进行二层隔离（也可以使用扁平家庭网络），并且仅在一台硬件设备（服务器/虚拟机）上进行。

KVM，即基于内核的虚拟机，是Linux内核的一种虚拟化技术。KVM支持在具有硬件虚拟化扩展的处理器上进行本地虚拟化。

### 前序要求

要完成本指南，您需要以下物品：

- 至少一台支持并启用硬件虚拟化的计算机。
- 一个 CentOS 7.9 minimal x86_64 安装 ISO，存储在可引导介质上。
- 一个 /24 网络，网关位于（例如）xxx.xxx.xxx.1，此网络不需要 DHCP，并且运行 CloudStack 的计算机都不会有动态地址。再次强调这是为了简单起见。

## 环境

### 操作系统

使用CentOS 7.9.2009 minimal x86_64安装ISO，您需要在硬件上安装CentOS 7。默认设置通常适用于此安装-但请确保配置IP地址/参数，以便稍后从互联网安装所需的软件包。稍后，我们将根据需要更改网络配置。

完成此安装后，您将希望通过SSH访问服务器。

开始之前始终更新系统是明智的：

```bash
yum -y upgrade
```

#### 配置网络

在继续之前，请确保已安装并可用“bridge-utils”和“net-tools”。

```bash
yum install bridge-utils net-tools -y
```

通过控制台或SSH连接，您应该以root身份登录。我们将从创建Cloudstack用于网络的桥接开始。创建并打开/etc/sysconfig/network-scripts/ifcfg-cloudbr0，并添加以下设置：

```conf
DEVICE=cloudbr0
TYPE=Bridge
ONBOOT=yes
BOOTPROTO=static
IPV6INIT=no
IPV6_AUTOCONF=no
DELAY=5
IPADDR=172.16.10.2 #(or e.g. 192.168.1.2)
GATEWAY=172.16.10.1 #(or e.g. 192.168.1.1 - this would be your physical/home router)
NETMASK=255.255.255.0
DNS1=8.8.8.8
DNS2=8.8.4.4
STP=yes
USERCTL=no
NM_CONTROLLED=no
```

> IP地址分配 - 在本文档中，我们假设您的CloudStack实施将使用/24网络。这可以是任何RFC 1918网络。但是，我们假设您将匹配我们正在使用的机器地址。因此，我们可能会使用172.16.10.2，并且由于您可能在192.168.55.0/24网络上使用例如192.168.55.2。另一个例子是如果您在本地家庭网络上使用即VirtualBox，在192.168.1.0/24网络上-在这种情况下，您可以从家庭范围内选择一个空闲IP地址（此实例的VirtualBox NIC应处于桥接模式以正确运行）。

保存配置并退出。然后我们将编辑网络接口卡，使其使用此桥接。

打开您的网络接口卡配置文件（例如：/etc/sysconfig/network-scripts/ifcfg-eth0），并按照以下方式进行编辑：

```conf
TYPE=Ethernet
BOOTPROTO=none
DEFROUTE=yes
NAME=eth0
DEVICE=eth0
ONBOOT=yes
BRIDGE=cloudbr0
```

> 接口名称（eth0）仅作为示例。请将eth0替换为您的默认以太网接口名称。

> 如果您的物理网卡（在我们的示例中为eth0）在按照本指南之前已经设置过，请确保/etc/config/network-scripts/ifcfg-cloudbr0和/etc/sysconfig/network-scripts/ifcfg-eth0的IP配置没有重复，否则会导致网络无法启动。基本上，应将eth0的IP配置移至桥接器，并将eth0添加到桥接器中。

既然我们已经正确设置了配置文件，现在需要运行一些命令来启动网络：

```bash
systemctl disable NetworkManager; systemctl stop NetworkManager
systemctl enable network
reboot
```
#### 主机名(Hostname)

CloudStack要求主机名正确设置。如果您在安装过程中使用了默认选项，则当前的主机名设置为localhost.localdomain。为了测试这个，我们将运行：

```bash
hostname --fqdn
```

这时候，会返回类似以下内容:

```txt
localhost
```

为了纠正这种情况 - 我们将通过编辑 /etc/hosts 文件来设置主机名，使其遵循类似于此示例的格式（记得用你的 IP 替换 IP，例如 192.168.1.2）：

```bash
127.0.0.1 localhost localhost.localdomain localhost4 localhost4.localdomain4
::1 localhost localhost.localdomain localhost6 localhost6.localdomain6
172.16.10.2 srvr1.cloud.priv
```

在您修改了该文件之后，继续使用以下命令重新启动网络：

```bash
systemctl enable network
# 再次检查修改是否生效
hostname --fqdn
```

#### SELinux

目前，为了使CloudStack正常工作，必须将SELinux设置为宽容或禁用。我们希望在未来的启动中配置此项，并修改当前运行的系统。

要在运行中的系统中配置SELinux为宽容模式，需要执行以下命令：

```bash
setenforce 0
```

为了确保它保持(持久化)在那个宽容状态，我们需要配置文件/etc/selinux/config以反映宽容状态，如此示例所示：

```conf
# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
# enforcing - SELinux security policy is enforced.
# permissive - SELinux prints warnings instead of enforcing.
# disabled - No SELinux policy is loaded.
SELINUX=permissive
# SELINUXTYPE= can take one of these two values:
# targeted - Targeted processes are protected,
# mls - Multi Level Security protection.
SELINUXTYPE=targeted
```
#### NTP

NTP配置是保持云服务器中所有时钟同步的必要条件。然而，默认情况下并未安装NTP。因此，我们将在此阶段安装和配置NTP。安装步骤如下：

```bash
yum -y install ntp
```

实际的默认配置对我们的目的来说很好，所以我们只需要按照以下步骤启用它并设置为开机自启动：

```bash
systemctl enable ntpd
systemctl start ntpd
```

#### 配置CloudStack的包仓库

我们需要配置机器以使用CloudStack软件包仓库。

> Apache CloudStack官方发布的是源代码。因此，没有“官方”的可执行文件可用。完整的安装指南描述了如何获取源代码并生成RPM包和yum仓库。本指南试图尽可能简化操作，并且使用了社区提供的一个yum仓库作为示例。此外，该示例假设进行的是4.19.0.0版本的Cloudstack安装 - 根据需要替换版本号即可。

要添加CloudStack存储库，请创建/etc/yum.repos.d/cloudstack.repo文件，并插入以下信息。

```conf
[cloudstack]
name=cloudstack
baseurl=http://download.cloudstack.org/centos/$releasever/4.19/
enabled=1
gpgcheck=0
```

### NFS

我们的配置将使用NFS作为主要和次要存储。我们打算为这些目的设置两个NFS共享。首先，我们将安装nfs-utils。

```bash
yum -y install nfs-utils
```

我们现在需要配置NFS来提供两个不同的共享。这是在/etc/exports文件中处理的。您应该确保它具有以下内容：

```txt
/export/secondary *(rw,async,no_root_squash,no_subtree_check)
/export/primary *(rw,async,no_root_squash,no_subtree_check)
```

您会注意到我们指定了两个系统上尚不存在的目录。我们将继续使用以下命令创建这些目录并适当设置权限：

```bash
mkdir -p /export/primary
mkdir -p /export/secondary
```

CentOS 7.x版本默认使用NFSv4。 NFSv4要求所有客户端的域设置匹配。在我们的情况下，域是cloud.priv，请确保/etc/idmapd.conf中的域设置已取消注释，并设置如下：

```conf
Domain = cloud.priv
```

现在您需要在文件/etc/sysconfig/nfs的底部添加配置值（或仅取消注释并设置它们）。

```conf
LOCKD_TCPPORT=32803
LOCKD_UDPPORT=32769
MOUNTD_PORT=892
RQUOTAD_PORT=875
STATD_PORT=662
STATD_OUTGOING_PORT=2020
```

为了简单起见，我们需要禁用防火墙，这样它就不会阻止连接。

> CentOS7上的防火墙配置超出了本指南的范围。

使用以下两个命令关闭防火墙:

```bash
systemctl stop firewalld
systemctl disable firewalld
```

我们现在需要配置nfs服务在启动时自动启动，并通过执行以下命令在主机上实际启动它：

```bash
systemctl enable rpcbind
systemctl enable nfs
systemctl start rpcbind
systemctl start nfs
```
## 管理服务器的安装

我们将安装CloudStack管理服务器和周边工具。
### 数据库安装和配置

我们将从安装MySQL并配置一些选项开始，以确保它与CloudStack良好运行。

首先，由于CentOS 7不再提供MySQL二进制文件，我们需要添加一个MySQL社区存储库，该存储库将提供MySQL服务器（以及稍后的Python MySQL连接器）：

```bash
yum -y install wget
wget http://repo.mysql.com/mysql-community-release-el7-5.noarch.rpm
rpm -ivh mysql-community-release-el7-5.noarch.rpm
yum -y install mysql-server
```

根据撰写本指南时的情况，应安装MySQL 5.x版本。现在已经安装了MySQL，我们需要对/etc/my.cnf进行一些配置更改。具体来说，我们需要将以下选项添加到\[mysqld\]部分：

```conf
innodb_rollback_on_timeout=1
innodb_lock_wait_timeout=600
max_connections=350
log-bin=mysql-bin
binlog-format = 'ROW'
```

> 对于Ubuntu 16.04及更高版本，请确保在您的`.cnf`文件中指定一个`server-id`以进行bin log记录。根据您的数据库设置，设置`server-id`。

```conf
server-id=source-01
innodb_rollback_on_timeout=1
innodb_lock_wait_timeout=600
max_connections=350
log-bin=mysql-bin
binlog-format = 'ROW'
```

既然MySQL已经正确配置好了，我们可以启动它并将其配置为开机自启动，具体操作如下：

```bash
systemctl enable mysqld
systemctl start mysqld
```
### MySQL Connector安装

从MySQL社区存储库中安装Python MySQL连接器（我们之前添加过的）。

```bash
yum -y install mysql-connector-python
```

请注意，以前需要的`mysql-connector-java`库现在已经与CloudStack管理服务器捆绑在一起，不再需要单独安装。

### 安装

我们现在要安装管理服务器。通过执行以下命令来完成：

```bash
 yum -y install cloudstack-management
```

CloudStack 4.19需要Java 11 JRE。安装管理服务器将自动安装 Java 11，但最好明确确认 Java 11 是选定的/激活的（如果您已经安装了以前的 Java 版本）：

```bash
alternatives --config java
```

确保选中 Java 11。

安装应用程序本身后，我们现在可以设置数据库，我们将使用以下命令和选项来执行此操作：

```bash
cloudstack-setup-databases cloud:password@localhost --deploy-as=root
```

当这个过程完成后，你应该会看到一条消息，比如“CloudStack已经成功初始化了数据库”。

现在，数据库已创建，我们可以通过发出以下命令来执行设置管理服务器的最后一步：

```bash
cloudstack-setup-management
```

### 系统模板设置

CloudStack使用大量的系统虚拟机来提供访问实例控制台、提供各种网络服务以及管理存储各个方面的功能。

我们需要下载 systemVM 模板并将其部署到次要存储。我们将使用本地路径 （/export/secondary），因为我们已经在 NFS 服务器上，但否则您需要将次要存储挂载到临时挂载点，并使用该挂载点而不是 /export/secondary 路径。

执行以下脚本：

```bash
/usr/share/cloudstack-common/scripts/storage/secondary/cloud-install-sys-tmplt -m /export/secondary -u [http://download.cloudstack.org/systemvm/4.19/systemvmtemplate-4.19.0-kvm.qcow2.bz2](http://download.cloudstack.org/systemvm/4.19/systemvmtemplate-4.19.0-kvm.qcow2.bz2) -h kvm -F
```

管理服务器的设置到此结束。我们仍然需要配置CloudStack，但是我们会在设置好虚拟机管理程序之后再做。

## KVM的设置和安装

### 预先要求

我们还使用管理服务器作为计算节点，这意味着在设置管理服务器时，我们已经执行了许多先决条件步骤，但为了清楚起见，我们将在此处列出它们。这些步骤是：

- [[#配置网络]]
- [[#主机名(Hostname)]]
- [[#NTP]]
- [[#配置CloudStack的包仓库]]

您现在不需要对管理服务器执行这些操作，因为我们已经这样做了。
### 安装

只需一个命令即可安装 KVM 代理，但之后我们需要配置一些东西。我们还需要安装 EPEL 存储库。

```bash
yum -y install epel-release
yum -y install cloudstack-agent
```
### KVM 配置

我们有两个不同的 KVM 部分需要配置，libvirt 和 QEMU。
#### QEMU配置

我们需要编辑 QEMU VNC 配置。这是通过编辑 /etc/libvirt/qemu.conf 并确保以下行存在且取消注释来完成的。

```conf
vnc_listen=0.0.0.0
```
#### Libvirt配置

CloudStack使用libvirt来管理实例。因此，正确配置 libvirt 至关重要。Libvirt 是 cloud-agent 的依赖项，应该已经安装好了。

1. 即使我们使用的是单个主机，也建议执行以下步骤来获得符合一般要求的 faimilar。为了使实时迁移正常工作，libvirt 必须侦听不安全的 TCP 连接。我们还需要关闭 libvirts 尝试使用组播 DNS 广告。这两个设置都在 /etc/libvirt/libvirtd.conf 中

    设置以下参数：

      ```conf
      listen_tls = 0
      listen_tcp = 1
      tcp_port = "16509"
      auth_tcp = "none"
      mdns_adv = 0
      ```

2. 在 libvirtd.conf 中打开“listen_tcp”是不够的，我们还必须更改参数，我们还需要修改 /etc/sysconfig/libvirtd：

取消注释以下行：

```conf
#LIBVIRTD_ARGS="--listen"
```

3. 重启libvirt:

```bash
systemctl restart libvirtd
```
#### KVM配置完成

为了完整起见，您应该检查 KVM 是否在您的机器上运行正常（您应该看到kvm_intel或kvm_amd模块显示为已加载）：

```bash
~# lsmod | grep kvm
kvm_intel              55496  0
kvm                   337772  1 kvm_intel
kvm_amd # if you are in AMD cpu
```

KVM的安装和配置到此结束，现在我们将转向使用CloudStack UI来实际配置我们的云。
## 配置

### 访问CloudStack的UI

要访问CloudStack的Web界面，只需将浏览器指向机器的IP地址，例如 http://172.16.10.2:8080/client 默认用户名是'admin'，默认密码是'password'。

## 建立一个Zone区域

### Zone的类型

Zone区域是CloudStack中最大的组织实体，我们将创建一个。

> 我们将配置一个高级Zone区域，允许我们同时访问云的“管理”网络和“公共”网络 - 我们将通过对“管理”（Pod）和“公共”网络使用相同的 CIDR（但其不同部分，即不同的 IP 范围）来做到这一点 - 这是您在生产中永远不会做的事情 - 这仅出于本指南中的测试目的而完成！

单击“继续安装”以继续 - 您将被要求更改您的root管理员密码 - 请这样做，然后单击确定。

将弹出一个新的Zone区域向导。请选择“高级”（不要勾选“安全组”），然后单击“下一步”。
### Zone的详情

在此页面上，我们输入DNS服务器所在的位置。CloudStack区分内部DNS和公共DNS。假定内部 DNS 能够解析仅限内部的主机名，例如 NFS 服务器的 DNS 名称。向访客实例提供公有 DNS 以解析公有 IP 地址。您可以为这两种类型输入相同的 DNS 服务器，但如果这样做，则必须确保内部和公共 IP 地址都可以路由到 DNS 服务器。在我们的特定情况下，我们不会在内部使用任何资源名称，并且我们确实会将它们设置为查找相同的外部资源，以免将nameserver设置添加到我们的需求列表中。

- Name - 我们将此设置为云的永远描述性的“Zone1”。
- IPv4 DNS 1 - 我们将云设置为 8.8.8.8。
- IPV4 DNS 2 - 我们将云设置为 8.8.4.4。
- Internal DNS1 - 我们还将云 设置为 8.8.8.8。
- Internal DNS2 - 我们还将云 设置为 8.8.4.4。
- Hypervisor - 这将是此Zone区域中使用的主要虚拟机管理程序。在我们的例子中，我们将选择 KVM。

点击Next继续。

### 物理网络

Cloudstack支持多种网络隔离方法。默认 VLAN 选项对于我们的目的来说已经足够了。为了提高性能和/或安全性，Cloudstack允许不同的流量类型在连接到Hypervisor的专用网络接口卡上运行。我们不会在这里做任何改变，默认设置对于Cloudstack的这个演示安装是可以的。

单击“Next”继续。

### 公共流量

在普通/公有云安装中，必须为此目的分配可公开访问的 IPs， 但由于我们只是部署一个演示/测试环境，我们将使用我们本地网络的一部分（例如，从 .11 到 .20 或其他自由范围）

- 网关 - 我们将使用 172.16.10.1 # or 您的物理网关，例如 192.168.1.1
- 网络掩码 - 我们将使用 255.255.255.0
- VLAN/VNI - 我们将这个留空
- 开始 IP - 我们将使用 172.16.10.11 # （或例如 192.168.1.11）
- 结束 IP - 我们将使用 172.16.10.20 # （或例如 192.168.1.20）

单击“添加”以添加范围。

单击“下一步”继续。
### Pod的配置

在这里，我们将为Cloudstack的内部管理流量配置一个范围——CloudStack会将这个范围的IP分配给系统虚拟机。这也将成为我们本地网络的一部分（即本地家庭网络的不同部分，从 .21 到 .30），其余 IP 参数 （netmaks/gateway） 与用于公共流量的参数相同。

- Pod 名称 - 我们将 Pod1 用于我们的云。
- 保留的系统网关 - 我们将使用 172.16.10.1 # （或任何物理网关，例如 192.168.1.1）
- 保留的系统网络掩码 - 我们将使用 255.255.255.0
- 开始保留的系统 IP - 我们将使用 172.16.10.21 # （或例如 192.168.1.21）
- 结束保留的系统 IP - 我们将使用 172.16.10.30 # （或例如 192.168.1.30）

点击“下一步”继续
### 访客流量

接下来，我们将为访客实例配置一系列 VLAN ID。

100 - 200 的范围就足够了。

单击“下一步”继续。

### 集群

一个 Pod 可以有多个集群，一个集群可以有多个主机。我们将有一个集群，我们必须为我们的集群命名。

输入 Cluster1

单击“下一步”继续。

### 主机(Host)

在这里，我们指定了hypervisor主机的详细信息。在我们的例子中，我们在将用作hypervisor的同一台机器上运行管理服务器。

- 主机名 - 我们将使用 IP 地址 172.16.10.2，因为我们没有设置用于名称解析的 DNS 服务器。（这是您的本地服务器，因此请使用正确的 IP 进行替换）
- 用户名 - 我们将使用 root
- 密码 - 输入 root 用户的操作系统密码

单击“下一步”继续。

#### 主存储

现在设置好集群后 - 系统应提示您输入主存储信息。在字段中输入以下值：

- 名称 - 我们将使用 Primary1
- 范围(Scope) - 我们将使用 Cluster，即使在这种情况下两者都很好。使用“Zone”范围时，所有群集中的所有主机都可以访问此存储池。
- 协议 - 我们将使用 NFS
- 服务器 - 我们将使用 IP 地址 172.16.10.2（这是您的本地服务器，因此请与正确的 IP 替换）
- 路径 - 很好地定义 /export/primary 作为我们正在使用的路径

单击“下一步”继续。

#### 次要存储

系统将提示你输入次要存储信息 - 按如下方式填充它：

- Provider - 选择 NFS
- 名称 - Secondary1
- NFS 服务器 - 我们将使用 IP 地址 172.16.10.2（这是您的本地服务器，因此请使用正确的 IP）
- 路径 - 我们将使用 /export/secondary

单击“下一步”继续。

现在，单击“Launch Zone”，您的云应该开始创建 - 创建可能需要几分钟才能完成。

完成后，单击“Enable Zone”，您的区域将准备就绪。

就是这样，你已经完成了Apache CloudStack演示云的安装。

要检查你的CloudStack安装的健康状况，请转到Infrasture - > System VMs，不时刷新用户界面 - 你应该在State=Running和Agent State=Up中看到“S-1-VM”和“V-2-VM”系统虚拟机（SSVM和CPVM） 之后，你可以进入Images –>Templates，点击名为“CentOS 5.5（64位）无GUI （KVM）”的内置模板， 然后单击“Zones”选项卡 - 并观察状态如何从下载的百分之几移动到完全下载，之后状态将显示为“下载完成”，而“就绪”列将显示为“是”。完成此操作后，您将能够从此模板部署实例。