## 前言

这几天把去年写的[k8s教程](https://github.com/AlexiaChen/AlexiaChen.github.io/issues/145)的坑填完了，趁热打铁。看看DevOps领域的新鲜概念。

## Infrastructure as code(Iac)

这里的基础设施是指VM server，网络，基础设施软件（DB Nginx等）

IaC叫基础设施即代码，就是以前我们创建安装配置基础设施大部分都是手工创建管理，现在我们要把Infra的构建和管理自动化，既然自动化，那么肯定是写代码（配置代码）了， 大概就是这样。简而言之就是别在准备环境的过程浪费过多的时间，尽可能快速开始干正事，为了达到这个目的，就是尽可能把准备环境自动化，几行配置搞定，用工具apply配置就可以了。

能达到这个目的的工具有Ansible， Terraform， Puppet，Chef。为啥需要这么多工具呢？ 因为一个工具不能覆盖整个IaC的流程功能。

IaC的主要功能大致有3类任务功能（按顺序）:

- Infra Provisioning, 就是提供基础设施环境的功能，快速创建server环境，比如快速准备几个VM的server。快速建立网络并创建LoadBalancer这些。
- configuration of provisioned Infra， 就是在已经准备好的基础设施上配置更上层的东西，比如在新建立好的几个server上快速自动化安装（管理，删除）某些基础软件（DB，Nginx等）或App
- Deployment of app， 就是在server上快速部署一些app或者基础软件

你看到上面，你会发现，第二步和第三部现在有Docker这样的容器化管理技术，已经可以通过Docker就解决了，第二步和第三部不存在了，只要第一步快速建立server，就可以安装Docker，Docker swarm这些东西，就可以通过一行命令，从DockerHub拉取你需要的App 或基础软件的image最后run起来。所以本质上，IaC主要的工具就是提供第一步的功能了，当然，也包括一点第二步的功能。

![_M9O7IZBQL ENY{G@9}BG5B](https://user-images.githubusercontent.com/8574915/168430575-3b02927e-3a94-4e76-9802-db42156b326e.png)

所以你看上图就知道，Terraform一般用于Infra Provision，提供Infra，变更删除Infra。Ansible就专注于安装和部署App。

Terraform是声明式配置，就是只要你配置不变，那么你多次apply，那么结果都是幂等的，因为状态没有变，跟k8s的声明式配置是一样的，都是 Desired state与Actual state做对比，有变动才会做状态更新。而ansible是过程式的，有点像你写shell自动化脚本一样，来用这些过程脚本做自动化。

## GitOps

本质上GitOps就是Infra as Code的这些配置代码提高到开发级别的同等高度，配置就是代码，所以需要把它们放到git repo中管理，对其不仅仅是做版本管理，也需要工程化对待，比如修改IaC的配置，必须发PR, 同样需要经过code review，并且通过CI/CD自动化测试部署这些IaC配置代码。所以你看到了，它就相当于IaCOps，叫GitOps反而有些误导，DevOps是开发方面。

每一次IaC的代码变更，CI/CD都会对其进行自动apply，自动构建基础设施环境。

GitOps = IaC + Version Control + Pull Request/ Merge Request + CI/CD pipeline

GitOps也是有相关的很多工具的，比如ArgoCD等等。


## References

- https://about.gitlab.com/topics/gitops/
- https://github.com/weaveworks/awesome-gitops
