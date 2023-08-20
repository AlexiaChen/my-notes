因为Sentry要Docker和Docker compose。而且公司的AWS环境居然是yum环境，看了下`cat /etc/os-release` 居然是Red hat 7.7 我用Docker的官方repo，用yum 的方式安装不了，所以只有手动下载yum的rpm包来下载。[Index of linux/centos/7/x86_64/stable/ (docker.com)](https://download.docker.com/linux/centos/7/x86_64/stable/) 

```bash
sudo yum install ./containerd.io-1.6.20-3.1.el7.x86_64.rpm

sudo yum install docker-ce-cli-24.0.1-1.el7.x86_64.rpm docker-com  
pose-plugin-2.19.1-1.el7.x86_64.rpm docker-buildx-plugin-0.11.1-1.el7.x86_64.rpm

sudo yum install docker-ce-rootless-extras-24.0.1-1.el7.x86_64.rp  
m docker-ce-24.0.1-1.el7.x86_64.rpm
```

```txt
docker-ce-cli-24.0.1-1.el7.x86_64.rpm  
containerd.io-1.6.20-3.1.el7.x86_64.rpm 
docker-ce-rootless-extras-24.0.1-1.el7.x86_64.rpm  
docker-buildx-plugin-0.11.1-1.el7.x86_64.rpm 
docker-compose-plugin-2.19.1-1.el7.x86_64.rpm  
docker-ce-24.0.1-1.el7.x86_64.rpm
```


```bash
sudo systemctl start docker
```

这样就可以使用Docker了。

还有就是对于Sentry这种强依赖Docker的服务，建议Docker和Docker compose都安装成尽可能最新的。因为在self host的Sentry安装过程中，我出现了:

```txt
container for service "postgres" is unhealthy
Error in install/set-up-and-migrate-database.sh:12.
'$dcr web upgrade' exited with status 1
```

运行多次未果，就是看日志就是postgres的容器不健康，就是启动起来有问题，或者直接没启动起来。