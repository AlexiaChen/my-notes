
> 一个需求场景，就是两个客户端都在访问一个服务端，服务端通过TCP拿到的remote IP如果没有NAT的隐藏，拿到的应该就是客户端的IP，但是如果都在NAT后面，那么拿到的都是同一个IP。但是我要在同一个局域网（不同IP机器）中模拟出这个场景，需要测试，有没有什么软件可以模拟NAT这个同一个IP的场景

你可以使用网络模拟工具如 `netns` 和 `iptables` 来模拟NAT环境。以下是一个简单的步骤来实现这个场景：

1. **创建网络命名空间**：使用 `ip netns` 创建两个网络命名空间来模拟两个客户端。
2. **配置虚拟网络接口**：为每个命名空间配置虚拟网络接口。
3. **设置NAT**：使用 `iptables` 在主机上设置NAT规则，使两个命名空间的流量通过同一个IP地址。

以下是详细步骤：

### 1. 创建网络命名空间

```bash
sudo ip netns add client1
sudo ip netns add client2
```


### 2. 配置虚拟网络接口


```bash
# 创建虚拟以太网对
sudo ip link add veth1 type veth peer name veth1-br
sudo ip link add veth2 type veth peer name veth2-br

# 将一端移动到命名空间
sudo ip link set veth1 netns client1
sudo ip link set veth2 netns client2

# 启用接口
sudo ip netns exec client1 ip link set veth1 up
sudo ip netns exec client2 ip link set veth2 up

sudo ip link set veth1-br up
sudo ip link set veth2-br up

# 分配IP地址
sudo ip netns exec client1 ip addr add 10.0.0.1/24 dev veth1
sudo ip netns exec client2 ip addr add 10.0.0.2/24 dev veth2

```


### 3. 设置NAT

```bash
# 启用IP转发
sudo sysctl -w net.ipv4.ip_forward=1

# 配置NAT规则 将匹配的数据包目标跳转到 `MASQUERADE` 目标。`MASQUERADE` 会动态地将源 IP 地址替换为外部接口的 IP 地址，并在连接关闭时自动清理。
sudo iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -o eth0 -j MASQUERADE
```



### 4. 测试连接

在每个命名空间中启动客户端程序，连接到服务端的IP地址。服务端将看到来自同一个NAT IP的连接。

```bash
# 在client1命名空间中运行客户端
sudo ip netns exec client1 <your-client-command>

# 在client2命名空间中运行客户端
sudo ip netns exec client2 <your-client-command>
```

这样，你就可以在同一个局域网中模拟出NAT环境，并测试服务端如何处理来自同一个NAT IP的多个客户端连接。


以上的客户端如果是有类似`./client install`会安装到宿主机相同目录的命令，那么最好用chroot隔离:

```bash
sudo mkdir -p /var/lib/machines/client1
sudo mkdir -p /var/lib/machines/client2
# 安装debootstrap
sudo apt-get install debootstrap
# 在目录上构建一个ubuntu Linux的目录
sudo debootstrap --arch=amd64 <focal | jammy> /var/lib/machines/client1 http://archive.ubuntu.com/ubuntu/
sudo debootstrap --arch=amd64 <focal | jammy> /var/lib/machines/client2 http://archive.ubuntu.com/ubuntu/
# 拷贝客户端的二进制到特定的目录下
sudo cp /path/to/your/binary_client /var/lib/machines/client1/usr/local/bin/
sudo cp /path/to/your/binary_client /var/lib/machines/client2/usr/local/bin/

sudo mount --bind /proc /var/lib/machines/client1/proc
sudo mount --bind /sys /var/lib/machines/client1/sys
sudo mount --bind /dev /var/lib/machines/client1/dev
sudo mount --bind /run /var/lib/machines/client1/run

sudo mount --bind /proc /var/lib/machines/client2/proc
sudo mount --bind /sys /var/lib/machines/client2/sys
sudo mount --bind /dev /var/lib/machines/client2/dev
sudo mount --bind /run /var/lib/machines/client2/run

# 通过chroot把客户端的程序放在特定的根目录执行
sudo chroot /var/lib/machines/client1 /usr/local/bin/<your-client-program> <cmd-lines>
sudo chroot /var/lib/machines/client2 /usr/local/bin/<your-client-program> <cmd-lines>

sudo chroot /var/lib/machines/client1 /usr/cloudDetect/detect install
sudo chroot /var/lib/machines/client2 /usr/cloudDetect/detect install
sudo chroot /var/lib/machines/client1 /usr/cloudDetect/detect run
sudo chroot /var/lib/machines/client2 /usr/cloudDetect/detect run
```