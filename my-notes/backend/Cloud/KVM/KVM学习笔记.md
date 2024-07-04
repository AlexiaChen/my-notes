
## 准备

```bash
grep -E 'svm|vmx' /proc/cpuinfo
```

以上命令主要查看CPU支不支持硬件虚拟化技术，vmx是Intel-VT。  svm是AMD-V。有一些本身支持，但是查看不到，可能是因为在BIOS里面把这个开关关闭了，需要在BIOS里面打开。

```
lsmod | grep kvm
```

以上是查看KVM内核模块是否被加载进内核了。如果有kvm_intel  kvm_intel的输出，那就说明KVM硬件虚拟化模块被加载进内核了。

> 以上其实就是查看软硬件环境是否满足，有了硬件还不够，需要软件来驱动硬件工作

## 安装

```
yum install qemu-kvm libvirt
```

以上命令其实会安装三个包，qemu-kvm是用户态的KVM模拟器，实现客户虚拟机与宿主机的通信并且模拟CPU和外设，qemu-img实现客户虚拟机的磁盘管理，libvirt其实是一个标准的虚拟化API实现，已经被广泛采用，是一个库，也可以作为一个服务，主要是提供宿主机与Hypervisor的通信，这里的Hypervisior是KVM本身。

```
yum install virt-install libvirt-python virt-manager  libvirt-client
```

以上主要是安装libvirt库服务的客户端工具，方便用户操作。virt-install用于创建虚拟机。libvirt-python提供给Python的libvirt API。virt-manager，通过libvirt-client的API来间接通过libvirt操作虚拟机的管理工具，有UI图形界面。libvert-client就是libvirt的API和附带的命令行来实现管理虚拟机的通信。

```bash
systemctl start libvirtd  
systemctl enable libvirtd
systemctl status libvirtd
```

因为libvirt它一般是以server的方式提供，所以要启动它的守护进程

```
# 查看当前有哪些用户组
groups
# 把用户添加到libvirt用户组中
usermod -a -G libvirt $(whoami)
# 退出当前shell，并重连
exit

```

关闭SELINUX:

```bash
setenforce 0
# SELINUX=disabled
vi /etc/selinux/config
```
## 创建虚拟机

```bash
# 检查下iso文件的可读权限，需要给qemu用户可读 -rw-r--r--. 1 qemu qemu 1020264448 Nov 3 2020 ./rom-iso/CentOS-7-x86_64-Minimal-2009.iso
ls -l ./rom-iso/CentOS-7-x86_64-Minimal-2009.iso

# 在安装之前，需要把下载好的iso镜像mv到libvirt的images下面，不然会有权限问题。
ls -l /var/lib/libvirt/images/

virt-install \
--name guest1-rhel7 \
--memory 2048 \
--vcpus 2 \
--disk size=8 \
--graphics none \
--console pty,target_type=serial \
--extra-args "console=ttyS0" \
--location /var/lib/libvirt/images/CentOS-7-x86_64-Minimal-2009.iso \
--os-variant rhel7

virt-install \
--name ubuntu-server \
--memory 2048 \
--vcpus 2 \
--disk size=20 \
--location /var/lib/libvirt/images/ubuntu-20.04.6-live-server-amd64.iso \
--os-variant ubuntu20.04 \
--graphics none \
--console pty,target_type=serial \
--extra-args "console=ttyS0"
```

> 注释以上用的是--location，不是--cdrom，  --cdrom大部分的时候与--location等价，但是cdrom灭提默认不会通过文本控制台打印输出，就不会看到OS的TUI文本安装提示。 --location支持本地和网络位置(http等)。当然，如果你不是在headless环境中折腾KVM，是在有GUI的Linux上折腾，或者可以远程VNC连接到GUI的server中，那么就可以用cdrom

如果以上创建了已存在虚拟机，你还是重新创建一个同名的，那么删除或者关机就可以了:

