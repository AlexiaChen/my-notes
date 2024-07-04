在CloudStack中，访客实例可以使用共享的基础设施相互通信，同时确保访客拥有私有局域网。CloudStack虚拟路由器是为访客流量提供网络功能的主要组件。
## 访客流量

一个网络只能在一个Zone区域内的实例之间传输访客流量。不同Zone可用区的实例无法通过其 IP 地址相互通信;它们必须通过公共 IP 地址（公网地址）进行路由来相互通信。

请参阅下面给出的典型访客流量设置：

![[Pasted image 20240307163416.png]]

通常，管理服务器(management server)会自动为每个网络创建一个虚拟路由器(Virtual Router)。虚拟路由器(Virtual Router)是在主机上运行的特殊实例。隔离网络中的每个虚拟路由器都有三个网络接口。如果使用多个公共 VLAN，则路由器将具有多个公共接口。其 eth0 接口用作访客流量的网关，IP 地址为 10.1.1.1。系统使用其 eth1 接口来配置虚拟路由器。它的 eth2 接口为公共流量分配了一个公共 IP 地址。如果使用多个公共 VLAN，则路由器将具有多个公共接口。

虚拟路由器提供 DHCP，并将自动为为网络分配的 IP 范围内的每个访客实例分配一个 IP 地址。用户可以手动重新配置访客实例以采用不同的 IP 地址。

source NAT 在虚拟路由器中自动配置，以转发所有访客实例的出站流量
## 在一个Pod中的网络

下图说明了单个Pod中的网络设置。主机连接到Pod级交换机(L2交换机)。主机至少应有一个到每个交换机的物理上行链路。还支持绑定 NIC。Pod 级交换机是一对具有 10 G 上行链路的冗余千兆交换机。

![[Pasted image 20240307163954.png]]

服务器的连接方式如下：

- 存储设备仅连接到承载管理流量的网络。
- 主机连接到网络，用于管理流量和公共流量。
- 主机还连接到一个或多个承载访客流量的网络。

我们建议使用多个物理以太网卡来实现每个网络接口以及冗余交换结构，以最大限度地提高吞吐量并提高可靠性。

## 在一个Zone中的网络

下图说明了单个Zone区域中的网络设置。

![[Pasted image 20240307164258.png]]

管理流量的防火墙在 NAT 模式下运行。通常在 192.168.0.0/16 B 类专用地址空间中为网络分配 IP 地址。每个 Pod 都在 192.168.\*.0/24 C 类专用地址空间中分配了 IP 地址。

每个Zone区域都有自己的一组公共 IP 地址。来自不同Zone区域的公共 IP 地址不会重叠。

## 基本的Zone物理网络配置

在基本网络中，配置物理网络相当简单。您只需配置一个访客网络即可传输访客实例生成的流量。当你第一次向CloudStack添加一个Zone区域时，你通过添加Zone区域屏幕来设置访客网络。

## 高级的Zone物理网络配置

在使用高级网络的Zone区域中，您需要告诉管理服务器如何设置物理网络以隔离传输不同类型的流量。

https://docs.cloudstack.apache.org/en/latest/adminguide/networking_and_traffic.html#advanced-zone-physical-network-configuration
## 编辑-重启-删除一个访客网络

https://docs.cloudstack.apache.org/en/latest/adminguide/networking_and_traffic.html#editing-restarting-and-removing-a-guest-network

## 使用多个访客网络

在使用高级网络的Zone区域中，可以在初始安装后随时为访客流量添加其他网络。您还可以通过为每个网络指定 DNS 后缀来自定义与网络关联的域名。

实例的网络是在创建实例时定义的。实例在创建后无法添加或删除网络，但用户可以进入访客机（guest）并从特定网络上的 NIC 中删除 IP 地址。

每个实例只有一个默认网络。虚拟路由器的 DHCP 应答会将访客机的默认网关设置为默认网络的网关。除了单个必需的默认网络外，还可以向客户机添加多个非默认网络。管理员可以控制哪些网络可用作默认网络。

其他网络可以可供所有帐户使用，也可以分配给特定帐户。可供所有帐户使用的网络是Zone区域范围的。任何有权访问该Zone区域的用户都可以创建有权访问该网络的实例。这些Zone区域范围的网络在访客之间提供很少或根本没有隔离。分配给特定帐户的网络提供强隔离。

https://docs.cloudstack.apache.org/en/latest/adminguide/networking_and_traffic.html#using-multiple-guest-networks
## 访客网络权限

https://docs.cloudstack.apache.org/en/latest/adminguide/networking_and_traffic.html#using-multiple-guest-networks

## 在隔离的访客网络中的IP保留

在隔离的访客网络中，可以为非CloudStack实例或物理服务器保留一部分访客IP地址空间。为此，您可以通过在访客网络处于“已实现”状态时指定 CIDR 来配置一系列保留 IP 地址。如果你的客户希望在同一网络上拥有非CloudStack控制的实例或物理服务器，他们可以共享一部分主要提供给访客网络的IP地址空间。

在高级Zone区域中，定义网络时，会将 IP 地址范围或 CIDR 分配给网络。CloudStack虚拟路由器充当DHCP服务器，并使用CIDR为访客实例分配IP地址。如果你决定为非CloudStack目的保留CIDR，你可以指定IP地址范围的一部分或CIDR，这些IP地址范围或CIDR应该只由虚拟路由器的DHCP服务分配给在CloudStack中创建的访客实例。该网络中的其余 IP 称为保留 IP 范围。配置IP预留后，管理员可以将不属于CloudStack的其他实例或物理服务器添加到同一网络中，并为其分配预留IP地址。CloudStack 访客实例无法从保留IP范围获取IP。

https://docs.cloudstack.apache.org/en/latest/adminguide/networking_and_traffic.html#ip-reservation-in-isolated-guest-networks

## 保留公有IP地址和账户的VLANs

CloudStack提供了为一个账户保留一组公共IP地址和VLAN的能力。在Zone区域创建期间，您可以继续定义一组 VLAN 和多个公共 IP 范围。此功能扩展了该功能，使你能够为租户专用一组固定的 VLAN 和访客 IP 地址。

请注意，如果一个帐户已消耗了所有专用于 IT 的 VLAN 和 IP，则该帐户可以从系统中再获取两个资源。CloudStack为root管理员提供了两个配置参数来修改这个默认行为：use.system.public.ips和use.system.guest.vlans。这些全局参数使 root 管理员能够禁止帐户从系统获取公共 IP 和访客 VLAN，前提是该帐户具有专用资源，并且这些专用资源已全部消耗。这两种配置都可以在账户级别进行配置。

此功能为您提供以下功能：

- 从高级Zone区域保留 VLAN 范围和公共 IP 地址范围，并将其分配给帐户
- 取消 VLAN 和公共 IP 地址范围与帐户的关联
- 查看分配给帐户的公共 IP 地址数
- 检查所需范围是否可用，是否符合账户限制。
- 每个帐户的最大 IP 限制不能被取代。

https://docs.cloudstack.apache.org/en/latest/adminguide/networking_and_traffic.html#reserving-public-ip-addresses-and-vlans-for-accounts

## 在单个网卡(NIC)上配置多个IP地址

CloudStack提供了为每个访客实例网卡关联多个私有IP地址的能力。除了主 IP 之外，您还可以将其他 IP 分配给访客实例 NIC。所有网络配置都支持此功能：基本网络、高级高级和 VPC。这些附加 IP 支持安全组、static NAT 和端口转发服务。

与往常一样，您可以从访客子网指定 IP;如果未指定，则会自动从访客实例子网中选取 IP。您可以在 UI 上查看与每个访客实例 NIC 关联的 IP。你可以通过使用CloudStack UI中的网络配置选项，在这些额外的访客IP上应用NAT。您必须指定应与 IP 关联的 NIC。

XenServer、KVM 和 VMware Hypervisor支持此功能。请注意，VMware 不支持基本Zone区域安全组。

https://docs.cloudstack.apache.org/en/latest/adminguide/networking_and_traffic.html#configuring-multiple-ip-addresses-on-a-single-nic
### 使用场景

下面介绍了一些用例：

- 网络设备（如防火墙和负载均衡器）通常在可以访问网络接口上的多个 IP 地址时效果最佳。
- 在接口或实例之间移动私有 IP 地址。绑定到特定 IP 地址的应用程序可以在实例之间移动。
- 在单个实例上托管多个 SSL 网站。您可以在单个实例上安装多个 SSL 证书，每个证书都与不同的 IP 地址相关联。

## 关于多个IP范围

> 只能是IPv4的地址

CloudStack提供了在基本Zone区域和启用了安全组的高级Zone区域中添加来自不同子网的访客IP范围的灵活性。对于启用了安全组的高级Zone区域，这意味着可以将多个子网添加到同一 VLAN。添加此功能后，当 IP 地址用尽时，您将能够从同一子网或从不同子网添加 IP 地址范围。这反过来又允许您使用更多数量的子网，从而减少地址管理开销。为了支持此功能，扩展了 `createVlanIpRange` API 的功能，以添加来自不同子网的 IP 范围。

请确保在添加 IP 范围之前手动配置新子网的网关。请注意，CloudStack只支持一个子网的网关;目前不支持重叠子网(overlapping subnet)。

使用 `deleteVlanRange` API 删除 IP 范围。如果正在使用删除范围中的 IP，则此操作将失败。如果删除范围包含运行DHCP服务器的IP地址，CloudStack将从同一子网获取一个新的IP。如果子网中没有可用的 IP，则删除操作将失败。

KVM、xenServer 和 VMware Hypervisor支持此功能。

