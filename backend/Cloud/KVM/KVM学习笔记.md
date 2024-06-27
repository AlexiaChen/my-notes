
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