```bash
# 查看所有VM运行状态
virsh list --all
# 关闭
virsh shutdown <vm-name>
# 如果关闭不掉，强行关闭(相当于硬件断电)
virsh destroy <vm-name>
# 删除虚拟机定义
virsh undefine <vm-name>
# 删除虚拟机的磁盘文件
rm /var/lib/libvirt/images/<vm-name>.qcow2
```


#### 安装完毕后

主要是TUI命令行的方式安装完成后，就可以输入用户名密码进入虚拟机中的Linux了，退出会话是`Ctrl + ]` 键， 如果要重连的话 `virsh console <vm-name>`, 然后就可以输入用户名和密码登录了。

#### 虚拟机的网络

默认的虚拟机网络是libvirt的NAT虚拟交换机。当然就是NAT的网络模式。也就是`virt-install` 未指定`--network`参数，如果显式指定就是`--network default`  


如果要弄成桥接(birdge)的网络模式，就要指定`--network br0`, 桥接模式意思就是虚拟机的网络与宿主机在一个网络里面，IP由DHCP分配。但是在运行`virt-install`之前，需要在宿主机上创建一个网桥(network bridge)， 然后`--network`才可以指定该网桥。

也可以给该网桥指定静态IP，不开启DHCP分配

```bash
--network br0 \
--extra-args "ip=192.168.1.2::192.168.1.1:255.255.255.0:test.example.com:eth0:none"
```

也可以让虚拟机没有网卡 `--network=none`

另外，在同一个宿主机上用KVM默认的NAT创建的虚拟机实例，其实例之间是在同一个内部网络中，也就是IP可以互通。当然，如果宿主机可以访问互联网，那么NAT模式下的虚拟机实例也可以访问互联网。

```bash
# 查看宿主机上所有定义的网络
virsh net-list --all
# 查看默认的一个网络，本质从看到的信息就是一个virbr0的桥接网络，内部网络的实例桥接在一起，设个virbr0实际上也充当一个NAT网关设备
virsh net-info default

```

## 管理运维

#### 管理实例

这里主要是管理虚拟机实例:

```bash
virsh list --all
# 关闭
virsh shutdown <vm-name>
# 如果关闭不掉，强行关闭(相当于硬件断电)
virsh destroy <vm-name>
# 删除虚拟机定义
virsh undefine <vm-name>
# 重启
virsh reboot <vm-name>
# 开机
virsh start <vm-name>
```

#### 导入其他虚拟机

从磁盘上导入

```bash
virt-install \
--name new-rh2l7 \
--memory 2048 \
--vcpus 2 \
--disk  /var/lib/libvirt/images/kk.qcow2 \
--import \
--os-variant rhel7 \
--graphics none \
--console pty,target_type=serial
```

然后就可以用户名密码登录了。其实以上也就是复用了一个其他的虚拟机镜像，制作镜像模板的时候有用。

#### 克隆虚拟机

相当于完成OS安装后，制作一个虚拟机模板了，这个模板拿来可以直接复用，启动。一些无论是公有云还是私有云都会用到

