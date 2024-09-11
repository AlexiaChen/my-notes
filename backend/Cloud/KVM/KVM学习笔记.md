
## 准备

> 准备在KVM中安装的镜像下载 [Index of /pub/centos-altarch/7.9.2009/isos/aarch64 (utah.edu)](https://mirror.chpc.utah.edu/pub/centos-altarch/7.9.2009/isos/aarch64/)   [Index of /pub/centos/7.9.2009/isos/x86_64 (utah.edu)](https://mirror.chpc.utah.edu/pub/centos/7.9.2009/isos/x86_64/)


```bash
grep -E 'svm|vmx' /proc/cpuinfo
```

以上命令主要查看CPU支不支持硬件虚拟化技术，vmx是Intel-VT。  svm是AMD-V。有一些本身支持，但是查看不到，可能是因为在BIOS里面把这个开关关闭了，需要在BIOS里面打开。

```
lsmod | grep kvm
```

以上是查看KVM内核模块是否被加载进内核了。如果有kvm_intel  kvm_intel的输出，那就说明KVM硬件虚拟化模块被加载进内核了。

> 以上其实就是查看软硬件环境是否满足，有了硬件还不够，需要软件来驱动硬件工作


如果你用的是windows 11上的WSL2，那么好消息，告诉你，这里是直接支持套娃式虚拟化的。

在你的C:/Users/\<user\>目录下编辑全局的.wslconfig文件

```txt
# Disables nested virtualization,默认是关闭的，添加打开即可 
nestedVirtualization=true
```

然后在你的虚拟机里面的Linux的`/etc/wsl.conf`的配置文件里面加入:

```txt
[boot]
command = /bin/bash -c 'chown -v root:kvm /dev/kvm && chmod 660 /dev/kvm'
```

重启WSL2

[hyper v - How to run KVM nested in WSL2 (or vmware)? - Server Fault](https://serverfault.com/questions/1043441/how-to-run-kvm-nested-in-wsl2-or-vmware)

最后进入虚拟机中运行`kvm-ok`, 如果输出如下:

```txt  
INFO: /dev/kvm exists  
KVM acceleration can be used
```

就可以了。
## 安装

```
# CentOS
yum install qemu-kvm libvirt
# ubuntu
sudo apt install -y qemu-kvm libvirt-daemon-system
```

以上命令其实会安装三个包，qemu-kvm是用户态的KVM模拟器，实现客户虚拟机与宿主机的通信并且模拟CPU和外设，qemu-img实现客户虚拟机的磁盘管理，libvirt其实是一个标准的虚拟化API实现，已经被广泛采用，是一个库，也可以作为一个服务，主要是提供宿主机与Hypervisor的通信，这里的Hypervisior是KVM本身。

```
# CentoS
yum install virt-install libvirt-python virt-manager  libvirt-client bridge-utils
# ubuntu
sudo apt install -y virt-manager  virtinst libvirt-clients bridge-utils
```

以上主要是安装libvirt库服务的客户端工具，方便用户操作。virt-install用于创建虚拟机。libvirt-python提供给Python的libvirt API。virt-manager，通过libvirt-client的API来间接通过libvirt操作虚拟机的管理工具，有UI图形界面。libvert-client就是libvirt的API和附带的命令行来实现管理虚拟机的通信。bridge-utils是管理桥接设备的

因为libvirt它一般是以server的方式提供，所以要启动它的守护进程

```
# CentOS
# 查看当前有哪些用户组
groups
# 把用户添加到libvirt用户组中
usermod -a -G libvirt $(whoami)
# 退出当前shell，并重连
exit

# ubuntu
sudo usermod -aG kvm $USER
sudo usermod -aG libvirt $USER
```

关闭SELINUX:

```bash
# CentOS
setenforce 0
# SELINUX=disabled
vi /etc/selinux/config
```

```bash
# centos or ubuntu
systemctl start libvirtd  
systemctl enable libvirtd
systemctl status libvirtd
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

# 如果是在ARM64下虚拟化x86的OS
virt-install \
--name guest1-rhel7 \
--memory 2048 \
--vcpus 2 \
--disk size=8 \
--graphics none \
--console pty,target_type=serial \
--extra-args "console=ttyS0" \
--location /var/lib/libvirt/images/centos7-aarch64.iso \
--os-variant rhel7.0 

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
# 一些新的版本需要这样删除
virsh undefine --nvram <vm-name>
# 删除虚拟机的磁盘文件
rm /var/lib/libvirt/images/<vm-name>.qcow2
```


#### 安装完毕后

主要是TUI命令行的方式安装完成后，就可以输入用户名密码进入虚拟机中的Linux了，退出会话是`Ctrl + ]` 键， 如果要重连的话 `virsh console <vm-name>`, 然后就可以输入用户名和密码登录了。

#### 虚拟机的网络

默认的虚拟机网络是libvirt的NAT虚拟交换机。当然就是NAT的网络模式。也就是`virt-install` 未指定`--network`参数，如果显式指定就是`--network default`  

> 注意，按理来说使用默认的网络，其实是NAT模式，也就是宿主机上的虚拟机实例之间如果用这个网络其实是一个子网，由DHCP分配，但是有些时候，虚拟机登录进去以后，ping不通百度，访问不了互联网，这是因为VM里面的网卡配置出了点问题，ip addr看不到eth0有子网IP，需要登录到对应的虚拟机里面，修改网卡配置`vi /etc/sysconfig/network-scripts/ifcfg-eth0`  这三个配置项很重要： 
> DEVICE=eth0
 BOOTPROTO=dhcp
 ONBOOT=yes
 我发现ONBOOT是no，修改成yes就好了，保存。重启VM，virsh shutdown \<vm\> virsh start \<vm\>  这样再登录进去，就可以ip addr看到网卡的IP了，这下测试也可以访问互联网了。 NAT模式，VM可以ping通宿主机的IP，宿主机也可以ping通虚拟机，虚拟机之间当然也可以相互ping通（虽然他们之间是不同网段的）， 


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

> 在WSL2的ubuntu 22.04启动default网络报错: error: internal error: Failed to apply firewall rules /usr/sbin/iptables -w --table filter --list-rules: iptables v1.8.7 (nf_tables): table `filter' is incompatible, use 'nft' tool.  
> as
> 其实可以把iptables的后端替换成nft的：
sudo update-alternatives --set  iptables /usr/sbin/iptables-nft  
sudo update-alternatives --set ip6tables /usr/sbin/ip6tables-nft  
sudo update-alternatives --set arptables /usr/sbin/arptables-nft  
sudo update-alternatives --set ebtables /usr/sbin/ebtables-nft
sudo systemctl restart libvirtd
sudo systemctl enable nftables
sudo systemctl start nftables
sudo virsh net-start default
asasa
如果还有其他一个叫dnsmasq的socket already use报错，那么其实就是你是在用虚拟机里面做KVM虚拟化，所以宿主机这个虚拟机占用了这个，你得自己建造一个网桥，然后virsh-install的时候，指定这个网桥设备。
sudo brctl addbr mybridge
sudo brctl addif mybridge eth0
sudo brctl addif mybridge eth1
sudo ip link set dev mybridge up
创建一个network的xml定义:

```xml
 <network>
  <name>mybridge</name>
  <forward mode='bridge'/>
  <bridge name='mybridge'/>
</network>
```

> sudo virsh net-define mybridge.xml
> sudo virsh net-start mybridge
> sudo virsh net-autostart mybridge
> ip link show mybridge

```bash
# WSL2中的ubuntu 22.04 安装centos7系统，安装到最后，WSL2会崩溃，所以这个方案没做继续尝试了
virt-install \
--name guest1-rhel7 \
--memory 2048 \
--vcpus 2 \
--disk size=8 \
--graphics none \
--console pty,target_type=serial \
--extra-args "console=ttyS0" \
--location /var/lib/libvirt/images/CentOS-7-x86_64-Minimal-2009.iso \
--os-variant rhl7 --network bridge=mybridge
```



安装成功以后，请关机，开机就可以连接了:

```bash
virsh shutdown <vm-name>
virsh start <vm-name>
virsh console <vm-name>
# 输入用户名密码就可以连接进去了
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
# 找到虚拟机对应的镜像
virsh dumpxml guest1-rhel7 | grep qcow2

virt-install \
--name new-rh2l7 \
--memory 2048 \
--vcpus 2 \
--disk  /var/lib/libvirt/images/centos7_aarch64_base.qcow2 \
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


在网桥模式下，宿主机充当一个虚拟交换设备，同时也开启了`ip_forwarding=1`的转发配置, 这个更接近真实环境，比如，宿主机A的IP是192.168.13.6 宿主机B是192.168.13.7 主机A上有两个libvirt KVM启动的虚拟机实例，IP分别是192.168.122.5 192.168.122.8 。如果是宿主机A上的ip_forwarding开启，在宿主机B上ping某个虚拟机实例的IP

```bash
# 在宿主机B上执行，添加路由规则，首先让宿主机B知道，122网段上哪台机器上的路由去找
sudo ip route add 192.168.122.0/24 via 192.168.13.6
# 在宿主机A上，允许FORWARD链中的流量，也就是让宿主机A转发，eth0是宿主机A的物理网卡设备，也就是流量输入端，virbr0是宿主机上的一个虚拟网桥设备，这个设备上挂着虚拟机的虚拟网卡，也就是对于FORWARD来说的所有流量都无脑转发到virbr0上
sudo iptables -A FORWARD -i eth0 -o virbr0 -j ACCEPT
```

#### 网络防火墙和网络过滤器


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

下面，正式的要开始了。

有三种类型的libvirt的功能可以来做某些类型的网络流量过滤，从高层次上来看，它们有:

- 虚拟网络驱动。这个提供了一个独立隔离的网桥设备（就是没有任何物理网卡挂载这个网桥上），客户虚拟机的TAP设备都挂载到这个桥接设备上。客户虚拟机之间可以互通，也可以跟宿主机互通。当然，甚至可以与外部互联网通信（看宿主机上没上网）。
- QEMU驱动程序MAC过滤。这个提供了对MAC地址的通用过滤，以防止客户虚拟机伪造其MAC地址。这基本上被下一项所淘汰。
- 网络过滤器驱动。这个提供了全部的可配置化，在客户虚拟机的网卡上可以实现流量的任意网络过滤。通用规则集在宿主机上定义，用于控制以某种方式进行的流量。然后规则集会关联到一个客户虚拟机的独立网卡上。虽然不如直接使用iptables/etables那么有表现力，但是仍然可以在客户虚拟机的网卡上执行几乎所有你想要执行的网络过滤

##### 虚拟网络驱动

客户虚拟机的典型配置是在主机上将客户虚拟机直接连接到宿主机所在的局域网。在 RHEL6 中，还有 可以使用 macvtap/sr-iov 和 VEPA 连接。这些东西都不是与无线网卡配合得很好，因为它们通常会静默地丢弃任何MAC地址与物理网卡MAC地址不匹配的流量。**这个其实就是最经典最传统的桥接模式，把虚拟机的虚拟网卡直接桥接到物理网卡。**

> 以上的桥接模式一般在windows下的Hyper-V，还有VMware  VirtualBox都可以见到

所以libvirt的虚拟网络驱动被发明了，对传统的桥接模式做了改进。它其实是一个独立的桥接设备（没有任何物理网卡挂在这个桥上）。客户虚拟机的TAP设备都挂载到这个桥接设备上。这样就可以立即让在同一台宿主机上的客户虚拟机可以相互通信，然后也可以与宿主操作系统相互通信（宿主机iptables的规则）。

> TAP设备是一种虚拟网络内核接口，在Linux和其他类Unix操作系统中用于网络虚拟化。它表现为一个虚拟网络设备，工作在数据链路层，可以处理以太网帧。TAP设备常用于虚拟化环境中，允许虚拟机直接发送和接收以太网帧，就好像它们是连接到物理网络的一部分。
> 
> TAP设备常用于创建虚拟网络，使得虚拟机或容器可以拥有自己的网络接口，仿佛它们是连接到实际的物理网络上。
> 
> 通过将TAP设备桥接到物理网络接口，虚拟机可以直接出现在物理网络上，拥有自己的IP地址，就像物理机一样。
> 
> TAP设备也可以用于创建网络隧道和VPN连接，允许远程网络或设备安全地连接和交换数据。

然后，libvirt使用iptables来控制可用的进一步连接，目前对于一个虚拟网络，下面有三种配置可以配置:

- 隔离(isolated): 所有的绕过主机的流量(off-node traffic)完全被阻止。 off-node traffic其实就是虚拟机实例可以直接把数据发到网络上，根本不通过宿主机内核。
- 网络地址转换(nat): 从虚拟机实例出去到宿主机的局域网流量被允许，还启动NAT伪装，相当于能共用一个出口IP。NAT伪装其实就是把数据包的source IP替换为防火墙的IP。
- 转发(forward): 从虚拟机实例出去到宿主机局域网的流量全部被允许，没有例外。

forward的情况要求虚拟网络相对于宿主机的局域网在一个分离的子网上，然后这样的话，该局域网管理员就可以给这个子网配置路由。在未来，我们打算添加IP子网和ARP代理的支持。这个允许虚拟网络使用相同子网的情况下，像使用宿主机局域网一样，这样可以避免该局域网管理员来配置特殊的路由规则。

libvirt也会选择性提供DHCP服务给虚拟网络（使用DNSMASQ）。对于所有的case来说，我们需要允许虚拟机实例向宿主机OS上发起的DNS/DHCP请求，因此我们不能假设宿主机防火墙的设置规则允许了这些请求，所以我们无脑在宿主机的INPUT Chain的头部插入了以下四条规则:

```txt
target     prot opt in     out     source               destination
ACCEPT     udp  --  virbr0 *       0.0.0.0/0            0.0.0.0/0           udp dpt:53
ACCEPT     tcp  --  virbr0 *       0.0.0.0/0            0.0.0.0/0           tcp dpt:53
ACCEPT     udp  --  virbr0 *       0.0.0.0/0            0.0.0.0/0           udp dpt:67
ACCEPT     tcp  --  virbr0 *       0.0.0.0/0            0.0.0.0/0           tcp dpt:67
```

> 上面的四条规则中，in 设备是virbr0，所以方向是出口方向，也就是虚拟机向宿主机的流量。

下面的规则依赖于连接的类型，规则在宿主机的FORWARD chain中:

- isolated类型， 允许虚拟机实例之间的流量，但是阻止入方向和出方向。也就是不允许虚拟机访问宿主机，不允许宿主机访问虚拟机

```txt
target     prot opt in     out     source               destination
ACCEPT     all  --  virbr1 virbr1  0.0.0.0/0            0.0.0.0/0
REJECT     all  --  *      virbr1  0.0.0.0/0            0.0.0.0/0           reject-with icmp-port-unreachable
REJECT     all  --  virbr1 *       0.0.0.0/0            0.0.0.0/0           reject-with icmp-port-unreachable
```

- nat类型，允许已经建立连接的入流量，允许我们指定的子网的出流量。允许虚拟机实例之间的流量。拒绝其入流量。拒绝其他出流量。

```txt
target     prot opt in     out     source               destination
ACCEPT     all  --  *      virbr0  0.0.0.0/0            192.168.122.0/24    state RELATED,ESTABLISHED
ACCEPT     all  --  virbr0 *       192.168.122.0/24     0.0.0.0/0
ACCEPT     all  --  virbr0 virbr0  0.0.0.0/0            0.0.0.0/0
REJECT     all  --  *      virbr0  0.0.0.0/0            0.0.0.0/0           reject-with icmp-port-unreachable
REJECT     all  --  virbr0 *       0.0.0.0/0            0.0.0.0/0           reject-with icmp-port-unreachable
```

- routed类型。允许我们指定子网的入流量。允许我们指定子网的出流量。允许虚拟机实例之间的流量。拒绝其他所有入流量，拒绝其他所有出流量。

```txt
target     prot opt in     out     source               destination
ACCEPT     all  --  *      virbr2  0.0.0.0/0            192.168.124.0/24
ACCEPT     all  --  virbr2 *       192.168.124.0/24     0.0.0.0/0
ACCEPT     all  --  virbr2 virbr2  0.0.0.0/0            0.0.0.0/0
REJECT     all  --  *      virbr2  0.0.0.0/0            0.0.0.0/0           reject-with icmp-port-unreachable
REJECT     all  --  virbr2 *       0.0.0.0/0            0.0.0.0/0           reject-with icmp-port-unreachable
```

- 最后对于nat类型，其实在POSTROUTING Chain中还有一个规则项，主要是启用NAT伪装:

```txt
target     prot opt in     out     source               destination
MASQUERADE all  --  *      *       192.168.122.0/24    !192.168.122.0/24
```

##### firewalld和虚拟网络驱动

如果firewalld服务在宿主机上被启动激活，libvirt就试图把libvirt虚拟网络的网桥接口放到firewalld zone中，叫"libvirt"（这样就让在那个虚拟网络中的所有虚拟机到宿主机的流量，都会关联到firewall的规则）。 这样做是因为，如果 firewalld 正在使用其 nftables 后端（从 firewalld 0.6.0 开始可用） 默认 firewalld 区域（如果 libvirt 未显式设置 zone） 阻止来自访客虚拟机的流量通过网桥转发，以及 阻止 DHCP、DNS 和从客户机到主机的大多数其他流量。名为 “libvirt”由 libvirt 安装到 firewalld 配置中（不是由 firewalld），并允许通过网桥和 DHCP 转发流量， 到主机的 DNS、TFTP 和 SSH 流量 - 取决于 firewalld 的后端 将通过 iptables 或 nftables 规则实现。libvirt 自己的规则 上面概述的将*始终*是 iptables 规则，无论哪个后端是 由 firewalld 使用。

注意：可以手动设置网络接口的防火墙区域 替换为网络“bridge”元素的“zone”属性。

注意：在 libvirt 5.1.0 之前，firewalld “libvirt” 区域不存在，并且 在 firewalld 0.7.0 之前，这是使“libvirt”区域运行的关键功能 在 Firewalld 中未正确实现（丰富的规则优先级设置）。在 两个包中的一个或另一个缺少必要的情况 功能性，仍然可以通过以下方式拥有功能性访客网络 将 firewalld 后端设置为“iptables”（在 0.6.0 之前的 firewalld 中，这 是唯一可用的后端）。

##### 网络过滤器驱动

此驱动程序提供完全可配置的网络筛选功能，可 利用 EBTables、IPTABLES 和 IP6Tables。这是由 libvirt 团队写的 虽然它的 XML 模式是由 libvirt 定义的，但概念模型 与用于网络筛选的 DMTF CIM标准模式密切相关：[Visio-CIM_Network.vsd (dmtf.org)](https://www.dmtf.org/sites/default/files/cim/cim_schema_v2230/CIM_Network.pdf)

过滤器被作为一个单独的对象被libvirt上层管理，也就是对象对用户可见。这就允许过滤器可以被任何的libvirt对象引用，这样就可以使用过滤器提供的功能了，而不仅仅只是客户虚拟机的网卡才可以使用。对于现在的实现，过滤器可以通过libvirt的domain XML格式被关联到独立的虚拟机的网卡上。未来，我们可能让过滤器被关联到虚拟网络对象上，这样我们就可以期待定义一个新的网络交换对象了，消除了配置bridge/sriov/vepa等网络模式的复杂性。这样就让网络过滤器物尽其用了。

有一些新的virsh命令来管理网络过滤器:

- `virsh nwfilter-define`  通过一个XML文件定义或者更新一个网络过滤器
- `virsh newfilter-undefine` 删除一个网络过滤器
- `virsh nwfilter-dumpxml` XML定义的网路过滤器的可视化信息
- `virsh nwfilter-list`  列出现在的网络过滤器
- `virsh nwfilter-edit` 编辑一个网络过滤器的XML配置

当然，这些命令其实都有对应的C语言的API。

正如所有被libvirt管理的对象一样，网络过滤器也是用一个XML格式配置的。大概是这样:

```xml
<filter name='no-spamming' chain='XXXX'>
  <uuid>d217f2d7-5a04-0e01-8b98-ec2743436b74</uuid>

  <rule ...>
    ....
  </rule>

  <filterref filter='XXXX'/>
</filter>
```

每个过滤器都有UUID，一个过滤器有一个或者多个规则rule。过滤器之间可能会被组织成一个有向无环图(DAG),  所以过滤器之间靠filterref来引用，但是环路是不被允许的。

rule标签是过滤器的核心，所有的流量控制都是在这个里面定义的，并附有优先级

```xml
<rule action='drop' direction='out' priority='500'>
```

rule标签里面有有很多protocol，支持很多协议，ip，tcp，udp，icmp等等

```xml
<protocol match='yes|no' attribute1='value1' attribute2='value2'/>
```

比如以下就是一个protocol标签协议示例，匹配TCP的0-1023的源端口

```xml
<tcp match='yes' srcportstart='0' srcportend='1023'/>
```

上面的一些标签属性还可以直接引用标量，比如客户虚拟机的XML格式允许每个网卡有一个MAC地址和IP，这样就可以通过变量来引用这些IP地址和MAC地址

```xml
<filter name='no-ip-spoofing' chain='ipv4'>
  <rule action='drop' direction='out'>
    <ip match='no' srcipaddr='$IP' />
  </rule>
</filter>
```

libvirt提供了一些内建的有用的规则, 它们被存在 `/etc/libvirt/nwfilter`

```bash
virsh nwfilter-list
```

要让某个客户虚拟机引用指定的过滤器，请在客户虚拟机的XML配置中的interface标签下加入filterref标签，引用过滤器的名称:

```xml
<interface type='bridge'>
  <mac address='52:54:00:56:44:32'/>
  <source bridge='br1'/>
  <ip address='10.33.8.131'/>
  <target dev='vnet0'/>
  <model type='virtio'/>
  <filterref filter='clean-traffic'/>
</interface>
```


我用我机器上写一个防火墙的具体例子,  注意nwfilter只支持虚拟网卡的类型是network或bridge的。一般libvirt默认创建的虚拟机的网卡是network模式的, 而且在宿主机上必须要开启`firewalld`服务，特别是RedHat 7上。`systemctl start firewalld`

```bash
vi  /etc/libvirt/nwfilter/block-all.xml
```

XML的内容:

```xml
<filter name='block-all' chain='root' priority='-700'>  
	<uuid>3cef420c-3a7d-11ef-ad33-00ff4c51429a</uuid>  
	<rule action='drop' direction='in' priority='100'>  
		<all/>  
	</rule>  
	<rule action='drop' direction='out' priority='100'>  
		<all/>  
	</rule>  
</filter>
```

UUID需要自己生成，可以用Linux的uuid命令行。name就是nwfilter的名称。priority是优先级，值越小优先级越高。然后就是规则，以上的XML的规则就是禁止入方向的所有流量，这里的方向是in和out，in是进入虚拟机的流量，out是从虚拟机出去的流量。

```bash
# 根据一个XML文件定义或更新一个过滤器
virsh nwfilter-define /etc/libvirt/nwfilter/block-all.xml
# 查看当前的过滤器列表
virsh nwfilter-list
# 查看这个过滤器的配置
virsh nwfilter-dumpxml block-all
# 编辑虚拟机实例的配置文件， 在interface标签下加入filterref 标签 <filterref filter='block-all'/>
virsh edit <vm-name>
# 必须重启虚拟机才可以使防火墙生效 
virsh shutdown <vm-name>
virsh start <vm-name>
# 查看防火墙绑定在哪个虚拟网卡上
virsh nwfilter-binding-list
# ping虚拟机的IP就IP不通了
ping 192.168.122.167
# ping另一个虚拟机的IP，就可以ping通
ping 192.168.122.144
```

> 编辑完成虚拟机的配置，必须重启虚拟机才可以使防火墙生效，[networking - Apply Network Filter to Ovirt Guest without Rebooting - Server Fault](https://serverfault.com/questions/947407/apply-network-filter-to-ovirt-guest-without-rebooting) [networking - Apply libvirt nwfilter without editing of XML - Server Fault](https://serverfault.com/questions/479144/apply-libvirt-nwfilter-without-editing-of-xml)


以上的现象是正常的，必须重启虚拟机实例。如果你的filter已经关联到一个虚拟机上了，虚拟机重启完毕以后，你再对这个filter进行修改，增加删除rule标签，那么就不需要重启虚拟机实例了。但是生效会需要一些时间。

> 从以上的这些任何操作你可以看到，很多virsh的操作都是配合edit的子命令来编辑XML文件的，但是这些对bash自动化脚本不友好，对直接传输远程执行的命令集合也不好，所以最好的方式就是bash脚本的源码需要类似 dumpxml 出来一个bak的XML文件，然后通过sed  xmlstarlet等工具进行XML的编辑，然后再define更新，然后还要找到对应的虚拟机实例名称等等 这样才生效，比如防护墙Hyper-V上其实就一行Powershell命令就可以了，但是在libvirt这里就不行，要更复杂。

更多内容请看 [libvirt: Firewall and network filtering in libvirt](https://libvirt.org/firewall.html)

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
[nwfilter规则-CSDN博客](https://blog.csdn.net/weixin_34032621/article/details/92863471)
[基于network filter的虚拟机访问控制 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/70078069)

[17.14. Applying Network Filtering | Red Hat Product Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/7/html/virtualization_deployment_and_administration_guide/sect-virtual_networking-applying_network_filtering#sect-Advanced_Filter_Configuration_Topics-Limiting_Number_of_Connections)

