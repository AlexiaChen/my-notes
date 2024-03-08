
## 普通安装

### 安装概述

#### 介绍

##### 谁应该读这个

对于那些已经经历过设计阶段并计划进行更复杂部署的人，或者那些准备开始扩大试验安安装 [[CloudStack快速安装指南]] 的人。通过以下步骤，你可以开始使用CloudStack更强大的功能，如高级VLAN网络、高可用性、额外的网络元素，如负载平衡器和防火墙，以及对多个虚拟机管理程序的支持，包括Citrix XenServer、KVM和VMware vSphere。

##### 安装步骤

对于简单的试用安装以外的任何内容，您将需要有关各种配置选择的指导。强烈建议您阅读以下内容：

- 选择部署体系结构  [[CloudStack的相关概念和术语#选择一个部署架构]]
- 选择虚拟机管理程序：支持的功能
- 网络设置
- 存储设置
- 最佳实践

- 确保您已准备好所需的硬件。请参阅  [[#最低系统要求]]
- 安装管理服务器（选择单节点或多节点）。请参阅  [[#管理服务器安装]]
- 配置您的云。参见  [[#配置你的CloudStack安装]]
    - 使用CloudStack UI。参见 *User Interface* ：ref：'log-in-to-ui
    - 添加Zone。包括第一个 Pod、集群和主机。请参阅 [[#添加一个Zone区域]]
    - 添加更多 Pod（可选）。请参阅  [[#添加一个Pod]]
    - 添加更多集群（可选）。请参阅  [[#添加一个集群]]
    - 添加更多主机（可选）。请参见  [[#添加一个主机]]
    - 添加更多主存储（可选）。请参见  [[#添加一个主存储]]
    - 添加更多次要存储（可选）。请参见 [[#添加一个次要存储]]

- 尝试使用云。请参阅  [[#初始化和测试]]

#### 最低系统要求

##### 管理服务器，数据库，存储系统的要求

将运行 Management Server 和 MySQL 数据库的计算机必须满足以下要求。同一台计算机还可用于提供主存储和次要存储，例如通过 localdisk 或 NFS。管理服务器可以放置在实例上。

- 操作系统：
    - 首选：CentOS/RHEL 7.2+ 或 Ubuntu 16.04（.2） 或更高版本
- 64 位 x86 CPU（内核越多，性能越好）
- 4 GB 内存
- 250 GB 本地磁盘（越多，功能越好;建议 500 GB）
- 至少 1 个 NIC
- 静态分配的 IP 地址
- hostname 命令返回的完全限定的域名

> 查看服务器版本  cat /etc/redhat-release
##### 主机/Hypervisor的系统要求

主机是云服务以访客虚拟机的形式运行的位置。每个主机都是一台满足以下要求的计算机：

- 必须支持 HVM（启用 Intel-VT 或 AMD-V）。
- 64 位 x86 CPU（内核越多，性能越好）
- 需要硬件虚拟化支持
- 4 GB 内存
- 36 GB 本地磁盘
- 至少 1 个 NIC
- 应用于虚拟机监控程序软件的最新修补程序
- 当你部署CloudStack时，Hypervisor主机不能有任何虚拟机已经运行
- 群集中的所有主机都必须是同构的。CPU 必须具有相同的类型、计数和功能标志。

主机有其他要求，具体取决于Hypervisor。请参阅所选Hypervisor的“安装”部分顶部列出的要求：

> 请确保满足本指南中提供的其他Hypervisor要求和安装步骤。Hypervisor主机必须做好充分的准备，才能与CloudStack一起工作。例如，XenServer 的要求列在 Citrix XenServer 安装下。


> egrep -o '(vmx|svm)' /proc/cpuinfo   输出vmx就是Intel-VT，输出svm就是AMD-V
#### 包仓库

CloudStack只从官方Apache镜像的源代码分发。然而，CloudStack社区的成员可以构建方便的二进制文件，这样用户就可以安装Apache CloudStack，而不需要从源代码构建。

如果您没有按照“[从源代码构建 RPM](https://docs.cloudstack.apache.org/en/4.19.0.0/installguide/building_from_source.html#building-rpms-from-source)”或“[构建 DEB 包](https://docs.cloudstack.apache.org/en/4.19.0.0/installguide/building_from_source.html#building-deb-packages)”部分中的步骤从源代码构建自己的软件包，您可能会在[下载页面]([Apache CloudStack Downloads | Apache CloudStack](https://cloudstack.apache.org/downloads/))中找到预先构建的 DEB 和 RPM 软件包，以便您方便。

> 这些存储库包含 Management Server 和 KVM Hypervisor 软件包。

### 管理服务器安装

#### 概览

本节介绍如何安装 Management Server。有两个略有不同的安装流程，具体取决于云中将有多少个管理服务器节点：

- 单个管理服务器节点，MySQL 位于同一节点上。
- 多个管理服务器节点，MySQL 位于与管理服务器分开的节点上。

无论哪种情况，每台计算机都必须满足  [[#最低系统要求]] 中所述的系统要求。

> 为了安全起见，请确保公共 Internet 无法访问管理服务器上的端口 8096 或端口 8250。

安装步骤如下:

- [[#准备操作系统]]
- [[#下载vdh-util]] (只有XenServer需求)
- [[#在第一台主机上安装管理服务器]]
- [[#安装数据库服务器]]
- [[#准备NFS Shares]]
- [[#额外的Management servers]]
- [[#准备系统VM模板]]
#### 准备操作系统

操作系统必须准备好使用以下步骤来托管管理服务器。必须在每个管理服务器节点上执行这些步骤。

- 以root用户登录
- 检查全限定域名

```bash
hostname --fqdn
```

- 确保机器能够访问互联网

```bash
ping cloudstack.apache.org
```

- 打开NTP，为了时钟同步

> 需要 NTP 守护程序来同步云中服务器的时钟。

安装chrony

```bash
# RHEL or CentOS
yum install chronysyst
systemctl enable chronyd  
systemctl start chronyd  
systemctl status chronyd
# Ubuntu
apt install chrony
# SUSE
zypeer install chrony
```

- 在每台托管管理服务器的主机上重复以上步骤

> CloudStack 4.19需要Java 11 JRE。安装CloudStack软件包将自动安装Java 11，但是在CloudStack软件包已经安装后，最好用 alternatives --config java明确确认Java 11是选定的/活动的（如果你已经安装了以前的Java版本）。
#### 在第一台主机上安装管理服务器

无论是在一台主机上还是在多台主机上安装 Management Server，安装的第一步都是在单个节点上安装软件。

> 如果计划在多个节点上安装 Management Server 以实现高可用性，请不要继续执行其他节点。这一步将在以后进行。

CloudStack管理服务器可以使用RPM或DEB软件包进行安装。这些包将取决于运行管理服务器所需的一切。

##### 配置包仓库

CloudStack只从官方Apache镜像的源代码分发。然而，CloudStack社区的成员可以构建方便的二进制文件，这样用户就可以安装Apache CloudStack，而不需要从源代码构建。

如果您没有按照“[从源代码构建 RPM](https://docs.cloudstack.apache.org/en/4.19.0.0/installguide/building_from_source.html#building-rpms-from-source)”或“[构建 DEB 包](https://docs.cloudstack.apache.org/en/4.19.0.0/installguide/building_from_source.html#building-deb-packages)”部分中的步骤从源代码构建自己的软件包，您可能会在[下载页面]([Apache CloudStack Downloads | Apache CloudStack](https://cloudstack.apache.org/downloads/))中找到预先构建的 DEB 和 RPM 软件包，以便您方便。

> 这些存储库包含 Management Server 和 KVM Hypervisor 软件包。

##### RPM包仓库

CloudStack有一个RPM软件包仓库，所以你可以很容易地安装在基于RHEL和SUSE的平台上。

如果你使用的是基于RPM的系统，你需要添加Yum存储库，这样你就可以用Yum安装CloudStack。

在 RHEL 或 CentOS 中：

Yum 存储库信息可在 `/etc/yum.repos.d` 下找到。您将在此目录中看到多个 `.repo` 文件，每个文件都表示一个特定的存储库。

要添加CloudStack仓库，请创建`/etc/yum.repos.d/cloudstack.repo`并插入以下信息。

在使用 RHEL 的情况下，您可以在 baseurl 的值中将 'centos' 替换为 'rhel'

```txt
[cloudstack]
name=cloudstack
baseurl=http://download.cloudstack.org/centos/$releasever/4.19/
enabled=1
gpgcheck=0
```

现在你应该能够使用yum来安装CloudStack了。

在 SUSE 中：

Zypper 存储库信息可在 `/etc/zypp/repos.d/` 下找到。您将在此目录中看到多个` .repo` 文件，每个文件都表示一个特定的存储库。

要添加CloudStack仓库，请创建`/etc/zypp/repos.d/cloudstack.repo`并插入以下信息。

```txt
[cloudstack]
name=cloudstack
baseurl=http://download.cloudstack.org/suse/4.19/
enabled=1
gpgcheck=0
```

现在你应该能够使用zypper来安装CloudStack了。
##### DEB包仓库

您可以使用以下命令将 DEB 软件包存储库添加到 apt 源。将代码名称替换为您的 Ubuntu LTS 版本：Ubuntu 16.04 （Xenial）、Ubuntu 18.04 （Bionic） 和 Ubuntu 20.04 （Focal）。不再支持 Ubuntu 14.04 （Trusty）。

使用您喜欢的编辑器并打开（或创建）`/etc/apt/sources.list.d/cloudstack.list`。将社区提供的存储库添加到文件中（如果是这种情况，请将“trusty”替换为“xenial”或“bionic”）：

```bash
deb https://download.cloudstack.org/ubuntu focal 4.19
```

我们现在必须将公钥添加到可信密钥中。

```bash
wget -O - https://download.cloudstack.org/release.asc |sudo tee /etc/apt/trusted.gpg.d/cloudstack.asc
```

然后更新apt cache:

```bash
apt update
```
##### 在CentOS/RHEL上安装

```bash
yum install epel-release
yum install cloudstack-management
```
##### 在SUSE上安装

```bash
zypper install cloudstack-management
```

##### 在ubuntu上安装

```bash
sudo apt install cloudstack-management
```

#### 下载vdh-util

只有在虚拟机管理程序主机上安装了 XenServer 的安装中，才需要执行此过程。

在设置管理服务器之前，请从 http://download.cloudstack.org/tools/vhd-util 下载 vhd-util。并将其复制到管理服务器的 `/usr/share/cloudstack-common/scripts/vm/hypervisor/xenserver` 中。

#### 安装数据库服务器

CloudStack管理服务器使用MySQL数据库服务器来存储其数据。在单个节点上安装管理服务器时，可以在本地安装 MySQL 服务器。对于具有多个管理服务器节点的安装，我们假设 MySQL 数据库也在单独的节点上运行。

CloudStack已经用MySQL 5.1和5.5进行了测试。这些版本包含在 RHEL/CentOS 和 Ubuntu 中。

##### 安装数据库到管理服务器节点上

本节介绍如何将MySQL与Management Server安装在同一台计算机上。此技术适用于具有单个管理服务器节点的简单部署。如果具有多节点管理服务器部署，则通常会对 MySQL 使用单独的节点。请参见在  [[#安装数据库到一个分开的节点上（多管理节点才用到这个）]]

- 从包仓库中安装MySQL

```bash
yum install mysql-server
zypper install mysql-server
sudo apt install mysql-server
```

- 打开MySQL配置文件。配置文件为 /etc/my.cnf 或 /etc/mysql/my.cnf，具体取决于您的操作系统。

在 \[mysqld\] 部分中插入以下行。

您可以将这些行放在 datadir 行的下方。`max_connections`参数应设置为 350 乘以要部署的管理服务器数。此示例假定有一个管理服务器。

```conf
innodb_rollback_on_timeout=1
innodb_lock_wait_timeout=600
max_connections=350
log-bin=mysql-bin
binlog-format='ROW'
```

> 对于 Ubuntu 16.04 及更高版本，请确保在 .cnf 文件中指定用于二进制日志记录的 server-id。根据数据库设置设置 server-id。

```conf
server-id=100
innodb_rollback_on_timeout=1
innodb_lock_wait_timeout=600
max_connections=350
log-bin=mysql-bin
binlog-format='ROW'
```

> 你也可以创建一个文件`/etc/mysql/conf.d/cloudstack.cnf`，并在其中添加这些指令。不要忘记在文件的第一行添加 [mysqld]。

- 启动或重新启动 MySQL 以使新配置生效。

```bash
systemctl start mysqld
sudo systemctl restart mysqld
```

- 这一步只有在CentOS RHEL上才需要，在ubuntu上不需要

> 写在mysql
rm -rf /var/lib/mysql/*
  yum remove $(rpm -qa | grep mysql)

> 在 RHEL 和 CentOS 上，MySQL 默认不设置 root 密码。强烈建议您设置 root 密码作为安全预防措施。

运行以下命令以保护安装。您可以回答所有问题的“Y”。

```bash
mysql_secure_installation
```

- CloudStack可以被安全机制阻止，比如SELinux。禁用 SELinux 以确保 + 代理具有所有必需的权限。

在RHEL或CentOS上配置SELinux ^776df0

1. 检查您的机器上是否安装了 SELinux。如果没有，您可以跳过此部分。在 RHEL 或 CentOS 中，SELinux 默认安装并启用。您可以通过以下命令进行验证：

```bash
rpm -qa | grep selinux
```

2. 将 `/etc/selinux/config` 中的 SELINUX 变量设置为“permissive”。这可确保在系统重新启动后保持宽松设置。

    在 RHEL 或 CentOS 中：

```bash
vi /etc/selinux/config

# change the line: SELINUX=enforcing  to SELINUX=permissive

setenforce permissive
```

- cloudstack-setup-databases脚本用于创建cloudstack数据库（cloud，cloud_usage），创建用户（cloud），向用户授予权限，并为管理服务器的首次启动准备表。

以下命令在数据库上创建“cloud”用户。

```bash
cloudstack-setup-databases cloud:<dbpassword>@localhost [ --deploy-as=root:<password> | --schema-only ] -e <encryption_type> -m <management_server_key> -k <database_key> -i <management_server_ip>
```

> cloudstack-setup-databases cloud:cloudlandui%LD123@localhost --deploy-as=root:landui@123LANDUI   -m password123 -k password123 -i  192.168.13.7


-  在 dbpassword 中，指定要分配给“cloud”用户的密码。您可以选择不提供密码，但不建议这样做。

    - 在 deploy-as 中，指定部署数据库的用户的用户名和密码。在以下命令中，假定 root 用户正在部署数据库并创建“cloud”用户。

    - （可选）有一个选项可以绕过数据库的创建，用户和向用户授予权限。如果您不想公开root凭据，但仍希望为首次启动做好准备，这将非常有用。在执行此脚本之前，必须手动完成这些跳过的步骤。可以通过传递 –schema-only 标志来调用此行为。此标志与 –deploy-as 标志冲突，因此两者不能一起使用。若要在执行带有标志的脚本之前手动设置数据库和用户，可以执行以下命令：

```sql
-- Create the cloud and cloud_usage databases
CREATE DATABASE `cloud`;
CREATE DATABASE `cloud_usage`;

-- Create the cloud user
CREATE USER cloud@`localhost` identified by '<password>';
CREATE USER cloud@`%` identified by '<password>';

-- Grant all privileges to the cloud user on the databases
GRANT ALL ON cloud.* to cloud@`localhost`;
GRANT ALL ON cloud.* to cloud@`%`;

GRANT ALL ON cloud_usage.* to cloud@`localhost`;
GRANT ALL ON cloud_usage.* to cloud@`%`;

-- Grant process list privilege for all other databases
GRANT process ON *.* TO cloud@`localhost`;
GRANT process ON *.* TO cloud@`%`;
```

- （可选）对于encryption_type，请使用 file 或 web 来指示用于传入数据库加密密码的技术。默认值：file。请参阅  [[#关于Password和Key加密]]
- （可选）对于management_server_key，替换用于加密CloudStack属性文件中机密参数的默认密钥。默认值：password。强烈建议您将其替换为更安全的值。请参阅  [[#关于Password和Key加密]]
- （可选）对于database_key，替换用于加密CloudStack数据库中机密参数的默认密钥。默认值：password。强烈建议您将其替换为更安全的值。请参阅  [[#关于Password和Key加密]]
- （可选）对于management_server_ip，可以显式指定群集管理服务器节点 IP。如果未指定，将使用本地 IP 地址。

此脚本完成后，您应该会看到类似“已成功初始化数据库”的消息。

> 如果脚本无法连接到MySQL数据库，请检查/etc/hosts中的“localhost”环回地址。它应该指向 IPv4 环回地址“`127.0.0.1`”，而不是 IPv6 环回地址 `::1`。或者，重新配置 MySQL 以绑定到 IPv6 环回接口。

- 如果与管理服务器在同一台计算机上运行 KVM Hypervisor程序，请编辑 `/etc/sudoers` 并添加以下行：

```conf
Defaults:cloud !requiretty
```

- 现在，数据库已设置完毕，您可以完成管理服务器的操作系统配置。此命令将设置 iptables、sudoers 并启动管理服务器。

```bash
cloudstack-setup-management
# 查看管理服务器是否在运行
systemctl status cloudstack-management
```

你应该得到输出消息“CloudStack管理服务器设置完成”。如果 servlet 容器是 Tomcat7，那么必须使用参数 `–tomcat7`。
##### 安装数据库到一个分开的节点上（多管理节点才用到这个）

忽略，查看官方文档 https://docs.cloudstack.apache.org/en/4.19.0.0/installguide/management-server/index.html#install-the-database-on-a-separate-node

#### 准备NFS Shares

CloudStack需要一个地方来保存主存储和次要存储（参见  [[CloudStack的相关概念和术语#云基础设施概览]] ）。这两者都可以是 NFS shares。本节将介绍如何在向CloudStack添加存储之前设置NFS shares。

> NFS 不是主存储或次要存储的唯一选择。例如，您可以使用 Ceph RBD、PowerFlex、GlusterFS、iSCSI、Fiberchannel 等。存储系统的选择将取决于虚拟机管理程序的选择以及您处理的是主存储还是次要存储。

主存储和次要存储的要求在以下位置进行了描述：

- [[CloudStack的相关概念和术语#主存储-Primary Storage]]
- [[CloudStack的相关概念和术语#次要存储-Secondary Storage]]

生产安装通常使用单独的 NFS 服务器。看 [[#使用一个分开单独的NFS Server]]

您还可以将管理服务器节点用作 NFS 服务器。这在试验安装中更为典型，但在更大的部署中技术上是可行的。请参  [[#使用管理服务器作为一个NFS Server]]
##### 使用一个分开单独的NFS Server

本部分介绍如何在独立于管理服务器的节点上运行的 NFS 服务器上为凑奥存储和（可选）主存储设置 NFS shares。

以下步骤的确切命令可能因操作系统版本而异。以下步骤表示您的存储系统上已经安装了 NFS 服务器。有关如何安装 NFS 服务器的信息，请参阅操作系统的指南。

> （仅限 KVM）确保 NFS 挂载点上尚未挂载任何卷。

- 在存储服务器上，为次要存储创建一个 NFS 共享，如果也将 NFS 用于主存储，请创建第二个 NFS 共享。例如：

```bash
mkdir -p /export/primary
mkdir -p /export/secondary
```

- 要将新目录配置为 NFS 导出，请编辑 `/etc/exports`。导出带有 rw，async，no_root_squash，no_subtree_check 的 NFS 共享。例如：

```bash
vi /etc/exports
# Insert the following line.
# /export  *(rw,async,no_root_squash,no_subtree_check)
```

- 导出/export目录

```bash
exportfs -a
```

- 在管理服务器上，为次要存储创建装入点。例如：

```bash
mkdir -p /mnt/secondary
```

- 在管理服务器上装载次要存储。将下面的示例 NFS 服务器名称和 NFS 共享路径替换为您自己的。

```bash
mount -t nfs nfsservername:/export/secondary /mnt/secondary
```

##### 使用管理服务器作为一个NFS Server

本部分介绍如何在与管理服务器相同的节点上为主存储和辅助存储设置 NFS 共享。这在试验安装中更为典型，但在更大的部署中技术上是可行的。假设主机上的存储空间小于 16TB。

以下步骤的确切命令可能因操作系统版本而异。

- 在RHEL/CentOS/SUSEshang 安装nfs-utils包

```bash
yum install nfs-utils
zypper install nfs-utils
apt install nfs-kernel-server
```

- 在管理服务器的主机上创建两个目录，用于主存储和次要存储

```bash
mkdir -p /export/primary
mkdir -p /export/secondary
```

-  要将新目录配置为 NFS 导出，请编辑 /etc/exports。导出带有 rw，async，no_root_squash，no_subtree_check 的 NFS 共享。例如：

```bash
vi /etc/exports
# Insert the following line.
# /export  *(rw,async,no_root_squash,no_subtree_check)
```

> `/etc/exports`是NFS（网络文件系统）服务器的主要配置文件。在这个文件中，你可以指定哪些文件系统应该被NFS导出以供远程主机访问，以及这些主机的访问权限。
>   
>   directory host1(option1,option2,...) host2(option1,option2,...) ...
>   
>   其中directory是你想要共享的文件系统路径，host是允许访问这个文件系统的远程主机，option是这个远程主机的访问权限。


- 导出/export目录

```bash
exportfs -a
```

> `exportfs`是一个管理NFS共享目录的命令。`exportfs -a`会将`/etc/exports`文件中列出的所有文件系统导出，如果有新的共享目录添加到`/etc/exports`文件中，你需要运行`exportfs -a`来应用这些更改。


- 编辑`/etc/sysconfig/nfs`文件

```bash
vi /etc/sysconfig/nfs
# Uncomment the following lines:
LOCKD_TCPPORT=32803
LOCKD_UDPPORT=32769
MOUNTD_PORT=892
RQUOTAD_PORT=875
STATD_PORT=662
STATD_OUTGOING_PORT=2020
```

> 这个配置文件`/etc/sysconfig/nfs`是用来设置NFS服务中各个组件的网络端口的。在某些情况下，你可能需要将这些服务绑定到特定的端口，例如在配置防火墙规则或满足特定的网络策略时。
> 
> 以下是每个设置的含义：
> - `LOCKD_TCPPORT=32803` 和 `LOCKD_UDPPORT=32769`：这两个设置是用来指定NFS的锁定服务（lockd）使用的TCP和UDP端口。lockd服务是用来处理文件锁定的，防止多个客户端同时修改同一个文件。
> - `MOUNTD_PORT=892`：这个设置是用来指定NFS的挂载守护进程（mountd）使用的端口。mountd服务是用来处理客户端的挂载请求的。
> - `RQUOTAD_PORT=875`：这个设置是用来指定NFS的报价守护进程（rquotad）使用的端口。rquotad服务是用来处理客户端的磁盘配额请求的。
> - `STATD_PORT=662` 和 `STATD_OUTGOING_PORT=2020`：这两个设置是用来指定NFS的状态守护进程（statd）使用的端口。statd服务是用来处理网络状态监控的。
>  
> 取消注释这些行并设置特定的值，意味着你正在将这些服务绑定到特定的端口。

- 编辑`/etc/sysconfig/iptables`文件

```bash
vi /etc/sysconfig/iptables
# Add the following lines at the beginning of the INPUT chain, where <NETWORK> is the Network that you’ll be using:
-A INPUT -s <NETWORK> -m state --state NEW -p udp --dport 111 -j ACCEPT
-A INPUT -s <NETWORK> -m state --state NEW -p tcp --dport 111 -j ACCEPT
-A INPUT -s <NETWORK> -m state --state NEW -p tcp --dport 2049 -j ACCEPT
-A INPUT -s <NETWORK> -m state --state NEW -p tcp --dport 32803 -j ACCEPT
-A INPUT -s <NETWORK> -m state --state NEW -p udp --dport 32769 -j ACCEPT
-A INPUT -s <NETWORK> -m state --state NEW -p tcp --dport 892 -j ACCEPT
-A INPUT -s <NETWORK> -m state --state NEW -p udp --dport 892 -j ACCEPT
-A INPUT -s <NETWORK> -m state --state NEW -p tcp --dport 875 -j ACCEPT
-A INPUT -s <NETWORK> -m state --state NEW -p udp --dport 875 -j ACCEPT
-A INPUT -s <NETWORK> -m state --state NEW -p tcp --dport 662 -j ACCEPT
-A INPUT -s <NETWORK> -m state --state NEW -p udp --dport 662 -j ACCEPT
```

```txt
-A INPUT -s 192.168.13.0/24 -m state --state NEW -p udp --dport 111 -j ACCEPT
-A INPUT -s 192.168.13.0/24 -m state --state NEW -p tcp --dport 111 -j ACCEPT
-A INPUT -s 192.168.13.0/24 -m state --state NEW -p tcp --dport 2049 -j ACCEPT
-A INPUT -s 192.168.13.0/24 -m state --state NEW -p tcp --dport 32803 -j ACCEPT
-A INPUT -s 192.168.13.0/24 -m state --state NEW -p udp --dport 32769 -j ACCEPT
-A INPUT -s 192.168.13.0/24 -m state --state NEW -p tcp --dport 892 -j ACCEPT
-A INPUT -s 192.168.13.0/24 -m state --state NEW -p udp --dport 892 -j ACCEPT
-A INPUT -s 192.168.13.0/24 -m state --state NEW -p tcp --dport 875 -j ACCEPT
-A INPUT -s 192.168.13.0/24 -m state --state NEW -p udp --dport 875 -j ACCEPT
-A INPUT -s 192.168.13.0/24 -m state --state NEW -p tcp --dport 662 -j ACCEPT
-A INPUT -s 192.168.13.0/24 -m state --state NEW -p udp --dport 662 -j ACCEPT
```

> 把上面的内容加入到INPUT chain的开始位置，也就是当前-A INPUT语句的前面，因为iptables是按照规则集的顺序处理的。

- 运行下列命令

```bash
service iptables restart
service iptables save
```

- 如果在客户端和服务器之间使用 NFS v4 通信，请将您的域添加到Hypervisor主机和管理服务器上的 `/etc/idmapd.conf`。

```bash
vi /etc/idmapd.conf
```

> rpm -q nfs-utils  如果`nfs-utils`的版本是`1.3.0`，那么NFS的版本可能是`4.1`或`4.2`。

> rpcinfo -p | grep nfs   。例如，如果你看到`100003 3 tcp 2049 nfs`，那么你的NFS版本就是3。 如果既有3又有4 你的NFS服务器支持版本3和版本4。
> 
> 这是因为在你的`rpcinfo`输出中，我们可以看到`100003 3`和`100003 4`，其中`100003`是NFS的程序号，后面的`3`和`4`分别表示NFS的版本号。
> 
> 具体使用哪个版本，取决于客户端的请求。如果客户端请求NFSv4，服务器就会使用NFSv4。如果客户端请求NFSv3，服务器就会使用NFSv3。
> 
> 所以，你的NFS服务器可以处理NFSv3和NFSv4的请求。具体使用哪个版本，取决于客户端的请求。如果客户端请求NFSv4，服务器就会使用NFSv4。如果客户端请求NFSv3，服务器就会使用NFSv3。

删除 idmapd.conf 中 Domain 行开头的字符 #, 并将文件中的值替换为您自己的 domain。在下面的示例中，域是 company.com。

```conf
Domain = company.com
```

> `/etc/idmapd.conf`是Linux NFS（网络文件系统）的一个配置文件，它用于NFSv4的用户和组ID映射。NFSv4使用字符串形式的用户和组名，而不是数字ID，这样可以在不同的机器之间更好地共享文件，因为字符串形式的用户名和组名更不容易发生冲突。
> 
> 在`/etc/idmapd.conf`文件中，`Domain`配置项用于指定NFSv4的域名。这个域名用于在用户和组名前添加一个前缀，以防止在不同的机器之间发生冲突。例如，如果你的`Domain`配置项设置为`mydomain.com`，那么一个用户`user1`在NFSv4中会被表示为`user1@mydomain.com`。
> 
> 这里的`Domain`与互联网上的域名是不同的。互联网上的域名是用于在网络上唯一标识一个网站的地址，而NFSv4的`Domain`只是用于在用户和组名前添加一个前缀，以防止在不同的机器之间发生冲突。NFSv4的`Domain`并不需要是一个真实存在的互联网域名，也不需要能够在网络上解析。

- 重启管理服务器主机

```bash
systemctl enable nfs  
systemctl enable rpcbind
systemctl start nfs
systemctl start rpcbind
chkconfig nfs on
chkconfig rpcbind on
```

现在设置了两个名为 /export/primary 和 /export/secondary 的 NFS 共享。

- 建议您进行测试，以确保前面的步骤已成功。

1. 登录Hypervisor的主机
2. 确保 NFS 和 rpcbind 正在运行。这些命令可能因操作系统而异。例如：

```bash
# 注意，Hypervisor的host也要关闭SELinux，并配置上idmapd.conf的Domain
yum install nfs-utils
systemctl  start rpcbind
systemctl  start nfs
chkconfig nfs on
chkconfig rpcbind on
reboot
```

3. 重新登录到Hypersivor主机，并尝试挂载 /export 目录。例如，替换你自己的管理服务器名称：

```bash
mkdir /primary
mount -t nfs <management-server-ip>:/export/primary /primary
mkdir /secondary
mount -t nfs <management-server-ip>:/export/secondary /secondary
mount | grep nfs
umount /primary
umount /secondary
```
#### 额外的Management servers

这个是用来做高可用的，不讲解，请参考 https://docs.cloudstack.apache.org/en/4.19.0.0/installguide/management-server/index.html#additional-management-servers

#### 准备系统VM模板

从Apache CloudStack v4.16开始，如果在开始升级之前没有完成，升级路径将处理SystemVM模板的注册。在全新安装CloudStack时，也可以选择省略SystemVM模板种子设定步骤，因为添加了支持，在添加第一个次要存储池时，为Zone区域中存在的所有Hypervisor启动SystemVM模板注册。次要存储必须使用用于CloudStack系统VM的模板进行播种。

> 复制和粘贴命令时，请确保在执行之前将命令粘贴为一行。某些文档查看器可能会在复制的文本中引入不需要的换行符。


- 在 Management Server 上，运行以下一个或多个 `cloud-install-sys-tmplt` 命令，以检索和解压缩系统 VM 模板。针对您希望终端用户在此Zone中运行的每种Hypervisor类型运行该命令。

如果次要存储挂载点未命名为 `/mnt/secondary`，请替换您自己的挂载点名称。


```bash
mkdir /mnt/secondary
mount -t nfs 127.0.0.1:/export/secondary /mnt/secondary/
```

如果你在设置数据库时将CloudStack数据库的加密类型设置为“web”，那么现在必须添加参数-s \<management-server-secret-key\>。请参阅  [[#关于Password和Key加密]]

此过程将需要本地文件系统上大约 5 GB 的可用空间，每次运行最多需要 30 分钟。

```bash
# Hyper-V
/usr/share/cloudstack-common/scripts/storage/secondary/cloud-install-sys-tmplt -m /mnt/secondary -u [http://download.cloudstack.org/systemvm/4.19/systemvmtemplate-4.19.0-hyperv.vhd.zip](http://download.cloudstack.org/systemvm/4.19/systemvmtemplate-4.19.0-hyperv.vhd.zip) -h hyperv -s <optional-management-server-secret-key> -F
# XenServer
/usr/share/cloudstack-common/scripts/storage/secondary/cloud-install-sys-tmplt -m /mnt/secondary -u [http://download.cloudstack.org/systemvm/4.19/systemvmtemplate-4.19.0-xen.vhd.bz2](http://download.cloudstack.org/systemvm/4.19/systemvmtemplate-4.19.0-xen.vhd.bz2) -h xenserver -s <optional-management-server-secret-key> -F
# vSphere
/usr/share/cloudstack-common/scripts/storage/secondary/cloud-install-sys-tmplt -m /mnt/secondary -u [http://download.cloudstack.org/systemvm/4.19/systemvmtemplate-4.19.0-vmware.ova](http://download.cloudstack.org/systemvm/4.19/systemvmtemplate-4.19.0-vmware.ova) -h vmware -s <optional-management-server-secret-key> -F
# KVM
/usr/share/cloudstack-common/scripts/storage/secondary/cloud-install-sys-tmplt -m /mnt/secondary -u http://download.cloudstack.org/systemvm/4.19/systemvmtemplate-4.19.0-kvm.qcow2.bz2 -h kvm -s <optional-management-server-secret-key> -F
# LXC
/usr/share/cloudstack-common/scripts/storage/secondary/cloud-install-sys-tmplt -m /mnt/secondary -u [http://download.cloudstack.org/systemvm/4.19/systemvmtemplate-4.19.0-kvm.qcow2.bz2](http://download.cloudstack.org/systemvm/4.19/systemvmtemplate-4.19.0-kvm.qcow2.bz2) -h lxc -s <optional-management-server-secret-key> -F
```

由于我想使用的是KVM这个Hypervisor，所以我是执行:

```bash
/usr/share/cloudstack-common/scripts/storage/secondary/cloud-install-sys-tmplt -m /mnt/secondary -u  
http://download.cloudstack.org/systemvm/4.19/systemvmtemplate-4.19.0-kvm.qcow2.bz2 -h kvm -F
```

- 如果您使用的是分开单独的 NFS 服务器，请执行此步骤。如果将管理服务器用作 NFS 服务器，则不得执行此步骤。

脚本完成后，卸载次要存储并删除创建的目录。

```bash
# 也就是如果在管理服务器上安装的NFS server，则不能执行以下命令
umount /mnt/secondary
rmdir /mnt/secondary
```

- 在每个次要存储服务器上重复执行执行这些步骤。

#### 安装完成，下一步

祝贺！你现在已经安装了CloudStack管理服务器和它用来持久化系统数据的数据库。

![[Pasted image 20240226111519.png]]

接下来应该怎么做？

- 即使不添加任何云基础设施，你也可以运行UI来了解所提供的内容，以及你将如何与CloudStack进行持续的交互。请参阅  [Log In to the UI — Apache CloudStack 4.19.0.0 documentation](https://docs.cloudstack.apache.org/en/4.19.0.0/adminguide/ui.html#log-in-to-ui)。
- 当你准备好了，添加云基础设施，并尝试在其上运行一些实例，这样你就可以观察CloudStack如何管理基础设施。请参阅    [[#Provisioning Steps概览]] 
- [部署CloudStack 4.18.0演示环境 - 简书 (jianshu.com)](https://www.jianshu.com/p/8a847cc7626c)
## 配置


本节介绍如何将Region区域、Zone可用区、Pod、集群、主机、存储和网络添加到您的云中。如果您不熟悉这些实体，请先浏览一下 [[CloudStack的相关概念和术语#云基础设施概览]]
### 配置你的CloudStack安装

#### Provisioning Steps概览

安装并运行管理服务器后，可以添加要管理的计算资源。关于CloudStack云基础设施的组织方式的概述，请参见  [[CloudStack的相关概念和术语#云基础设施概览]]

若要预配(provision)云基础设施，或随时扩展，请遵循以下过程：

- 定义Region区域（可选）。请参阅  [[#添加Regions区域(可选)]] 
- 将Zone区域添加到Region区域。请参阅 [[#添加一个Zone区域]]
- 向Zone区域添加更多 Pods（可选）。请参阅  [[#添加一个Pod]]
- 向 Pod 添加更多集群（可选）。请参阅  [[#添加一个集群]]
- 向群集添加更多主机Host（可选）。请参见 [[#添加一个主机]]
- 将主存储添加到集群。请参见添加  [[#添加一个主存储]]
- 将次要存储添加到指定的Zone区域。请参见添加  [[#添加一个次要存储]]
- 初始化并测试新云。请参阅  [[#初始化和测试]]

完成这些步骤后，您将拥有具有以下基础设施的部署：

![[Pasted image 20240228162928.png]]
#### 添加Regions区域(可选)

在预配(Provision)云时，将云资源分组到地理区域(region)是一个可选步骤。有关区域的概述，请参阅  [[CloudStack的相关概念和术语#Regions]]

##### 第一个Region区域-默认的Region

如果不采取措施来定义Region区域，则云中的所有Zone区域将自动分组到单个默认的Region区域中。此Region区域分配的Region区域 ID 为 1。你可以通过在CloudStack UI中显示默认Region区域并点击编辑按钮来更改默认区域的名称或URL。

##### 添加一个Region

使用以下步骤添加除了默认Region区域之外的第二个Region区域。

- 每个Region区域都有自己的CloudStack实例。因此，创建新Region区域的第一步是在要设置新Region区域的地理区域(比如，昆明，东京等)中的一个或多个节点上安装 Management Server 软件。使用安装指南中的步骤。当您进入设置数据库的步骤时，请使用附加命令行标志 `-r <region_id>` 为新Region区域设置Region区域 ID。系统会自动为默认区域分配区域 ID 1，因此您的第一个添加的区域可能是区域 2。

```bash
cloudstack-setup-databases cloud:<dbpassword>@localhost --deploy-as=root:<password> -e <encryption_type> -m <management_server_key> -k <database_key> -r <region_id>
```

- 在安装过程结束时，管理服务器应已启动。确保管理服务器安装成功且完整。
- 现在将新Region区域添加到CloudStack中的区域1。
    - 登录CloudStack的区域1，用root管理员登录， http://region-1-ip-address:8080/client
    - 在左侧的导航栏中，点击Regions
    - 点击Add Region。在对话框中，填写以下字段:
        - ID 唯一的标识号。使用在新Region区域中安装 Management Server 期间在数据库中设置的相同编号;例如，2。也就是区域号
        - Name。为新Region区域指定一个描述性名称。
        - Endpoint 。可在新Region区域中登录管理服务器的 URL。其格式为 http://region.2.IP.address:8080/client 

- 现在执行同样的操作，只不过是相反的，登录Region 2的management server的UI，添加Region 1。 是一个相互的过程
- 将account表、user表和domain表从区域 1 数据库复制到区域 2 数据库。在以下命令中，假设你已经在数据库上设置了root密码，这是CloudStack推荐的最佳实践。替换您自己的 MySQL root 密码。
    - 首先，运行以下命令拷贝数据库的内容
```bash
mysqldump -u root -p<mysql_password> -h <region1_db_host> cloud account user domain > region1.sql
```

     - 然后，运行以下命令把数据放到region 2的数据库中

```bash
mysql -u root -p<mysql_password> -h <region2_db_host> cloud < region1.sql
```

- 删除project accounts，在region 2的数据上运行以下命令

```sql
mysql> delete from account where type = 5;
```

- 设置默认Zone为null

```sql
mysql> update account set default_zone_id = null;
```

- 重启Region 2上的管理服务器。

```bash
systemctl restart cloudstack-management
```
##### 添加第三个和后续的Regions

https://docs.cloudstack.apache.org/en/4.19.0.0/installguide/configuration.html#adding-third-and-subsequent-regions

##### 删除一个Region

登录到其他每个Region区域，导航到要删除的Region区域，然后单击删除Region区域。例如，要删除 3 个区域云中的第三个区域，请执行以下操作：

- 登录 http://region.1.IP.address:8080/client
- 在左侧导航栏中，单击Regions。
- 单击要删除的Region区域的名称。
- 单击“删除Region”按钮。
- 对 http://region.2.IP.address:8080/client 重复这些步骤。

#### 添加一个Zone区域

添加新Zone区域时，系统将提示您配置该Zone区域的物理网络并添加第一个 Pod、集群、主机、主存储和次要存储。

- 以root管理员身份登录CloudStack UI。
- 在左侧导航栏中，选择 Infrastructure。
- 在“Zones”上，单击“View more”。
- 单击添加Zone区域。将出现Zone区域创建向导。
- 选择以下网络类型之一：
    - Basic 用于 AWS 风格的网络。提供单一网络，其中每个实例直接从网络分配一个 IP。可以通过安全组（IP address source filtering）等第 3 层方式提供访客机器之间的隔离。
    - Advanced 适用于更复杂的网络拓扑。此网络模型在定义访客网络和提供自定义网络产品（如防火墙、VPN 或负载均衡器支持）方面提供了最大的灵活性。
    - 安全组  您可以选择在您的Zone区域中启用安全组。有关安全组和先决条件的更多信息，请参阅文档中的“安全组”部分。
- 其余步骤会有所不同，具体取决于您选择的是“基本”还是“高级”。继续执行适用于您的步骤：
    - [[#基本的Zone配置]]
    - [[#高级的Zone配置]]
##### 基本的Zone配置

- 在“添加Zone区域”向导中选择“Basic”并单击“下一步”后，系统将要求您输入以下详细信息。然后单击“下一步”。
    - Name   Zone的名字
    - DNS 1和 2    这些是供Zone区域中的访客实例使用的 DNS 服务器。这些 DNS 服务器将通过您稍后添加的公网进行访问。Zone区域的公共 IP 地址必须具有指向此处命名的 DNS 服务器的路由。
    - Internal DNS 1和 Internal DNS 2 这些是供Zone区域中的系统虚拟机使用的DNS服务器（这些是CloudStack本身使用的实例，如虚拟路由器、控制台代理和次要存储虚拟机）。这些 DNS 服务器将通过系统 VM 的管理流量网络接口进行访问。您为 Pod 提供的专用 IP 地址必须具有指向此处指定的内部 DNS 服务器的路由。
    - Hypervisor （在 3.0.1 版中引入）为Zone区域中的第一个集群选择Hypervisor程序。在完成Zone区域添加后，您可以稍后添加具有不同Hypervisor的集群。
    - Network Offering 您在此处的选择决定了网络上将哪些网络服务可用于访客实例。

![[Pasted image 20240228173912.png]]

- continue
    - Network Domain （可选）如果要为访客实例网络分配特殊域名，请指定 DNS 后缀。
    - Public 公共Zone区域可供所有用户使用。非公共区域将被分配给特定的domain。仅允许该domain中的用户在此Zone区域中创建访客实例。

- 选择物理网络将承载的流量类型。

流量类型包括管理流量、公共流量、访客流量和存储流量。有关类型的详细信息，请滚动图标以显示其工具提示，或参阅基本区域网络流量类型。此屏幕开始时已分配一些流量类型。要添加更多流量类型，请将流量类型拖放到网络上。如果需要，您还可以更改网络名称。

- 为物理网络上的每种流量类型分配网络流量标签。这些标签必须与您已在Hypervisor Host主机上定义的标签匹配。要分配每个标签，请单击流量类型图标下方的编辑按钮。此时将显示一个弹出对话框，您可以在其中键入标签，然后单击“确定”。

这些流量标签将仅针对为第一个集群选择的Hypervisor定义。对于所有其他Hypervisor，可以在创建Zone区域后配置标签。

- 单击下一步
- (NetScaler Only) 这个步骤可以跳过，因为暂时没有安装CitrixNetScaler  [Configuring your CloudStack Installation — Apache CloudStack 4.19.0.0 documentation](https://docs.cloudstack.apache.org/en/4.19.0.0/installguide/configuration.html)
- (NetScaler Only) 这个步骤可以跳过，因为暂时没有安装CitrixNetScaler  [Configuring your CloudStack Installation — Apache CloudStack 4.19.0.0 documentation](https://docs.cloudstack.apache.org/en/4.19.0.0/installguide/configuration.html)
- 在一个新的Zone区域中，CloudStack会为你添加第一个pod。您以后可以随时添加更多 Pod。有关 Pod 的概述，请参阅  [[CloudStack的相关概念和术语#Pods]]  要配置第一个 Pod，请输入以下内容，然后单击下一步：
    - Pod Name   Pod的名称
    - Reserved system gateway  在Pod中所有hosts主机的网关
    - Reserved system netmask 定义Pod的子网的网络前缀，用CIDR的格式描述
    - Start/End Reserved System IP  CloudStack用于管理各种系统虚拟机的IP范围，如次要存储虚拟机、控制台代理虚拟机和DHCP。有关详细信息，请参阅系统保留 IP 地址。

- 为访客流量配置网络。提供以下内容，然后单击“下一步”：
    - Guest Gateway 访客应该使用的网关
    - Guest netmask   访客(the guests)将使用的子网的子网掩码
    - Guest start/end IP 输入第一个和最后一个IP地址，这些IP地址定义了CloudStack可以分配给guest的范围。
        - 强烈建议使用多个 NIC。如果使用多个 NIC，它们可能位于不同的子网中。
        - 如果使用一个网卡，则这些 IP 应与 Pod CIDR 位于同一 CIDR 中。

- 在一个新的pod中，CloudStack为你添加第一个集群。您以后可以随时添加更多集群。有关什么是集群的概述，请参阅  [[CloudStack的相关概念和术语#Clusters]] 。要配置第一个集群，请输入以下内容，然后单击下一步：
    - Hypervisor （仅限版本 3.0.0;在 3.0.1 中，此字段为只读）选择此群集中所有主机都将运行的Hypervisor软件类型。如果选择 VMware，则会显示其他字段，以便您提供有关 vSphere 集群的信息。对于vSphere服务器，我们建议在vCenter中创建主机集群，然后将整个集群添加到CloudStack中。请参见  [[#添加一个集群]]
    - Cluster Name 输入集群的名称。这可以是你选择的文本，CloudStack不会使用。

- 在一个新的集群中，CloudStack会为你添加第一个主机。您以后可以随时添加更多主机。有关什么是主机的概述，请参阅  [[CloudStack的相关概念和术语#Hosts]]

> 当你向CloudStack添加一个虚拟机管理程序主机时，该主机不能有任何实例已经运行。

在配置主机之前，您需要在主机上Hypervisor软件。你需要知道CloudStack支持哪个版本的Hypervisor软件版本，以及需要哪些额外的配置来确保主机能够与CloudStack一起工作。若要查找这些安装详细信息，请参阅：

- Citrix XenServer 安装和配置
- VMware vSphere 安装和配置
- KVM vSphere 安装和配置

要配置第一个主机，请输入以下内容，然后单击下一步：

- Host Name   主机的IP地址或者主机的域名，DNS可以解析的域名
- Username 用户名是root
- Password 这是上面指定的用户的密码（来自 XenServer 或 KVM 安装）。在KVM的情况下，还有一个额外的功能是，也可以使用CloudStack的SSH密钥添加主机，而不必提供主机密码。

    在CloudStack中添加主机之前，请执行以下操作：
     - 从管理服务器上的 /var/cloudstack/management/.ssh/id_rsa.pub 复制 SSH 公钥
     - 将复制的密钥添加到主机上的 /root/.ssh/authorized_keys 文件中
    选择“系统 SSH 密钥”并继续执行后续步骤。
- Host Tags （可选）用于对主机进行分类以便于维护的任何标签。例如，如果您希望此主机仅用于启用了“高可用性”功能的实例，则可以将其设置为云的 HA 标记（在 ha.tag 全局配置参数中设置）。有关更多信息，请参阅启用 HA 的实例以及主机的 HA。


- 在一个新的集群中，CloudStack会为你添加第一个主存储服务器。您以后可以随时添加更多服务器。有关什么是主存储的概述，请参见关于  [[CloudStack的相关概念和术语#主存储-Primary Storage]]   要配置第一个主存储服务器，请输入以下内容，然后单击下一步：
    - Name 存储设备的名称
    - Protocol 对于 XenServer，选择 NFS、iSCSI 或 PreSetup。对于 KVM，请选择 NFS、SharedMountPoint、CLVM、RBD、FiberChannel 或自定义（对于 PowerFlex）。对于 vSphere，选择 VMFS（iSCSI 或 FiberChannel）或 NFS。屏幕中的其余字段会有所不同，具体取决于您在此处选择的内容。
##### 高级的Zone配置

https://docs.cloudstack.apache.org/en/4.19.0.0/installguide/configuration.html#id4
###### Core Zone

https://docs.cloudstack.apache.org/en/4.19.0.0/installguide/configuration.html#core-zone
###### 边缘Zone(Edge Zone)

https://docs.cloudstack.apache.org/en/4.19.0.0/installguide/configuration.html#edge-zone
#### 添加一个Pod

当你创建一个新Zone区域时，CloudStack会为你添加第一个pod。您可以随时使用本节中的过程添加更多Pods。

- 登录到CloudStack UI。请参阅 https://docs.cloudstack.apache.org/en/4.19.0.0/adminguide/ui.html#log-in-to-ui
- 在左侧导航栏中，选择 Infrastructure。在“Zones”中，单击“view more”，然后单击要向其添加Pod的Zone区域。
- 单击“计算和存储”选项卡。在diagram的 Pods 节点中，单击“查看全部”。
- 单击添加Pod。
- 在对话框中输入以下详细信息。
    - Name  Pod的名称
    - Gateway  Pod中所有主机的网关
    - Netmask  Pod的子网的网络前缀，用CIDR格式描述
    - Start/End Reserved System IP  CloudStack用于管理各种系统虚拟机的IP范围，如次要存储虚拟机、控制台代理虚拟机和DHCP。有关详细信息，请参阅系统保留 IP 地址。
- 单击OK

#### 添加一个集群

你需要告诉CloudStack它将要管理的主机。主机存在于集群中，因此在开始向云添加主机之前，必须至少添加一个集群。

##### 添加KVM或者XenServer集群

这些步骤假设你已经在主机上已经安装了KVM 或者XenServer程序，并登录到了CloudStack的UI。

- 在左侧导航栏中，选择 Infrastructure。在“区域”中，单击“查看更多”，然后单击要在其中添加群集的区域。
- 单击“计算”选项卡。
- 在diagram的“集群”节点中，单击“查看全部”。
- 单击添加集群。
- 选择此集群的Hypervisor类型。
- 选择要在其中创建集群的 Pod。
- 输入集群的名称。这可以是你选择的文本，CloudStack不会使用。
- 单击“确定”。

##### 添加vSphere集群

忽略，请参考官网

#### 添加一个主机

- 在将主机添加到CloudStack配置之前，你必须首先在主机上安装你选择的虚拟机管理程序。CloudStack可以管理在各种虚拟机管理程序下运行实例的主机。

 [[CloudStack完全安装指南]] 提供了如何安装每个支持的Hypervisor，并配置它以用于CloudStack的说明。请参阅安装指南中的 [[#Hypervisor设置]] ，了解你所选择的Hypervisor的哪个版本是受支持的，以及配置虚拟机管理程序主机以与CloudStack一起使用的关键额外步骤。

> 确保你已经执行了你的特定Hypervisor程序安装部分描述的其他CloudStack特定的配置步骤。

- 现在将Hypervisor主机添加到CloudStack中。要使用的技术因Hypervisor而异。
    - [[#添加一个KVM主机或者XenServer主机]]
    - [[#添加一个vSphere主机]]
##### 添加一个KVM主机或者XenServer主机

可以随时将 XenServer 和 KVM 主机添加到群集中。

> 在你把Hypervisor主机添加到CloudStack之前，确保它没有任何实例在运行。

配置要求：

- 每个集群必须仅包含具有相同的Hypervisor的主机。（要么全是KVM，要么全是XenServer）
- 对于 XenServer，不要在一个群集中放置超过 8 个主机。
- 对于 KVM，不要在一个集群中放置超过 16 个主机。

关于硬件要求，请参看 [[CloudStack完全安装指南]] 中  [[CloudStack完全安装指南#Hypervisor设置]] 。

如果正在使用网络绑定(network bonding)，则管理员必须以相同的方式将新主机连接到群集中的其他主机。

对于要添加到群集的所有其他主机，请运行以下命令。这将导致主机加入 XenServer 池中的master

```bash
xe pool-join master-address=[master IP] master-username=root master-password=[your password]
```

> 复制和粘贴命令时，请确保在执行之前将命令粘贴为一行。某些文档查看器可能会在复制的文本中引入不需要的换行符。

将所有主机添加到 XenServer pool后，运行 cloud-setup-bond 脚本。此脚本将在群集中的新主机上完成绑定的配置和设置。

- 将脚本从管理服务器的 /usr/share/cloudstack-common/scripts/vm/hypervisor/xenserver/cloud-setup-bonding.sh 复制到master主机，并确保它是可执行的。
- 运行脚本  `./cloud-setup-bonding.sh`
    - 如果正在使用共享挂载点存储，管理员应确保新主机具有与群集中其他主机相同的挂载点（已挂载存储）。
    - 确保新主机具有与群集中其他主机相同的网络配置（访客、专用和公用网络）。
    - 如果你使用的是OpenVswitch bridge，在将主机添加到CloudStack之前，请编辑KVM主机上的agent.properties文件，并将参数network.bridge.type设置为openvswitch
    - 如果您使用非 root 用户添加 KVM 主机，请将该用户添加到 sudoers 文件：

```conf
cloudstack ALL=NOPASSWD: /usr/bin/cloudstack-setup-agent
defaults:cloudstack !requiretty
```

- 如果尚未执行此操作，请在主机上安装Hypervisor。你需要知道CloudStack支持哪个版本的Hypervisor软件版本，以及需要哪些额外的配置来确保主机能够与CloudStack一起工作。要找到这些安装细节，请参看 [[CloudStack完全安装指南#Hypervisor设置]]
- 以管理员身份登录CloudStack UI。
- 在左侧导航栏中，选择 Infrastructure。在“Zones”中，单击“View more”，然后单击要在其中添加主机的Zone区域。
- 单击“计算”选项卡。在“群集”节点中，单击“查看全部”。
- 单击要添加主机的集群。
- 单击查看Hosts
- 单击添加Host
- 提供以下信息
    - Host Name   主机的IP地址或者主机的域名，DNS可以解析的域名
    - Username 用户名是root
    - Password 这是上面指定的用户的密码（来自 XenServer 安装）
    - Host Tags （可选）用于对主机进行分类以便于维护的任何标签。例如，如果您希望此主机仅用于启用了“高可用性”功能的实例，则可以将其设置为云的 HA 标记（在 ha.tag 全局配置参数中设置）。有关更多信息，请参阅启用 HA 的实例以及主机的 HA。
- 重复以上步骤，添加额外的主机

添加 KVM 主机的步骤与添加 XenServer 主机的步骤相同，如上一节所述。在KVM的情况下，还有一个额外的功能是，也可以使用CloudStack的SSH密钥添加主机，而不必提供主机密码。

在CloudStack中添加主机之前，请执行以下操作：

- 从管理服务器上的 /var/cloudstack/management/.ssh/id_rsa.pub 复制 SSH 公钥
- 将复制的密钥添加到主机上的 /root/.ssh/authorized_keys 文件中

从CloudStack UI添加主机时，选择“System SSH Key”，如下图所示

![[Pasted image 20240301102726.png]]



##### 添加一个vSphere主机

忽略，请参考官网

#### 添加一个主存储

##### 主存储的系统要求

硬件要求：

- 底层Hypervisor支持的任何符合标准的 iSCSI、SMB 或 NFS 服务器。
- 存储服务器应该是具有大量磁盘的计算机。理想情况下，磁盘应由硬件 RAID 控制器管理。
- 所需的最小容量取决于您的需求。

设置主存储时，请遵循以下限制：

- 在将主机添加到群集之前，无法添加主存储。
- 如果未置备(provision)共享主存储，则必须将全局配置参数 system.vm.local.storage.required 设置为 true，否则将无法启动实例。
##### 添加主存储

创建新Zone区域时，将添加第一个主存储作为该过程的一部分。您可以随时添加主存储服务器，例如在添加新集群或向现有集群添加更多服务器时。

> 将预分配（preallocated）的存储用于主存储时，请确保存储上没有任何内容（例如，您有一个空的 SAN 卷或一个空的 NFS 共享）。将存储添加到CloudStack将破坏任何现有的数据。

- 登录到CloudStack UI 登录UI。
- 在左侧导航栏中，选择 Infrastructure。在“区域”中，单击“View all”，然后单击要在其中添加主存储的区域。
- 单击“计算”选项卡。
- 在Diagram的主存储节点中，单击“查看全部”。
- 单击添加主存储。
- 在对话框中提供以下信息。所需信息因您在协议中的选择而异。
    - Scope 指示存储是Zone区域中的所有主机使用，还是仅供单个群集中的主机使用。
    - Pod （仅当您在“作用域”字段中选择“群集”时才可见。存储设备的 Pod。
    - Cluster （仅当您在“作用域”字段中选择“群集”时才可见。存储设备的集群。
    - Name 存储设备的名称
    - Protocol 对于 XenServer，选择 NFS、iSCSI 或 PreSetup。对于 KVM，选择 NFS、SharedMountPoint 或自定义（对于 PowerFlex）。对于 vSphere，选择 NFS、PreSetup （VMFS - iSCSI/FiberChannel、vSAN、VVOLS） 或 DatastoreCluster。对于“Hyper-V”，请选择“SMB”。
    -  **Server (for NFS, iSCSI, or PreSetup).** The IP address or DNS name of the storage device.
    -  **Server (for PreSetup or DatastoreCluster).** The IP address or DNS name of the vCenter server.
    -  **Path (for NFS).** In NFS this is the exported path from the server.
    -  **Path (for PreSetup or DatastoreCluster).** In vSphere this is a combination of the datacenter name and the datastore or datastore cluster name. The format is “/” datacenter name “/” datastore or datastore cluster name. For example, “/cloud.dc.VM/cluster1datastore”.
    -  **Path (for SharedMountPoint).** With KVM this is the path on each host that is where this primary storage is mounted. For example, “/mnt/primary”.
    - **RADOS Monitor (for RBD).** With KVM, this is Ceph host domain/IP with port. For example, “round-robin.ceph-cluster.xyz:6789”.
    - **RADOS Pool (for RBD).** With KVM, this is Ceph pool name. For example, “cloudstack”.
    - **RADOS User (for RBD).** With KVM, this is Ceph client user name. For example, “cloudstack”.
    - **RADOS Secret (for RBD).** With KVM, this is the Ceph pool secret authorised for a client username. For example, “AQC3u/JfhipzGBAACiILEFKembN8gTJsIvu6nQ\=\=”.
    - **SMB Username** (for SMB/CIFS): Applicable only if you select SMB/CIFS provider. The username of the account which has the necessary permissions to the SMB shares. The user must be part of the Hyper-V administrator group.
    - **SMB Password** (for SMB/CIFS): Applicable only if you select SMB/CIFS provider. The password associated with the account.
    - **SMB Domain**(for SMB/CIFS): Applicable only if you select SMB/CIFS provider. The Active Directory domain that the SMB share is a part of.
    - **SR Name-Label (for PreSetup).** Enter the name-label of the SR that has been set up outside CloudStack.
    - **Target IQN (for iSCSI).** In iSCSI this is the IQN of the target. For example, iqn.1986-03.com.sun:02:01ec9bb549-1271378984.
    - **Lun # (for iSCSI).** In iSCSI this is the LUN number. For example, 3.
    - **Tags (optional).** The comma-separated list of tags for this storage device. It should be an equivalent set or superset of the tags on your disk offerings..

> Zone区域中跨集群的主存储上的标签集必须相同。例如，如果集群 A 提供具有标签 T1 和 T2 的主存储，则区域中的所有其他集群也必须提供具有标签 T1 和 T2 的主存储。

- 单击OK
##### 配置一个存储Plug-in

> 基于自定义插件（例如SolidFire）的主存储必须通过CloudStack API（本节后面介绍）添加。目前不支持通过CloudStack UI添加这种类型的主存储（尽管它的大部分功能都可以通过CloudStack UI获得）。

###### SolidFire Plug-in

忽略

###### PowerFlex Plug-in

忽略
###### StorPool Plug-in

忽略
###### HPE Primera/3PAR Plug-in

忽略
###### Pure Flasharray API

忽略

#### 添加一个次要存储

##### 次要存储的系统要求

- NFS 存储设备或 Linux NFS 服务器
- SMB/CIFS （Hyper-V）
- （可选）OpenStack 对象存储 （Swift）（参见 http://swift.openstack.org）
- 最小容量 100GB
- 次要存储设备必须与其所服务的访客实例位于同一Zone区域中。
- 每个次要存储服务器必须可供Zone区域中的所有主机使用。
##### 添加次要存储

创建新Zone区域时，将添加第一个次要存储作为该过程的一部分。您可以随时添加二级存储服务器，以将更多服务器添加到现有Zone区域。

> 确保服务器上未存储任何内容。将服务器添加到CloudStack将销毁任何现有的数据。

- 若要准备基于区域的辅助暂存存储，应在 Management Server 安装期间创建并装载 NFS 共享。请参阅 [[#准备NFS Shares]] 。如果您使用的是 Hyper-V 主机，请确保已创建 SMB 共享。
- 请确保在 Management Server 安装过程中准备好了系统 VM 模板。请参阅 [[#准备系统VM模板]]
- 以root管理员登录CloudStack UI
- 左侧导航栏，点击Infrastructure
- 在次要存储中，点击View all
- 点击添加次要存储
- 填写以下字段
    - Name 给存储一个描述性的名称
    - Provider 选择 S3、Swift、NFS 或 CIFS，然后填写显示的相关字段。这些字段将因存储提供商而异;有关更多信息，请参阅提供商的文档（例如 S3 或 Swift 网站）。NFS 可用于基于区域的存储，其他 NFS 可用于区域范围的存储。对于 Hyper-V，请选择 SMB/CIFS。

> 区域不支持异构辅助存储。每个区域只能使用一个 NFS、S3 或 Swift 账户。

- 填写以下字段
    - 创建 NFS次要暂存(staging)存储。必须始终选中此框。

> 即使 UI 允许您取消选中此框，也不要这样做。必须填写此复选框及其下方的三个字段。即使将 Swift 或 S3 用作次要存储提供程序，仍需要每个Zone区域中的 NFS 暂存存储。


- 填写以下字段
    - Zone NFS 次要暂存存储所在的Zone区域。
    - SMB Username 仅当选择 SMB/CIFS 提供商时才适用。对 SMB 共享具有必要权限的帐户的用户名。用户必须是 Hyper-V 管理员组的一部分。
    - SMB Password 仅当选择 SMB/CIFS 提供商时才适用。与帐户关联的密码。
    - SMB Domain 仅当选择 SMB/CIFS 提供商时才适用。SMB 共享所属的 Active Directory 域。
    - NFS Server Zone区域的次要暂存存储的名称
    - Path Zone区域的次要暂存存储的路径
###### 为每个Zone区域添加一个NFS 次要 Staging Store

每个Zone区域必须至少预配一个 NFS 存储;每个Zone区域允许多个 NFS 服务器。要为Zone区域预配 NFS 暂存存储，请执行以下操作：

1. Log in to the CloudStack UI as root administrator.
2. In the left navigation bar, click Infrastructure.
3. In Secondary Storage, click View All.
4. In Select View, choose Secondary Staging Store.
5. Click the Add NFS Secondary Staging Store button.
6. Fill out the dialog box fields, then click OK:
    - Zone. The zone where the NFS Secondary Staging Store is to be located.
    - NFS server. The name of the zone’s Secondary Staging Store
    - Path. The path to the zone’s Secondary Staging Store.

###### 添加对象存储

https://docs.cloudstack.apache.org/en/4.19.0.0/installguide/configuration.html#adding-object-storage

#### 初始化和测试

一切配置完成后，CloudStack将进行初始化。这可能需要 30 分钟或更长时间，具体取决于您的网络速度。当初始化成功完成后，管理员的Dashboard应该会显示在CloudStack的UI中。

- 验证系统是否已准备就绪。在左侧导航栏中，选择“模板”。单击 CentOS 5.5 （64bit） no Gui （KVM） 模板。检查以确保状态为“下载完成”。在显示此状态之前，不要继续执行下一步。
- 去Instances tab页面，过滤我的Instances
- 点击添加实例，然后遵循以下步骤
    -  选择你之前添加的zone
    - 在模板选项，选择要在实例中使用的模板，如果这是全新安装，可能只有提供的 CentOS 模板可用。
    - 选择服务offering。确保您拥有的硬件允许启动所选的服务offering。
    - In data disk offering, if desired, add another data disk. This is a second volume that will be available to but not mounted in the guest. For example, in Linux on XenServer you will see /dev/xvdb in the guest after rebooting the instance. A reboot is not required if you have a PV-enabled OS kernel in use.
    - In default network, choose the primary network for the guest. In a trial installation, you would have only one option here.
    - Optionally give your instance a name and a group. Use any descriptive text you would like.
    - Click Launch instance. Your instance will be created and started. It might take some time to download the Template and complete the instance startup. You can watch the instance’s progress in the Instances screen.
- to use the instance, click the View Console button.

有关使用实例的更多信息，包括有关如何允许传入网络流量进入实例、启动、停止和删除实例以及将实例从一台主机移动到另一台主机的说明，请参阅《管理员指南》中的使用虚拟机。

祝贺！你已经成功完成了CloudStack的安装。

如果您决定扩展部署，则可以添加更多主机、主存储、Zone区域、Pod 和集群。
#### 配置参数

##### 关于配置参数

CloudStack提供了多种设置，你可以用它来设置限制、配置功能，以及启用或禁用云中的功能。管理服务器运行后，可能需要设置其中一些配置参数，具体取决于要设置的可选功能。您可以在全局级别设置默认值，除非在较低级别覆盖它们，否则该默认值将在整个云中生效。您可以在账户，Zone区域，集群或主存储级别进行本地设置，这些设置将覆盖全局配置参数值。

每个CloudStack功能的文档都应该引导你找到适用参数的名称。下表显示了一些更有用的参数。

https://docs.cloudstack.apache.org/en/4.19.0.0/installguide/configuration.html#about-configuration-parameters
##### 设置全局配置参数

https://docs.cloudstack.apache.org/en/4.19.0.0/installguide/configuration.html#setting-global-configuration-parameters

##### 设置本地配置参数

https://docs.cloudstack.apache.org/en/4.19.0.0/installguide/configuration.html#setting-local-configuration-parameters

#### 粒度(Granular)全局配置参数

https://docs.cloudstack.apache.org/en/4.19.0.0/installguide/configuration.html#granular-global-configuration-parameters

## Hypervisor设置

https://docs.cloudstack.apache.org/en/4.19.0.0/installguide/index.html#hypervisor-setup

### 自定义Hypervisor安装

https://docs.cloudstack.apache.org/en/4.19.0.0/installguide/hypervisor/custom.html#custom-hypervisor-installation

### 主机Hyper-V安装

https://docs.cloudstack.apache.org/en/4.19.0.0/installguide/hypervisor/hyperv.html#host-hyper-v-installation
### 主机KVM安装

#### KVM Hypervisor主机的系统要求

KVM 包含在各种基于 Linux 的操作系统中。尽管您不需要运行这些发行版，但建议执行以下操作：

- CentOS / RHEL：7.X
- CentOS / RHEL / 二进制兼容变体：8.X
- Ubuntu的版本：18.04 +
- openSUSE / SLES：15.2 +

KVM Hypervisor的主要要求是 libvirt 和 Qemu 版本。无论您使用哪种 Linux 发行版，请确保满足以下要求：

- libvirt：1.2.0 或更高版本
- Qemu/KVM：1.5 或更高版本（推荐 2.5 或更高版本）

CloudStack中的默认桥接是Linux原生的桥接实现（bridge模块）。CloudStack包含一个与OpenVswitch一起工作的选项，要求如下

- libvirt：1.2.0 或更高版本
- OpenvSwitch：1.7.1 或更高版本

并非所有版本的 Qemu/KVM 都支持实例的动态扩展。某些组合可能会导致实例部署期间出现与 CPU 或内存相关的故障。

另外，硬件相关的要求如下:

- 在单个集群中，主机必须具有相同的分发版本。
- 群集中的所有主机都必须是同构的。CPU 必须具有相同的类型、计数和功能标志。
- 必须支持 HVM（启用 Intel-VT 或 AMD-V）
- 64 位 x86 CPU（内核越多，性能越好）
- 4 GB 内存
- 至少 1 个 NIC
- 当你部署CloudStack时，Hypersivor主机不能有任何已经运行的实例。这些将被CloudStack销毁。
#### KVM安装概览

如果要使用 Linux 内核虚拟机 （KVM） Hypervisor来运行访客实例，请在云中的主机上安装 KVM（也就是KVM host运行在一个KVM虚拟机中，套娃）。本节中的材料不会重复 KVM 安装文档。它提供了CloudStack特定的步骤，这些步骤是准备一个KVM主机来与CloudStack一起工作所需要的。

> 在继续操作之前，请确保已将最新更新应用于主机。

> 不建议在这个不受CloudStack控制的主机上运行服务。

> 某些服务器（如戴尔）提供了选择电源管理配置文件的选项。有源电源控制器启用了Dell System DBPM（基于需求的电源管理），这可以限制操作系统可用的最大CPU时钟速度的可见性，这反过来又可能导致CloudStack获取服务器不正确的CPU速度。为了确保CloudStack始终能够在服务器上获取最大的CPU速度，请确保将“操作系统控制”设置为电源管理配置文件。

安装 KVM Hypervisor主机的过程如下：

- 准备操作系统
- 安装和配置 libvirt
- 配置安全策略（AppArmor 和 SELinux） ^4e9011
- 安装和配置代理

##### KVM 安装

[Centos7安装KVM虚拟化-CSDN博客](https://blog.csdn.net/u012830695/article/details/126555312)[

[CentOS7.0部署KVM虚拟机-腾讯云开发者社区-腾讯云 (tencent.com)](https://cloud.tencent.com/developer/article/2224839)

```bash
cat /proc/cpuinfo | grep [vmx | smv]
lsmod | grep kvm
systemctl stop firewalld
systemctl disable firewalld
yum install epel* -y
yum install libvirt qemu-kvm virt-viewer bridge-utils avahi dmidecode qemu-kvm-tools virt-manager qemu-img virt-install net net-tools openssl
systemctl enable libvirtd
systemctl start libvirtd
```
#### 准备操作系统

这里与 之前的 [[#准备操作系统]] 一致，不做赘述。

#### 安装和配置代理(Agent)

为了管理主机上的KVM实例，CloudStack使用Agent。此代理与管理服务器通信并控制主机上的所有实例。

> 根据你的发行版，你可能需要为CloudStack添加相应的软件包存储库。

```bash
# RHEL or CentOS
yum install -y epel-release
yum install cloudstack-agent
# ubuntu
apt install cloudstack-agent
# SUSE
zypper install cloudstack-agent
```

主机现在已准备好添加到集群中。这将在后面的部分中介绍，请参阅 [[#添加一个主机]] 。建议您在添加主机之前继续阅读文档！

如果您使用非 root 用户添加 KVM 主机，请将该用户添加到 sudoers 文件：

```conf
cloudstack ALL=NOPASSWD: /usr/bin/cloudstack-setup-agent
Defaults:cloudstack !requiretty
```

##### 为KVM guest配置CPU model(可选)

https://docs.cloudstack.apache.org/en/4.19.0.0/installguide/hypervisor/kvm.html#configure-cpu-model-for-kvm-guest-optional

#### 安装和配置libvirt

CloudStack使用libvirt来管理实例。因此，正确配置 libvirt 至关重要。Libvirt 是 cloudstack-agent 的依赖项，应该已经安装好了。

> 请注意，Cloudstack会在添加主机时自动执行agent和libvirt的基本配置。如果您计划自动部署和配置 KVM 主机，这一点很重要。

- 为了避免对实例的潜在安全攻击，我们需要关闭 libvirt 来监听不安全的 TCP 端口。当主机被添加到cloudstack时，CloudStack会自动设置云密钥库和证书。我们还需要关闭 libvirts 尝试使用组播 DNS 广播。这两个设置都在 /etc/libvirt/libvirtd.conf 中

```conf
listen_tls = 0
listen_tcp = 0
tls_port = "16514"
tcp_port = "16509"
auth_tcp = "none"
mdns_adv = 0
```

- 我们还必须更改参数：

在 RHEL 或 CentOS 或 SUSE 上，请修改 /etc/sysconfig/libvirtd：

取消注释以下行：

```conf
#LIBVIRTD_ARGS="--listen"
```

> `--listen`参数告诉libvirtd守护进程在启动时打开TCP/IP监听（默认情况下，libvirtd只接受本地UNIX套接字连接）。这意味着libvirtd将接受远程主机上的连接，这对于管理远程虚拟机是必需的。

在 RHEL 8 / CentOS 8 / SUSE 上运行以下命令：

```bash
systemctl mask libvirtd.socket libvirtd-ro.socket libvirtd-admin.socket libvirtd-tls.socket libvirtd-tcp.socket
```

> 禁用一些服务，使其不能被启动，即使是手动也不行。这是通过创建一个指向`/dev/null`的符号链接来实现的。. 在这个命令中，`libvirtd.socket`，`libvirtd-ro.socket`，`libvirtd-admin.socket`，`libvirtd-tls.socket`和`libvirtd-tcp.socket`都是libvirt的socket服务。这些服务用于接收来自其他进程的连接请求。
> 
> CloudStack文档中的这个命令的目的是禁用这些socket服务，这可能是因为CloudStack希望直接控制libvirtd服务，而不是通过这些socket服务。这样可以避免其他进程干扰CloudStack管理虚拟机。

在 Ubuntu 20.04 或更早版本上，修改 /etc/default/libvirtd

取消注释并更改以下行

```conf
#libvirtd_opts=""
```

像这样:

```conf
libvirtd_opts="-l"
```


> 以上配置是老版本的libvirt的配置，其实与上面的--listen是一个意思

在 Ubuntu 22.04 或更高版本上，修改 /etc/default/libvirtd：

取消注释以下行：

```conf
#LIBVIRTD_ARGS="--listen"
```

- 重启libvirt

```bash
systemctl restart libvirtd
```
#### 配置安全策略

如果Hypervisor的Host是CentOS RHEL  SUSE系统，需要永久关闭SELINUX

[[#^776df0]]

如果Hypervisor的Host是ubuntu，需要配置Apparmor [[#^4e9011]]
#### 配置网络

> 这个章节非常重要，需要仔细阅读

> 本节详细介绍了如何在 Linux 中使用本机实现配置网桥(bridge)。如果您打算使用 OpenVswitch，请参考下一节

CloudStack将网桥与KVM结合使用，将访客实例彼此连接并连接到外部世界。它们还用于将系统虚拟机连接到你的Infrasturecture。

默认情况下，这些网桥称为 cloudbr0 和 cloudbr1 等，但可以将其更改为更具描述性。

> 确保用于配置网桥的接口名称与以下模式之一匹配：'eth*'、'bond*'、'team*'、'vlan*'、'em*'、'p*p*'、'ens*'、'eno*'、'enp*'、'enx*'。否则，KVM 代理将无法正确配置网桥。

> 在所有Hypervisor中保持配置一致至关重要。

有多种方法可以配置网络。即使在给定网络模式的范围内。以下是一些简单的示例。

> 从 Ubuntu 20.04 开始，管理网络连接的标准是使用 NetPlan YAML 文件。有关详细信息，请参阅 Ubuntu 手册页，并形象地设置网络连接。
##### 网络例子之基本网络

在基本网络中，给定Pod中的所有访客机都将位于同一 VLAN/子网上。通常将本地（untagged）VLAN 用于专用/管理网络，因此在此示例中，我们将有两个 VLAN，一个（本地）用于专用/管理网络，另一个用于访客网络。

我们假设Hypervisor有一个 NIC （eth0），这个网卡中包含一个从交换机中继的tagged VLAN：

-  Native VLAN for Management Network (cloudbr0)
-  VLAN 200 for guest network of the Instances (cloudbr1)

在以下示例中，我们为Hypervisor提供 IP 地址 192.168.42.11/24，网关为 192.168.42.1

> Hypervisor和管理服务器不必位于同一子网中

> "Native VLAN"是一个特殊的VLAN，它用于处理在交换机端口上未标记的流量。在这个配置中，本地VLAN被用于管理网络，这意味着所有未标记的流量都被认为是管理网络的流量。

> 这个配置允许Hypervisor通过一个物理网卡同时处理管理网络和访客网络的流量，这大大简化了网络配置和管理。
###### 为基本网络配置Network bridges

这取决于您使用的发行版如何配置这些，您将在下面找到 RHEL/CentOS、SUSE 和 Ubuntu 的示例。

> 目标是在此部分之后拥有两个名为“cloudbr0”和“cloudbr1”的网桥。这仅应用作指南。确切的配置将取决于您的网络布局。
###### 在  RHEL或CentOS上配置基本网络

在安装 libvirt 时安装了所需的软件包，我们可以继续配置网络。

首先，我们配置 eth0

```bash
 vi /etc/sysconfig/network-scripts/ifcfg-eth0
```

让配置像下面这样:

```conf
DEVICE=eth0
HWADDR=00:04:xx:xx:xx:xx
# 这意味着在系统启动时自动启用这个接口。
ONBOOT=yes
# 这禁止了热插拔，意味着这个接口不能在系统运行时被添加或删除。
HOTPLUG=no
# 这意味着这个接口不使用任何协议来获取IP地址。因为这个接口是一个桥接接口，所以它不需要自己的IP地址。
BOOTPROTO=none
# 这是网络接口的类型。
TYPE=Ethernet
# 这是这个接口所连接的桥的名称。
BRIDGE=cloudbr0
```

> 这个配置文件是用来设置`eth0`网络接口的。在这个配置中，`eth0`被设置为一个桥接接口，它将被添加到`cloudbr0`桥上。这意味着所有通过`eth0`接口的流量都将被转发到`cloudbr0`桥。

> 在CloudStack中，这个配置通常用于设置管理网络。管理网络是用于CloudStack内部通信的，包括CloudStack管理服务器和hypervisor之间的通信。通过将`eth0`接口添加到`cloudbr0`桥，CloudStack可以通过这个接口来管理hypervisor和虚拟机。

然后我们必须配置VLAN接口

```bash
yum install vconfig
 vi /etc/sysconfig/network-scripts/ifcfg-eth0.200
```

```conf
# 这个配置主要注意，我我的机器上eth0.200  systemctl restart network启动失败，说是这个VLAN 200的设备配置错误，但是我DEVICE的配置修改成eth0后，就restart成功了
DEVICE=eth0.200
HWADDR=00:04:xx:xx:xx:xx
ONBOOT=yes
HOTPLUG=no
BOOTPROTO=none
TYPE=Ethernet
VLAN=yes
BRIDGE=cloudbr1
```

> 以上配置在我的机器上启动报错，所以我用nmcli的命令加的VLAN :  `nmcli con add type vlan con-name eth0.200 dev eth0 id 200` 

然后修改一下生成的配置，像以下这样:

```conf
VLAN=yes  
TYPE=Vlan  
PHYSDEV=eth0 
VLAN_ID=200  
#REORDER_HDR=yes  
#GVRP=no  
#MVRP=no  
HWADDR=d4:be:d9:fa:49:e8  
#PROXY_METHOD=none  
#BROWSER_ONLY=no  
BOOTPROTO=none  
#DEFROUTE=yes  
#IPV4_FAILURE_FATAL=no  
#IPV6INIT=yes  
#IPV6_AUTOCONF=yes  
#IPV6_DEFROUTE=yes  
#IPV6_FAILURE_FATAL=no  
#IPV6_ADDR_GEN_MODE=stable-privacy  
NAME=eth0.200  
UUID=baf991af-bf3c-403d-9a81-15846dd7efc8  
ONBOOT=yes  
BRIDGE=cloudbr1
```

现在我们已经配置了 VLAN 接口，我们可以在它们之上添加网桥。

```bash
vi /etc/sysconfig/network-scripts/ifcfg-cloudbr0
```

现在，我们配置 cloudbr0 并包含虚拟机管理程序的管理 IP。

> 虚拟机管理程序的管理 IP 不必与管理网络位于同一子网/VLAN 中，但很常见。

```conf
DEVICE=cloudbr0
TYPE=Bridge
ONBOOT=yes
BOOTPROTO=none
# 这两个选项禁用了IPv6。
IPV6INIT=no
# 这两个选项禁用了IPv6。
IPV6_AUTOCONF=no
# 这是桥接接口启动的延迟时间，单位是秒。这个选项可以防止循环，特别是在STP（生成树协议）启用的情况下。
DELAY=5
# 这是桥接接口的IP地址
IPADDR=192.168.42.11
GATEWAY=192.168.42.1
NETMASK=255.255.255.0
# 这启用了STP（生成树协议）。STP是一种防止网络循环的协议，特别是在有多个桥接接口的情况下
STP=yes
```

我们将 cloudbr1 配置为没有 IP 地址的普通网桥 `vi /etc/sysconfig/network-scripts/ifcfg-cloudbr1`

```conf
DEVICE=cloudbr1
TYPE=Bridge
ONBOOT=yes
BOOTPROTO=none
IPV6INIT=no
IPV6_AUTOCONF=no
DELAY=5
STP=yes
```

使用此配置，您应该能够重新启动网络，但建议重新启动以查看是否一切正常。

```bash
# VLAN需要驱动支持，按理说已经有了，不过可以通过lsmod看看,有一个类似8006a的内核模块
lsmod | grep "*a"
systemctl restart network
systemctl status network -l
# 如果是单独启动某个接口设备
ifup eth0.200
# 查看配置的接口列表，你可以看到VLAN 和桥接设备这些都已经启动
ip link show
```

> 确保您有其他方式（如 IPMI 或 ILO）来访问计算机，以防您出现配置错误并且网络停止运行！
###### 在SUSE上配置基本网络

https://docs.cloudstack.apache.org/en/4.19.0.0/installguide/hypervisor/kvm.html#configure-suse-for-basic-networks

###### 在ubuntu上配置基本网络

https://docs.cloudstack.apache.org/en/4.19.0.0/installguide/hypervisor/kvm.html#configure-ubuntu-for-basic-networks

##### 网络例子之高级网络

在高级网络模式下，最常见的情况是每个 hypervior-host 具有（至少）两个物理接口。我们将使用链接到网桥“cloudbr0”的接口 eth0，使用untagged的（本机）VLAN 进行Hypervisor。此外，我们还配置了第二个接口，以便与网桥“cloudbr1”一起用于公共和访客流量。这次我们没有应用VLAN，CloudStack会在实际使用过程中根据需要添加VLAN。

我们再次为Hypervisor提供 IP 地址 192.168.42.11/24，网关为 192.168.42.1

> Hypervisor和管理服务器不必位于同一子网中

###### 为高级网络配置Network bridges

https://docs.cloudstack.apache.org/en/4.19.0.0/installguide/hypervisor/kvm.html#configuring-the-network-bridges-for-advanced-networks

###### 在RHEL或CentOS上配置高级网络

https://docs.cloudstack.apache.org/en/4.19.0.0/installguide/hypervisor/kvm.html#configure-rhel-centos-for-advanced-networks

###### 在SUSE上配置高级网络

https://docs.cloudstack.apache.org/en/4.19.0.0/installguide/hypervisor/kvm.html#configure-suse-for-advanced-networks

###### 在ubuntu上配置高级网络

https://docs.cloudstack.apache.org/en/4.19.0.0/installguide/hypervisor/kvm.html#configure-ubuntu-for-advanced-networks

##### 使用OpenVswitch配置网络

> 这个章节特别重要，请仔细阅读

为了将流量转发到您的实例，您至少需要两个网桥：公有网桥和私有网桥。

默认情况下，这些网桥称为 cloudbr0 和 cloudbr1，但您必须确保它们在每个Hypervisor Host上都可用。

最重要的因素是，在所有Hypervisor Hots上保持配置一致。
###### 准备

为了确保原生网桥模块不会干扰 openvswitch，应将网桥模块添加到拒绝列表（可能命名为“denylist”），请参阅您的发行版的 modprobe 文档，了解在哪里可以找到拒绝列表。在执行后续步骤之前，请确保未通过重新启动或执行 rmmod bridge 来加载模块。

下面的网络配置取决于 ifup-ovs 和 ifdown-ovs 脚本，它们是 openvswitch 安装的一部分。它们应该安装在 /etc/sysconfig/network-scripts/ 中

###### OpenVswitch 网络例子

有多种方法可以配置您的网络。在基本网络模式下，您应该有两个 VLAN，一个用于专用网络，一个用于公用网络。

我们假设Hypervisor host有一个 NIC （eth0） ，并且这个网卡上有三个 tagged VLAN：

- VLAN 100 for management of the hypervisor
- VLAN 200 for public network of the Instances (cloudbr0)
- VLAN 300 for private network of the instances (cloudbr1)

在 VLAN 100 上，我们为Hypervisor提供 IP 地址 192.168.42.11/24，网关为 192.168.42.1

> Hypervisor和管理服务器不必位于同一子网中
###### 为OpenVswitch配置network bridges

这取决于您使用的发行版如何配置这些，您将在下面找到 RHEL/CentOS 的示例。

> 目标是在本节之后有三个网桥，分别称为“mgmt0”、“cloudbr0”和“cloudbr1”。这仅应用作指南。确切的配置将取决于您的网络布局。
###### 配置OpenVswitch

使用 OpenVswitch 的网络接口是使用 ovs-vsctl 命令创建的。此命令将配置接口并将它们保存到 OpenVswitch 数据库中。

首先，我们创建一个连接到 eth0 接口的主桥(main bridge)。接下来，我们创建三个假网桥（fake bridge），每个网桥都连接到一个特定的 vlan 标签(tag)。

```bash
ovs-vsctl add-br cloudbr
ovs-vsctl add-port cloudbr eth0
ovs-vsctl set port cloudbr trunks=100,200,300
ovs-vsctl add-br mgmt0 cloudbr 100
ovs-vsctl add-br cloudbr0 cloudbr 200
ovs-vsctl add-br cloudbr1 cloudbr 300
```

###### 在RHEL或CentOS中配置OpenVswitch

在安装 openvswitch 和 libvirt 时安装了所需的软件包，我们可以继续配置网络。

首先，我们配置 eth0 `vi /etc/sysconfig/network-scripts/ifcfg-eth0`

确保配置看起来如下:

```bash
DEVICE=eth0
HWADDR=00:04:xx:xx:xx:xx
ONBOOT=yes
HOTPLUG=no
BOOTPROTO=none
TYPE=Ethernet
```

我们必须使用中继配置基桥（the base bridge with trunck）。`vi /etc/sysconfig/network-scripts/ifcfg-cloudbr`

```conf
DEVICE=cloudbr
ONBOOT=yes
HOTPLUG=no
BOOTPROTO=none
DEVICETYPE=ovs
TYPE=OVSBridge
```

现在，我们必须配置三个 VLAN 网桥：`vi /etc/sysconfig/network-scripts/ifcfg-mgmt0`

```conf
DEVICE=mgmt0
ONBOOT=yes
HOTPLUG=no
BOOTPROTO=static
DEVICETYPE=ovs
TYPE=OVSBridge
IPADDR=192.168.42.11
GATEWAY=192.168.42.1
NETMASK=255.255.255.0
```

`vi /etc/sysconfig/network-scripts/ifcfg-cloudbr0`

```conf
DEVICE=cloudbr0
ONBOOT=yes
HOTPLUG=no
BOOTPROTO=none
DEVICETYPE=ovs
TYPE=OVSBridge
```

`vi /etc/sysconfig/network-scripts/ifcfg-cloudbr1`

```conf
DEVICE=cloudbr1
ONBOOT=yes
HOTPLUG=no
BOOTPROTO=none
TYPE=OVSBridge
DEVICETYPE=ovs
```

使用此配置，您应该能够重新启动网络，但建议重新启动以查看是否一切正常。

> 确保您有其他方式（如 IPMI 或 ILO）来访问计算机，以防您出现配置错误并且网络停止运行！

###### 在SUSE上配置OpenVswitch

https://docs.cloudstack.apache.org/en/4.19.0.0/installguide/hypervisor/kvm.html#configure-openvswitch-in-suse
#### 配置防火墙

Hypervisor需要能够与其他Hypervisor hosts进行通信，并且管理服务器需要能够访问Hypervisor host。

为此，我们必须打开以下TCP端口（如果您使用的是防火墙）：

- 22 （SSH）
- 1798
- 16514 （libvirt）
- 5900 - 6100（VNC 控制台）
- 49152 - 49216（libvirt 实时迁移）

这取决于您使用的防火墙如何打开这些端口。您将在下面找到如何在 RHEL/CentOS 和 Ubuntu 中打开这些端口的示例。

##### 在RHEL/CentOS/SUSE上打开端口

RHEL 和 CentOS 使用 iptables 为系统设置防火墙，您可以通过执行以下 iptable 命令来打开额外的端口：

```bash
iptables -I INPUT -p tcp -m tcp --dport 22 -j ACCEPT
iptables -I INPUT -p tcp -m tcp --dport 1798 -j ACCEPT
iptables -I INPUT -p tcp -m tcp --dport 16514 -j ACCEPT
iptables -I INPUT -p tcp -m tcp --dport 5900:6100 -j ACCEPT
iptables -I INPUT -p tcp -m tcp --dport 49152:49216 -j ACCEPT
```

这些 iptable 设置在重新启动后不会持久化，我们必须先保存它们。

```bash
iptables-save > /etc/sysconfig/iptables
```

> 在 RHEL 8 / CentOS 8 / SUSE 上，firewalld 是默认的防火墙管理器并控制 iptables。建议禁用 systemctl stop firewalld ;systemctl 禁用防火墙

> 在 SUSE 上，iptables 不会在重新启动时保留，因此建议创建 iptables 和 ip6tables 服务以确保它们持久存在
##### 在ubuntu上打开端口

https://docs.cloudstack.apache.org/en/4.19.0.0/installguide/hypervisor/kvm.html#open-ports-in-ubuntu

##### 额外的功能包

###### 次要存储的Bypass

4.11 中的新功能是能够绕过将模板存储在次要存储上，而是直接从备用远程位置下载“模板”。为此，必须在所有 KVM 主机上安装 Aria2 （https://aria2.github.io/） 软件包。

由于此包在标准分发存储库中通常不可用，因此需要从首选源安装该包。

###### Volume快照

CloudStack使用qemu-img来执行Snapshots。在 CentOS >= 6.5 中，RedHat/CentOS 提供的 qemu-img 不再包含执行快照的 '-s' 开关。“-s”开关已在最新的 CentOS/RHEL 7.x 版本中恢复。

为了能够在 CentOS 6.x（高于 6.4）上执行卷快照，您必须将 qemu-img 版本替换为已修补以包含“-s”开关的版本。

###### Live Migration

对于客户机的实时迁移，最好在 KVM 主机的同一接口上配置客户机网桥。如果在 KVM 主机的不同接口上配置了访客网桥，请确保目标主机在源主机中没有具有访客网桥接口名称的接口。

###### UEFI legacy/secure boot

要使用 UEFI 旧版/安全启动部署实例，还需要执行一些其他任务。你可以在CloudStack Wiki（https://cwiki.apache.org/confluence/display/CLOUDSTACK/Enable+UEFI+booting+for+Instance）上找到关于先决条件的更多信息，以及在CloudStack中使用UEFI的限制。可以在实例部署向导的“高级模式”部分中找到使用 UEFI 部署实例的选项。

#### 把主机添加到CloudStack

主机现在已准备好添加到集群中。这将在后面的部分中介绍，请参阅  [[#添加一个主机]]  。建议您在添加主机之前继续阅读文档！

### 主机LXC安装

忽略，看官方

### 主机VMware vSphere安装

忽略，看官网

### 主机 Citrix XenServer安装

忽略，看官网

## 可选安装

https://docs.cloudstack.apache.org/en/4.19.0.0/installguide/index.html#optional-installation

### 附加的安装选项

https://docs.cloudstack.apache.org/en/4.19.0.0/installguide/optional_installation.html#additional-installation-options

### 关于Password和Key加密

https://docs.cloudstack.apache.org/en/4.19.0.0/installguide/encryption.html#about-password-and-key-encryption