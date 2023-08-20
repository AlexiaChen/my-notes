## 前言

在GitOps那篇文章提到过IaC了，TF就是IaC的工具，它主要解决Infra Provisioning的问题。接下来我们直接步入主题吧。

## Terraform的基本原理

![BG1WE P%} _0B 9YH`UYK26](https://user-images.githubusercontent.com/8574915/168454671-39bcc75e-bf82-4df4-97fd-f3368757a874.png)

就是跟之前说的一样，原理和k8s这样基于declarative 的原理一样，都是Desired state跟actual state做对比，做diff，然后变更diff部门，幂等执行。这样就是只要配置文件相同，那么能保证提供的infra环境（VPS, VM等）是一致的。

从大部分公司使用TF来看，主要是用来构建IaaS服务，比如用TF 向Azure，GCP(Google Cloud Platform)，AWS, Alibaba Cloud这样的Coud Provider申请配置Infra资源。当然还可以配置本地私有的Infra资源，比如k8s，Docker，自建的Openstack等。所以在TF生态下，有很多TF provider，有了这些provider，就可以通过TF配置对应Provider提供的Infra资源了。这些Provider你可以理解是一些第三方实现的插件，TF只是提供了接口和标准。

下面就是TF配置文件的样例:

![~N6A FOVWH$PFY)CIG2VBIO](https://user-images.githubusercontent.com/8574915/168454808-fddf1354-c823-49a5-a858-72e174951b2d.png)

上图是向AWS申请并创建一个VPC资源，并且给这台VPC配置了一个IPv4 CIDR block ，这个详情可以看AWS的VPC Provider参数 https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/vpc#cidr_block

https://en.wikipedia.org/wiki/Classless_Inter-Domain_Routing#IPv4_CIDR_blocks

![%JI2ZLN3W6GK)N$YXOO6}MK](https://user-images.githubusercontent.com/8574915/168454813-192c277d-04fe-46cc-88d0-e8182fdee968.png)

上图是向指定的k8s集群申请并创建了一个叫“my-first-namespace”的namespace资源。


## Terraform for Docker

由于我没有AWS账号之类的，暂时不弄最流行的Terraform + AWS的教程。我尝试用本地的Docker来完成这个工作。主要是翻译terraform官网的Docker教程。 learning by doing。

好了，我们就根据官网的Interactive Lab作为我们本机环境的学习教程。先我们在WSL2中安装terraform, https://www.terraform.io/cli/install/apt

```bash
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
sudo apt-add-repository "deb [arch=$(dpkg --print-architecture)] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
sudo apt install terraform
```

这样就安装上了。首先先启动Docker的Daemon `sudo service docker start`， 然后查看下terrafrom是否正常 `terraform version`

```bash
mathxh@MathxH:~/terraform-docker$ terraform version
Terraform v1.1.9
on linux_amd64
```

- 在这个目录下创建一个main.tf的文件，然后加入以下内容

```terraform
terraform {
  required_providers {
    docker = {
      source  = "kreuzwerker/docker"
      version = "~> 2.13.0"
    }
  }
}

provider "docker" {}

resource "docker_image" "nginx" {
  name         = "nginx:latest"
  keep_locally = false
}

resource "docker_container" "nginx" {
  image = docker_image.nginx.latest
  name  = "tutorial"
  ports {
    internal = 80
    external = 8000
  }
}

```
- 执行 `terraform init` 它会自动安装你指定的Provider插件（kreuzwerker/docker） 

```bash
mathxh@MathxH:~/terraform-docker$ terraform init

Initializing the backend...

Initializing provider plugins...
- Finding kreuzwerker/docker versions matching "~> 2.13.0"...
- Installing kreuzwerker/docker v2.13.0...
- Installed kreuzwerker/docker v2.13.0 (self-signed, key ID 24E54F214569A8A5)

Partner and community providers are signed by their developers.
If you'd like to know more about provider signing, you can read about it here:
https://www.terraform.io/docs/cli/plugins/signing.html

Terraform has created a lock file .terraform.lock.hcl to record the provider
selections it made above. Include this file in your version control repository
so that Terraform can guarantee to make the same selections by default when
you run "terraform init" in the future.

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
```

- 执行`terraform apply` 它会执行 main.tf, 拉取Nginx镜像资源并运行容器

```bash
mathxh@MathxH:~/terraform-docker$ terraform apply

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the
following symbols:
  + create

