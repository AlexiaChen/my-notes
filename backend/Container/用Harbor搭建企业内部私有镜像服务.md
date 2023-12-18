
## 简介

目前在公司内部自己搭建Docker并兼容Kubernetes的镜像服务，最流行和主流的方案是使用Harbor。

Harbor是一个开源的镜像仓库，它提供了一个可扩展的平台，用于存储、分发和管理Docker镜像。Harbor支持Docker和OCI（Open Container Initiative）格式的镜像，并且与Kubernetes紧密集成，可以作为Kubernetes集群的镜像仓库。

使用Harbor搭建公司内部的镜像服务有以下几个优势：

1. 安全性：Harbor提供了丰富的安全功能，包括镜像签名、漏洞扫描和访问控制等，可以确保镜像的安全性。
    
2. 可扩展性：Harbor支持水平扩展，可以根据需求增加存储容量和处理能力，适应不断增长的镜像需求。
    
3. 多云支持：Harbor可以部署在私有云、公有云或混合云环境中，支持多云环境下的镜像管理和分发。
    
4. 高可用性：Harbor支持多节点部署，可以实现高可用性和容错性，确保镜像服务的稳定性和可靠性。
    
5. 用户管理：Harbor提供了灵活的用户和权限管理功能，可以根据角色和权限设置访问控制，确保镜像的安全访问。
    

总结来说，使用Harbor搭建公司内部的镜像服务是目前最流行和主流的方案之一。它提供了丰富的功能和灵活的部署选项，可以满足公司对镜像管理的需求，并与Kubernetes紧密集成，方便在Kubernetes集群中使用和管理镜像。

#### Harbor与Nexus的区别

Harbor和Nexus都可以用于搭建私有镜像服务，但它们有一些区别：

1. **专注领域**：
    
    - Harbor专注于Docker镜像管理，提供了丰富的Docker镜像存储、安全扫描和访问控制功能，以及与Kubernetes的紧密集成。
    - Nexus是一个通用的仓库管理器，支持多种软件包管理格式，包括Maven、npm、Docker等，因此在镜像管理方面相对更为通用。
2. **安全功能**：
    
    - Harbor在安全方面提供了专门针对Docker镜像的漏洞扫描、镜像签名和访问控制等功能，使得它更适合于Docker镜像的安全管理。
    - Nexus同样提供了安全功能，但可能在特定的Docker镜像安全特性方面相对较弱。
3. **集成性**：
    
    - Harbor与Kubernetes的集成更为紧密，可以作为Kubernetes集群的镜像仓库，支持Helm Chart等。
    - Nexus也可以与Kubernetes集成，但在Docker镜像管理方面可能没有Harbor那么深入的集成。
4. **用户体验**：
    
    - Harbor专注于Docker镜像管理，因此在Docker镜像相关的用户体验和功能方面可能更为专业和完善。
    - Nexus作为通用的仓库管理器，可能在多种软件包管理格式的支持和管理方面更为全面。

综上所述，Harbor更适合于专注于Docker镜像管理的场景，提供了更专业和完善的Docker镜像管理功能；而Nexus则更适合于需要管理多种软件包格式的通用场景。

> Harbor还可以配置为自动同步Docker Hub上的镜像到私有镜像服务中。这样，除了自己构建的私有镜像之外，还可以在Harbor中使用公共的基础镜像。


## Harbor的安装和配置

本节介绍如何执行 Harbor 的新安装。

