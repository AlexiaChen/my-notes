
## 概览

使用云基础设施的人对云提供的网络服务有各种需求和偏好。作为CloudStack管理员，你可以做以下的事情来为你的用户设置网络。

- 在区域(Zones)中设置物理网络
- 在单个物理网络上为同一服务设置多个不同的提供商（例如，Cisco 和瞻博网络防火墙）
- 将不同类型的网络服务捆绑到网络产品中，以便用户可以为任何给定的虚拟机选择所需的网络服务
- 随着时间的推移添加新的网络产品，以便最终用户可以在其网络上升级到更好的服务等级
- 为用户访问网络提供更多方式，例如通过用户所属的项目

## 关于虚拟网络

虚拟网络是一种逻辑构造，可在单个物理网络上实现多租户。在CloudStack中，虚拟网络可以是共享的，也可以是隔离的。

### 隔离网络

隔离网络只能由单个账户的实例访问。隔离网络具有以下属性。

- 动态分配VLAN等资源，回收垃圾
- 整个网络都有一个网络Offering
- 网络Offering可以升级或降级，但它适用于整个网络

更多信息请参考 https://docs.cloudstack.apache.org/en/latest/adminguide/networking_and_traffic.html#configure-guest-traffic-in-an-advanced-zone
### 共享网络

共享网络可以由属于许多不同账户的实例访问。共享网络上的网络隔离是通过使用安全组等技术实现的，这些技术仅在基本Zone区域或具有安全组的高级Zone区域中受支持。

- 共享网络由终端用户或管理员创建。允许网络创建者指定 VLAN 的网络Offering只能由 root 管理员创建。
- 可以将共享网络指定到某个域(Domain)
- 共享网络资源（如 VLAN 和它映射到的物理网络）由管理员指定
- 共享网络可以按安全组隔离
- 公用网络是不向终端用户显示的共享网络

当服务提供商是虚拟路由器时，共享网络不支持每个Zone区域的source NAT。但是，支持每个账户的source NAT。

更多信息参考 https://docs.cloudstack.apache.org/en/latest/adminguide/networking_and_traffic.html#configuring-a-shared-guest-network
### L2(二层)网络

L2 网络提供网络隔离，无需任何其他服务。这意味着不会有虚拟路由器。假定最终用户将拥有自己的 IPAM，或者他们将静态分配 IP 地址。

L2 网络可以由最终用户创建，但允许网络创建者指定 VLAN 的网络产品只能由 root 管理员创建。

CloudStack不会给实例分配IP地址。

用户数据和元数据可以使用配置驱动器传递到实例（必须在网络服务产品中启用）

示例 GUI 对话框（对于普通用户帐户）如下所示：

![[Pasted image 20240307085811.png]]
### 虚拟网络资源的运行时分配

当你定义一个新的虚拟网络时，你对这个网络的所有设置都存储在CloudStack中。只有当第一个实例在网络中启动时，才会激活实际的网络资源。当所有实例都离开虚拟网络时，将对网络资源进行垃圾回收，以便可以再次分配它们。这有助于节省网络资源。

## 网络服务提供者(Provider)

> 关于最新的网络服务供应商列表，参见CloudStack UI或调用listNetworkServiceProviders。

服务提供者（也称为网元）是使网络服务成为可能的硬件或虚拟设备; 例如，可以在云中安装防火墙设备以提供防火墙服务。在单个网络上，多个提供者可以提供相同的网络服务。例如，防火墙服务可能由同一物理网络中的 Cisco 或Juniper网络设备提供。

网络中可以有同一服务提供商的多个实例（例如，多个Juniper网络 SRX 设备）。

如果设置了不同的提供者以在网络上提供相同的服务，则管理员可以创建网络产品，以便用户可以指定他们喜欢的网络服务提供商（以及网络产品中提供的其他选项）。否则，CloudStack将选择在需要服务时使用哪个提供者。

支持的网络服务提供商

CloudStack附带了一个内部支持的服务提供商列表，你可以在创建网络产品时从这个列表中进行选择。

