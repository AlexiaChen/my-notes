# Docker学习笔记

## Cheat Sheet

![[a78bddf85124d00525a73f382c3db45.png]]

### 容器网络

[Container Networking Is Simple! (iximiuz.com)](https://iximiuz.com/en/posts/container-networking-is-simple/)

TODO

#### Dockerfile中的EXPOSE指令

> The EXPOSE instruction informs Docker that the container listens on the specified network ports at runtime. EXPOSE does not make the ports of the container accessible to the host.

以上是官方文档的解释。

EXPOSE指令暴露了指定的端口，使其仅用于容器间的通信。让我们借助一个例子来理解这一点。

假设我们有两个容器，一个nodejs应用程序和一个redis服务器。我们的node应用需要与redis服务器进行通信，这有几个原因。

为了使node应用能够与redis服务器对话，redis容器需要暴露端口。看一下官方redis镜像的Docker文件，你会看到一行`EXPOSE 6379`。这就是帮助两个容器相互通信的原因。

因此，当你的nodejs应用容器试图连接到redis容器的6379端口时，EXPOSE指令是使其成为可能的原因。

注意：为了使node应用服务器能够与redis容器通信，重要的是两个容器都在同一个docker网络中运行。

所以EXPOSE有助于容器间的通信。如果需要将容器的端口与运行该容器的主机的端口绑定，该怎么办？

将-p（小写p）作为选项传递给docker运行指令，如下所示

`docker run -p <HOST_PORT>:<CONTAINER:PORT> IMAGE_NAME`

如果docker镜像只是一个网络应用，我认为没有必要在自己的Dockerfile中使用EXPOSE指令。当你把你的Docker文件分发给其他人并让他们连接到应用程序时，你可能想使用它。

使用EXPOSE指令的一个好处是，无论你是否在运行一个多容器应用程序，它都可以帮助其他人了解应用程序监听的端口，他们只需阅读Dockerfile，而不需要翻阅你的代码。

除非你在同一台机器上运行一个多容器应用程序，否则你不应该担心这个问题。如果你需要容器间的通信，请查阅docker网络文档。[Networking overview | Docker Documentation](https://docs.docker.com/network/) 

如果你觉得docker网络概念有点复杂，不用担心，有docker compose来救你。Docker compose自己为你处理网络层，并允许你通过docker-compose.yml文件中提到的服务名称来引用其他容器。

## ubuntu安装

```bash
sudo apt-get update && sudo apt-get install -y apt-transport-https ca-certificates curl software-properties-common && curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add - && sudo add-apt-repository "deb [arch=amd64] http://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable" && sudo apt-get update && sudo apt-get install -y docker-ce
```

用`apt-key add`的方式可能有警告，过时了 [apt key - Warning: apt-key is deprecated. Manage keyring files in trusted.gpg.d instead - Stack Overflow](https://stackoverflow.com/questions/68992799/warning-apt-key-is-deprecated-manage-keyring-files-in-trusted-gpg-d-instead) 

#### 免sudo

[How can I use docker without sudo? - Ask Ubuntu](https://askubuntu.com/questions/477551/how-can-i-use-docker-without-sudo)

```bash
sudo gpasswd -a ${USER} docker
# or
sudo usermod -aG docker $USER

sudo service docker restart

# 可能后面要退出重进下shell才可以，也可能不需要。


# maybe need, maybe not
sudo chown root:docker /var/run/docker.sock
sudo chown -R root:docker /var/run/docker

```

#### Docker设置国内镜像源

创建活修改`/etc/docker/daemon.json`  添加国内镜像

```json
{
  "registry-mirrors": [
        "https://cr.console.aliyun.com/",
        "http://hub-mirror.c.163.com",
        "https://docker.mirrors.ustc.edu.cn",
        "https://mirror.ccs.tencentyun.com",
        "https://registry.docker-cn.com"
  ],
  "insecure-registries":["docker.harbor.com"]
}
```

```bash
sudo service docker restart
docker info
```

#### Docker 推送Image到Dockerhub

首先要有个账号，然后建立repo。就可以推送了。

```bash

docker build -t whenry/fedora-jboss:latest -t whenry/fedora-jboss:v2.1 .
docker login
docker push whenry/fedora-jboss:v2.1
docker push whenry/fedora-jboss:latest
docker build -t <myuser>/hellodocker:mytag .
docker tag hellodocker:v1.0.0 <myuser>/hellodocker:1.0.0
docker push <myuser>/hellodocker:1.0.0

# or
docker build -t ${IMAGE}:${VERSION} .
docker tag ${IMAGE}:${VERSION} ${IMAGE}:latest
```


#### 清理没用的容器和镜像网络和Voume

```bash
docker system prune
# or
docker container prune
docker image prune
docker network prune
docker volume prune
```

#### 杀死所有容器

```bash
containers=$(docker ps -q)  
docker kill $containers
# or
c=$(docker ps -q) && [[ $c ]] && docker kill $c
```


## 手动下载Docker hub上的image

[dockerhub - Downloading Docker Images from Docker Hub without using Docker - DevOps Stack Exchange](https://devops.stackexchange.com/questions/2731/downloading-docker-images-from-docker-hub-without-using-docker)

把这个bash保存在本地，[raw.githubusercontent.com/moby/moby/master/contrib/download-frozen-image-v2.sh](https://raw.githubusercontent.com/moby/moby/master/contrib/download-frozen-image-v2.sh)

然后运行:

```bash
./download-image.sh ./ nvidia/cuda:12.1.0-runtime-ubuntu20.04
./download-image.sh ./ ubuntu:latest
```

## Docker导出导入镜像到本机

```bash
docker save -o your-image-file.tar your-image:tag
docker load < your-image-file.tar
```