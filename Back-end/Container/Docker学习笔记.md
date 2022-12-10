# Docker学习笔记

## Cheat Sheet

![[a78bddf85124d00525a73f382c3db45.png]]

### 容器网络

[Container Networking Is Simple! (iximiuz.com)](https://iximiuz.com/en/posts/container-networking-is-simple/)

TODO

## ubuntu安装

```bash
sudo apt-get update && sudo apt-get install -y apt-transport-https ca-certificates curl software-properties-common && curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add - && sudo add-apt-repository "deb [arch=amd64] http://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable" && sudo apt-get update && sudo apt-get install -y docker-ce
```

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