[Chapter 4. Cloning Virtual Machines | Red Hat Product Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/7/html/virtualization_deployment_and_administration_guide/cloning_virtual_machines#Cloning_virtual_machines)


## KVM半虚拟化驱动

半虚拟化驱动程序增强了客户机的性能，减少了客户机的 I/O 延迟，并将吞吐量提高到几乎到裸机的水平。建议将半虚拟化驱动程序用于运行 I/O 密集型任务和应用程序的完全虚拟化客户机。

Virtio 驱动程序是 KVM 的半虚拟化设备驱动程序，可用于在 KVM 主机上运行的客户机虚拟机。这些驱动程序包含在软件包中。virtio 软件包支持块（存储）设备和网络接口控制器。

[Chapter 5. KVM Paravirtualized (virtio) Drivers | Red Hat Product Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/7/html/virtualization_deployment_and_administration_guide/chap-kvm_para_virtualized_virtio_drivers#chap-KVM_Para_virtualized_virtio_Drivers)

## 网络配置

[Chapter 6. Network Configuration | Red Hat Product Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/7/html/virtualization_deployment_and_administration_guide/chap-network_configuration#chap-Network_configuration)

这里主要介绍基于libvirt的客户机虚拟机使用的常见网络配置。

Red Hat Linux企业版7支持以下虚拟化网络设置。

- 使用NAT的虚拟化网络
- 使用PCI设备分配技术直接分配物理设备
- 使用PCIE SR-IOV直接分配虚拟化功能
- 桥接网络(bridge network)

你必须开机NAT, 网络桥接或者直接分配一个PCI设备，这样才可以允许外部主机访问虚拟机实例中的网络服务。

#### libvirt的NAT

共享网络连接的最常见方法之一是使用网络地址转换 （NAT） 转发（也称为虚拟网络）。

##### 主机配置

每个标准的libvirt安装提供虚拟机之间的NAT连接，也就是默认的虚拟网络，你可以运行以下看到

```bash
virsh net-list --all

Name State Autostart Persistent  
----------------------------------------------------------  
default active yes yes
```

你会看到一个default的默认网络，这个默认网络定义在主机目录`/etc/libvirt/qemu/networks/default.xml`

```xml
<!--  
WARNING: THIS IS AN AUTO-GENERATED FILE. CHANGES TO IT ARE LIKELY TO BE  
OVERWRITTEN AND LOST. Changes to this xml configuration should be made using:  
virsh net-edit default  
or other application using the libvirt API.  
-->  
  
<network>  
	<name>default</name>  
	<uuid>088e69d5-663f-41ef-94d2-4587326aacfa</uuid>  
	<forward mode='nat'/>  
	<bridge name='virbr0' stp='on' delay='0'/>  
	<mac address='52:54:00:9a:86:6e'/>  
	<ip address='192.168.122.1' netmask='255.255.255.0'>  
		<dhcp>  
			<range start='192.168.122.2' end='192.168.122.254'/>  
		</dhcp>  
	</ip>  
</network>
```

可以看到，默认的就是NAT转发的模式，在同一主机上新建的虚拟机实例，默认就是连接到这个网络中，这个网络是通过一个叫`virbr0`的虚拟网桥与主机进行通信的。可以看到DHCP对这个网络的分配的IP范围， 也就是默认的虚拟机实例分配的IP地址，是从122这个地址段分配的。注意，这个文件你是不能手工编辑的，需要`virsh net-edit default` 来编辑。

当然这个目录下还有相关的客户虚拟的配置:

```bash
[root@localhost ~]# ll /etc/libvirt/qemu/  
total 8  
-rw------- 1 root root 3807 Jun 27 01:33 guest1-rhel7.xml  
drwx------. 3 root root 42 Jul 3 06:06 networks  
-rw------- 1 root root 3435 Jun 27 02:37 new-rh2l7.xml  
[root@localhost ~]# virsh list --all  
setlocale: No such file or directory  
Id Name State  
----------------------------------------------------  
16 new-rh2l7 running  
17 guest1-rhel7 running
```

你可以看到，这些以虚拟机实例名称的xml文件就是虚拟机实例配置，可以看到实例的配置里面也指定了这个实例使用的网络:

```xml
<interface type='network'>  
	<mac address='52:54:00:27:e7:30'/>  
	<source network='default'/>  
	<model type='virtio'/>  
	<address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>  
</interface>
```


好了，我们回过头来看libvirt管理的网络

```bash
# 让指定网络自动开启
virsh net-autostart <net-name>
# 手动开启指定网络
virsh net-start <net-name>
# 查看网桥设备,就可以看到虚拟网路的桥接设备
brctl show
```

> 好了，到现在，我们了解了virbr0这个虚拟的网桥设备，它是用来桥接宿主机和虚拟机实例网络的。所以你可以从`brctl show`中显示出来的interfaces那列看到挂载到这个网桥设备有哪些网卡，一般是一个宿主机的虚拟网卡`virbr0-nic`, 还有一些虚拟机实例的虚拟网卡 `vnet0` `vnet1` 这样就把虚拟机和宿主机的网络打通了，毕竟虚拟机里面连通外部网络，还是需要经过宿主机的，它没有意念神波来传递信息。

> 当然`vnet0` `vnet1`的虚拟网卡配置你可以通过`ip addr show dev <interface-name>`来看到，比如它的MAC地址，这个MAC地址你可以在虚拟机实例的XML配置的看到。注意网桥设备只是网卡的桥接器，它转发的数据包是根据MAC地址转发的，它类似于交换机，它工作在二层，所以它有交换机类的STP(生成树)协议。

libvirt会把iptables的INPUT FORWARD OUTPUT POSTROUTING chain上的出口入口的流量规则Apply到`virbr0` 这个虚拟的网桥设备上。  libvirt会试图开启Linux的`ip_forward`参数，一些系统关闭了，所以最好还是自己手动修改`/etc/sysctl.conf`的配置:

```conf
net.ipv4.ip_forward = 1
```

使配置生效 `sudo sysctl -p` 

> 这个文件是一个Linux系统的配置文件，用于在系统启动时设置内核参数。通过编辑这个文件，管理员可以调整和优化系统的运行参数，以适应不同的使用需求和提高系统的性能和安全性。

> 这个配置项，用于控制Linux系统是否允许IP转发。IP转发是指在两个网络接口之间转发IP数据包的能力，这对于路由器和其他网络设备来说是必要的功能。0是默认值，1是开启。关闭时，系统不会转发收到的数据包，即它不允许网络之间的通信，这适用于大多数单一网络接口的主机。打开时，系统会转发收到的数据包，允许不同网络之间的通信，这对于执行路由功能的设备来说是必要的，比如作为网关的服务器或者运行KVM虚拟机的宿主机，需要转发来自虚拟机的流量到外部网络。

##### 客户虚拟机配置

一旦主机配置完毕，那么客户虚拟机就可以通过网络名字，连接到这个网络上了。为了让客户虚拟机连接到这个`default`的默认NAT网络，以下的配置项就在客户虚拟机的配置文件中:

```xml
<interface type='network'>
   <source network='default'/>
</interface>
```

> 定义 MAC 地址是可选的。如果未定义 MAC 地址，则会自动生成一个 MAC 地址，并将其用作网络使用的桥接设备的 MAC 地址。手动设置 MAC 地址可能有助于在整个环境中保持一致性或易于引用，或者避免发生冲突的可能性很小。这里的MAC地址是虚拟机的MAC地址。

```xml
<interface type='network'>
  <source network='default'/>
  <mac address='00:16:3e:1a:b3:4a'/>
</interface>
```

#### 关闭vhost-net

`vhost-net`该模块是 virtio 网络的内核级后端，通过将 virtio 数据包处理任务移出用户空间（QEMU 进程）并移入内核（驱动程序）来减少虚拟化开销。vhost-net 仅适用于 Virtio 网络接口。如果加载了 vhost-net 内核模块，则默认情况下会为所有 virtio 接口启用它，但如果使用 vhost-net 时特定工作负载的性能下降，则可以在接口配置中禁用该模块。

具体而言，当 UDP 流量从主机发送到该主机上的客户机虚拟机时，如果客户机虚拟机处理传入数据的速度慢于主机发送数据的速度，则可能会发生性能下降。在这种情况下，启用vhost-net会导致 UDP 套接字的接收缓冲区更快地溢出，从而导致更大的数据包丢失。因此，在这种情况下最好禁用vhost-net以减慢流量并提高整体性能。

为了关闭vhost-net，需要编辑客户虚拟机对应的XML配置，在`<interface>`标签中:

```xml
<interface type="network">
   ...
   <model type="virtio"/>
   <driver name="qemu"/>
   ...
</interface>
```

需要设置driver name为`qemu`， 这样就可以强制网络包的处理放在QEMU的用户空间中，这样其实就是为这个虚拟机上的网络接口关闭了vhost-net功能。

#### 打开vhost-net zero-copy

在 Red Hat Enterprise Linux 7 中，vhost-net zero-copy 默认处于禁用状态。永久启用此操作需要在`/etc/modprobe.d`中添加一个`vhost-net.conf`文件

```conf
options vhost_net  experimental_zcopytx=1
```

```bash
modprobe -r vhost_net
modprobe vhost_net
```

如果你要再次关闭这个配置:

```bash
# 卸载这个模块
modprobe -r vhost_net
# 再次以这个参数，安装该模块
modprobe vhost_net experimental_zcopytx=0
```

确认配置是否生效:

```bash
cat /sys/module/vhost_net/parameters/experimental_zcopytx
```


#### 桥接网络(Bridged Networking)

桥接网络(也称为网络桥接技术或者虚拟网络交换技术)，主要是用来把同一个宿主机上的虚拟机的网卡都放在与物理网卡所在的相同网络中。桥接仅仅只需要少量的配置就可以让一个虚拟机实例出现在一个已经存在的网络上，这样降低了管理负担和网络复杂性。因为桥接只包含少量的组件和配置项，它们提供了一个透明的配置项，如果需要，可以简单理解和排除故障。

桥接可以配置在一个虚拟环境中，使用Red Hat企业版的Linux工具，virt-manager或者libvirt和一下小结描述的工具。

然后，即使在虚拟环境中，桥接也可以使用宿主操作系统的网络工具轻易创建。

##### 在Red Hat企业版Linux 7的宿主机中配置桥接网络

可以为 Red Hat Enterprise Linux 主机上的虚拟机配置桥接网络，而与虚拟化管理工具无关。当虚拟化网桥是宿主机的唯一网络接口或主机的管理网络接口时，主要建议使用此配置。参考 [Chapter 9. Configure Network Bridging | Red Hat Product Documentation](https://docs.redhat.com/en/documentation/Red_Hat_Enterprise_Linux/7/html/Networking_Guide/ch-Configure_Network_Bridging.html#ch-Configure_Network_Bridging)

##### 用virt-manager来配置桥接网络

主要是用virt-manager来配置一个从宿主机的网卡到客户虚拟机的一个网桥。

> 根据您的环境，在 Red Hat Enterprise Linux 7 中使用 libvirt 工具设置网桥可能需要禁用 Network Manager，Red Hat 不建议这样做。使用 libvirt 创建的网桥还要求运行 libvirtd 才能保持网络连接。 建议按照[《Red Hat Enterprise Linux 7 网络指南](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/Networking_Guide/ch-Configure_Network_Bridging.html)》中所述在物理 Red Hat Enterprise Linux 主机上配置桥接网络，同时在创建桥接后使用 libvirt 将虚拟机接口添加到网桥。

[6.4. Bridged Networking | Red Hat Product Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/7/html/virtualization_deployment_and_administration_guide/sect-network_configuration-bridged_networking#sect-Network_configuration-Bridged_networking)

##### 用libvirt来配置桥接网络

同上，因为virt-manager也是libvirt的GUI，所以本质是一样的。


> libvirt 现在能够利用新的内核可调参数来管理主机网桥转发数据库 （FDB） 条目，从而有可能在桥接多个虚拟机时提高系统网络性能。在主机的 XML 配置文件中将网络_`的 <bridge>`_ 元素的 _`macTableManager`_ 属性设置为：`'libvirt'`

```xml
<bridge name='br0' macTableManager='libvirt'/>
```

> 这将关闭所有网桥端口上的学习（泛洪）模式，libvirt 将根据需要向 FDB 添加或删除条目。除了消除学习MAC地址的正确转发端口的开销外，这还允许内核在将网桥连接到网络的物理设备上禁用混杂模式，从而进一步降低开销。


#### 网络防火墙

> libvirt创建出来的KVM虚拟机实例，连接的是默认的default网络。宿主机和虚拟机实例网落ping可以互通。

宿主机向虚拟机发起的流量防火墙示例， 注意，宿主机对虚拟机发出的请求，流量是不会通过FORWARD Chain的，走的是OUTPUT Chain。FORWARD Chain是不生效的。当然即使走OUTPUT Chain，流量始终要经过virbr0虚拟网桥设备， `ip link set dev virbr0 down`把网桥停止掉，就不能ping通了，重新开启`ip link set dev virbr0 up`又可以通了。

```bash
# 禁止宿主机自身向某个虚拟机的IP发起ping命令,宿主机
iptables -I OUTPUT -p icmp --icmp-type echo-request -d 192.168.122.167 -j DROP
# 查看OUTPUT的所有规则
iptables -L OUTPUT -n --line-numbers
# 删除OUTPUT链的某条规则
iptables -D OUTPUT <line-number>
```

在网桥模式下，宿主机充当一个虚拟交换设备，同时也开启了`ip_forwarding=1`的转发配置, 这个更接近真实环境，比如，宿主机A的IP是192.168.13.6 宿主机B是192.168.13.7 主机A上有两个libvirt KVM启动的虚拟机实例，IP分别是192.168.122.5 192.168.122.8 。如果是宿主机A上的ip_forwarding开启，在宿主机B上ping某个虚拟机实例的IP

```bash
# 在宿主机B上执行，添加路由规则，首先让宿主机B知道，122网段上哪台机器上的路由去找
sudo ip route add 192.168.122.0/24 via 192.168.13.6
# 在宿主机A上，允许FORWARD链中的流量，也就是让宿主机A转发，eth0是宿主机A的物理网卡设备，也就是流量输入端，virbr0是宿主机上的一个虚拟网桥设备，这个设备上挂着虚拟机的虚拟网卡，也就是对于FORWARD来说的所有流量都无脑转发到virbr0上
sudo iptables -A FORWARD -i eth0 -o virbr0 -j ACCEPT
```

还可以通过virsh，请参考 [libvirt: Firewall and network filtering in libvirt](https://libvirt.org/firewall.html)
#### 虚拟机的磁盘管理

```bash
# 先进入虚拟机实例内部，看一下磁盘设备，你会看到vda的虚拟存储设备
fdisk -l
```

给虚拟机实例里面加入一个数据盘, 需要先创建一个存储池

> 存储池是虚拟化环境中用于管理存储资源的一个概念。它是一组存储卷的集合，可以是本地硬盘、网络共享存储（如NFS或iSCSI）或其他类型的存储设备。存储池提供了一种方便的方式来组织、分配和管理虚拟机的存储资源。

```bash
# 在宿主机上创建一个存储池的目录
mkdir libvirt-vol-pool  
# 在这个存储池目录上真正定义个叫my-pool的存储池,包括其类型（如目录、NFS、iSCSI等）、位置和其他相关设置。
virsh pool-define-as --name my-pool --type dir --target /root/libvirt-vol-pool
# 手动启动存储池
virsh pool-start my-pool 
# 让这个存储池在宿主机开机也自动启动
virsh pool-autostart my-pool
```

> - 在创建存储池之前，请确保你有足够的权限来访问和修改指定的存储资源。
> - 存储池的类型和配置选项可能会根据其类型（如NFS、iSCSI等）而有所不同。请根据你的具体需求选择合适的类型和配置。


```bash
# 在这个之前创建的存储池中上创建一个10G的存储卷(volume)
virsh vol-create-as --pool my-pool --name new-data-disk.img --format qcow2 --capacity 10G
# 查看一下这个存储卷
virsh vol-list --pool my-pool
# 确定存储卷的路径
virsh vol-path --pool my-pool new-data-disk.img
```

```bash
# 把存储池中的存储卷修改成qemu或libvirt-qemu用户能够访问，不然以下挂载的时候会报错
chown qemu:qemu /root/libvirt-vol-pool/new-data-disk.img
# 把存储卷作为磁盘挂载进虚拟机实例,vdb是这个虚拟机内部的设备名，也就是你这个磁盘设备即将在虚拟机里面识别的一个设备名，之前存在一个vda，所以这里我们需要挂载成vdb，以免冲突，以此类推，vdc，vdd
virsh attach-disk <vm-name> --source /root/libvirt-vol-pool/new-data-disk.img --target vdb --persistent
```

如果以上自定义的存储卷不行，那么就在默认存储池中创建存储卷:

```bash
# 查看当前存储池，发现default是激活运行中的
virsh pool-list
# 查看default的详细信息
virsh pool-info default
# 查看defaut存储池的路径, 一般来说，是/var/lib/libvirt/images/
virsh pool-dumpxml default | grep 'path'
# 在默认的存储池中创建存储卷，你会发现，存储池中的目录下多出一个new-data-disk.img的镜像文件
virsh vol-create-as --pool default --name new-data-disk.img --format qcow2 --capacity 10G
# 挂载到指定虚拟机实例
virsh attach-disk <vm-name> --source /var/lib/libvirt/images/new-data-disk.img --target vdb --persistent
# 进入到虚拟机实例中，查看存储设备,可以看到这个设备其实并没有10个G，是因为qcow2的格式是稀疏分配，它不会立即分配10G的物理磁盘给这个文件，所以看到是，实际的数据卷文件的大小，这是不应该的。
virsh console <vm-name>
fdisk -l
# 退出虚拟机，查看数据卷文件，可以看到virtual size是10G，Disk Size却是很小。
qemu-img info /var/lib/libvirt/images/new-data-disk.img
# 重新创建一个raw格式的存储卷
virsh vol-create-as --pool default --name new-data-raw-disk.img --format raw --capacity 2G
# 挂载一个raw格式的存储卷作为vdc
virsh attach-disk <vm-name> --source /var/lib/libvirt/images/new-data-raw-disk.img --target vdc --persistent
# 进入虚拟机查看磁盘设备, 这下可以看到 Disk /dev/vdc: 2147 MB, 2147483648 bytes, 4194304 sectors
virsh console <vm-name>
fdisk -l
# 退出虚拟机
```

卸载磁盘设备

```bash
# 如果虚拟机正在运行的，必须要加入--live参数，不然就得重启虚拟机，才会生效
virsh detach-disk <vm-name> --target vdc --live --config
virsh detach-disk <vm-name> --target vdb --live --config

# 如果虚拟机已关机，必须去掉--live
virsh detach-disk <vm-name> --target vdb  --config
```

另外，需要注意就是，如果虚拟机里面没有/dev/vdb的设备，你挂载指定的target是vdc， 那么你进去虚拟机里面查看，其实挂载成为了/dev/vdb，其实就是按照abcd顺序来的，b没有的情况下，不能有c。

## KVM的超额使用

KVM 虚拟机管理程序会自动过量使用 CPU 和内存。这意味着可以分配给虚拟机的虚拟化 CPU 和内存比系统上的物理资源多。这是可能的，因为大多数进程不会始终访问其分配的资源的 100%。

因此，未充分利用的虚拟化服务器或桌面可以在更少的主机上运行，从而节省大量系统资源，从而减少功耗、冷却和服务器硬件投资。

[Chapter 7. Overcommitting with KVM | Red Hat Product Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/7/html/virtualization_deployment_and_administration_guide/chap-overcommitting_with_kvm#sect-Overcommitting_with_KVM-Introduction)


## 管理

[Part II. Administration | Red Hat Product Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/7/html/virtualization_deployment_and_administration_guide/part-administration)




## 参考资料

[Virtualization basics and an introduction to KVM (mkdev.me)](https://mkdev.me/posts/virtualization-basics-and-an-introduction-to-kvm)

[digitalocean/go-libvirt: Package libvirt provides a pure Go interface for interacting with Libvirt. Apache 2.0 Licensed. (github.com)](https://github.com/digitalocean/go-libvirt)
[virt-manager/virt-manager: Desktop tool for managing virtual machines via libvirt (github.com)](https://github.com/virt-manager/virt-manager)

[KVM/Networking - Community Help Wiki (ubuntu.com)](https://help.ubuntu.com/community/KVM/Networking)

[Open vSwitch with KVM — Open vSwitch 3.3.90 documentation](https://docs.openvswitch.org/en/latest/howto/kvm/)
[libvirt: virsh](https://www.libvirt.org/manpages/virsh.html)
[libvirt: Network XML format](https://libvirt.org/formatnetwork.html#nat-based-network)

[libvirt: Firewall and network filtering in libvirt](https://libvirt.org/firewall.html)

