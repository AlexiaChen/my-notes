
# 1  介绍

  

OpenStack是热门的云计算IaaS开源软件，但是现在已经很多成员退出了，进入了衰退期，但是作为云计算领域的标杆开源项目，还是值得拿来研究的。比如国内的多家云厂商，其实早期，甚至至今，都是OpenStack的二次开发版。比如阿里云早期就用了OpenStack，但是经过他们内部随着对OPenStack的理解认知，目前现在肯定不是了。但是或多或少都有OpenStack的架构设计的影子在里面。

  

与阿里云相比入场云计算较晚的腾讯云，早期底层也是直接用的开源的OpenStack，据传也给OpenStack多了一定的二次开发。但是后期腾讯也自研了自己的TStack  IaaS软件，为了弥补OpenStack的不足。

  

UCloud据说是完全看不上OpenStack，因为创始人声称，目前在公有云领域，没有看到特别成功的OpenStack案例，因为OpenStack确实只是很多大厂拿来做自己公司的私有云，服务于内部的系统的。

  

华为云，华为入局云计算几乎最晚。所以直接采用了开源的OpenStack。fork了OpenStack的某个版本，自己维护一个自己的内部分支为公有云做支撑。并且是OpenStack的白金会员。因为单纯的开源OpenStack，很难完全面向公有云。

  

天翼云。央企中国电信入局的云计算品牌。其实就是买了华为云的底座，自然就是华为内部维护的OpenStack分支了。 可以说，根本上就是华为的产品，只是换了层皮。

  

从以上的介绍来看，对于公有云大厂来说，OpenStack对他们意味着是一个业界参考模型，他们无论是自研，还是二次开发，都有OpenStack的影子。为什么要自研和二次开发呢？其实本质还是直接用开源的OpenStack问题多多，OpenStack的开源社区现今也不太活跃了。然后就是OPenStack

并不适合用做公有云。  所以，OpenStack问题多多，然后又庞大复杂，不是大厂，直接拿开源的OpenStack用，运维，维护成本大，所以国内的信创（国内信创热潮之前很火）就自己做了一个IaaS的软件，有些口碑也还行，比如ZStack[云轴科技ZStack-产品化的云基础软件提供商](https://www.zstack.io/)，StarVCenter [StarVCenter (starvcs.com)](http://www.starvcs.com/)  这些小的IaaS软件厂商开发的IaaS软件也在政务云领域有出现，提供社区版，企业版等。

  

上面提到的OpenStack的场景大部分是大厂的私有云领域，非公有云领域。公有云是OpenStack的只有华为云和天翼云。

  

# 2 优势

  

- 尽管已经进入衰退期，但是开源社区相对其他IaaS软件还是比较大。业界元老标杆。
- 高可用，OpenStack支持高可用部署方式
- OpenStack后端虚拟化技术支持VMware，KVM，Xen等虚拟化技术，复杂灵活。
- OpenStack可以降低云计算成本，一些大厂直接拿过来，二次开发就可以做自己的云计算产品了。
- OpenStack为云计算业界提供了参考模型，所以无论是国外还是国内都有其他轻量级开源的IaaS代替品。国内的ZStack，StarVConeter。 国外的Apache CloudStack，OpenNebula。其中CloudStack的客户包括英国电信等大公司。
- 各种教程和官方搭建，都是基于ubuntu来演示。

  

# 3 劣势

  

- 部署和运维OpenStack非常非常复杂，并且耗时，而且需要不少相关专业知识。
- 知乎上有个技术爱好者，把OpenStack部署直到运行起来，花了一个多月： [没事不要在家里搭建 OpenStack - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/440120871) 
- 只有社区版，没有商业化支持，出了问题，只有靠公司自己解决，难以获得专业帮助
- 因为进入衰退期，所以OpenStack的安全问题。性能问题，一直进展很慢。部署后，因为各个组件太多，通信复杂，非常容易出一些问题。
- 进入衰退期，有OpenStack经验的人很少了，连NASA都放弃了。
- OpenStack是Python编写的，Python的包，还有各种依赖冲突共存的问题，做过Python开发的有目共睹。安装期间会出现很多问题，运行维护期也会出现很多问题。非常复杂。
- 对于中小公司，无论是做私有云还是公有云，一般都不推荐用OpenStack，OpenStack需要一个专业的DevOps团队来做支撑。
- 各种教程和官方搭建都是基于ubuntu演示，这个其实对蓝队云也是劣势，因为蓝队云的OS经验大部分是CentOS.   其他OS搭建OpenStack，很难成功。

# 4 网络上技术人员对OpenStack的评价

  

- 40到50人的团队搞OpenStack，失败的案例不计其数。失败的原因通常是把功能描绘的太邪乎，对团队的能力盲目信任。
- 40-50人的团队，老老实实 webvirtmgr 或者 proxmoxve 即可，openstack 这种，等服务器过 100 台再说。
- 我也是一个人搞openstack，从kilo升级到liberty以后就换了深信服的超融合

  

# 5 OpenStack与其他IaaS的对比表格

![[Pasted image 20240204100602.png]]


![[Pasted image 20240204100613.png]]

# 6 总结

  

**总结:**

- OpenStack功能最丰富，但也最复杂，需要专业知识才能部署和管理。
- Proxmox VE相对简单，易于使用，适合小型企业和个人使用。(各方面口碑都很好，并且是商业公司免费开源)
- CloudStack功能和易用性介于OpenStack和Proxmox VE之间，适合中型企业使用。

**具体选择哪个平台，需要根据您的具体需求和技术能力进行评估。**

**以下是一些建议:**

- 如果您需要一个功能丰富、可扩展的公有云平台，并且具备专业知识，可以选择OpenStack。
- 如果您需要一个相对简单、易于使用的虚拟化平台，可以选择Proxmox VE。 [Features - Proxmox Virtual Environment](https://www.proxmox.com/en/proxmox-virtual-environment/features)
- 如果您需要一个功能和易用性兼顾的云平台，可以选择CloudStack。也支持VPC [Configuring a Virtual Private Cloud — Apache CloudStack Administration Documentation 4.11.0.0 documentation](https://docs.cloudstack.apache.org/projects/archived-cloudstack-administration/en/latest/networking/virtual_private_cloud_config.html)

# 参考资料

  

- [(51 封私信 / 80 条消息) 国内的云计算平台有没有不是依靠 OpenStack 搭建的？ - 知乎 (zhihu.com)](https://www.zhihu.com/question/34511860)
- [(51 封私信 / 80 条消息) 如何用openstack搭建云平台？ - 知乎 (zhihu.com)](https://www.zhihu.com/question/54549481)
- [KVM 虚拟化环境搭建 - WebVirtMgr - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/49120559)
- [KVM 虚拟化环境搭建 - ProxmoxVE - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/49118355)
- [Proxmox Virtual Environment - Open-Source Server Virtualization Platform](https://www.proxmox.com/en/proxmox-virtual-environment/overview)
- [Proxmox——虚拟机使用_proxmoxve中文使用手册-CSDN博客](https://blog.csdn.net/wang__sepcial/article/details/121990466)
- [Proxmox(pve)集群(1)安装(1)系统安装 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/650900358?utm_id=0)
- [Software-Defined Network (proxmox.com)](https://pve.proxmox.com/pve-docs/chapter-pvesdn.html)
- [Apache CloudStack: Open Source Cloud Computing](https://cloudstack.apache.org/cloud-builders.html)
- [Apache CloudStack: Open Source Cloud Computing](https://cloudstack.apache.org/api.html)