如果从 Harbor 的旧版本升级，可能需要更新配置文件并迁移数据，以适应新版本的数据库模式。有关升级的信息，请参阅[升级 Harbor](https://goharbor.io/docs/2.9.0/administration/upgrade/)。

安装 Harbor 之前，可以在 Harbor 团队维护的演示环境中测试最新版本的 Harbor。有关信息，请参阅使用[[#在Demo server上测试Harbor]]。

Harbor 支持与不同的第三方复制适配器（用于复制数据）、OIDC 适配器（用于 authN/authZ）和扫描器适配器（用于扫描容器镜像的漏洞）集成。有关支持的适配器的信息，请参阅 [[#Harbor的兼容列表]]。

### 安装步骤


#### 把Harbor部署在k8s上

把Harbor部署在k8s集群上，以实现高可用。https://goharbor.io/docs/2.9.0/install-config/harbor-ha-helm/

#### 安装后的配置

有关如何管理已部署的 Harbor 实例的信息，请参阅 [[#重新配置Harbor和管理Harbor的生命周期]]

默认情况下，Harbor 使用自己的私钥和证书与 Docker 进行身份验证。有关如何自定义配置以使用自己的密钥和证书的信息，请参阅 [[#自定义Harbor的令牌服务]]。

安装完成后，通过网络控制台登录 Harbor，在 "配置 "下配置实例。Harbor 还提供命令行界面（CLI），允许你 在命令行中配置[[#Harbor配置]] 。

#### Harbor的组件

下表列出了部署 Harbor 时要部署的一些关键组件。

|Component|Version|
|---|---|
|Postgresql|13.3.0|
|Redis|6.0.13|
|Beego|1.9.0|
|Docker/distribution|2.7.1|
|Helm|2.9.1|
|Swagger-ui|3.22.1|


### 在Demo server上测试Harbor

Harbor 团队提供了一个 Harbor 演示实例，您可以使用该实例试用 Harbor 并测试其功能。
使用演示服务器时，请注意使用条件。

#### Demo server的使用条件

- 演示服务器仅供试验使用，允许您测试港湾功能。
- 请勿将敏感图像上传到演示服务器。
- 演示服务器不是生产环境。Harbor 团队对使用演示服务器可能造成的任何数据、功能或服务损失概不负责。
- 演示服务器每两天清理和重置一次。
- 演示服务器只允许您测试用户功能。无法测试管理员功能。要测试管理员功能和高级功能，请设置 Harbor 实例。
- 请勿向演示服务器推送大于 100MB 的图像，因为演示服务器的存储容量有限。

#### 访问Demo server

1. 访问 https://demo.goharbor.io。
2. 单击注册账户。
3. 提供用户名、电子邮件地址、姓名和密码，创建用户账户。
4. 使用创建的账户登录 Harbor 界面。
5. 浏览默认项目: `library`。
6. 单击 "新建项目 "创建自己的项目。
     有关如何创建项目的信息，请参阅创建项目。[Harbor docs | Create Projects (goharbor.io)](https://goharbor.io/docs/2.9.0/working-with-projects/create-projects/)
7. 打开 Docker 客户端，并使用上文创建的凭据登录 Harbor。

```bash
docker login demo.goharbor.io
```
8. 创建一个非常简单的 Dockerfile，内容如下。

```dockerfile
FROM busybox:latest
```

9. 从这个 Dockerfile 生成一个镜像，并打上标签。

```bash
docker build -t demo.goharbor.io/your-project/test-image .
```

10. 将镜像推送到 Harbor 中的项目。

```bash
docker push demo.goharbor.io/your-project/test-image
```

在 Harbor 界面，转到Porjects > your project > repositories，查看你推送到 Harbor 项目的镜像仓库。
### Harbor的兼容列表

[Harbor docs | Harbor Compatibility List (goharbor.io)](https://goharbor.io/docs/2.9.0/install-config/harbor-compatibility-list/)

### Harbor安装的预置要求

Harbor 以多个 Docker 容器的形式部署。因此，你可以在任何支持 Docker 的 Linux 发行版上部署它。目标主机需要安装 Docker 和 Docker Compose。

#### 硬件

下表列出了部署 Harbor 的最低和推荐硬件配置。

|Resource|Minimum|Recommended|
|---|---|---|
|CPU|2 CPU|4 CPU|
|Mem|4 GB|8 GB|
|Disk|40 GB|160 GB|

#### 软件

下表列出了目标主机上必须安装的软件版本。

|Software|Version|Description|
|---|---|---|
|Docker Engine|Version 17.06.0-ce+ or higher|For installation instructions, see [Docker Engine documentation](https://docs.docker.com/engine/installation/)|
|Docker Compose|docker-compose (v1.18.0+) or docker compose v2 (docker-compose-plugin)|For installation instructions, see [Docker Compose documentation](https://docs.docker.com/compose/install/)|
|OpenSSL|Latest is preferred|Used to generate certificate and keys for Harbor|

#### 网络端口

Harbor 要求在目标主机上打开以下端口。

|Port|Protocol|Description|
|---|---|---|
|443|HTTPS|Harbor portal and core API accept HTTPS requests on this port. You can change this port in the configuration file.|
|4443|HTTPS|Connections to the Docker Content Trust service for Harbor. You can change this port in the configuration file.|
|80|HTTP|Harbor portal and core API accept HTTP requests on this port. You can change this port in the configuration file.|



### 下载Harbor安装器

您可以从[官方发布页面](https://github.com/goharbor/harbor/releases)下载Harbor安装程序。下载在线安装程序或离线安装程序。

- 在线安装程序： 在线安装程序从 Docker hub 下载 Harbor 映像。因此，安装程序的体积非常小。
- 离线安装程序： 如果部署 Harbor 的主机无法连接互联网，请使用离线安装程序。脱机安装程序包含预制映像，因此比在线安装程序大。

在线安装程序和离线安装程序的安装过程几乎相同。

#### 下载和解压安装器

1. 访问 https://github.com/goharbor/harbor/releases
2. 下载离线版或者在线版的安装器
3. 可选择下载相应的 \*.asc 文件，以验证软件包的真实性。

\*.asc 文件是 OpenPGP 密钥文件。执行以下步骤验证下载的捆绑包是否真实。

       1. Obtain the public key for the `*.asc` file.
    
      ```bash
       gpg --keyserver hkps://keyserver.ubuntu.com --receive-keys 644FF454C0B4115C
       ```
    
    
2. Verify that the package is genuine by running one of the following commands.
    
    - Online installer:
        
        ```sh
        gpg -v --keyserver hkps://keyserver.ubuntu.com --verify harbor-online-installer-version.tgz.asc
        ```
        
    - Offline installer:
        
        ```sh
        gpg -v --keyserver hkps://keyserver.ubuntu.com --verify harbor-offline-installer-version.tgz.asc
        ```
        
    
    The `gpg` command verifies that the bundle’s signature matches that of the `*.asc` key file. You should see confirmation that the signature is correct.
    
    ```sh
    gpg: armor header: Version: GnuPG v1
    gpg: assuming signed data in 'harbor-online-installer-v2.0.2.tgz'
    gpg: Signature made Tue Jul 28 09:49:20 2020 UTC
    gpg:                using RSA key 644FF454C0B4115C
    gpg: using pgp trust model
    gpg: Good signature from "Harbor-sign (The key for signing Harbor build) <jiangd@vmware.com>" [unknown]
    gpg: WARNING: This key is not certified with a trusted signature!
    gpg:          There is no indication that the signature belongs to the owner.
    Primary key fingerprint: 7722 D168 DAEC 4578 06C9  6FF9 644F F454 C0B4 115C
    gpg: binary signature, digest algorithm SHA1, key algorithm
    ```
      
1. 使用tar来提取安装器中的包
    -  在线
 ```sh
tar xzvf harbor-online-installer-version.tgz
```
   - 离线

```sh
tar xzvf harbor-offline-installer-version.tgz
```

### 配置https访问Harbor

默认情况下，Harbor 不附带证书。可以在没有安全保障的情况下部署 Harbor，这样就可以通过 HTTP 与 Harbor 连接。不过，只有在没有外部互联网连接的空中屏蔽测试或开发环境中，才可以使用 HTTP。在没有空气屏蔽的环境中使用 HTTP 会使您遭受中间人攻击。在生产环境中，请始终使用 HTTPS。

要配置 HTTPS，必须创建 SSL 证书。可以使用由可信第三方 CA 签名的证书，也可以使用自签名证书。本节将介绍如何使用 OpenSSL 创建 CA，以及如何使用 CA 签署服务器证书和客户端证书。你也可以使用其他 CA 提供商，例如  [Let's Encrypt]()https://letsencrypt.org/ 。

以下步骤假定 Harbor 注册表的主机名是 `yourdomain.com`，其 DNS 记录指向运行 Harbor 的主机。

#### 生成一个CA证书

在生产环境中，应从 CA 获取证书。在测试或开发环境中，可以生成自己的 CA。要生成 CA 证书，请运行以下命令。

1. 生成CA证书私钥:

```bash
openssl genrsa -out ca.key 4096
```

2. 生成CA证书

调整 `-subj` 选项中的值，以反映你的组织。如果使用 FQDN 连接 Harbor 主机，则必须将其指定为通用名称 (CN) 属性。

```bash
openssl req -x509 -new -nodes -sha512 -days 3650 \
 -subj "/C=CN/ST=Beijing/L=Beijing/O=example/OU=Personal/CN=yourdomain.com" \
 -key ca.key \
 -out ca.crt
```


#### 生成一个服务器证书

证书通常包含一个  `.crt` 文件和一个 `.key` 文件，例如，`yourdomain.com.crt` 和 `yourdomain.com.key`。

1. 生成服务端证书私钥:

```bash
openssl genrsa -out yourdomain.com.key 4096
```

2. 生成证书签名请求(CSR)

调整 `-subj`选项中的值，以反映你的组织。如果使用 FQDN 连接 Harbor 主机，则必须将其指定为通用名称 (CN) 属性，并在密钥和 CSR 文件名中使用。

```bash
openssl req -sha512 -new \
    -subj "/C=CN/ST=Beijing/L=Beijing/O=example/OU=Personal/CN=yourdomain.com" \
    -key yourdomain.com.key \
    -out yourdomain.com.csr
```

3. 生成一个x509 v3的扩展文件

无论使用 FQDN 还是 IP 地址连接 Harbor 主机，都必须创建此文件，以便为 Harbor 主机生成符合主题备选名称 (SAN) 和 x509 v3 扩展要求的证书。替换 `DNS` 条目以反映你的域名。

```bash
cat > v3.ext <<-EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1=yourdomain.com
DNS.2=yourdomain
DNS.3=hostname
EOF
```

4. 使用`v3.ext`文件来生成Harbor主机的证书

将 CSR 和 CRT 文件名中的 `yourdomain.com` 替换为 Harbor 主机名。

```bash
openssl x509 -req -sha512 -days 3650 \
    -extfile v3.ext \
    -CA ca.crt -CAkey ca.key -CAcreateserial \
    -in yourdomain.com.csr \
    -out yourdomain.com.crt
```

#### 提供刚才生成的证书给Harbor和Docker

生成 `ca.crt`、`yourdomain.com.crt` 和 `yourdomain.com.key` 文件后，必须将它们提供给 Harbor 和 Docker，并重新配置 Harbor 以使用它们。

1. 拷贝服务端的证书和私钥到在horbor主机上的证书文件夹

```bash
cp yourdomain.com.crt /data/cert/
cp yourdomain.com.key /data/cert/
```

2. 转换 `yourdomain.com.crt` 到 `yourdomain.com.cert` 给Docker使用

因为Docker的Daemon进程会把`.crt` 的文件认为是CA证书，`.cert`的文件认为是客户端的证书。

```bash
openssl x509 -inform PEM -in yourdomain.com.crt -out yourdomain.com.cert
```

3. 拷贝服务端证书，服务端私钥和CA证书文件到Docker的证书文件夹，这个证书文件夹也在Harbor主机上，你必须优先创建相应的文件夹

```bash
cp yourdomain.com.cert /etc/docker/certs.d/yourdomain.com/
cp yourdomain.com.key /etc/docker/certs.d/yourdomain.com/
cp ca.crt /etc/docker/certs.d/yourdomain.com/
```

如果您将默认 nginx 端口 443 映射到其他端口，请创建文件夹 `/etc/docker/certs.d/yourdomain.com:port` 或 `/etc/docker/certs.d/harbor_IP:port`。

4. 重启Docker Engine

```bash
systemctl restart docker
```

您可能还需要在操作系统层面信任证书。更多信息，请参阅 [[#Harbor安装的troubleshooting]] 。

下面的示例说明了使用自定义证书的配置。

```fallback
/etc/docker/certs.d/
    └── yourdomain.com:port
       ├── yourdomain.com.cert  <-- Server certificate signed by CA
       ├── yourdomain.com.key   <-- Server key signed by CA
       └── ca.crt               <-- Certificate authority that signed the registry certificate
```

#### 部署或重新配置Harbor

如果您尚未部署 Harbor，请参阅配置  [[#配置Harbor YML文件]] 以获取有关如何通过在`harbor.yml` 中指定`hostname`和 `https` 属性来配置 Harbor 以使用证书的信息

如果已经使用 HTTP 部署了 Harbor，但想重新配置为使用 HTTPS，请执行以下步骤。

1. 运行`prepare`脚本以启用 HTTPS。

Harbor 使用 `nginx` 实例作为所有服务的反向代理。您可以使用`prepare`脚本将 nginx 配置为使用 HTTPS。`prepare`位于 Harbor 安装程序包中，与 `install.sh` 脚本处于同一级别。

```bash
./prepare
```

2. 如果 Harbor 正在运行，请停止并删除现有实例。

您的图像数据保留在文件系统中，因此不会丢失任何数据。

```bash
docker-compose down -v
```

3. 重启Harbor:

```bash
docker-compose up -d
```
#### 校验https连接

为 Harbor 设置 HTTPS 后，您可以通过执行以下步骤来验证 HTTPS 连接。


- 打开浏览器并输入 https://yourdomain.com 它应该显示Harbor 界面。
- 某些浏览器可能会显示警告，指出证书颁发机构 (CA) 未知。当使用并非来自受信任的第三方 CA 的自签名 CA 时，会发生这种情况。您可以将 CA 导入浏览器以消除警告。
- 在运行 Docker 守护程序的计算机上，检查 `/etc/docker/daemon.json` 文件以确保未为 https://yourdomain.com 设置 `-insecure-registry` 选项。
- 从 Docker 客户端登录 Harbor。

```bash
docker login yourdomain.com
```

如果您已将 `nginx` 443 端口映射到其他端口，请在`login`命令中添加该端口。

```bash
docker login yourdomain.com:port
```



### 配置Harbor组件之间的内部TLS通信

默认情况下，Harbor 组件（`harbor-core、harbor-jobservice、proxy、harbor-portal、registry、registryctl、trivy_adapter、chartmuseum`）之间的内部通信使用 HTTP 协议，该协议对于某些生产环境可能不够安全。 从Harbor v2.0开始，TLS可以用于这个内部网络。 在生产环境中，始终使用 HTTPS 是建议的最佳实践。

此功能是通过`harbor.yml` 文件中的`internal_tls` 引入的。 要启用内部 TLS，请将启用设置为 `true` 并将 `dir` 值设置为包含内部证书文件的目录路径。 所有证书都可以通过`prepare`工具自动生成。

```bash
docker run -v /:/hostfs goharbor/prepare:<current_harbor_version> gencert -p /path/to/internal/tls/cert
```

用户还可以提供自己的 CA 来生成其他证书。 只需将CA的证书和CA私钥放在内部tls cert目录中，并将它们命名为`harbor_internal_ca.key`和`harbor_internal_ca.crt`。 此外，用户还可以提供所有组件的证书。 但是，证书有一些限制：

- 首先，所有证书必须由单个唯一的 CA来进行签名
- 其次，证书文件上的内部证书和 `CN` 字段的文件名必须遵循下面列出的约定
- 第三，由于Golang 1.5中不推荐使用不带SAN的自签名证书，因此您在自己生成证书时必须将SAN扩展添加到您的证书文件中，否则Harbor实例将无法正常启动。 SAN 扩展中的 DNS 名称应与下表中的 CN 字段相同。 有关更多信息，请参阅 [golang 1.5 发行说明](https://golang.org/doc/go1.15#commonname)和[issue](https://github.com/golang/go/issues/24151)。

|  name | usage  | CN  |
|---|---|---|
|`harbor_internal_ca.key`|ca’s key file for internal TLS|N/A|
|`harbor_internal_ca.crt`|ca’s certificate file for internal TLS|N/A|
|`core.key`|core’s key file|N/A|
|`core.crt`|core’s certificate file|`core`|
|`job_service.key`|job_service’s key file|N/A|
|`job_service.crt`|job_service’s certificate file|`jobservice`|
|`proxy.key`|proxy’s key file|N/A|
|`proxy.crt`|proxy’s certificate file|`proxy`|
|`portal.key`|portal’s key file|N/A|
|`portal.crt`|portal’s certificate file|`portal`|
|`registry.key`|registry’s key file|N/A|
|`registry.crt`|registry’s certificate file|`registry`|
|`registryctl.key`|registryctl’s key file|N/A|
|`registryctl.crt`|registryctl’s certificate file|`registryctl`|
|`trivy_adapter.key`|trivy_adapter.’s key file|N/A|
|`trivy_adapter.crt`|trivy_adapter.’s certificate file|`trivy-ada`|

### 配置Harbor YML文件

您可以在安装程序包中包含的`harbor.yml` 文件中设置Harbor 的系统级参数。 这些参数在您运行`install.sh`脚本安装或重新配置Harbor时生效。

初始部署和启动 Harbor 后，您可以在 Harbor Web Portal 中执行其他配置。

#### 需要的参数

下表列出了部署Harbor时必须设置的参数。 默认情况下，所有必需的参数均在`harbor.yml`文件中均没有被注释。 可选参数用＃注释。 您不一定需要从提供的默认值中更改所需参数的值，但是这些参数必须保持未注释。 至少，您必须更新`hostname`参数。

> 重要提示：Harbor不提供任何证书。在1.9.x及以下的版本中，默认情况下，Harbor使用HTTP为注册表请求提供服务。这只能在气隙测试或开发环境中接受。在生产环境中，始终使用HTTPS。

您可以使用由受信任的第三方CA签名的证书，也可以使用自签名证书。有关如何创建CA以及如何使用CA对服务器证书和客户端证书进行签名的信息，请参阅  [[#配置https访问Harbor]] 

需要的参数请参考 [Harbor docs | Configure the Harbor YML File (goharbor.io)](https://goharbor.io/docs/2.9.0/install-config/configure-yml-file/)

### 运行安装器脚本

一旦配置了从`harbor.yml.tmpl`复制的`harbor.yml`，并可以选择设置存储后端，就可以使用`install.sh`脚本安装并启动harbor。请注意，在线安装程序可能需要一些时间才能从Dockerhub下载所有Harbor映像。

您可以在不同的配置中安装Harbor：

- 只是Harbor，没有Trivy
- Trivy + Harbor的方式

#### 默认安装不包含Trivy

默认的Harbor安装不包括Trivy服务。运行以下命令

```bash
sudo ./install.sh
```

如果安装成功，您可以打开浏览器访问Harbor界面，网址为 `http://reg.yourdomain.com`，将`reg.yourdomain.com`更改为您在`harbor.yml`中配置的hostname。如果您没有在`harbor.yml`中更改它们，则默认的管理员用户名和密码为`admin`和`Harbor12345`。

登录到管理门户并创建一个新项目，例如`myproject`。然后，您可以使用Docker命令登录到Harbor，标记图像，并将它们推送到Harbor。

```bash
docker login reg.yourdomain.com
docker push reg.yourdomain.com/myproject/myrepo:mytag
```

> - 如果您的Harbor安装使用HTTPS，则必须向Docker客户端提供Harbor证书。有关信息，请参阅  [[#配置https访问Harbor]]
> - 如果Harbor的安装使用HTTP，则必须将选项`--unsecurity-registry`添加到客户端的Docker守护进程中，然后重新启动Docker服务。有关更多信息，请参阅下面的  [[#用HTTP连接Harbor]] 。

#### 附带Trivy的安装

要使用Trivy服务安装Harbor，请在运行`install.sh`时添加`--with-Trivy`参数：

```bash
sudo ./install.sh --with-trivy
```

有关Trivy的更多信息，请参阅[Trivy文档](https://github.com/aquasecurity/trivy)。有关如何在网络代理环境中使用Trivy的更多信息，请参阅[为Trivy配置自定义证书颁发机构](https://goharbor.io/docs/2.9.0/install-config/run-installer-script/administration/vulnerability-scanning/configure-custom-certs.md)

#### 用HTTP连接Harbor

重要提示：如果Harbor的安装使用HTTP而不是HTTPS，则必须将选项`--unsecurity-registry`添加到客户端的Docker守护进程中。默认情况下，守护程序文件位于`/etc/docker/daemon.json`。

```json
{
    "insecure-registries" : ["myregistrydomain.com:5000", "0.0.0.0"]
}
```

```bash
systemctl restart docker
docker-compose down -v
docker-compose up -d
```

### 用Helm来部署Harbor以实现高可用

这个暂时不用关心，这个场景是把Harbor部署在k8s集群里面，实现高可用，一般用不上 [Harbor docs | Deploying Harbor with High Availability via Helm (goharbor.io)](https://goharbor.io/docs/2.9.0/install-config/harbor-ha-helm/)

### Harbor安装的troubleshooting

以下部分将帮助您解决安装Harbor时的问题。

#### 访问Harbor的日志

默认情况下，注册表数据持久存在主机的`/data/`目录中。即使删除和`/`或重新创建了Harbor的容器，这些数据也保持不变，您可以编辑`harbor.yml`文件中的`data_volume`来更改此目录。

此外，Harbor使用`rsyslog`来收集每个容器的日志。默认情况下，这些日志文件存储在目标主机上的目录`/var/log/harbor/`中以进行故障排除，您也可以更改`harbor.yml`中的日志目录。

#### Harbor没有启动或其功能不正确

如果Harbor没有启动或功能不正确，请运行以下命令检查Harbor的所有容器是否都处于`Up`状态。

```bash
docker-compose ps
```

如果容器未处于Up状态，请检查`/var/log/harbor`中该容器的日志文件。例如，如果`harbor-core`容器没有运行，请查看`core.log`日志文件。

#### 使用nginx或者负载均衡

如果Harbor在`nginx`代理或弹性负载平衡的后面 运行，请打开`common/config/nginx/nginx.conf`文件并搜索以下行。

```conf
proxy_set_header X-Forwarded-Proto $scheme;
```

如果代理已经有类似的设置，请将其从`location  / `   `location  /v2/`和  `location  /service/` sections中删除，然后重新部署Harbor。有关如何重新部署Harbor的说明，请参阅  [[#重新配置Harbor和管理Harbor的生命周期]]
#### Troubleshoot Https连接

如果您使用来自证书颁发者的中间证书，请将该中间证书与您自己的证书合并以创建证书捆绑包。运行以下命令。

```bash
cat intermediate-certificate.pem >> yourdomain.com.crt
```

当Docker守护进程在某些操作系统上运行时，您可能需要在操作系统级别信任证书。例如，运行以下命令。

- Ubuntu

```bash
cp yourdomain.com.crt /usr/local/share/ca-certificates/yourdomain.com.crt 
update-ca-certificates
```

- Red Hat(CentOS etc)

```bash
cp yourdomain.com.crt /etc/pki/ca-trust/source/anchors/yourdomain.com.crt
update-ca-trust
```

### 重新配置Harbor和管理Harbor的生命周期

您使用`docker-compose`来管理Harbor的生命周期。本主题提供了一些有用的命令。您必须在`docker-compose.yml`所在的目录中运行命令。

#### 停止Harbor

```bash
docker-compose stop
```

#### 重启Harbor

```bash
docker-compose start
```

#### 重新配置Harbor

```bash
docker-compose down -v
vim harbor.yml
# 生成配置
sudo ./prepare
# 或者生成配置的同时安装trivy
sudo ./prepare --with-trivy
docker-compose up -d
```


#### 其他命令

删除Harbor的容器，但将所有图像数据和Harbor数据库文件保留在文件系统中：

```bash
docker-compose down -v
```

在执行干净的重新安装之前，请删除Harbor数据库和镜像数据：

```bash
rm -rf /data/database
rm -rf /data/registry
rm -rf /data/redis
```

### 自定义Harbor的令牌服务

Harbor有自己的私钥和证书，如果你要替换成自己的，那么就参考这里 [Harbor docs | Customize the Harbor Token Service (goharbor.io)](https://goharbor.io/docs/2.9.0/install-config/customize-token-service/)

### Harbor配置

某些Harbor配置是与  [[#配置Harbor YML文件]] 分开配置的。您可以通过HTTP请求或使用环境变量更改Harbor接口中的配置。本页介绍了可用的配置项，以及如何使用命令行或环境变量更新配置。

[Harbor docs | Harbor Configuration (goharbor.io)](https://goharbor.io/docs/2.9.0/install-config/configure-system-settings-cli/)

### Harbor的API

[Harbor docs | View the Harbor REST API (goharbor.io)](https://goharbor.io/docs/2.9.0/working-with-projects/using-api-explorer/)