![[Pasted image 20240307090613.png]]

> 注意，以上Virtual Router不支持弹性IP，也就是IP不可以动态地分配给虚拟机实例。
## 网络产品(Offerings)

> 有关支持的网络服务的最新列表，请参阅CloudStack UI或调用listNetworkServices。


网络产品是一组命名的网络服务，例如：

- DHCP
- DNS
- source NAT
- static NAT
- 端口转发(port forwarding)
- 负载均衡
- 防火墙
- VPN
- （可选）说出要用于给定服务的多个可用提供程序之一
- （可选）用于指定要使用的物理网络的网络tag

创建新实例时，用户选择一种可用的网络offering，这决定了实例可以使用哪些网络服务。

除了CloudStack提供的默认网络产品(Offering)之外，CloudStack管理员还可以创建任意数量的自定义网络产品。通过创建多个自定义网络产品，您可以将云设置为在单个多租户物理网络上提供不同类别的服务。例如，虽然两个租户的基础物理布线可能相同，但租户 A 可能只需要对其网站进行简单的防火墙保护，而租户 B 可能正在运行 Web 服务器场，并且需要可扩展的防火墙解决方案、负载平衡解决方案和备用网络来访问数据库后端。

> 如果你在使用包含外部负载均衡器设备（如NetScaler）的网络服务产品时创建负载平衡规则，然后将网络服务产品更改为使用CloudStack虚拟路由器的产品，那么你必须在虚拟路由器上为每个现有的负载平衡规则创建一个防火墙规则，以便它们继续运行。

在创建新的虚拟网络时，CloudStack管理员会选择为该网络启用哪个网络产品。每个虚拟网络都与一个网络产品/服务相关联。可以通过更改虚拟网络的关联网络产品/服务来升级或降级虚拟网络。如果执行此操作，请确保重新编程物理网络以匹配。

CloudStack也有内部网络产品供CloudStack系统VM使用。这些网络产品对用户不可见，但可由管理员修改。


### 创建一个新的网络产品(Offering)

为了创建一个新的网络产品:

1. 以管理员权限登录CloudStack UI
    
2. 在左侧导航栏中，点击 Service Offerings 和 选择 Network Offering.
    
3.  点击Add Network Offering.
    
