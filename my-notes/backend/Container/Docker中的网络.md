
## 网络概览

容器网络是指容器之间或容器与非 Docker 工作负载之间相互连接和通信的能力。

容器不知道它连接的是哪种网络，也不知道它们的对端是否也是 Docker 工作负载。容器只能看到带有 IP 地址的网络接口、网关、路由表、DNS 服务和其他网络细节。也就是说，除非容器使用 `none` 网络驱动程序。

本章从容器的角度介绍网络连接，以及与容器网络连接有关的概念。本章不描述有关 Docker 网络如何工作的特定于操作系统的细节。有关 Docker 如何在 Linux 上操纵 `iptables` 规则的信息，请参阅[包过滤和防火墙]([Packet filtering and firewalls | Docker Docs](https://docs.docker.com/network/packet-filtering-firewalls/))。

### 已发布的端口

默认情况下，当你使用 `docker create` 或 `docker run` 创建或运行一个容器时，容器不会向外界公开它的任何端口。使用 `--publish` 或 `-p` 标志，可以将端口提供给 Docker 以外的服务。这会在主机中创建一条防火墙规则，将容器端口映射到 Docker 主机上对外的端口。下面是一些例子：

|Flag value|Description|
|---|---|
|`-p 8080:80`|Map port `8080` on the Docker host to TCP port `80` in the container.|
|`-p 192.168.1.100:8080:80`|Map port `8080` on the Docker host IP `192.168.1.100` to TCP port `80` in the container.|
|`-p 8080:80/udp`|Map port `8080` on the Docker host to UDP port `80` in the container.|
|`-p 8080:80/tcp -p 8080:80/udp`|Map TCP port `8080` on the Docker host to TCP port `80` in the container, and map UDP port `8080` on the Docker host to UDP port `80` in the container.|

> 重要： 发布容器端口默认情况下是不安全的。这意味着，当你发布一个容器的端口时，不仅 Docker 主机可以访问它，外部世界也可以访问它。
> 
> 如果在发布标记中包含本地主机 IP 地址（127.0.0.1），则只有 Docker 主机才能访问已发布的容器端口。
>
> `docker run -p 127.0.0.1:8080:80 nginx`

> 警告: 同一 L2 网段内的主机（例如，连接到同一网络交换机的主机）可以访问发布到 localhost 的端口。更多信息，请参阅 [Publishing ports explicitly to private networks should not be accessible from LAN hosts · Issue #45610 · moby/moby (github.com)](https://github.com/moby/moby/issues/45610)

如果想让其他容器访问某个容器，就没有必要发布容器的端口。您可以通过将容器连接到同一网络（通常是[桥接网络]([Bridge network driver | Docker Docs](https://docs.docker.com/network/drivers/bridge/))）来启用容器间通信。

### IP地址和hostname

默认情况下，容器会为其连接的每个 Docker 网络获取一个 IP 地址。容器从网络的 IP 子网中获得一个 IP 地址。Docker 守护进程会为容器执行动态子网划分和 IP 地址分配。每个网络还有一个默认子网掩码和网关。

当容器启动时，它只能使用 `--network` 标志连接到一个网络。你可以使用 docker 网络连接（`docker network connect`）命令，将运行中的容器连接到其他网络。在这两种情况下，你都可以使用 `--ip` 或 `--ip6` 标志来指定容器在特定网络上的 IP 地址。

同样，容器的hostname默认为容器在 Docker 中的 ID。您可以使用 `--hostname` 来覆盖主机名。当使用 docker 网络连接（`docker network connect`）连接到现有网络时，可以使用 `--alias` 标志为该网络上的容器指定一个额外的网络别名。

### DNS服务

默认情况下，容器会继承 `/etc/resolv.conf` 配置文件中定义的主机 DNS 设置。连接到默认桥接网络的容器会收到该文件的副本。连接到[自定义网络](https://docs.docker.com/network/network-tutorial-standalone/#use-user-defined-bridge-networks)的容器会使用 Docker 的嵌入式 DNS 服务器。嵌入式 DNS 服务器会将外部 DNS 查询转发给主机上配置的 DNS 服务器。

你可以使用用于启动容器的 `docker run` 或 `docker create` 命令的标记，按容器配置 DNS 解析。下表描述了与 DNS 配置相关的可用 docker 运行标记。

|Flag|Description|
|---|---|
|`--dns`|The IP address of a DNS server. To specify multiple DNS servers, use multiple `--dns` flags. If the container can't reach any of the IP addresses you specify, it uses Google's public DNS server at `8.8.8.8`. This allows containers to resolve internet domains.|
|`--dns-search`|A DNS search domain to search non-fully qualified hostnames. To specify multiple DNS search prefixes, use multiple `--dns-search` flags.|
|`--dns-opt`|A key-value pair representing a DNS option and its value. See your operating system's documentation for `resolv.conf` for valid options.|
|`--hostname`|The hostname a container uses for itself. Defaults to the container's ID if not specified.|

### Nameservers with IPv6地址

如果主机系统上的 `/etc/resolv.conf` 文件包含一个或多个带有 IPv6 地址的 nameserver 条目，这些 nameserver 条目就会被复制到运行的容器中的 `/etc/resolv.conf` 文件。

对于使用 musl libc 的容器（换句话说，Alpine Linux），这会导致主机名查找出现竞争条件。因此，如果外部 IPv6 DNS 服务器在与嵌入式 DNS 服务器的竞争条件中获胜，主机名解析可能会偶尔失败。

外部 DNS 服务器比嵌入式 DNS 服务器快的情况很少见。但垃圾回收或大量并发 DNS 请求等情况有时会导致外部服务器的往返速度快于本地解析。

### 自定义主机

在主机上的 `/etc/hosts` 文件中定义的自定义主机不会被容器继承。要向容器传递额外的主机，请参阅 `docker run` 参考文档中的向[容器主机文件添加条目]([docker run | Docker Docs](https://docs.docker.com/engine/reference/commandline/run/#add-host))。

### 代理服务器

如果你的容器虚拟使用一个代理服务器，请参考 [Configure Docker to use a proxy server | Docker Docs](https://docs.docker.com/network/proxy/)

## 网络驱动器(drivers)

### 概览

Docker 的网络子系统是可插拔的，使用驱动程序。默认情况下存在几个驱动程序，提供核心网络功能：

-  `bridge`  默认网络驱动程序。如果您没有指定驱动程序，这就是您要创建的网络类型。当应用程序在需要与同一主机上的其他容器通信的容器中运行时，通常会使用桥接网络。请参见  [[#Bridge]]
- `host` 移除容器与 Docker 主机之间的网络隔离，直接使用主机网络。请参阅 [[#Host]] 。
- `overlay` overlay网络将多个 Docker 守护进程连接在一起，使 Swarm 服务和容器能够跨节点通信。这种策略无需进行操作系统级路由选择。[[#Overlay]]
- `ipvlan`  IPvlan 网络可让用户完全控制 IPv4 和 IPv6 寻址。在此基础上，VLAN 驱动程序还能让运营商完全控制第 2 层 VLAN 标记，甚至还能为对底层网络集成感兴趣的用户提供 IPvlan L3 路由。[[#IPvlan]]
- `macvlan` Macvlan 网络允许你为容器分配一个 MAC 地址，使它看起来像网络上的一个物理设备。Docker 守护进程会根据容器的 MAC 地址将流量路由到容器。在处理希望直接连接到物理网络而不是通过 Docker 主机的网络堆栈路由的传统应用程序时，使用 `macvlan` 驱动程序有时是最佳选择。[[#Macvlan]]
- `none` 将容器与主机和其他容器完全隔离。[[#None(no networking)]]
- [网络插件]([Plugins and Services | Docker Docs](https://docs.docker.com/engine/extend/plugins_services/)), 您可以使用 Docker 安装和使用第三方网络插件。

#### 网络驱动器总结

- 默认桥接网络适用于运行不需要特殊网络功能的容器。
- 用户定义的桥接网络能让同一 Docker 主机上的容器相互通信。用户定义的网络通常为属于一个共同项目或组件的多个容器定义一个隔离网络。
- 主机网络与容器共享主机网络。使用此驱动程序时，容器的网络不会与主机隔离。
- 当你需要运行在不同 Docker 主机上的容器进行通信，或者多个应用程序使用 Swarm 服务协同工作时，最好使用overlay网络。
- Macvlan 网络适用于从虚拟机设置迁移的情况，或者需要让容器看起来像网络上的物理主机，每个容器都有一个唯一的 MAC 地址。
- IPvlan 与 Macvlan 类似，但不会为容器分配唯一的 MAC 地址。如果对可分配给网络接口或端口的 MAC 地址数量有限制，可以考虑使用 IPvlan。
- 第三方网络插件允许你将 Docker 与专门的网络堆栈集成。

#### 网络教程

- [Networking with standalone containers | Docker Docs](https://docs.docker.com/network/network-tutorial-standalone/)
- [Networking using the host network | Docker Docs](https://docs.docker.com/network/network-tutorial-host/)
- [Networking with overlay networks | Docker Docs](https://docs.docker.com/network/network-tutorial-overlay/)
- [Networking using a macvlan network | Docker Docs](https://docs.docker.com/network/network-tutorial-macvlan/)

### Bridge

就网络而言，网桥是一种链路层设备，用于转发网段之间的流量。网桥可以是硬件设备，也可以是在主机内核中运行的软件设备。

就 Docker 而言，桥接网络使用软件桥接器，让连接到同一桥接网络的容器进行通信，同时提供与未连接到该桥接网络的容器的隔离。Docker 桥接驱动程序会自动在主机中安装规则，使不同桥接网络上的容器无法直接相互通信。

桥接网络适用于在同一 Docker 守护进程主机上运行的容器。对于运行在不同 Docker 守护进程主机上的容器之间的通信，你可以在操作系统层面管理路由，也可以使用 [[#Overlay]]。

启动 Docker 时，会自动创建一个默认的桥接网络（也称为bridge），除非另有指定，否则新启动的容器会连接到该网络。**你也可以创建用户自定义的bridge网络。用户定义的bridge网络优于默认bridge网络。** 

#### 用户自定义的bridge网络和默认的Bridge网络的区别

- 用户自定义的Bridge网络提供同网络中容器间的自动DNS解析

默认桥接网络上的容器只能通过 IP 地址相互访问，除非使用 --link 选项（这被视为传统选项）。在用户定义的桥接网络上，容器可以通过名称（容器名）或别名相互解析。

想象一下，一个应用程序有一个网络前端和一个数据库后端。如果将容器称为 web 和 db，那么无论应用堆栈运行在哪台 Docker 主机上，web 容器都能连接到 db 容器的 db。

如果在默认桥接网络上运行同一个应用栈，则需要在容器之间手动创建链接（使用传统的 `--link` 标志）。这些链接需要双向创建，因此如果有两个以上的容器需要通信，情况就会变得复杂。或者，你也可以在容器内操作 `/etc/hosts` 文件，但这会产生难以调试的问题。

- 用户自定义的Bridge网络提供更好的隔离性

所有未指定 `--network` 的容器都会连接到默认桥接网络。这可能会带来风险，因为无关的堆栈/服务/容器都能进行通信。

使用用户定义的网络可以提供一个有范围的网络，只有连接到该网络的容器才能进行通信。

- 容器可以在用户自定义的Bridge网络上自由加入或者离开

在容器的生命周期内，可以即时连接或断开容器与用户定义网络的连接。要从默认桥接网络中移除容器，需要停止容器并使用不同的网络选项重新创建它。

- 每个用户定义的网络都会创建一个可配置的网桥

如果您的容器使用默认bridge网络，您可以对其进行配置，但所有容器都使用相同的设置，如 MTU 和 `iptables` 规则。此外，配置默认bridge网络发生在 Docker 本身之外，需要重新启动 Docker。

用户定义的桥接网络是通过 `docker network create` 创建和配置的。如果不同的应用程序组有不同的网络要求，可以在创建时分别配置每个用户定义的bridge。

- 默认bridge网络上的链接容器共享环境变量。

最初，在两个容器之间共享环境变量的唯一方法是使用 `--link` 标志将它们链接起来。这种变量共享方式在用户定义的网络中是不可能实现的。不过，共享环境变量还有更优越的方法。以下是一些想法：

> 多个容器可以使用 Docker volume 挂载包含共享信息的文件或目录。
    多个容器可以使用 `docker-compose` 一起启动，而组成文件可以定义共享变量。
    可以使用 swarm 服务来代替独立容器，并利用共享的[秘密]([Manage sensitive data with Docker secrets | Docker Docs](https://docs.docker.com/engine/swarm/secrets/))和[配置]([Store configuration data using Docker Configs | Docker Docs](https://docs.docker.com/engine/swarm/configs/))。

连接到同一用户定义bridge网络的容器会有效地相互暴露所有端口。要让不同网络上的容器或非 Docker 主机访问某个端口，必须使用 `-p` 或 `--publish` 标记发布该端口。

#### 选项

下表描述了在使用网桥驱动程序创建自定义网络时，可以传递给 `--option` 的特定于驱动程序的选项。

|Option|Default|Description|
|---|---|---|
|`com.docker.network.bridge.name`||Interface name to use when creating the Linux bridge.|
|`com.docker.network.bridge.enable_ip_masquerade`|`true`|Enable IP masquerading.|
|`com.docker.network.bridge.enable_icc`|`true`|Enable or Disable inter-container connectivity.|
|`com.docker.network.bridge.host_binding_ipv4`||Default IP when binding container ports.|
|`com.docker.network.driver.mtu`|`0` (no limit)|Set the containers network Maximum Transmission Unit (MTU).|
|`com.docker.network.container_iface_prefix`|`eth`|Set a custom prefix for container interfaces.|


其中一些选项也可以作为 `dockerd CLI` 的标志，你可以在启动 Docker 守护进程时使用它们来配置默认的 `docker0` bridge。下表显示了哪些选项在 `dockerd` CLI 中有对应的标志。

|Option|Flag|
|---|---|
|`com.docker.network.bridge.name`|-|
|`com.docker.network.bridge.enable_ip_masquerade`|`--ip-masq`|
|`com.docker.network.bridge.enable_icc`|`--icc`|
|`com.docker.network.bridge.host_binding_ipv4`|`--ip`|
|`com.docker.network.driver.mtu`|`--mtu`|
|`com.docker.network.container_iface_prefix`|-|


Docker 守护进程支持`--bridge`标志，你可以用它来定义自己的 `docker0` 桥。如果你想在同一台主机上运行多个守护进程实例，请使用该选项。详情请参阅[运行多个守护进程]([dockerd | Docker Docs](https://docs.docker.com/engine/reference/commandline/dockerd/#run-multiple-daemons))。

#### 管理一个用户自定义的bridge网络

创建 `docker network create <network-name>`

你可以指定子网、IP 地址范围、网关和其他选项。详情请参考 [docker network create 参考资料]([docker network create | Docker Docs](https://docs.docker.com/engine/reference/commandline/network_create/#specify-advanced-options))或 `docker network create --help` 的输出结果。

使用 `docker network rm` 命令移除用户定义的Bridge网络。如果容器当前连接到网络，请先[断开它们的连接]([Bridge network driver | Docker Docs](https://docs.docker.com/network/drivers/bridge/#disconnect-a-container-from-a-user-defined-bridge))。

`docker network rm <network-name>`

> 到底发生了什么？
> 
   当您创建或移除用户定义的bridhe，或从用户定义的网桥连接或断开容器时，Docker 会使用操作系统特有的工具来管理底层网络基础设施（例如在 Linux 上添加或移除网桥设备或配置 `iptables` 规则）。这些细节应视为实施细节。让 Docker 为您管理用户定义的网络吧。

#### 把一个容器连接到一个用户自定义的bridge网络

创建时直接连接:

```bash
docker create --name my-nginx \
  --network <network-name> \
  --publish 8080:80 \
  nginx:latest
```

已在运行中的容器连接:

```bash
docker network connect <network-name> my-nginx
```

#### 断开一个容器与某个bridge网络的连接

```bash
docker network disconnect <network-name> my-nginx
```

#### 使用IPv6


### Overlay

### Host

### IPvlan

### Macvlan

### None(no networking)