Terraform will perform the following actions:

  # docker_container.nginx will be created
  + resource "docker_container" "nginx" {
      + attach           = false
      + bridge           = (known after apply)
      + command          = (known after apply)
      + container_logs   = (known after apply)
      + entrypoint       = (known after apply)
      + env              = (known after apply)
      + exit_code        = (known after apply)
      + gateway          = (known after apply)
      + hostname         = (known after apply)
      + id               = (known after apply)
      + image            = (known after apply)
      + init             = (known after apply)
      + ip_address       = (known after apply)
      + ip_prefix_length = (known after apply)
      + ipc_mode         = (known after apply)
      + log_driver       = "json-file"
      + logs             = false
      + must_run         = true
      + name             = "tutorial"
      + network_data     = (known after apply)
      + read_only        = false
      + remove_volumes   = true
      + restart          = "no"
      + rm               = false
      + security_opts    = (known after apply)
      + shm_size         = (known after apply)
      + start            = true
      + stdin_open       = false
      + tty              = false

      + healthcheck {
          + interval     = (known after apply)
          + retries      = (known after apply)
          + start_period = (known after apply)
          + test         = (known after apply)
          + timeout      = (known after apply)
        }

      + labels {
          + label = (known after apply)
          + value = (known after apply)
        }

      + ports {
          + external = 8000
          + internal = 80
          + ip       = "0.0.0.0"
          + protocol = "tcp"
        }
    }

  # docker_image.nginx will be created
  + resource "docker_image" "nginx" {
      + id           = (known after apply)
      + keep_locally = false
      + latest       = (known after apply)
      + name         = "nginx:latest"
      + output       = (known after apply)
      + repo_digest  = (known after apply)
    }

Plan: 2 to add, 0 to change, 0 to destroy.

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes

docker_image.nginx: Creating...
docker_image.nginx: Still creating... [10s elapsed]
docker_image.nginx: Creation complete after 17s [id=sha256:7425d3a7c478efbeb75f0937060117343a9a510f72f5f7ad9f14b1501a36940cnginx:latest]
docker_container.nginx: Creating...
docker_container.nginx: Creation complete after 8s [id=b60c7266a7b8b9c0b7d3e306ada54292757e278abd4822c1234a6c95560a14cf]

Apply complete! Resources: 2 added, 0 changed, 0 destroyed.
mathxh@MathxH:~/terraform-docker$ docker ps
CONTAINER ID   IMAGE          COMMAND                  CREATED              STATUS              PORTS                  NAMES
b60c7266a7b8   7425d3a7c478   "/docker-entrypoint.…"   About a minute ago   Up About a minute   0.0.0.0:8000->80/tcp   tutorial
``` 

从上面可以看到container运行起来了，好了因为是在WSL2中运行的terraform， 从TF的配置上看到container暴露的external端口也是8000，所以我们从宿主机上访问WSL2中的8000，首先通过 `ip address` 拿到 eth0的 IP地址，然后通过IP:8000形式访问

- 访问 `http://172.28.230.233:8000/`