4. 在对话框中, 作出以下选择:
    
    - **Name**. 网络产品的名称
        
    - **Description**.  网络产品的描述
        
    - **Network Rate**. 允许数据每秒MB的传输速率
        
    - **Guest Type**. 选择访客网络是否是隔离的还是共享的
        
        For a description of this term, see [“About Virtual Networks”](https://docs.cloudstack.apache.org/en/latest/adminguide/networking.html#about-virtual-networks).
        
    - **Persistent**. 访客网络是否是持久化的. 无需在其上部署实例即可预置的网络称为持久性网络。, see [“Persistent Networks”](https://docs.cloudstack.apache.org/en/latest/adminguide/networking_and_traffic.html#persistent-networks).
        
    - **Specify VLAN**. 指示在使用此产品时是否可以指定 VLAN。如果您选择此选项，并在创建 VPC 网络层或隔离网络时使用此网络产品，您将能够为您创建的网络指定 VLAN ID。
        
    - **VPC**. 此选项指示访客网络是否启用了 Virtual Private Cloud。虚拟私有云（VPC）是CloudStack的一个私有的、隔离的部分。VPC 可以拥有自己的虚拟网络拓扑，类似于传统的物理网络。, see [“About Virtual Private Clouds”](https://docs.cloudstack.apache.org/en/latest/adminguide/networking_and_traffic.html#about-virtual-private-clouds).
        
    - **Promiscuous Mode**. 仅适用于 VMware 虚拟机管理程序上的访客网络。它接受以下值来表示网元的所需行为：
        
        _Reject_ - The switch drops any outbound frame from an Instance adapter with a source MAC address that is different from the one in the .vmx configuration file.
        
        _Accept_ - The switch does not perform filtering, and permits all outbound frames.
        
        _None_ - Default to value from global setting - `network.promiscuous.mode`.
        
    - **Forged Transmits**. 仅适用于 VMware 虚拟机管理程序上的访客网络。它接受以下值，以表示网络元素的所需行为:
        
        _Reject_ - The switch drops any outbound frame from an Instance adapter with a source MAC address that is different from the one in the .vmx configuration file.
        
        _Accept_ - The switch does not perform filtering, and permits all outbound frames.
        
        _None_ - Default to value from global setting - `network.forged.transmits`.
        
    - **MAC Address Changes**. 仅适用于 VMware 虚拟机管理程序上的来宾网络。它接受以下值，以表示网元的所需行为:
        
        _Reject_ - If the guest OS changes the effective MAC address of the Instance to a value that is different from the MAC address of the instance network adapter (set in the .vmx configuration file), the switch drops all inbound frames to the adapter.
        
        If the guest OS changes the effective MAC address of the Instance back to the MAC address of the instance network adapter, the Instance receives frames again.
        
        _Accept_ - If the guest OS changes the effective MAC address of the Instance to a value that is different from the MAC address of the instance network adapter, the switch allows frames to the new address to pass.
        
        _None_ - Default to value from global setting - `network.mac.address.changes`.
        
    - **MAC Learning**. 仅适用于具有 VMware Distributed Virtual Switches 版本 6.6.0 及更高版本和 vSphere 版本 6.7 及更高版本的 VMware 虚拟机管理程序上的访客网络。它接受以下值，以表示网络元素的所需行为:
        
        _Reject_ - Network connectivity for multiple MAC address behind a single vNIC will not work.
        
        _Accept_ - Enables network connectivity for multiple MAC addresses behind a single vNIC.
        
        _None_ - Default to value from global setting - `network.mac.learning`.
        
    - **Supported Services**. 选择一个或多个可能的网络服务。对于某些服务，您还必须选择服务提供商;例如，如果你选择Load Balancer，你可以选择CloudStack虚拟路由器或在云中配置的任何其他负载均衡器。根据您选择的服务，对话框的其余部分可能会显示其他字段。.
        
        根据所选的访客网络类型，您可以看到以下支持的服务：
		![[Pasted image 20240307092018.png]]
	- **System Offering**. 如果在“支持的服务”中选择的任何服务的服务提供商是虚拟路由器，则会显示“system offering”字段。选择您希望虚拟路由器在此网络中使用的系统服务产品。例如，如果你在“支持的服务”中选择了“负载均衡器”，并选择了一个虚拟路由器来提供负载均衡，那么就会出现“系统产品”字段，这样你就可以在CloudStack默认的系统服务产品和CloudStack根管理员定义的任何自定义系统服务产品之间进行选择。
    
    For more information, see [“System Service Offerings”](https://docs.cloudstack.apache.org/en/latest/adminguide/service_offerings.html#system-service-offerings).
    
    - **LB Isolation**: 指定要为网络提供的负载均衡器隔离类型：共享或专用。
    
        - **Dedicated**: 如果选择专用 LB 隔离，则会从区域中预配的专用负载均衡器设备池中为网络分配专用负载均衡器设备。如果区域中没有足够的专用负载均衡器设备可用，则网络创建将失败。对于充分利用设备资源的高流量网络来说，专用设备是一个不错的选择。
        
        - **Shared**: 如果选择共享 LB 隔离，则会从区域中预配的共享负载均衡器设备池中为网络分配共享负载均衡器设备。在配置时，CloudStack会选择由最少的账户使用的共享负载均衡器设备。设备达到最大容量后，设备将不会分配给新帐户。

## 使用CloudStack Virtual Router配置自动伸缩(AutoScale) 

[Configuring AutoScale with using CloudStack Virtual Router — Apache CloudStack 4.19.0.0 documentation](https://docs.cloudstack.apache.org/en/latest/adminguide/autoscale_with_virtual_router.html)