## 关于弹性IP

弹性公网IP（Elastic IP，简称EIP）是与账户关联的IP地址，作为静态IP地址。账户所有者对属于该账户的弹性 IP 地址拥有完全控制权。作为账户所有者，您可以从账户的 EIP 池中将弹性 IP 分配给您选择的实例。稍后，如果需要，您可以将 IP 地址重新分配给其他实例。此功能在实例故障期间非常有用。无需替换已关闭的实例，而是可以将 IP 地址重新分配给您账户中的新实例。

与公有 IP 地址类似，弹性 IP 地址使用 Static NAT 映射到其关联的私有 IP 地址。弹性公网IP服务在支持弹性公网IP的基本Zone可用区中配备Static NAT（1：1）服务。如果在您的Zone区域中部署了 NetScaler 设备，则默认网络产品 DefaultSharedNetscalerEIPandELBNetworkOffering 可为您的网络提供 EIP 和 ELB 网络服务。有关详细信息，请考虑下图。

![[Pasted image 20240307170927.png]]

在图中，NetScaler设备是CloudStack实例的默认入口或出口点，防火墙是数据中心其余部分的默认入口或出口点。Netscaler 为访客网络提供 LB 服务和静态 NAT 服务。容器和管理服务器中的访客流量位于不同的子网/VLAN 上。数据中心核心交换机中基于策略的路由通过 NetScaler 发送公共流量，而数据中心的其余部分则通过防火墙。

弹性公网IP工作流程如下：

- 部署用户实例时，会自动从Zone区域中配置的公共 IP 池中获取公共 IP。此 IP 归实例的账户所有。
- 每个实例都有自己的私有 IP。当用户实例启动时，将使用公有 IP 和专用 IP 之间的入站网络地址转换 （INAT） 和反向 NAT （RNAT） 规则在 NetScaler 设备上置备静态 NAT。
	> 入站 NAT （INAT） 是 NetScaler 支持的一种 NAT 类型，其中来自公用网络（如 Internet）的数据包中的目标 IP 地址将替换为专用网络中实例的专用 IP 地址。反向 NAT （RNAT） 是 NetScaler 支持的一种 NAT，其中源 IP 地址在专用网络中的实例生成的数据包中替换为公有 IP 地址。
- 此默认公共 IP 将在两种情况下发布：
    - 当实例停止时。当实例启动时，它会再次从公有 IP 池中接收一个新的公有 IP，不一定是最初分配的公有 IP。
    - 用户获取公网IP（弹性公网IP）。此公共 IP 与帐户关联，但不会映射到任何私有 IP。但是，用户可以启用静态 NAT 以将此 IP 关联到账户中实例的私有 IP。可以随时禁用公共 IP 的静态 NAT 规则。禁用静态 NAT 后，将从池中分配一个新的公共 IP，该 IP 不一定与最初分配的 IP 相同。

对于公共 IP 资源有限的部署，可以灵活地选择默认情况下不分配公共 IP。您可以使用关联公网IP选项在启用弹性公网IP的基本Zone可用区中打开或关闭自动公网IP分配。如果您在创建网络产品（Network Offering）时关闭自动公有 IP 分配，则在使用该网络产品部署实例时，只会将私有 IP 分配给该实例。稍后，用户可以获取实例的 IP 并启用静态 NAT。