![7FNQJ0M@WK_I@ 1SXU@MZ)M](https://user-images.githubusercontent.com/8574915/168461462-7e08173f-7e5c-47ba-b34a-12f73fa343d5.png)

- 销毁资源 `terraform destroy`

```bash
mathxh@MathxH:~/terraform-docker$ terraform destroy
docker_image.nginx: Refreshing state... [id=sha256:7425d3a7c478efbeb75f0937060117343a9a510f72f5f7ad9f14b1501a36940cnginx:latest]
docker_container.nginx: Refreshing state... [id=b60c7266a7b8b9c0b7d3e306ada54292757e278abd4822c1234a6c95560a14cf]

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the
following symbols:
  - destroy

Terraform will perform the following actions:

  # docker_container.nginx will be destroyed
  - resource "docker_container" "nginx" {
      - attach            = false -> null
      - command           = [
          - "nginx",
          - "-g",
          - "daemon off;",
        ] -> null
      - cpu_shares        = 0 -> null
      - dns               = [] -> null
      - dns_opts          = [] -> null
      - dns_search        = [] -> null
      - entrypoint        = [
          - "/docker-entrypoint.sh",
        ] -> null
      - env               = [] -> null
      - gateway           = "172.17.0.1" -> null
      - group_add         = [] -> null
      - hostname          = "b60c7266a7b8" -> null
      - id                = "b60c7266a7b8b9c0b7d3e306ada54292757e278abd4822c1234a6c95560a14cf" -> null
      - image             = "sha256:7425d3a7c478efbeb75f0937060117343a9a510f72f5f7ad9f14b1501a36940c" -> null
      - init              = false -> null
      - ip_address        = "172.17.0.2" -> null
      - ip_prefix_length  = 16 -> null
      - ipc_mode          = "private" -> null
      - links             = [] -> null
      - log_driver        = "json-file" -> null
      - log_opts          = {} -> null
      - logs              = false -> null
      - max_retry_count   = 0 -> null
      - memory            = 0 -> null
      - memory_swap       = 0 -> null
      - must_run          = true -> null
      - name              = "tutorial" -> null
      - network_data      = [
          - {
              - gateway                   = "172.17.0.1"
              - global_ipv6_address       = ""
              - global_ipv6_prefix_length = 0
              - ip_address                = "172.17.0.2"
              - ip_prefix_length          = 16
              - ipv6_gateway              = ""
              - network_name              = "bridge"
            },
        ] -> null
      - network_mode      = "default" -> null
      - privileged        = false -> null
      - publish_all_ports = false -> null
      - read_only         = false -> null
      - remove_volumes    = true -> null
      - restart           = "no" -> null
      - rm                = false -> null
      - security_opts     = [] -> null
      - shm_size          = 64 -> null
      - start             = true -> null
      - stdin_open        = false -> null
      - sysctls           = {} -> null
      - tmpfs             = {} -> null
      - tty               = false -> null

      - ports {
          - external = 8000 -> null
          - internal = 80 -> null
          - ip       = "0.0.0.0" -> null
          - protocol = "tcp" -> null
        }
    }

  # docker_image.nginx will be destroyed
  - resource "docker_image" "nginx" {
      - id           = "sha256:7425d3a7c478efbeb75f0937060117343a9a510f72f5f7ad9f14b1501a36940cnginx:latest" -> null
      - keep_locally = false -> null
      - latest       = "sha256:7425d3a7c478efbeb75f0937060117343a9a510f72f5f7ad9f14b1501a36940c" -> null
      - name         = "nginx:latest" -> null
      - repo_digest  = "nginx@sha256:2c72b42c3679c1c819d46296c4e79e69b2616fa28bea92e61d358980e18c9751" -> null
    }

Plan: 0 to add, 0 to change, 2 to destroy.

Do you really want to destroy all resources?
  Terraform will destroy all your managed infrastructure, as shown above.
  There is no undo. Only 'yes' will be accepted to confirm.

  Enter a value: yes

docker_container.nginx: Destroying... [id=b60c7266a7b8b9c0b7d3e306ada54292757e278abd4822c1234a6c95560a14cf]
docker_container.nginx: Destruction complete after 2s
docker_image.nginx: Destroying... [id=sha256:7425d3a7c478efbeb75f0937060117343a9a510f72f5f7ad9f14b1501a36940cnginx:latest]
docker_image.nginx: Destruction complete after 0s

Destroy complete! Resources: 2 destroyed.
mathxh@MathxH:~/terraform-docker$ docker ps
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
mathxh@MathxH:~/terraform-docker$ netstat -anl | grep 8000
mathxh@MathxH:~/terraform-docker$
```

下面我们来对配置文件`main.tf`进行详解。

```terraform
terraform {
  required_providers {
    docker = {
      source  = "kreuzwerker/docker"
      version = "~> 2.13.0"
    }
  }
}
```

terraform {}块包含了Terraform的设置，包括Terraform用来配置基础设施所需的Provider。对于每个Provider，源属性定义了一个可选的主机名、一个命名空间和Provider类型。Terraform默认从Terraform registry中安装Provider。在这个配置示例中，docker的Provider的来源被定义为kreuzwerker/docker，这是对registry.terraform.io/kreuzwerker/docker的简写。

你还可以为在required_providers块中定义的每个provider设置一个版本约束。版本属性是可选的，但我们建议使用它来约束提供者的版本，这样Terraform就不会安装不适合你的配置的provider的版本。如果你不指定provider的版本，Terraform会在初始化时自动下载最新的版本。

详情参考 https://www.terraform.io/language/providers/requirements


```terraform
provider "docker" {}
```

Provider模块配置指定的Provider，在这里是指docker。提供者是一个插件，Terraform使用它来创建和管理你的Infra资源。

你可以在你的Terraform配置中使用多个Provider块来管理不同提供者的资源。你甚至可以同时使用不同的Provider。例如，你可以将Docker镜像ID传递给Kubernetes服务。

```terraform
resource "docker_image" "nginx" {
  name         = "nginx:latest"
  keep_locally = false
}

resource "docker_container" "nginx" {
  image = docker_image.nginx.latest
  name  = "tutorial"
  ports {
    internal = 80
    external = 8000
  }
}
```

使用resource块来定义你的基础设施的组件。一个resource可能是一个物理或虚拟组件，如Docker容器，也可能是一个逻辑资源，如Heroku应用程序。

resource块前面有两个字符串：resource type和resource name。在这个例子中，第一个资源类型是docker_image，名称是nginx。resource type的前缀映射到提供者的名称。在这个配置例子中，Terraform用docker provider管理docker_image resource。resource type和resource name共同构成了resource的唯一ID。例如，你的Docker镜像的ID是docker_image.nginx。

resource块包含参数，你可以用它来配置资源。参数可以包括诸如机器尺寸、磁盘镜像名称或VPC ID等。我们的provider 的文档中 https://www.terraform.io/language/providers 记载了每个resource的必要和可选参数。对于你的容器，该配置示例将Docker镜像作为docker_container资源的镜像源。

## Terraform 的 input variable

这样使用会更灵活，毕竟有些工程的配置很复杂。

创建一个`variables.tf`的文件，在之前的目录下，并加入以下内容

```terraform
variable "container_name" {
  description = "Value of the name for the Docker container"
  type        = string
  default     = "ExampleNginxContainer"
}
```

然后修改main.tf 引用该变量, 并再次 `terraform apply`

```terraform
resource "docker_container" "nginx" {
  image = docker_image.nginx.latest
  name  = var.container_name
  ports {
    internal = 80
    external = 8000
  }
}
```

结果就是容器名变成了 `ExampleNginxContainer` :

```bash
mathxh@MathxH:~/terraform-docker$ docker ps
CONTAINER ID   IMAGE          COMMAND                  CREATED         STATUS         PORTS                  NAMES
624c4ad2d453   7425d3a7c478   "/docker-entrypoint.…"   9 seconds ago   Up 4 seconds   0.0.0.0:8000->80/tcp   ExampleNginxContainer
```

如果你要覆盖默认的这个变量值，你可以通过terraform的 `-var`来传递进去修改变量值， ` terraform apply -var="container_name=MathxHChen"`

这样terraform就会先destroy之前的container，再启动一个新的名为MathxHChen的新container。Terraform还支持很多关于variable的用法，详情请看 https://learn.hashicorp.com/tutorials/terraform/variables?in=terraform/configuration-language

## Terraform的Output Data

之前已经说过利用Terraform提供的input variables让配置文件更灵活，使其参数化。你将使用output value来组织数据，以方便查询和显示给Terraform用户。也算是为了方便调试吧。

在目录下创建一个`outputs.tf`的文件，并且`terraform apply`

```bash
mathxh@MathxH:~/terraform-docker$ ls
main.tf  outputs.tf  terraform.tfstate  terraform.tfstate.backup  variables.tf
```
文件中定义你需要的container id和image id的输出

```terraform
output "container_id" {
  description = "ID of the Docker container"
  value       = docker_container.nginx.id
}

output "image_id" {
  description = "ID of the Docker image"
  value       = docker_image.nginx.id
}
```

apply完成你就可以看到output了，你也可以用`terraform output`来查看 

![EX@$E0)JNQ1SH43EB2O67C0](https://user-images.githubusercontent.com/8574915/168462891-aa579289-878d-47de-9ab6-7a26fc30a5b5.png)

## 下一步的深入学习

- [配置语言](https://learn.hashicorp.com/collections/terraform/configuration-language) - 需要更熟悉variables、outputs、 dependencies, meta-arguments和其他语言功能，以编写更复杂的Terraform配置。

- [模块](https://learn.hashicorp.com/tutorials/terraform/module) - 用模块来组织和复用Terraform配置。

- [provision](https://learn.hashicorp.com/collections/terraform/provision) - 使用Packer或Cloud-init自动配置SSH密钥和Web服务器，并 把它们配置到由Terraform在AWS创建的Linux虚拟机上

- [导入](https://learn.hashicorp.com/tutorials/terraform/state-import) --将现有基础设施导入Terraform。

- 还有一个course https://github.com/wardviaene/terraform-course 和一个example https://github.com/futurice/terraform-examples

## References

- https://learn.hashicorp.com/collections/terraform/docker-get-started
- https://learn.hashicorp.com/collections/terraform/configuration-language
- https://learn.hashicorp.com/collections/terraform/cli
- https://www.terraform.io/intro/use-cases
- https://registry.terraform.io/providers/kreuzwerker/docker/latest
- https://www.terraform.io/cli/commands/state


