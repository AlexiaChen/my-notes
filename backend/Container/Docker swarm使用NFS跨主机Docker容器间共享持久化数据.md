可以解决docker swarm集群下容器的漂移共享不到历史数据问题。（因为区块链的节点容器不是无状态的服务，容器一旦漂移到另外的swarm worker节点上去，以前存在本地LevelDB的历史数据就找不到了，使用上会出问题）

这种解决办法都是需要外挂一个存储。把状态放到一个地方共享，这个地方可以是NFS，也可以是分布式文件系统，也可以是AWS提供的存储服务。我想这点上k8s提供的比较强大，这里不做讨论，我也暂时不会k8s。

没有采用Docker最推荐的方式volume。而是使用了bind。比较笨，主要是使用volume报出了/var/lib/docker/volumes/nfs_volume/的权限问题，时间紧急，一下子没搞明白，就直接用了bind方式。

在特定节点上搭建NFS服务器，其他的swarm集群节点上搭建NFS客户端，然后df -h查看以下是否有
[nfs server ip]:/[nfs server root dir] 20G   19G  1.3G  94% /mnt/[client mount point] 这样的项就表示成功了。

可以先自己写个不停写文件的程序，用Dockerfile打包成镜像，测试下，然后docker run bind到客户端挂载点：

```bash
docker run -d  --mount type=bind,src=/mnt/[client mount point],dst=/home/[data-dir] [image id or tag name]
```
如果可以了，那么swarm service create的go API也是与命令行接口（--mount 参数）保持一致的。

```go
serviceSpec := swarm.ServiceSpec{
		Annotations: swarm.Annotations{
			Name:   serviceName,
			Labels: map[string]string{},
		},
		TaskTemplate: swarm.TaskSpec{
			ContainerSpec: swarm.ContainerSpec{
				Image:    cfg.Image,
				Hostname: serviceName,
				Args:     strings.Split(args, " "),
				Mounts: []mount.Mount{
					{
						Type:   mount.TypeBind,
						Source: cfg.NFSBindSource,
						Target: target,
					},
				},
			},
			Networks: []swarm.NetworkAttachmentConfig{
				{Target: cfg.Overlay},
			},
		},
	}
	res, err := cli.ServiceCreate(context.Background(), serviceSpec, types.ServiceCreateOptions{})

```
**参考**

- https://www.cnblogs.com/Honeycomb/p/10148715.html
- https://docs.docker.com/engine/reference/commandline/service_create/
- https://docs.docker.com/storage/volumes/
- https://godoc.org/github.com/docker/docker/api/types/swarm#ServiceMode
- https://docs.docker.com/engine/reference/builder/
- http://collabnix.com/docker-1-12-swarm-mode-persistent-storage-using-nfs/