关联公有IP的选项请看 [[CloudStack为用户设置网络#创建一个新的网络产品(Offering)]]

> 关联公共 IP 功能仅用于用户实例。默认情况下，无论网络产品配置如何，系统 VM 都会继续获取公共 IP 和专用 IP。

使用默认共享网络产品和 EIP 和 ELB 服务在基本Zone区域中创建共享网络的新部署将继续为每个用户实例分配公共 IP。
## Portable IP

https://docs.cloudstack.apache.org/en/latest/adminguide/networking_and_traffic.html#portable-ips
### 关于Portable IP

CloudStack中的可移植IP是Region区域级的IP池，本质上是弹性的，可以跨地理上分开的Region区域传输。作为管理员，可以在Region区域级别预配可移植公共 IP 池，并可供用户使用。如果管理员在用户所属的Region区域级别预配了可移植 IP，则用户可以获取可移植 IP。这些 IP 可用于高级Zone区域内的任何服务。您也可以在基础Zone可用区使用可移植IP进行弹性公网IP业务。

便携式IP的突出特点如下：

- IP 是静态分配的
- IP 不需要与网络关联
- IP 关联可跨网络传输
- IP 可在基本和高级Zone区域之间传输
- IP 可跨 VPC、非 VPC 隔离和共享网络传输
- 可移植 IP 传输仅适用于static NAT。

> 新 UI 当前不支持可移植 IP。要管理可移植的IP，请直接调用相应的API，或者使用cloudstack的CLI工具cloudmonkey

## 共享网络中的多个子网

CloudStack提供了在基本Zone区域和启用了安全组的高级Zone区域中添加来自不同子网的访客IP范围的灵活性。对于启用了安全组的高级Zone区域，这意味着可以将多个子网添加到同一 VLAN。添加此功能后，当 IP 地址用尽时，您将能够从同一子网或从不同子网添加 IP 地址范围。这反过来又允许您使用更多数量的子网，从而减少地址管理开销。您可以删除已添加的 IP 范围。

https://docs.cloudstack.apache.org/en/latest/adminguide/networking_and_traffic.html#multiple-subnets-in-shared-network

### 前置要求和指南

-  此功能只能被实现如下
    - IPv4地址
    - 虚拟路由器是一个DHCP的提供者
    - 在KVM xenServer VMware Hypervisor
- 在添加IP范围之前，需要手动配置新子网的网关
- CloudStack一个子网只支持一个网关。overlapping 子网当前不支持

## 在高级的Zone网络中使用私有的VLANs做隔离

https://docs.cloudstack.apache.org/en/latest/adminguide/networking_and_traffic.html#isolation-in-advanced-zone-using-private-vlans

### 关于PVLANs(Secondary VLANs)

PVLAN 的主要用例是共享备份网络，您希望所有用户的主机都能够与备份主机通信，但不能相互通信。

![[Pasted image 20240307172646.png]]

For further reading:

- [Understanding Private VLANs](http://www.cisco.com/en/US/docs/switches/lan/catalyst3750/software/release/12.2_25_see/configuration/guide/swpvlan.html#wp1038379)  
- [Cisco Systems’ Private VLANs: Scalable Security in a Multi-Client Environment](http://tools.ietf.org/html/rfc5517)
- [Private VLAN (PVLAN) on vNetwork Distributed Switch - Concept Overview (1010691)](http://kb.vmware.com/)
### 支持的Secondary VLAN 类型

在三种类型的私有VLAN（混杂的、社区的和隔离的）中，CloudStack支持每个primary VLAN一个混杂的PVLAN、一个隔离的PVLAN和多个社区PVLAN。PVLAN 目前在共享网络和第 2 层网络上受支持。KVM（使用 OVS 时）、XenServer（使用 OVS 时）和 VMware Hypervisor支持 PVLAN 概念

> XenServer 和 KVM 上的 OVS 本身不支持 PVLAN。因此，CloudStack通过修改flow table，成功地模拟了XENServer和KVM的OVS上的PVLAN。
#### 前置要求

- 使用一个支持PVLAN的交换机 http://www.cisco.com/en/US/products/hw/switches/ps708/products_tech_note09186a0080094830.shtml
- 所有可识别 PVLAN 的第 2 层交换机都相互连接，其中一个连接到路由器。连接到主机的所有端口都将配置为中继模式。开放管理 VLAN、主 VLAN（公共）和辅助隔离 VLAN 端口。在 PVLAN 混杂中继模式下配置连接到路由器的交换机端口，这会将隔离的 VLAN 转换为无法识别 PVLAN 的路由器的主 VLAN。
    > 请注意，只有 Cisco Catalyst 4500 具有 PVLAN 混合中继模式，可将普通 VLAN 和 PVLAN 连接到无法识别 PVLAN 的交换机。对于另一个 Catalyst PVLAN 支持交换机，请使用电缆将交换机连接到上部交换机，PVLAN 对各一根。
    
- 在物理交换机上配置带外(out-of-band)专用 VLAN。
- 在 XenServer 和 KVM 上使用 PVLAN 之前，请启用 Open vSwitch （OVS）。

## 安全组

https://docs.cloudstack.apache.org/en/latest/adminguide/networking_and_traffic.html#security-groups

### 关于安全组

安全组提供了一种隔离实例流量的方法。安全组是一组实例，它们根据一组规则（称为入口和出口规则）过滤其传入和传出流量。这些规则根据尝试与实例通信的 IP 地址过滤网络流量。安全组在使用基本网络的Zone区域中特别有用，因为所有访客实例都有单的一个访客网络。在高级Zone区域中，仅在 KVM Hypervisor上支持安全组。

> 在使用高级网络的Zone区域中，您可以改为定义多个访客网络来隔离到实例的流量。

每个CloudStack账户都有一个默认的安全组，它拒绝所有的入站流量，并允许所有的出站流量。可以修改默认安全组，以便所有新实例都继承其他一些所需的规则集。

任何CloudStack用户都可以设置任意数量的额外安全组。启动新实例时，除非指定了另一个用户定义的安全组，否则该实例将分配给默认安全组。一个实例可以是任意数量的安全组的成员。将实例分配给安全组后，该实例将在整个生命周期内保留在该组中;您无法将正在运行的实例从一个安全组移动到另一个安全组。

您可以通过删除或添加任意数量的入口和出口规则来修改安全组。执行此操作时，新规则将应用于组中的所有实例，无论是正在运行的还是已停止的。

如果未指定入口规则，则不允许任何流量进入，但对通过出口规则允许的任何流量的响应除外。
## 外部防火墙和负载均衡

CloudStack能够将其一些虚拟路由器网络功能转移到一些外部网络服务提供商，例如NetScaler。

https://docs.cloudstack.apache.org/en/latest/adminguide/networking_and_traffic.html#external-firewalls-and-load-balancers

## 全局服务器负载均衡支持

CloudStack支持全球服务器负载平衡（GSLB）功能，以提供业务连续性，并实现CloudStack环境中的无缝资源移动。CloudStack通过扩展其与NetScaler Application Delivery Controller（ADC）集成的功能来实现这一点，ADC也提供了各种GSLB功能，如灾难恢复和负载均衡。DNS重定向技术用于在CloudStack中实现GSLB。

为了支持此功能，引入了Region区域级服务和服务提供商。新服务“GSLB”作为Region区域级服务引入。引入了 GSLB 服务提供商，它将提供 GSLB 服务。目前，NetScaler是CloudStack中支持的GSLB提供商。GSLB 功能适用于主动-主动数据中心环境。

> 新 UI 中当前不支持全局服务器负载平衡。要管理全局服务器负载均衡，请直接调用相应的 API 或使用 cloudstack 的 CLI 工具 cloudmonkey

https://docs.cloudstack.apache.org/en/latest/adminguide/networking_and_traffic.html#global-server-load-balancing-support

### 关于全局服务器负载均衡

全局服务器负载平衡 （GSLB） 是负载平衡功能的扩展，在避免停机方面非常有效。根据部署的性质，GSLB 代表了一组用于各种目的的技术，例如负载共享、灾难恢复、性能和法律义务。借助 GSLB，工作负载可以分布在位于Region地理位置不同的多个数据中心。GSLB 还可以提供在发生故障时访问资源的备用位置，或提供一种轻松转移流量以简化维护的方法，或两者兼而有之。

#### GSLB的组件

典型的 GSLB 环境由以下组件组成：

- GSLB站点：在CloudStack的术语中，GSLB站点由映射到数据中心的Zone区域表示，每个Zone区域都有不同的网络设备。每个 GSLB 站点都由该站点本地的 NetScaler 设备管理。这些设备中的每一个都将自己的站点视为本地站点，而由其他设备管理的所有其他站点都视为远程站点。它是 GSLB 部署中的中心实体，由名称和 IP 地址表示。

- GSLB 服务：GSLB 服务通常由负载平衡或内容交换虚拟服务器表示。在 GSLB 环境中，您可以拥有本地和远程 GSLB 服务。本地 GSLB 服务表示本地负载平衡或内容交换虚拟服务器。远程 GSLB 服务是在 GSLB 设置中的其他站点之一上配置的服务。在 GSLB 设置中的每个站点上，您可以创建一个本地 GSLB 服务和任意数量的远程 GSLB 服务。

- GSLB虚拟服务器：GSLB虚拟服务器是指一个或多个GSLB服务，通过使用CloudStack功能来平衡多个Zone区域中实例之间的流量。它评估配置的 GSLB 方法或算法，以选择要向其发送客户端请求的 GSLB 服务。来自不同Zone区域的一个或多个虚拟服务器绑定到 GSLB 虚拟服务器。GSLB 虚拟服务器没有与之关联的公共 IP，而是具有 FQDN DNS 名称。

- 负载平衡或内容交换虚拟服务器：根据 Citrix NetScaler 术语，负载平衡或内容交换虚拟服务器表示本地网络上的一个或多个服务器。客户端将其请求发送到负载平衡或内容交换虚拟服务器的虚拟 IP （VIP） 地址，虚拟服务器在本地服务器之间平衡负载。在 GSLB 虚拟服务器选择表示本地或远程负载平衡或内容交换虚拟服务器的 GSLB 服务后，客户端会将请求发送到该虚拟服务器的 VIP 地址。

- DNS VIP：DNS 虚拟 IP 表示 GSLB 服务提供商上的负载平衡 DNS 虚拟服务器。GSLB 服务提供商具有权威性的域的 DNS 请求可以发送到 DNS VIP。

- 权威DNS：ADNS（权威域名服务器）是一种为DNS查询提供实际答案的服务，例如网站IP地址。在 GSLB 环境中，ADNS 服务仅响应 GSLB 服务提供商具有权威性的域的 DNS 请求。配置 ADNS 服务后，服务提供商拥有该 IP 地址并通告该 IP 地址。创建 ADNS 服务时，NetScaler 会响应配置的 ADNS 服务 IP 和端口上的 DNS 查询。

#### 在CloudStack中GSLB是如何工作的

全局服务器负载平衡用于管理托管在两个独立Zone区域上的网站的流量，理想情况下，这两个Zone区域位于不同的地理位置。以下是在CloudStack中如何提供GLSB功能的图示： 一个组织xyztelco已经建立了一个公共云，它跨越了两个区域，Zone-1和Zone-2，跨越了由CloudStack管理的地理上分开的数据中心。云的租户 A 使用 xyztelco 云启动高可用性解决方案。为此，它们在两个区域中分别启动两个实例：区域 1 中的 VM1 和 VM2，以及区域 2 中的 VM5 和 VM6。租户 A 在区域 1 中获取公共 IP-1，并配置负载均衡器规则以对 VM1 和 VM2 实例之间的流量进行负载均衡。CloudStack编排在Zone-1的LB服务提供商上设置一个虚拟服务器。在区域 1 中的 LB 服务提供商上设置的虚拟服务器 1 表示客户端在 IP-1 下访问的可公开访问的虚拟服务器。以 IP-1 传输到虚拟服务器 1 的客户端流量将在 VM1 和 VM2 实例之间进行负载均衡。

租户 A 在区域 2 中获取另一个公共 IP，即 IP-2，并设置负载均衡器规则以对 VM5 和 VM6 实例之间的流量进行负载均衡。类似地，在Zone-2中，CloudStack编排在LB服务提供商上设置一个虚拟服务器。在 Zone-2 的 LB 服务提供商上设置的虚拟服务器 2 表示客户端以 IP-2 访问的可公开访问的虚拟服务器。以 IP-2 到达虚拟服务器 2 的客户端流量在 VM5 和 VM6 实例之间进行负载均衡。此时，租户 A 在两个区域中都启用了该服务，但如果其中一个区域发生故障，则无法设置灾难恢复计划。此外，租户 A 无法根据负载、邻近度等智能地将流量负载均衡到其中一个区域。xyztelco 的云管理员为这两个区域预置了 GSLB 服务提供商。GSLB 提供程序通常是一个 ADC，它能够充当 ADNS（权威域名服务器），并具有监视本地和远程站点虚拟服务器运行状况的机制。云管理员为使用区域 1 和 2 的租户启用 GSLB 即服务。

![[Pasted image 20240307174215.png]]

租户 A 希望利用 xyztelco 云提供的 GSLB 服务。租户 A 配置 GSLB 规则，以在区域 1 的虚拟服务器 1 和区域 2 的虚拟服务器 2 之间对流量进行负载均衡。域名按 A.xyztelco.com 提供。CloudStack编排在Zone-1的GSLB服务提供商上设置GSLB虚拟服务器1。CloudStack将Zone-1的虚拟服务器1和Zone-2的虚拟服务器2绑定到GLSB虚拟服务器1。GSLB 虚拟服务器 1 配置为开始监视区域 1 中虚拟服务器 1 和 2 的运行状况。CloudStack还将在Zone-2的GSLB服务提供商上协调设置GSLB虚拟服务器2。CloudStack会将Zone-1的虚拟服务器1和Zone-2的虚拟服务器2绑定到GLSB虚拟服务器2。GSLB 虚拟服务器 2 配置为开始监视虚拟服务器 1 和 2 的运行状况。CloudStack会将域 A.xyztelco.com 绑定到GSLB虚拟服务器1和2。此时，租户 A 服务将在 A.xyztelco.com 时在全球范围内访问。域 xyztelcom.com 的专用 DNS 服务器由带外管理员配置，以解析两个区域的 GSLB 提供程序的域 A.xyztelco.com，这些区域配置为域 A.xyztelco.com 的 ADNS。客户端在发送 DNS 请求以解析 A.xyztelcom.com 时，最终将 DNS 委派到区域 1 和 2 的 GSLB 提供商地址。GSLB 提供商将收到客户端 DNS 请求。GSLB 提供程序将选取与该域关联的 GSLB 虚拟服务器，具体取决于它需要解析的域。根据正在进行负载平衡的虚拟服务器的运行状况，域的 DNS 请求将解析为与所选虚拟服务器关联的公共 IP。
## 访客IP范围

访客网络流量的 IP 范围由用户基于每个帐户进行设置。这允许用户配置其网络，从而在其访客网络和客户端之间启用 VPN 链接。

在基本Zone区域和启用了安全组的高级网络中的共享网络中，您可以灵活地从不同的子网添加多个访客 IP 范围。您可以一次添加或删除一个 IP 范围。有关详细信息，请参阅  [[#关于多个IP范围]]

## 申请一个新的IP地址

https://docs.cloudstack.apache.org/en/latest/adminguide/networking_and_traffic.html#acquiring-a-new-ip-address

## 释放一个IP地址

https://docs.cloudstack.apache.org/en/latest/adminguide/networking_and_traffic.html#releasing-an-ip-address

## 保留一个公共的IP地址

https://docs.cloudstack.apache.org/en/latest/adminguide/networking_and_traffic.html#reserving-a-public-ip-address

## 释放一个保留的公共IP地址

https://docs.cloudstack.apache.org/en/latest/adminguide/networking_and_traffic.html#releasing-a-reserved-public-ip-address

## Static NAT

静态 NAT 规则将公有 IP 地址映射到实例的私有 IP 地址，以允许 Internet 流量进入实例。公共 IP 地址始终保持不变，这就是它被称为静态 NAT 的原因。本节介绍如何为特定 IP 地址启用或禁用静态 NAT。

### 启动和关闭Static NAT

https://docs.cloudstack.apache.org/en/latest/adminguide/networking_and_traffic.html#enabling-or-disabling-static-nat

## IP转发和防火墙

默认情况下，所有传入到公共 IP 地址的流量都将被拒绝。默认情况下，来自访客的所有传出流量也会被阻止。

要允许传出流量，请按照高级Zone区域中的出口防火墙规则中的过程操作。

为了允许传入流量，用户可以设置防火墙规则和/或端口转发规则。例如，可以使用防火墙规则在公共 IP 地址上打开一系列端口，例如 33 到 44。然后，使用端口转发规则将流量从该范围内的各个端口定向到用户实例上的特定端口。例如，一个端口转发规则可以将公共 IP 端口 33 上的传入流量路由到一个用户实例的私有 IP 上的端口 100。

https://docs.cloudstack.apache.org/en/latest/adminguide/networking_and_traffic.html#ip-forwarding-and-firewalling

## IP负载均衡

用户可以选择为多个访客关联相同的公共 IP。CloudStack通过以下策略实现了TCP级别的负载均衡器。

- Round-Robin轮询
- 最少连接
- source IP

这类似于端口转发，但目标可能是多个 IP 地址。

## DNS和DHCP

虚拟路由器为访客提供 DNS 和 DHCP 服务。它将 DNS 请求代理到Zone可用区上配置的 DNS 服务器。

## 远程访问VPN

CloudStack账户所有者可以创建虚拟专用网络（VPN）来访问他们的实例。如果访客网络是从提供远程访问 VPN 服务的网络产品(Network Offering)实例化的，则虚拟路由器（基于系统虚拟机）将用于提供服务。CloudStack为访客虚拟网络提供了基于L2TP-over-IPsec的远程访问VPN服务。由于每个网络都有自己的虚拟路由器，因此 VPN 不会在网络之间共享。Windows、Mac OS X 和 iOS 原生的 VPN 客户端可用于连接到访客网络。帐户所有者可以为其 VPN 创建和管理用户。CloudStack不使用其account数据库来达到这个目的，而是使用一个单独的表。VPN user数据库在CloudStack帐户所有者创建的所有 VPN 之间共享。所有 VPN 用户都可以访问CloudStack帐户所有者创建的所有 VPN。

> 确保并非所有流量都通过 VPN。也就是说，VPN 安装的路由应仅针对访客网络，而不是针对所有流量。

- Road Warrior / 远程访问。用户希望能够安全地从家庭或办公室连接到云中的专用网络。通常，连接客户端的 IP 地址是动态的，不能在 VPN 服务器上进行预配置。
- 站点到站点。在此方案中，两个专用子网通过安全 VPN 隧道通过公共 Internet 连接。云用户的子网（例如，办公网络）通过网关连接到云中的网络。用户网关的地址必须在云端的VPN服务器上预先配置。请注意，尽管 L2TP-over-IPsec 可用于设置站点到站点 VPN，但这不是此功能的主要目的。有关详细信息，请参阅 https://docs.cloudstack.apache.org/en/latest/adminguide/networking_and_traffic.html#setting-s2s-vpn-conn

https://docs.cloudstack.apache.org/en/latest/adminguide/networking_and_traffic.html#remote-access-vpn

## 关于VLAN之间的路由(nTier Apps)

VLAN 间路由（nTier 应用）是在 VLAN 之间路由网络流量的功能。借助此功能，您可以构建 Virtual Private Cloud （VPC），这是云的一个隔离部分，可以容纳多层应用程序。这些层部署在可以相互通信的不同 VLAN 上。您可以将 VLAN 预置到您创建的层（tier），并且实例可以部署在不同的层上。VLAN 连接到虚拟路由器，便于实例之间的通信。实际上，您可以通过 VLAN 将实例分段到可以托管多层应用程序（如 Web、应用程序或数据库）的不同网络中。这种通过VLAN进行的分段在逻辑上将应用程序实例分开，以实现更高的安全性和更低的广播，同时保持与同一设备的物理连接。

XenServer、KVM 和 VMware Hypervisor支持此功能。

主要优点是：

- 管理员可以部署一组 VLAN，并允许用户在这些 VLAN 上部署实例。访客 VLAN 从一组预先指定的访客 VLAN 中随机分配给一个帐户。帐户的某一层的所有实例都驻留在分配给该帐户的访客 VLAN 上。
    > 为一个账户分配的VLAN不能在多个账户之间共享。
- 管理员可以允许用户创建自己的 VPC 并部署应用程序。在此方案中，属于该帐户的实例部署在分配给该帐户的 VLAN 上。
- 管理员和用户都可以创建多个VPC。当第一个实例部署在层中时，访客网络网卡将插入 VPC 虚拟路由器。
- 管理员可以创建以下网关来向实例发送流量或从实例接收流量：
    - **VPN Gateway**: For more information, see [“Creating a VPN gateway for the VPC”](https://docs.cloudstack.apache.org/en/latest/adminguide/networking_and_traffic.html#creating-a-vpn-gateway-for-the-vpc).
    - **Public Gateway**: 在为VPC创建虚拟路由器时，VPC的公网网关会添加到虚拟路由器中。公共网关不会向最终用户公开。您不得列出它，也不允许创建任何静态路由。
    - **Private Gateway**: For more information, see “[Adding a Private Gateway to a VPC](https://docs.cloudstack.apache.org/en/latest/adminguide/networking_and_traffic.html#adding-priv-gw-vpc)”.
- 管理员和用户都可以创建各种可能的目标-网关组合。但是，在部署中只能使用每种类型的一个网关。
    For example:
    
    - **VLANs and Public Gateway**: 例如，应用程序部署在云中，Web 应用程序实例与 Internet 通信。
    - **VLANs, VPN Gateway, and Public Gateway**: 例如，应用程序部署在云中;Web 应用程序实例与 Internet 通信;数据库实例与本地设备通信。
- 管理员可以在虚拟路由器上定义网络访问控制列表（ACL），以过滤VLAN之间或Internet与VLAN之间的流量。您可以根据 CIDR、端口范围、协议、类型代码（如果选择了 ICMP 协议）和入口/出口类型来定义 ACL。

下图显示了 VLAN 间设置的可能部署方案：

![[Pasted image 20240307180224.png]]

为了建立一个多层的VLAN间部署，请看  [[#配置一个虚拟私有云(VPC)]]

## 配置一个虚拟私有云(VPC)

### 关于VPC

CloudStack Virtual Private Cloud是CloudStack的一个私有的、独立的部分。VPC 可以拥有自己的虚拟网络拓扑，类似于传统的物理网络。可以在虚拟网络中启动实例，这些实例的专用地址可以在所选范围内，例如：10.0.0.0/16。您可以在 VPC 网络范围内定义网络层，这样您就可以根据 IP 地址范围对类似类型的实例进行分组。

例如，如果 VPC 的专用范围为 10.0.0.0/16，则其访客网络可以具有网络范围 10.0.1.0/24、10.0.2.0/24、10.0.3.0/24 等。

### 一个VPC主要组件

VPC由以下网络组件组成：

- VPC：VPC充当多个隔离网络的容器，这些网络可以通过其虚拟路由器相互通信。
- 网络层：每个网络层都充当具有自己的 VLAN 和 CIDR 列表的隔离网络，您可以在其中放置资源组，例如实例。网络层通过 VLAN 进行分段。每个网络层的 NIC 充当其网关。
- 虚拟路由器：创建VPC时会自动创建并启动虚拟路由器。虚拟路由器连接网络层，并在公有网关、VPN 网关和 NAT 实例之间定向流量。对于每个网络层，虚拟路由器中都存在相应的 NIC 和 IP。虚拟路由器通过其 IP 提供 DNS 和 DHCP 服务。
- 公网网关：通过公网网关路由到VPC的流量。在VPC中，公有网关不会暴露给终端用户;因此，公共网关不支持静态路由。
- 私有网关：通过私有网关路由到VPC的所有进出私有网络的流量。有关更多信息，请参阅  [[#添加一个私有网关到VPC]]
- VPN网关：VPN连接的VPC端。
- Site-to-Site VPN 连接：VPC 与数据中心、家庭网络或主机托管设施之间基于硬件的 VPN 连接。
- 客户网关：VPN 连接的客户端。有关详细信息，请参阅“创建和更新 VPN 客户网关”。
- NAT实例：为实例提供端口地址转换，通过公网网关访问公网。有关更多信息，请参阅 [[#在一个VPC上打开或者关闭Static NAT]]
- 网络ACL：网络ACL是一组网络ACL项。网络 ACL 项只不过是按顺序评估的编号规则，从编号最低的规则开始。这些规则确定是否允许流量进出与网络 ACL 关联的任何网络层。有关详细信息，请参阅 [[#配置网络访问控制列表(ACL)]]

### 一个VPC的网络架构

在VPC中，存在以下四个基本网络架构选项：

- 仅具有公有网关的 VPC
- 具有公有网关和私有网关的 VPC
- 具有公有和私有网关以及站点到站点 VPN 访问的 VPC
- 仅具有私有网关和站点到站点 VPN 访问权限的 VPC

### 一个VPC的Connectivity选项

您可以将 VPC 连接到：

- 通过公共网关的 Internet。
- 通过 VPN 网关使用站点到站点 VPN 连接的企业数据中心。
- 使用公共网关和 VPN 网关的 Internet 和公司数据中心。

### VPC网络的考量

在创建一个VPC之前，你需要考虑以下问题:

- 默认情况下，VPC创建时处于启用状态。
- VPC只能在高级Zone可用区中创建，一次不能属于多个Zone可用区。
- 一个账户可以创建的默认 VPC 数量为 20 个。但是，您可以使用 max.account.vpcs 全局参数对其进行更改，该参数控制允许账户创建的最大 VPC 数。
- 账户可以在 VPC 中创建的默认网络层(tier)数为 3。可以使用 vpc.max.networks 参数配置此数字。
- 每个网络层在 VPC 中都应具有唯一的 CIDR。确保网络层的 CIDR 应在 VPC CIDR 范围内。
- 一个网络层只属于一个VPC。
- VPC 内的所有网络层(tier)都应属于同一账户。
- 创建VPC时，默认为VPC分配一个Source NAT IP。仅当VPC被移除时，才会释放Source NAT IP。
- 公共 IP 一次只能用于一个目的。如果 IP 是 source NAT，则不能用于 Static NAT 或端口转发。
- 实例只能具有您预置的私有 IP 地址。要与 Internet 通信，请为您在 VPC 中启动的实例启用 NAT。
- VPC只能添加新的网络。每个 VPC 的最大网络数受您在 vpc.max.networks 参数中指定的值的限制。默认值为 3。
- 负载均衡服务只能由 VPC 内的一个网络层(tier)支持。
- 如果一个IP地址被分配到了一个网络层(tier):
    - 该 IP 不能同时由多个网络层在 VPC 中使用。例如，如果您有网络层 A 和 B 以及公共 IP1，则可以使用 A 或 B 的 IP（但不能同时使用两者）来创建端口转发规则。
    - 该 IP 不能用于 VPC 内其他访客网络的静态 NAT、负载均衡或端口转发规则。
- 远程访问VPN，不支持VPC网络

### 添加一个VPC

创建 VPC 时，您只需为 VPC 网络地址空间提供区域和一组 IP 地址即可。您可以以无类别域间路由 （CIDR） 块的形式指定这组地址。

1. 以管理员或者终端用户登录CloudStack UI
2. 在左侧导航栏，选择Network
3. 选择VPC
4. 点击添加VPC，页面如下

    ![[Pasted image 20240310105202.png]]
1. 提供以下信息
    - **Name**: VPC的简要名称
    - **Description**: VPC的简要描述
    - **Zone**: 选择VPC所属的Zone区域
    - **CIDR**: 定义 VPC 中所有网络层tier（访客网络）的 CIDR 范围。创建网络层(tier)时，请确保其 CIDR 在您输入的父 CIDR 值范围内。CIDR 必须符合RFC1918。
    - **Network Domain**: 如果要分配特殊域名，请指定 DNS 后缀。此参数将应用于 VPC 内的所有网络层tier。这意味着，您在 VPC 中创建的所有网络层tier都属于同一个 DNS 域。如果未指定该参数，则自动生成DNS域名。
    - **VPC Offering**: 如果管理员配置了多个 VPC 产品(offering)，请选择要用于此 VPC 的offering。
    - **DNS**: 此 VPC 将使用的一组自定义 DNS。如果未提供，则将使用为Zone区域指定的 DNS。仅当所选VPC产品支持DNS服务时才可用。
    - **IPv6 DNS**: 此 VPC 将使用的一组自定义 IPv6 DNS。如果未提供，则将使用为Zone区域指定的 IPv6 DNS。仅当所选 VPC 产品启用 IPv6 并支持 DNS 服务时才可用。
    - **IPv4 address for the VR in this VPC**: 访客网络要使用的source NAT 地址或primary公用网络地址。如果未提供，则将使用可用地址池中的随机地址。
    - **Public MTU**: 在VPC网络VR的公网接口上配置的MTU
2. 点击OK

> - 在启用安全组的高级Zone可用区和基本Zone可用区中，不支持创建VPC和隔离网络。
> - 公共 MTU 选项将显示在 UI 中，仅当区域配置 allow.end.users.to.specify.vr.mtu 设置为 true 时才会考虑。公共 MTU 的最大允许值可由Zone区域级配置 - vr.public.interface.max.mtu 控制。


### 添加网络层(Network tiers)

网络层tier是 VPC 中的不同位置，充当隔离网络(isolated network)，默认情况下无法访问其他网络层tier。网络层tier设置在不同的 VLAN 上，这些 VLAN 可以使用虚拟路由器相互通信。网络层tier提供与 VPC 内其他网络层tier的低成本、低延迟网络连接。

1. 管理员账户或者终端用户登录CloudStack UI
2. 左侧导航栏，选择Network
3. 选择VPC
    在这个账号下创建的所有VPC都会列出来

> 最终用户可以查看自己的 VPC，而ROOT用户和Domain管理员可以查看他们有权查看的任何 VPC。

1. 单击要为其设置网络层tier的 VPC 的 Configure （配置） 按钮。
2. 点击创建网络
    
    点击添加新的网络层tier，如下
   ![[Pasted image 20240310105352.png]]
1. 如果您已经创建了网络层，则会显示 VPC 图表。单击“创建网络层tier”以添加新的网络层tier。
2. 指定以下内容:
    所有字段都是必填项。
    
    - **Name**: 网络层的名称，必须唯一
    - **Network Offering**: 列出了以下默认网络产品/服务：Internal LB、DefaultIsolatedNetworkOfferingForVpcNetworksNoLB、DefaultIsolatedNetworkOfferingForVpcNetworks
        在 VPC 中，使用启用 LB 的网络产品(Offering)只能创建一个网络层tier。
    - **Gateway**: 您创建的网络层tier的网关。确保网关位于您在创建 VPC 时指定的父级 CIDR 范围内，并且不与 VPC 内任何现有网络层tier的 CIDR 重叠。
    - **VLAN**: root管理员创建的网络层tier的 VLAN ID。
        仅当您选择的网络产品Offering启用了 VLAN 时，此选项才可见。
        
        For more information, see [“Assigning VLANs to Isolated Networks”](https://docs.cloudstack.apache.org/en/latest/adminguide/hosts.html#assigning-vlans-to-isolated-networks).
    
    - **Netmask**: 你创建的网络层tier的子网掩码
        例如，VPC网段为10.0.0.0/16，网络层tier网段为10.0.1.0/24，则网络层tier网关为10.0.1.1，网络层tier网段为255.255.255.0。
        
3. 点击OK
4. 继续为网络层Tier配置访问控制列表。
### 配置网络访问控制列表(ACL)

仅当创建的VPC支持“NetworkACL”服务时，才能创建网络访问控制列表。

定义网络访问控制列表 （ACL） 以控制关联的网络层tier与外部网络（VPC 的其他网络层tier以及公共网络）之间的传入（入口，ingress）和传出（出口 egress）流量。

#### 关于网络ACL

在CloudStack术语中，网络ACL是一组网络ACL规则。网络 ACL 规则按其顺序进行处理，从编号最低的规则开始。每个规则至少定义一个受影响的协议、流量类型、操作和受影响的detination/source网络。下表显示了“default_deny”ACL 的示例性内容。

|Rule|Protocol|Traffic type|Action|CIDR|
|---|---|---|---|---|
|1|All|Ingress|Deny|0.0.0.0/0|
|2|All|Egress|Deny|0.0.0.0/0|

每个网络ACL都与一个VPC关联，可以分配给多个VPC网络层tier。每个网络层tier都需要与网络 ACL 相关联。一次只能将一个 ACL 与网络层tier关联。如果在创建网络层tier时没有可用的自定义网络 ACL，则必须改用默认网络 ACL。目前有两个默认 ACL 可用。“default_allow”ACL 允许流入和流出，而“default_deny”阻止所有流入和流出流量。默认网络 ACL 无法删除或修改。新创建的 ACL 虽然显示为空，但会拒绝所有传入流量到关联的网络层tier，并允许所有传出流量。要更改默认值，请向 ACL 添加“拒绝所有出口目标”和/或“允许所有入口源”规则。之后，流量可以被列入白名单或黑名单。

- Cloudstack中的ACL规则是有状态的
- source/destination CIDR 始终是外部网络
- 在VPC的虚拟路由器上也可以看到ACL规则。入口规则列在表 iptables 表“filter”中，而出口规则列在表 “mangle” 表中
- 入口和出口的 ACL 规则不相关。例如，出口“全部拒绝”不会影响流量以响应允许的入口连接
#### 创建网络ACL

1. 用终端用户账号或者管理员账号登录CloudStack UI
2. 在左侧导航栏中，选择网络。
3. 选择VPC
    
    所有此账号创建的VPC都会列出
    
4. 点击VPC的配置按钮
    
    对于每个网络层tier，以下选项都会列出
    - Internal LB
    - Public LB IP
    - Static NAT
    - Instances
    - CIDR
        
    以下路由信息也会列出:
    - Private Gateways
    - Public IP Addresses
    - Site-to-Site VPNs
    - Network ACL Lists
        
5. 选择网络ACL列表.
    
    “网络 ACL”页面中显示以下默认规则：default_allow、default_deny。
    
6. 单击“添加 ACL 列表”，然后指定以下内容：
    - **ACL List Name**: ACL列表的名称
    - **Description**: 可向用户显示的 ACL 列表的简短说明。

#### 创建一个ACL规则

1. 终端用户或者管理员登录CloudStack UI
2. 左侧导航栏选择Network
3. 选择VPC
    
    所有这个账号创建的VPC都已经列出
    
4. 点击VPC配置按钮
5. 选择Network ACL列表
    
    除了您创建的自定义 ACL 列表外，“网络 ACL”页面中还会显示以下默认规则：default_allow、default_deny。
    
6. 选择你期望的ACL列表
7. 选择ACL规则Tab页面
    
    要添加 ACL 规则，请填写以下字段以指定 VPC 中允许的网络流量类型。
    
    - **Rule Number**:  计算规则的顺序。
    - **CIDR**: CIDR 充当入口规则的源 CIDR 和出口规则的目标 CIDR。要仅接受来自或流向特定地址块内 IP 地址的流量，请输入 CIDR 或逗号分隔的 CIDR 列表。CIDR 是传入流量的基本 IP 地址。例如，192.168.0.0/22。要允许所有 CIDR，请设置为 0.0.0.0/0。
    - **Action**: 要采取什么行动。允许流量或阻止。
    - **Protocol**: 源用于将流量发送到层tier的网络协议。TCP 和 UDP 协议通常用于数据交换和最终用户通信。ICMP 协议通常用于发送错误消息或网络监控数据。全部支持所有流量。另一个选项是协议编号。
    - **Start Port**, **End Port** （仅限 TCP、UDP）：作为传入流量目标的侦听端口范围。如果要打开单个端口，请在两个字段中使用相同的数字。
    - **Protocol Number**: 与 IPv4 或 IPv6 关联的协议编号。有关详细信息，请参阅[协议编号](http://www.iana.org/assignments/protocol-numbers/protocol-numbers.xml)。
    - **ICMP Type**, **ICMP Code** （仅限 ICMP）：将发送的消息类型和错误代码。
    - **Traffic Type**: 流量类型：传入或传出。
        
8. 点击添加
    
    您可以编辑分配给 ACL 规则的标签并删除已创建的 ACL 规则。单击“详细信息”选项卡中的相应按钮。

#### 用一个自定义的ACL规则创建一个Tier层

1. 创建一个VPC
2. 创建一个自定义的VPC列表
3. 给这个列表添加一个ACL规则
4. 在VPC中创建一个Tier层
    
    当创建一个tier层时，选择期望的ACL列表
    
5. 点击OK

#### 把一个自定义的ACL列表分配给一个Tier层

1. 创建一个VPC
2. 在VPC中创建一个tier层
3. 将默认的ACL列表关联至该tier层
4. 创建一个自定义的ACL列表
5. 添加一些ACL规则到这个ACL列表中
6. 选择一个tier层，这个层是你想分配自定义ACL列表的层
7. 点击替换ACL图标. ![button to replace an ACL list](https://docs.cloudstack.apache.org/en/latest/_images/replace-acl-icon.png)
    
    替换的ACL列表已经显示
    
8. 选择期望的ACL列表
9. 点击OK

### 添加一个私有网关到VPC

私有网关可由root管理员和用户添加。VPC内网与物理网的网卡是1：1的关系。您可以将多个私有网关配置到单个 VPC。同一数据中心中不允许具有重复 VLAN 和 IP 的网关。

1. 管理员或者终端用户登录CloudStack
2. 在左边的导航栏上，选择Network
3. 选择VPC
    
    所有这个账号创建的VPC都已经列出
    
4. 单击要配置负载均衡规则的VPC的配置按钮。
    
    这个VPC下所有的网络tier层被列出
    
5. 点击设置图标
    
    以下选项被显示
    
    - Internal LB
    - Public LB IP
    - Static NAT
    - Instances
    - CIDR
    
    以下路由信息被显示
    
    - Private Gateways
    - Public IP Addresses
    - Site-to-Site VPNs
    - Network ACL Lists
        
6. 选择 Private Gateways.
    
    网关页面被显示
    
7. 点击添加一个新的网关
   ![[Pasted image 20240310114718.png]]
   1. 指定以下内容
    
    - **Physical Network**: （仅限管理员）您在Zone区域中创建的物理网络。
    - **VLAN**: （仅限管理员）与VPC网关关联的VLAN。
    - **IP Address**: VPC网关关联的IP地址
    - **Gateway**: 流量通过其路由到 VPC 或从 VPC 路由的网关。
    - **Netmask**: VPC网关的子网掩码
    - **Source NAT**: 选择此选项可在 VPC 私有网关上启用源 NAT 服务。
     参阅 [[#在私有网关上的Source NAT]]
        
    - **Bypass VLAN id/range overlap**: （仅限管理员）绕过 VLAN 重叠检查。这样，可以创建具有相同 VLAN 的多个网络
    - **Associated Network**: 此专用网关关联的 L2 或隔离网络。此专用网络将使用与关联网络相同的 VLAN。
    - **ACL**: 控制 VPC 私有网关上的入口和出口流量。默认情况下，所有流量都会被阻止。
        
        参阅 [[#在私有网关上的ACL]]
        
    
    新网关将显示在列表中。您可以重复这些步骤，为此 VPC 添加更多网关。

### 在私有网关上的Source NAT

您可能希望部署具有相同父级 CIDR 和访客层 CIDR 的多个 VPC。因此，来自不同 VPC 的多个 Guest 实例可以具有相同的 IP，以通过私有网关到达企业数据中心。在这种情况下，需要在私有网关上配置NAT服务以避免IP冲突。如果开启了source NAT，则VPC中的访客实例将通过NAT服务通过私有网关IP地址到达企业网络。

在添加私有网关时，可以启用私有网关上的source NAT 服务。删除私有网关时，将删除特定于该私有网关的source NAT 规则。

要在现有私有网关上启用source NAT，请将其删除并使用source NAT 重新创建。

### 在私有网关上的ACL

VPC 私有网关上的流量通过创建入口和出口网络 ACL 规则来控制。ACL 包含允许和拒绝规则。根据规则，将阻止所有进入私有网关接口的流量和从私有网关接口流出的所有出口流量。

您可以在创建私有网关时更改此默认行为。或者，您可以执行以下操作：

1. 在 VPC 中，确定要使用的私有网关。
2. 在 Private Gateway （私有网关） 页面中，执行以下任一操作：
    
    - Use the Quickview. See 3.
        
    - Use the Details tab. See 4 through .
        
3. 在所选私有网关的概览中，单击替换 ACL，选择 ACL 规则，然后单击确定
4. 单击要使用的私有网关的 IP 地址。
5. 在“详细信息”选项卡中，单击“替换 ACL”按钮 ![button to replace an ACL list](https://docs.cloudstack.apache.org/en/latest/_images/replace-acl-icon.png)
    
    The Replace ACL dialog is displayed.
    
6. 选择 ACL 规则，然后单击确定。
    
    等待几秒钟。您可以看到新的ACL规则显示在详情页面中。

### 创建一个静态路由

CloudStack允许你为你创建的VPN连接指定路由。您可以输入一个 CIDR 地址或 CIDR 地址，以指示要路由回网关的流量。


1. In a VPC, identify the Private Gateway you want to work with.
    
2. In the Private Gateway page, click the IP address of the Private Gateway you want to work with.
    
3. Select the Static Routes tab.
    
4. Specify the CIDR of destination network.
    
5. Click Add.
    
    Wait for few seconds until the new route is created.

#### Denylisting路由

CloudStack使你能够阻止路由列表，这样它们就不会被分配给任何VPC私有网关。在“denied.routes”全局参数中指定要列入拒绝列表的路由列表。请注意，参数更新仅影响新的静态路由创建。如果阻止现有静态路由，则该路由将保持不变并继续运行。如果静态路由被拒绝，则无法添加该Zone区域的路由。

### 部署实例到特定的Tier层

2. Log in to the CloudStack UI as an administrator or end User.
    
3. In the left navigation, choose Network.
    
4. In the Select view, select VPC.
    
    All the VPCs that you have created for the Account is listed in the page.
    
5. Click the Configure button of the VPC to which you want to deploy the Instances.
    
    The VPC page is displayed where all the tiers you have created are listed.
    
6. Click Instances tab of the tier to which you want to add an Instance.

![[Pasted image 20240310114845.png]]

The Add Instance page is displayed.

Follow the on-screen instruction to add an Instance. For information on adding an Instance, see the Installation Guide.

### 部署实例到一个VPC Tier和共享网络

CloudStack允许你在VPC层和一个或多个共享网络上部署实例。借助此功能，部署在多层应用程序中的实例可以通过服务提供商提供的共享网络接收监控服务。

1. Log in to the CloudStack UI as an administrator.
    
2. In the left navigation, choose Instances.
    
3. Click Add Instance.
    
4. Select a zone.
    
5. Select a Template or ISO, then follow the steps in the wizard.
    
6. Ensure that the hardware you have allows starting the selected service offering.
    
7. Under Networks, select the desired networks for the Instance you are launching.
    
    You can deploy an Instance to a VPC tier and multiple shared networks.
     ![[Pasted image 20240310114925.png]]
1. Click Next, review the configuration and click Launch.
    
    Your Instance will be deployed to the selected VPC tier and shared network.

### 为一个VPC申请一个新的IP地址

获取 IP 地址时，所有 IP 地址都将分配给 VPC，而不是分配给 VPC 内的访客网络。仅当为 IP 或网络创建第一个端口转发、负载平衡或静态 NAT 规则时，IP 才会关联到访客网络。IP 不能一次关联到多个网络。


1. Log in to the CloudStack UI as an administrator or end User.
    
2. In the left navigation, choose Network.
    
3. In the Select view, select VPC.
    
    All the VPCs that you have created for the Account is listed in the page.
    
4. Click the Configure button of the VPC to which you want to deploy the Instances.
    
    The VPC page is displayed where all the tiers you created are listed in a diagram.
    
    The following options are displayed.
    
    - Internal LB
        
    - Public LB IP
        
    - Static NAT
        
    - Instances
        
    - CIDR
        
    
    The following router information is displayed:
    
    - Private Gateways
        
    - Public IP Addresses
        
    - Site-to-Site VPNs
        
    - Network ACL Lists
        
5. Select IP Addresses.
    
    The Public IP Addresses page is displayed.
    
6. Click Acquire New IP, and click Yes in the confirmation dialog.
    
    系统会提示您进行确认，因为通常 IP 地址是有限的资源。片刻之后，新的 IP 地址将显示为“已分配”状态。您现在可以在端口转发、负载均衡和静态 NAT 规则中使用 IP 地址。

### 释放一个已经分配给一个VPC的IP地址

IP 地址是有限的资源。如果您不再需要特定 IP，则可以取消其与 VPC 的关联，并将其返回到可用地址池。仅当删除此 IP 地址的所有网络（端口转发、负载平衡或 StaticNAT）规则时，才能从其层中释放 IP 地址。释放的 IP 地址仍将属于同一 VPC。

1. Log in to the CloudStack UI as an administrator or end User.
    
2. In the left navigation, choose Network.
    
3. In the Select view, select VPC.
    
    All the VPCs that you have created for the Account is listed in the page.
    
4. Click the Configure button of the VPC whose IP you want to release.
    
    The VPC page is displayed where all the tiers you created are listed in a diagram.
    
    The following options are displayed.
    
    - Internal LB
        
    - Public LB IP
        
    - Static NAT
        
    - Instances
        
    - CIDR
        
    
    The following router information is displayed:
    
    - Private Gateways
        
    - Public IP Addresses
        
    - Site-to-Site VPNs
        
    - Network ACL Lists
        
5. Select Public IP Addresses.
    
    The IP Addresses page is displayed.
    
6. Click the IP you want to release.
    
7. In the Details tab, click the Release IP button ![button to release an IP.](https://docs.cloudstack.apache.org/en/latest/_images/release-ip-icon.png)

### 在一个VPC上打开或者关闭Static NAT

静态 NAT 规则将公有 IP 地址映射到 VPC 中实例的私有 IP 地址，以允许 Internet 流量流向该实例。本节介绍如何为 VPC 中的特定 IP 地址启用或禁用静态 NAT。

如果端口转发规则已对某个 IP 地址生效，则无法启用该 IP 的静态 NAT。

如果访客实例是多个网络的一部分，则静态 NAT 规则只有在默认网络上定义时才会起作用。

1. Log in to the CloudStack UI as an administrator or end User.
    
2. In the left navigation, choose Network.
    
3. In the Select view, select VPC.
    
    All the VPCs that you have created for the Account is listed in the page.
    
4. Click the Configure button of the VPC to which you want to deploy the Instances.
    
    The VPC page is displayed where all the tiers you created are listed in a diagram.
    
    For each tier, the following options are displayed.
    
    - Internal LB
        
    - Public LB IP
        
    - Static NAT
        
    - Instances
        
    - CIDR
        
    
    The following router information is displayed:
    
    - Private Gateways
        
    - Public IP Addresses
        
    - Site-to-Site VPNs
        
    - Network ACL Lists
        
5. In the Router node, select Public IP Addresses.
    
    The IP Addresses page is displayed.
    
6. Click the IP you want to work with.
    
7. In the Details tab,click the Static NAT button. ![button to enable Static NAT.](https://docs.cloudstack.apache.org/en/latest/_images/enable-disable.png) The button toggles between Enable and Disable, depending on whether static NAT is currently enabled for the IP address.
    
8. If you are enabling static NAT, a dialog appears as follows:
    ![[Pasted image 20240310115030.png]]
1. Select the tier and the destination Instance, then click Apply.

### 在一个VPC上添加负载均衡规则

在 VPC 中，您可以配置两种类型的负载均衡：外部 LB 和内部 LB。 外部 LB 只不过是一个 LB 规则，用于重定向在 VPC 虚拟路由器的公有 IP 上接收的流量。流量根据您的配置在层内进行负载均衡。外部 LB 支持 Citrix NetScaler 和 VPC 虚拟路由器。当您使用内部 LB 服务时，在某个层接收的流量会在该层内的不同实例之间进行负载均衡。例如，在 Web 层到达的流量将重定向到该层中的另一个实例。内部 LB 不支持外部负载平衡设备。该服务由在目标层上配置的内部 LB 实例提供。

#### 在一个Tier层上加入负载均衡(External LB)

CloudStack用户或管理员可以创建负载均衡规则，将公有IP接收到的流量均衡到属于VPC中提供负载均衡服务的网络层的一个或多个实例。用户创建规则，指定算法，并将该规则分配给层中的一组实例。

##### 使NetScaler在一个VPC的tier层上作为一个LB Provider

1. Add and enable Netscaler VPX in dedicated mode.
    
    Netscaler can be used in a VPC environment only if it is in dedicated mode.
    
2. Create a network offering, as given in “[Creating a Network Offering for External LB](https://docs.cloudstack.apache.org/en/latest/adminguide/networking_and_traffic.html#create-net-offering-ext-lb)”.
    
3. Create a VPC with Netscaler as the Public LB provider.
    
    For more information, see [“Adding a Virtual Private Cloud”](https://docs.cloudstack.apache.org/en/latest/adminguide/networking_and_traffic.html#adding-a-virtual-private-cloud).
    
4. For the VPC, acquire an IP.
    
5. Create an external load balancing rule and apply, as given in [Creating an External LB Rule](https://docs.cloudstack.apache.org/en/latest/adminguide/networking_and_traffic.html#create-ext-lb-rule).

##### 为external LB创建一个网络产品(Offering)

To have external LB support on VPC, create a network offering as follows:

1. Log in to the CloudStack UI as a user or admin.
    
2. Navigate to Service Offerings and choose Network Offering.
    
3. Click Add Network Offering.
    
4. In the dialog, make the following choices:
    
    - **Name**: Any desired name for the network offering.
        
    - **Description**: A short description of the offering that can be displayed to users.
        
    - **Network Rate**: Allowed data transfer rate in MB per second.
        
    - **Traffic Type**: The type of network traffic that will be carried on the network.
        
    - **Guest Type**: Choose whether the guest network is isolated or shared.
        
    - **Persistent**: Indicate whether the guest network is persistent or not. The network that you can provision without having to deploy an Instance on it is termed persistent network.
        
    - **VPC**: This option indicate whether the guest network is Virtual Private Cloud-enabled. A Virtual Private Cloud (VPC) is a private, isolated part of CloudStack. A VPC can have its own virtual network topology that resembles a traditional physical network. For more information on VPCs, see :ref: about-vpc.
        
    - **Specify VLAN**: (Isolated guest networks only) Indicate whether a VLAN should be specified when this offering is used.
        
    - **Supported Services**: Select Load Balancer. Use Netscaler or VpcVirtualRouter.
        
    - **Load Balancer Type**: Select Public LB from the drop-down.
        
    - **LB Isolation**: Select Dedicated if Netscaler is used as the external LB provider.
        
    - **System Offering**: Choose the system service offering that you want virtual routers to use in this network.
        
    - **Conserve mode**: Indicate whether to use conserve mode. In this mode, network resources are allocated only when the first virtual machine starts in the network.
        
5. Click OK and the network offering is created.

##### 创建一个external LB规则

1. Log in to the CloudStack UI as an administrator or end user.
    
2. In the left navigation, choose Network.
    
3. In the Select view, select VPC.
    
    All the VPCs that you have created for the Account is listed in the page.
    
4. Click the Configure button of the VPC, for which you want to configure load balancing rules.
    
    The VPC page is displayed where all the tiers you created listed in a diagram.
    
    For each tier, the following options are displayed:
    
    - Internal LB
        
    - Public LB IP
        
    - Static NAT
        
    - Instances
        
    - CIDR
        
    
    The following router information is displayed:
    
    - Private Gateways
        
    - Public IP Addresses
        
    - Site-to-Site VPNs
        
    - Network ACL Lists
        
5. In the Router node, select Public IP Addresses.
    
    The IP Addresses page is displayed.
    
6. Click the IP address for which you want to create the rule, then click the Configuration tab.
    
7. In the Load Balancing node of the diagram, click View All.
    
8. Select the tier to which you want to apply the rule.
    
9. Specify the following:
    
    - **Name**: A name for the load balancer rule.
        
    - **Public Port**: The port that receives the incoming traffic to be balanced.
        
    - **Private Port**: The port that the Instances will use to receive the traffic.
        
    - **Algorithm**. Choose the load balancing algorithm you want CloudStack to use. CloudStack supports the following well-known algorithms:
        
        - Round-robin
            
        - Least connections
            
        - Source
            
    - **Stickiness**. (Optional) Click Configure and choose the algorithm for the stickiness policy. See Sticky Session Policies for Load Balancer Rules.
        
    - **Add Instances**: Click Add Instances, then select two or more Instances that will divide the load of incoming traffic, and click Apply.
        

The new load balancing rule appears in the list. You can repeat these steps to add more load balancing rules for this IP address.

#### 跨Tier的负载均衡

CloudStack支持在VPC内的不同层之间共享工作负载。假设在环境中设置了多个层，例如 Web 层和应用层。每个层的流量在公共端的 VPC 虚拟路由器上进行平衡，如 [[#在一个VPC上添加负载均衡规则]] 中所述。如果你想平衡从Web层到应用层的流量，请使用CloudStack提供的内部负载均衡功能。

#### 内部LB在VPC中是如何工作的

在此图中，为公共 IP 72.52.125.10 创建了一个公共 LB 规则，具有公共端口 80 和专用端口 81。在 VPC 虚拟路由器上创建的 LB 规则将应用于从 Internet 到 Web 层上的实例的流量。在应用程序层上，将创建两个内部负载均衡规则。在实例 InternalLBVM1 上配置了具有负载均衡器端口 23 和实例端口 25 的客户机 IP 10.10.10.4 的内部 LB 规则。具有负载均衡器端口 45 和实例端口 46 的客户机 IP 10.10.10.4 的另一个内部 LB 规则在实例 InternalLBVM1 上配置。客户机 IP 10.10.10.6 的另一个内部 LB 规则（负载均衡器端口 23 和实例端口 25）在实例 InternalLBVM2 上配置。

![[Pasted image 20240310115500.png]]

##### 指南

- 内部 LB 和公共 LB 在层上是互斥的。如果该层在公共端具有 LB，则它不能具有内部 LB。
    
- 在CloudStack 4.2版本中，只有VPC网络支持内部LB。
    
- 在CloudStack 4.2版本中，只有内部LB实例可以作为内部LB提供者。
    
- 不支持从具有内部 LB 的网络产品到具有公共 LB 的网络产品/服务的网络升级。
    
- VPC中可以支持多个层的内部LB。
    
- VPC中只能有一个层支持公有LB。

##### 使Internal LB在一个VPC网络tier层上

1. Create a network offering, as given in [Creating a Network Offering for Internal LB](https://docs.cloudstack.apache.org/en/latest/adminguide/networking_and_traffic.html#creating-net-offering-internal-lb).
    
2. Create an internal load balancing rule and apply, as given in [Creating an Internal LB Rule](https://docs.cloudstack.apache.org/en/latest/adminguide/networking_and_traffic.html#create-int-lb-rule).

##### 为Internal LB创建一个网络产品(Offering)

要在 VPC 上获得内部 LB 支持，请使用默认产品 DefaultIsolatedNetworkOfferingForVpcNetworksWithInternalLB，或按如下方式创建网络产品：

1. Log in to the CloudStack UI as a user or admin.
    
2. Navigate to Service Offerings and choose Network OfferingPublic IP Addresses.
    
3. Click Add Network Offering.
    
4. In the dialog, make the following choices:
    
    - **Name**: Any desired name for the network offering.
        
    - **Description**: A short description of the offering that can be displayed to users.
        
    - **Network Rate**: Allowed data transfer rate in MB per second.
        
    - **Traffic Type**: The type of network traffic that will be carried on the network.
        
    - **Guest Type**: Choose whether the guest network is isolated or shared.
        
    - **Persistent**: Indicate whether the guest network is persistent or not. The network that you can provision without having to deploy an Instance on it is termed persistent network.
        
    - **VPC**: This option indicate whether the guest network is Virtual Private Cloud-enabled. A Virtual Private Cloud (VPC) is a private, isolated part of CloudStack. A VPC can have its own virtual network topology that resembles a traditional physical network. For more information on VPCs, see [“About Virtual Private Clouds”](https://docs.cloudstack.apache.org/en/latest/adminguide/networking_and_traffic.html#about-virtual-private-clouds).
        
    - **Specify VLAN**: (Isolated guest networks only) Indicate whether a VLAN should be specified when this offering is used.
        
    - **Supported Services**: Select Load Balancer. Select `InternalLbVM` from the provider list.
        
    - **Load Balancer Type**: Select Internal LB from the drop-down.
        
    - **System Offering**: Choose the system service offering that you want virtual routers to use in this network.
        
    - **Conserve mode**: Indicate whether to use conserve mode. In this mode, network resources are allocated only when the first virtual machine starts in the network.
        
5. Click OK and the network offering is created.

##### 创建一个Internal LB规则

当您创建内部 LB 规则并应用于实例时，将创建一个负责负载均衡的内部 LB 实例。

如果导航到“**基础架构**”>“**区域**”> <zone_名称> > <physical_network_name> >“**网络服务提供商**”>“**内部LB实例**”，则可以在“实例”页面中查看创建的内部 LB 实例。您可以根据需要从该位置管理内部 LB 实例。

1. Log in to the CloudStack UI as an administrator or end user.
    
2. In the left navigation, choose Network.
    
3. In the Select view, select VPC.
    
    All the VPCs that you have created for the Account is listed in the page.
    
4. Locate the VPC for which you want to configure internal LB, then click Configure.
    
    The VPC page is displayed where all the tiers you created listed in a diagram.
    
5. Locate the Tier for which you want to configure an internal LB rule, click Internal LB.
    
    In the Internal LB page, click Add Internal LB.
    
6. In the dialog, specify the following:
    
    - **Name**: A name for the load balancer rule.
        
    - **Description**: A short description of the rule that can be displayed to users.
        
    - **Source IP Address**: (Optional) The source IP from which traffic originates. The IP is acquired from the CIDR of that particular tier on which you want to create the Internal LB rule. If not specified, the IP address is automatically allocated from the network CIDR.
        
        For every Source IP, a new Internal LB Instance is created for load balancing.
        
    - **Source Port**: The port associated with the source IP. Traffic on this port is load balanced.
        
    - **Instance Port**: The port of the internal LB Instance.
        
    - **Algorithm**. Choose the load balancing algorithm you want CloudStack to use. CloudStack supports the following well-known algorithms:
        
        - Round-robin
            
        - Least connections
            
        - Source
### 在一个VPC上添加端口转发规则

1. Log in to the CloudStack UI as an administrator or end user.
    
2. In the left navigation, choose Network.
    
3. In the Select view, select VPC.
    
    All the VPCs that you have created for the Account is listed in the page.
    
4. Click the Configure button of the VPC to which you want to deploy the Instances.
    
    The VPC page is displayed where all the tiers you created are listed in a diagram.
    
    For each tier, the following options are displayed:
    
    - Internal LB
        
    - Public LB IP
        
    - Static NAT
        
    - Instances
        
    - CIDR
        
    
    The following router information is displayed:
    
    - Private Gateways
        
    - Public IP Addresses
        
    - Site-to-Site VPNs
        
    - Network ACL Lists
        
5. In the Router node, select Public IP Addresses.
    
    The IP Addresses page is displayed.
    
6. Click the IP address for which you want to create the rule, then click the Configuration tab.
    
7. In the Port Forwarding node of the diagram, click View All.
    
8. Select the tier to which you want to apply the rule.
    
9. Specify the following:
    
    - **Public Port**: The port to which public traffic will be addressed on the IP address you acquired in the previous step.
        
    - **Private Port**: The port on which the Instance is listening for forwarded public traffic.
        
    - **Protocol**: The communication protocol in use between the two ports.
        
        - TCP
            
        - UDP
            
    - **Add Instance**: Click Add Instance. Select the name of the Instance to which this rule applies, and click Apply.
        
        You can test the rule by opening an SSH session to the Instance.

### 删除Tiers

您可以从 VPC 中删除tier层。无法撤消已删除的层。删除某个层时，仅清除该层的资源。将删除所有网络规则（端口转发、负载平衡和静态 NAT）以及与层关联的 IP 地址。该 IP 地址仍属于同一 VPC。
1. Log in to the CloudStack UI as an administrator or end user.
    
2. In the left navigation, choose Network.
    
3. In the Select view, select VPC.
    
    All the VPC that you have created for the Account is listed in the page.
    
4. Click the Configure button of the VPC for which you want to set up tiers.
    
    The Configure VPC page is displayed. Locate the tier you want to work with.
    
5. Select the tier you want to remove.
    
6. In the Network Details tab, click the Delete Network button. ![button to remove a tier](https://docs.cloudstack.apache.org/en/latest/_images/del-tier.png)
    
    Click Yes to confirm. Wait for some time for the tier to be removed.

### 编辑-重启-删除一个VPC

> 在删除 VPC 之前，请确保删除所有tier层。

1. Log in to the CloudStack UI as an administrator or end user.
    
2. In the left navigation, choose Network.
    
3. In the Select view, select VPC.
    
    All the VPCs that you have created for the Account is listed in the page.
    
4. Select the VPC you want to work with.
    
5. In the Details tab, click the Remove VPC button ![button to remove a VPC](https://docs.cloudstack.apache.org/en/latest/_images/remove-vpc.png)
    
    You can remove the VPC by also using the remove button in the Quick View.
    
    You can edit the name and description of a VPC. To do that, select the VPC, then click the Edit button. ![button to edit.](https://docs.cloudstack.apache.org/en/latest/_images/edit-icon.png)
    
    To restart a VPC, select the VPC, then click the Restart button. ![button to restart a VPC](https://docs.cloudstack.apache.org/en/latest/_images/restart-vpc.png)

## 持久化网络

https://docs.cloudstack.apache.org/en/latest/adminguide/networking_and_traffic.html#persistent-networks

## 建立一个Palo网络防火墙

https://docs.cloudstack.apache.org/en/latest/adminguide/networking_and_traffic.html#setup-a-palo-alto-networks-firewall

## 使用远程访问VPN

https://docs.cloudstack.apache.org/en/latest/adminguide/networking/using_remote_access.html#using-remote-access-vpn

## 虚拟网络功能(VNFs)模板以及应用

https://docs.cloudstack.apache.org/en/latest/adminguide/networking/vnf_templates_appliances.html#vnf-templates-and-appliances