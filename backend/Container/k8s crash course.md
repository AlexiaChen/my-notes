## 什么是Kubernetes(k8s)

就是一个开源的容器编排系统，与Docker开发的swarm差不多。但是现在k8s已经是主流社区的事实标准了，让我不得不学习。
它源自于Google内部的Borg，后来由Google重新用golang开发并开源，具体历史可以去查。

为什么会有这种编排工具呢？一个是由于运行在网络上的大型业务已经逐渐从单体应用发展成了成百上千的分布式的微服务，一个就是容器的流行化，需要方便地管理大规模的容器应用，所以类似于k8s这样的管理容器的工具应运而生。

编排工具的设计直接就提供了高可用，可扩展性还有故障恢复。

## k8s提供了哪些主要的组件

这些组件的功能基本就可以把k8s用起来了。

### Pod

说到Pod之前需要提到Node的概念，这里的Node表示一台虚拟机或者物理机器这样的节点。Pod是k8s的最小调度单元，同时它可以被认为是容器之上的抽象，说人话就是，你可以认为容器就跑在Pod里面（它创建了容器的运行时环境）。这样也就意味着有这么一个抽象的中间层，k8s就提供了不同的容器运行时的插件机制，比如Docker，还可以换成其他的容器技术。反正容器的CRI标准都已经定义好了。k8s完全可以不依赖Docker。一般来说是一个应用服务（最多）跑在一个Pod中，你可以多个应用跑在一个Pod中，但是一般不推荐这样做。

每个Pod都有自己的IP地址，在一个k8s集群内的所有Pod通过VxLAN这样的大二层技术组成了一个大的二层局域网络，每个Pod都可以通过IP地址访问其他的Pod。当然，这个IP地址外部不可访问，是一个内部地址。Pod有生命周期，会死亡，容器里面的进程会崩溃比如访问越界导致Pod崩溃重启，会被调度到别的node上，这时Pod的IP地址会变化，这些IP地址不是固定的。所以这些IP地址显然不是给人使用的，所以Service这个概念诞生了。

### Service

Service在集群内就有一个“永久”的固定IP（假设集群不死的话），这才是给人用的。Service还提供了解耦Pod的功能（它与Pod的消亡和创建调度无关），你可以理解为Service是一个Pod或某几个Pod的前端稳定统一的网关入口。创建一个Service你就需要把这个Service给attach到指定的Pod上。当然，Service有几种类型，本质上来说姑且分为两种，一种是外部Service，可以让外部浏览器访问Service关联的Pod内部容器提供的服务，一种是内部Service，不允许外部访问，主要是内部沟通。比如数据库的端口服务就不允许外部访问。

![图片](https://user-images.githubusercontent.com/8574915/140706128-2539b3ca-e50c-46d7-930a-449766d63e4f.png)

上面的my-app-service-ip就是Service的固定IP地址，也就是所谓的Service的ClusterIP。

上面提到的外部类型的service可能叫法不专业，一般来说就是k8s中LoadBalancer类型的Service了，但是你会发现，如果你的多个服务都需要暴露给外部访问，那么你就需要创建多个对应的LoadBalancer类型的Service，多么不方便啊，而且每个LoadBalancer Service的IP都不同，太麻烦了。这样Ingress这个组件就应运而生了。

### Ingress

你可以使用一个Ingress，它位于这些服务的前面，进行路由选择。这样就不用创建LoadBalancer的Service了，以下就是Ingress的yaml配置例子

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: test-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - http:
      paths:
      - path: /cart
        backend:
          serviceName: cart
          servicePort: 80
     - path: /payment
        backend:
          serviceName: payment
          servicePort: 80
```

以上一看就明白，你就可以在一个IP下的Ingress通过路由选择各个服务了。用http://ingress-ip/cart 和 http://ingress-ip/payment 访问cart和payment服务。简而言之就是更人性化了。

### ConfigMaps

其实就是用yaml配置的一堆键值对的配置项，专门给Pod内部的容器用的，为了避免修改配置跟着修改容器镜像，又重新commit一个新的到类似于Dockerhub的镜像仓库。配置Pod的yaml可以从ConfigMap里面取得相关的配置。简单来说就是给Pod存放外部配置的东西了，注意，不要存储密码，用户名相关的认证之类的配置。更推荐用后面提到的Secrets。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: game-demo
data:
  # property-like keys; each key maps to a simple value
  player_initial_lives: "3"
  ui_properties_file_name: "user-interface.properties"

  # file-like keys
  game.properties: |
    enemy.types=aliens,monsters
    player.maximum-lives=5    
  user-interface.properties: |
    color.good=purple
    color.bad=yellow
    allow.textmode=true    
```

### Secrets

类似ConfigMaps，只是存放一些比较敏感的配置的。值类型一般存为Base64编码的配置。这些键值对，在创建Pod的时候都可以引用为环境变量的值或者commandline什么的。

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: bootstrap-token-5emitj
  namespace: kube-system
type: bootstrap.kubernetes.io/token
data:
  auth-extra-groups: c3lzdGVtOmJvb3RzdHJhcHBlcnM6a3ViZWFkbTpkZWZhdWx0LW5vZGUtdG9rZW4=
  expiration: MjAyMC0wOS0xM1QwNDozOToxMFo=
  token-id: NWVtaXRq
  token-secret: a3E0Z2lodnN6emduMXAwcg==
  usage-bootstrap-authentication: dHJ1ZQ==
  usage-bootstrap-signing: dHJ1ZQ==
```

### Volumes

这个概念跟Docker里面的Volume类似了，都是为了持久化Pod内部容器生成的数据的，因为Pod重启，崩溃什么的，Pod里面的数据就会消失，所以需要通过Volumes这样的机制给Pod外挂一个存储。这个存储可以是一个本地的物理存储，也可以是NFS之类的远程外部存储，当然也还可以是Amazon之类提供的云存储。

反正记住，k8s是不管理数据持久化的，这都需要运维人员自己操心备份数据之类的。


### Deployments

虽然使用Pod就可以部署应用了，但是如果在分布式微服务环境下，一些无状态的应用服务，为了扩展提高性能，会Replicate 多个Pod的副本来处理，我们不可能一个一个创建，太累了。所以它是管理应用Pods的一个设计蓝图。如果说Pod是容器的一个抽象层，那么Deployments就是Pods的抽象层，你可以通过Deployments提升Pod Replica的数量也可以降低。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

所以，从实践上来看，以后大部分使用的都是Deployment，而不是Pod，你可以认为Deployment是k8s提供出来实现高可用的一个组件。

注意：这里Replicate的一般是无状态的应用服务，不可以是数据库的，因为状态都存储在数据库上了，如果数据库也弄出多个Replica副本，那就会导致数据库数据不一致的问题。所以数据库一般不用Deployment来部署。如果非要用Deployment部署多个数据库Pod副本，那么必须让这些Pods共享一个外挂存储，比如通过NFS挂到同一个物理存储上。所以，后面的Stateful Set就为此应运而生了。

### Stateful Set

可以理解为StatefulSet就是为MySQL mongoDB，ElasticSearch这样有状态的数据服务而生的特殊的Deployment了。所以它也可以scale数据服务的replica数量，但是这些replica后的这组Pods，StatefulSets会为这组Pods提供排序和唯一性保证。这些Pod是s根据相同的规格创建的，但不能互换：每个Pod都有一个“永久”的标识符，它在任何重新组织中（比如Pod重启，这个永久标识符还会被分配给新的Pod）都保持不变。利用这个标识符，就可以与Volumes更好匹配了。

当然，你可以直接规避此问题，就是把这些数据库服务，全部搬到k8s集群外面，不受k8s管理。所以在容器刚开始流行的时候，就有一句话，一般更推荐容器化的是无状态应用。即使是现在，专门为云原生设计的数据库也没有多少，这里推荐一篇Google的文章[《To run or not to run a database on Kubernetes: What to consider》](https://cloud.google.com/blog/products/databases/to-run-or-not-to-run-a-database-on-kubernetes-what-to-consider)，里面有提到什么样的数据库更适合上k8s。

![图片](https://user-images.githubusercontent.com/8574915/140858422-bbe7a6e8-e501-4e41-ac84-982e08f89bc7.png)

## k8s的架构

### worker node

这里提两个概念，master node和workder node，与Docker swarm类似。之前提到的node一般就是指workder node了，上面是主要运行应用任务的Pods的，可以运行很多Pods，因为本来就是做实际工作的。workder node上至少跑了（安装）三个进程。一个k8s集群里面有成千上百个workder node。

- 容器运行时进程，创建Pod的时候就需要用到了，或者你用到了其他除Docker外容器运行时技术
- kubelet，k8s本身的node进程，用来处理node和容器之间交互的进程。启动Pod就是kubelet调用的了，比如分配CPU等一些资源给容器
- kube proxy。用来转发service到关联Pods的请求的，因为之前提到过了，Pods组成的大二层网络里面，主要是通过service入口来与各组Pods沟通的。当然，它还做了一点点负载均衡的功能。

### master node

与集群交互，新节点加入集群，Pod在workder nodes之间的调度，重启等主要是管理集群的功能的进程就在master node上了。

master node上至少跑了（安装）了四个进程：

- API server， client会与它通信，请求API server，client就有很多了，Web的，commandline形式的都有，比如k8s dashboard，kubectl，k8s API这种可编程的接口。在部署应用到k8s集群的时候都是通过API server这个网关入口。API server就会把相关请求转发到其他相关的进程上。
- scheduler，Pod调度器，比如通过kubectl调度一个新Pod到集群内，那么API server其实就是发送请求给scheduler了。scheduler自然会选择一个worker node上启动一个Pod。把Pod放到哪里，都是scheduler的事情，所以scheduler有很多决策策略，它需要知道各个workder node的负载状况，而进行Pod的恰当调度。至于实际启动Pod，是workder node上的kubelet这个节点代理进程。
- controller manager，检测集群的状态变化，比如发Pods的崩溃，它就需要尽快收集这些Pods的信息发送给scheduler重新启动Pods。这样周而复始。
- etcd，这个进程很多人即使没有接触过k8s的人都听过，它就是k8s集群的大脑，它存储集群的各种状态，用了boltDB这个B tree的存储引擎。其他很多进程都需要根据etcd保存的数据进行决策。

### 一个集群的例子

一个k'8s集群可能有好几个master node，这里提高了集群的可用性，避免master node出现单点故障。master node的机器可以不太用高性能的，workder node因为是做实际事情的，需要用性能高一点的机器。workder node越多，master node一般就要与之增多，一个增加可用性，一个是增加了API server的处理能力。生产环境肯定是多个master node的，至少2个。

## 使用Minikube搭建本地k8s集群

主要是学习和测试用。

下载安装minikube:  https://minikube.sigs.k8s.io/docs/start/

注意，要启动Docker的守护进程 `sudo service docker start`或者其他类似的虚拟化容器技术（VirtualBox，Podman，KVM，VMware），不然minikube会启动失败。

启动minikube

```bash
minikube start
```

然后它会下载一些组件和镜像，最后成功。记得翻墙。

```txt
mathxh@MathxH:~$ minikube start
😄  minikube v1.24.0 on Ubuntu 20.04 (amd64)
✨  Using the docker driver based on existing profile
❗  Local proxy ignored: not passing HTTP_PROXY=http://127.0.0.1:10809 to docker env.
❗  Local proxy ignored: not passing HTTPS_PROXY=http://127.0.0.1:10809 to docker env.
👍  Starting control plane node minikube in cluster minikube
🚜  Pulling base image ...
💾  Downloading Kubernetes v1.22.3 preload ...
    > gcr.io/k8s-minikube/kicbase: 0 B [______________________] ?% ? p/s 14m56s
    > preloaded-images-k8s-v13-v1...: 144.11 MiB / 501.73 MiB  28.72% 164.36 Ki
    > index.docker.io/kicbase/sta...: 355.77 MiB / 355.78 MiB  100.00% 557.12 K
❗  minikube was unable to download gcr.io/k8s-minikube/kicbase:v0.0.28, but successfully downloaded docker.io/kicbase/stable:v0.0.28 as a fallback image
🤷  docker "minikube" container is missing, will recreate.
🔥  Creating docker container (CPUs=2, Memory=2200MB) ...
❗  Local proxy ignored: not passing HTTP_PROXY=http://127.0.0.1:10809 to docker env.
❗  Local proxy ignored: not passing HTTPS_PROXY=http://127.0.0.1:10809 to docker env.
❗  Local proxy ignored: not passing HTTP_PROXY=http://127.0.0.1:10809 to docker env.
❗  Local proxy ignored: not passing HTTPS_PROXY=http://127.0.0.1:10809 to docker env.
🌐  Found network options:
    ▪ http_proxy=http://127.0.0.1:10809
❗  You appear to be using a proxy, but your NO_PROXY environment does not include the minikube IP (192.168.49.2).
📘  Please see https://minikube.sigs.k8s.io/docs/handbook/vpn_and_proxy/ for more details
    ▪ https_proxy=http://127.0.0.1:10809
❗  This container is having trouble accessing https://k8s.gcr.io
💡  To pull new external images, you may need to configure a proxy: https://minikube.sigs.k8s.io/docs/reference/networking/proxy/
    > kubectl.sha256: 64 B / 64 B [--------------------------] 100.00% ? p/s 0s
    > kubelet.sha256: 64 B / 64 B [--------------------------] 100.00% ? p/s 0s
    > kubeadm.sha256: 64 B / 64 B [--------------------------] 100.00% ? p/s 0s
    > kubectl: 44.73 MiB / 44.73 MiB [-----------] 100.00% 41.04 KiB p/s 18m36s
    > kubeadm: 43.71 MiB / 43.71 MiB [-----------] 100.00% 39.94 KiB p/s 18m41s
    > kubelet: 115.57 MiB / 115.57 MiB [----------] 100.00% 93.61 KiB p/s 21m4s

    ▪ Generating certificates and keys ...
    ▪ Booting up control plane ...
    ▪ Configuring RBAC rules ...
🔎  Verifying Kubernetes components...
    ▪ Using image gcr.io/k8s-minikube/storage-provisioner:v5
🌟  Enabled addons: default-storageclass, storage-provisioner
💡  kubectl not found. If you need it, try: 'minikube kubectl -- get pods -A'
🏄  Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default
```

看见以上消息，就意味着成功了。

然后尝试获取一下集群信息（kubectl不是minikube的一部分，要单独安装才可以直接用kubectl命令）:

```bash
mathxh@MathxH:~$ minikube kubectl -- get pods -A
NAMESPACE     NAME                               READY   STATUS    RESTARTS        AGE
kube-system   coredns-78fcd69978-dzd2v           1/1     Running   0               3m51s
kube-system   etcd-minikube                      1/1     Running   0               4m8s
kube-system   kube-apiserver-minikube            1/1     Running   0               4m8s
kube-system   kube-controller-manager-minikube   1/1     Running   1 (4m20s ago)   4m8s
kube-system   kube-proxy-rxwxl                   1/1     Running   0               3m51s
kube-system   kube-scheduler-minikube            1/1     Running   0               4m8s
kube-system   storage-provisioner                1/1     Running   1 (3m14s ago)   3m56s
mathxh@MathxH:~$ minikube kubectl -- get nodes
NAME       STATUS   ROLES                  AGE     VERSION
minikube   Ready    control-plane,master   4m36s   v1.22.3
```
OK,大功告成！

单独安装kubectl以后:

```bash
mathxh@MathxH:~$ kubectl get nodes
NAME       STATUS   ROLES                  AGE   VERSION
minikube   Ready    control-plane,master   10m   v1.22.3
mathxh@MathxH:~$ kubectl version
Client Version: version.Info{Major:"1", Minor:"22", GitVersion:"v1.22.3", GitCommit:"c92036820499fedefec0f847e2054d824aea6cd1", GitTreeState:"clean", BuildDate:"2021-10-27T18:41:28Z", GoVersion:"go1.16.9", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"22", GitVersion:"v1.22.3", GitCommit:"c92036820499fedefec0f847e2054d824aea6cd1", GitTreeState:"clean", BuildDate:"2021-10-27T18:35:25Z", GoVersion:"go1.16.9", Compiler:"gc", Platform:"linux/amd64"}
```

好了，后面我都会有kubectl操作，minikube只是启动，删除，停止k8s集群的工具，操作集群一般用kubectl。

### kubectl的基本命令

- `kubectl get nodes`,  获得所有节点信息
- `kubectl get pods`， 获得所有pods信息
- `kubectl get services`， 获得所有service信息
- `kubectl create xxxxx`, 创建k8s各种组件，你会发现没有Pod，因为之前提到过了，我们不直接与Pod打交道，它比较底层，我们一般用Deployment间接与Pod打交道。

下面我们来创建一个Nginx的deployment。

```bash
mathxh@MathxH:~$ kubectl create deployment nginx-deploy --image=nginx
deployment.apps/nginx-deploy created
mathxh@MathxH:~$ kubectl get deployments
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deploy   0/1     1            0           72s
mathxh@MathxH:~$ kubectl get pod
NAME                           READY   STATUS              RESTARTS   AGE
nginx-deploy-8588f9dfb-j9z2n   0/1     ContainerCreating   0          2m45s
```

上面的信息表示还没有创建好。因为要从Docker hub拉取nginx的latest镜像。

直到等到下面信息显示，就说明成功, 同时你还会发现这个Pod只有一个replica:

```bash
mathxh@MathxH:~$ kubectl get deployments
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deploy   1/1     1            1           13m
mathxh@MathxH:~$ kubectl get pod
NAME                           READY   STATUS    RESTARTS   AGE
nginx-deploy-8588f9dfb-j9z2n   1/1     Running   0          14m
mathxh@MathxH:~$ kubectl get replicaset
NAME                     DESIRED   CURRENT   READY   AGE
nginx-deploy-8588f9dfb   1         1         1       15m
```

如果你想对deployment做配置变更，可以通过编译dump出的yaml文件进行:

```bash
kubectl edit deployment nginx-deploy
```

这样会自动用vi或vim打开一个yaml文件供你编辑。

```bash
mathxh@MathxH:~$ kubectl edit deployment nginx-deploy

deployment.apps/nginx-deploy edited
mathxh@MathxH:~$ kubectl get deployments
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deploy   1/2     2            1           25m
mathxh@MathxH:~$ kubectl get deployments
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deploy   1/2     2            1           25m
mathxh@MathxH:~$ kubectl get deployments
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deploy   2/2     2            2           25m
mathxh@MathxH:~$ kubectl get pod
NAME                           READY   STATUS    RESTARTS   AGE
nginx-deploy-8588f9dfb-j9z2n   1/1     Running   0          25m
nginx-deploy-8588f9dfb-nlc6h   1/1     Running   0          24s
```

我重新编辑replica为2，所以就启动了2个nginx-deploy的Pod。

最后可以`kubectl delete deployment nginx-deploy` 来删除该deployment

### 调试Pod

使用`kubectl logs <pod-name>` 可以看Pod里面容器内的应用打印出的日志, 这样就可以看日志错误了:

```bash
mathxh@MathxH:~$ kubectl logs nginx-deploy-8588f9dfb-j9z2n
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: info: Getting the checksum of /etc/nginx/conf.d/default.conf
10-listen-on-ipv6-by-default.sh: info: Enabled listen on IPv6 in /etc/nginx/conf.d/default.conf
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/30-tune-worker-processes.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
2021/11/10 03:44:09 [notice] 1#1: using the "epoll" event method
2021/11/10 03:44:09 [notice] 1#1: nginx/1.21.4
2021/11/10 03:44:09 [notice] 1#1: built by gcc 10.2.1 20210110 (Debian 10.2.1-6)
2021/11/10 03:44:09 [notice] 1#1: OS: Linux 5.10.60.1-microsoft-standard-WSL2
2021/11/10 03:44:09 [notice] 1#1: getrlimit(RLIMIT_NOFILE): 1048576:1048576
2021/11/10 03:44:09 [notice] 1#1: start worker processes
2021/11/10 03:44:09 [notice] 1#1: start worker process 31
2021/11/10 03:44:09 [notice] 1#1: start worker process 32
2021/11/10 03:44:09 [notice] 1#1: start worker process 33
2021/11/10 03:44:09 [notice] 1#1: start worker process 34
2021/11/10 03:44:09 [notice] 1#1: start worker process 35
2021/11/10 03:44:09 [notice] 1#1: start worker process 36
2021/11/10 03:44:09 [notice] 1#1: start worker process 37
2021/11/10 03:44:09 [notice] 1#1: start worker process 38
```

还可以用`kubectl describe pod <pod-name>`查看显示出来的Events那栏，查看k8s做了哪些事件，卡在了哪个事件上:

```bash
 kubectl describe pod nginx-deploy-8588f9dfb-j9z2n

...

Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  39m   default-scheduler  Successfully assigned default/nginx-deploy-8588f9dfb-j9z2n to minikube
  Normal  Pulling    39m   kubelet            Pulling image "nginx"
  Normal  Pulled     31m   kubelet            Successfully pulled image "nginx" in 7m57.2982567s
  Normal  Created    31m   kubelet            Created container nginx
  Normal  Started    31m   kubelet            Started container nginx
```

你最后还可以用类似docker exec的命令`kubectl exec -it <pod-name> -- /bin/bash`进入到Pod的容器内:

```bash
mathxh@MathxH:~$ kubectl exec -it nginx-deploy-8588f9dfb-j9z2n -- /bin/bash
root@nginx-deploy-8588f9dfb-j9z2n:/# ls
bin  boot  dev  docker-entrypoint.d  docker-entrypoint.sh  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
```

实在不行就看网上的deployment排错指南：

![mmexport1637683176925.jpg](https://user-images.githubusercontent.com/8574915/143059599-7b03ac33-d24b-4024-b367-8f5df83c80b6.jpg)

![mmexport1637683169733.jpg](https://user-images.githubusercontent.com/8574915/143059672-4d67dcb1-e33d-429a-9c70-826a1d021d21.jpg)

下面会列举一个Nginx deployment的yaml配置：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.16
        ports:
        - containerPort: 80
```

以上就是一个nginx deployment的配置样例，第一个spec是deployment，第二个spec项是关联在该Deployment的Pods的配置。

对于以上yaml文件，我们可以通过`kubectl apply -f <yaml-file>` 来应用更新配置或者通过`kubectl delete -f <yaml-file>来删除关于这个配置的服务`，以后这些配置在实际工作中都是需要提交到repo中的。k8s主要用法几乎都可以用配置来完成，所以也有一种说法就是，k8s是一个容器的操作系统，以后都是针对配置来编程。

### k8s的yaml配置文件

首先解释下，无论是用yaml还是CLI，API等方式与k8s交互，都需要使用k8s 对象，这个对象就是Service，Deployment这些类的实体，在Yaml里是Kind项。这些对象的status是持久化到etcd中的。你需要通过cli或者API等方式创建，删除，更新对象。你在yaml里看到的第一个spec项就是对象的规约了。k8s对象（yaml配置）一般有三部分：metadata，spec，status，详情请看 https://kubernetes.io/docs/concepts/overview/working-with-objects/kubernetes-objects/ metadata你看到了，以上的配置文件，spec就是你描述的Service Deployment等的规范。status您会发现，一般你手工编写的yaml上没有啊？这是怎么回事？原因是status是k8s自动生成添加的，也就是含有status的配置才是最终的配置，k8s对我们写的yaml进行了一次转换（编译）。严格意义上说，spec部分就是你期望的对象的status，当你apply 一个yaml配置的时候，它会把该对象当前的status与yaml中的期望status做对比，如果不同，就对当前实际的status做出修改。所以yaml文件一般也是与代码一样存放在repo中的（或者跟代码存在一起），因为DevOps的流行，这些配置文件跟代码同样重要，开发和运维，部署一体化，需要共同维护，这也符合Infra as a code这样的理念。

Deployment中的spec中的template相面的labels项，是Pods的标签，template也是关于容器的模板，所以template下面也有属于自己的spec，所以template中的labels是Deployment中的spec中的selector的matchLabels匹配关联的，也是就说Deployment指定要关联的Pods。而Deployment中metadata下的labels，是Service要匹配的，Service sepc中的selector指定需要关联哪些Deployments。

所以关联的组件是这样，Service关联指定的一组Deployments，而Deloyment关联指定的一组Pods。这些组件与组件之间的链接通信就是靠labels，selector，ports这些匹配来连接的了。不明白就看以下图示：

![1](https://user-images.githubusercontent.com/8574915/143557280-4b2a2456-63bf-4135-b3fb-7c131c831fd6.jpg)

![2](https://user-images.githubusercontent.com/8574915/143557341-b8c817df-cdd0-4e7b-97da-bb137aada881.jpg)

Service中的targetPort连接的是Deployment中template的容器的containerPort：

![3](https://user-images.githubusercontent.com/8574915/143557373-b7f3f58b-6ce5-4777-8d25-0688e9f1d4f6.jpg)

![4](https://user-images.githubusercontent.com/8574915/143557496-dc6fa42d-ef66-4d53-b887-ef9e1fe038c5.jpg)

如果你想查看你的Service有没有连接正确相应的port端口，你可以`cubectl describe service <service-name>`来查看，会有相关的端口转发配置和IP显示出来，再与`cubectl get pod -o wide`显示出来的Pod的IP对比就知道了。

还有一种可以查看的方式，就是把当前某个k8s对象的当前status显示出来，以yaml的格式:

```bash
kubectl get <deployment | service | ...> <object-name> -o yaml
```

### 一个完整的Demo

其实就是搭建一个mongo-express 访问mongo-db的应用，express是一个Web App可以以网页UI的形式管理mongo-db，所以它需要请求mongo DB的服务，这个简单场景就可以部署在k8s上，让我们来搭建看看吧。

首先我们分析下，DB服务必须是放在k8s内部的，不能轻易暴露在外部，所以需要一个mongoDB的Pod 关联的Deployment和一个对接该Deployment的internal service，然后我们还需要一个mongo-express这个Web App的Pod关联的Deployment，
express 要连接mongoDB的服务，就需要一个DB的URL连接，还有User和Password，这些参数要配置化，所以还需要一个ConfigMap还有Secret。ConfigMao保存DB的URL， Secret保存DB的User和Password，之后express的Deployment的yaml配置文件里面的Env variable要引用ConfigMap里的URL，Secret中的User和Password。最后又因为express需要浏览器访问这个Web页面，所以express的Deployment需要一个关联的external service。

![}OGZIYIQU`3$CR0TA6S9 OI](https://user-images.githubusercontent.com/8574915/167296749-3cc3dfbc-abf6-4dda-a210-57c2eb55d1d1.png)

![I0P}Q3~9ILI%G@4KH@O`0EH](https://user-images.githubusercontent.com/8574915/167296817-f2b253a5-d1ab-4a61-892e-55c809a0abc9.png)

好了，设计草稿完成，Lets get started:

首先看看minikube的状态，假设它已经在运行了哈:

```bash
mathxh@MathxH:~/k8s-learn$ kubectl get all
NAME                               READY   STATUS             RESTARTS        AGE
pod/nginx-deploy-8588f9dfb-j9z2n   0/1     ImagePullBackOff   0 (3m19s ago)   179d
pod/nginx-deploy-8588f9dfb-nlc6h   0/1     Error              0               179d

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   180d

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-deploy   0/2     2            0           179d

NAME                                     DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-deploy-8588f9dfb   2         2         0       179d
```
哈哈，还有一些之前实验已经完成的Deployment和Pods，所以我们要把它删除:

```bash
kubectl delete deployment nginx-deploy
```

这样就只剩 kubernetes的service了。这次我们真正开始了, 首先创建mongoDB的deployment：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb-deployment
  labels:
    app: mongodb
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
        - name: mongodb
          image: mongo:latest
          ports:
            - containerPort: 27017
          env:
            - name: MONGO_INITDB_ROOT_USERNAME
              valueFrom:
                secretKeyRef:
                  name: mongodb-secret    #  引用Secret的配置
                  key: mongo-root-username
            - name: MONGO_INITDB_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mongodb-secret     # 引用Secret的配置
                  key: mongo-root-password
```

注意，上面的env和ports都是mongo的Docker hub官方文档上找到的，不能随意指定哈， https://hub.docker.com/_/mongo 。

至于env的用户名和密码，一般这个yaml在DevOps的体系下是要放到git的repo里面的，所以不能写在这里，所以就需要引用Secret里面的配置了，所以接下来我们要创建Secret:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mongodb-secret
type: Opaque
data:
  mongo-root-username: bWF0aHho                     # echo -n 'mathxh' | base64
  mongo-root-password: bWF0aHhoMTIzNA==             # echo -n 'mathxh1234' | base64
```
因为mongoDB的deployment需要引用这个Secret，所以要先创建Secret:

```bash
mathxh@MathxH:~/k8s-learn$ kubectl apply -f ./mongo-secret.yaml
secret/mongodb-secret created
mathxh@MathxH:~/k8s-learn$ kubectl get secret
NAME                  TYPE                                  DATA   AGE
default-token-wzcgs   kubernetes.io/service-account-token   3      180d
mongodb-secret        Opaque                                2      54s
```

最终就可以启动mongodb的Deployment了:

```bash
mathxh@MathxH:~/k8s-learn$ kubectl apply -f ./mongodb-deploy.yaml
deployment.apps/mongodb-deployment created
mathxh@MathxH:~/k8s-learn$ kubectl get deployment
NAME                 READY   UP-TO-DATE   AVAILABLE   AGE
mongodb-deployment   0/1     1            0           38s
```
看到上面的的deployment状态了嘛，还没有准备好，八成是正在拉取mongo的镜像，所以需要等一等，你可以通过 以下命令来观察确认:

```bash
mathxh@MathxH:~/k8s-learn$ kubectl get pod --watch
NAME                                  READY   STATUS              RESTARTS   AGE
mongodb-deployment-6c587ddcbb-cjt4c   0/1     ContainerCreating   0          5m38s
```
一旦容器创建好了，就OK了。

```bash
mathxh@MathxH:~/k8s-learn$ kubectl get pod
NAME                                  READY   STATUS    RESTARTS   AGE
mongodb-deployment-6c587ddcbb-cjt4c   1/1     Running   0          18m
```

接下来，我们就需要创建mongoDB Deployment对应的internal service了。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongodb-service
spec:
  selector:
    app: mongodb
  ports:
    - protocol: TCP
      port: 27017    # service port, 这两个端口可以不同，这里相同是为了方便
      targetPort: 27017  # container port of deployment
```
```bash
mathxh@MathxH:~/k8s-learn$ kubectl apply -f ./mongodb-deploy.yaml
deployment.apps/mongodb-deployment unchanged
service/mongodb-service created
```
上面显示的两行消息是因为我把mongo的Deployment和Service放一个yaml文件了，在工程上一般会这样做:

![RL9TJP$23EXAZZBRR % NKP](https://user-images.githubusercontent.com/8574915/167300212-78c07262-f752-4ce4-8e56-326691034db9.png)

```bash
mathxh@MathxH:~/k8s-learn$ kubectl get service
NAME              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)     AGE
kubernetes        ClusterIP   10.96.0.1       <none>        443/TCP     180d
mongodb-service   ClusterIP   10.110.27.119   <none>        27017/TCP   2m20s
mathxh@MathxH:~/k8s-learn$ kubectl describe service mongodb-service
Name:              mongodb-service
Namespace:         default
Labels:            <none>
Annotations:       <none>
Selector:          app=mongodb
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                10.110.27.119
IPs:               10.110.27.119
Port:              <unset>  27017/TCP
TargetPort:        27017/TCP
Endpoints:         172.17.0.3:27017
Session Affinity:  None
Events:            <none>
mathxh@MathxH:~/k8s-learn$ kubectl get pod -o wide
NAME                                  READY   STATUS    RESTARTS   AGE   IP           NODE       NOMINATED NODE   READINESS GATES
mongodb-deployment-6c587ddcbb-cjt4c   1/1     Running   0          28m   172.17.0.3   minikube   <none>           <none>
```

从上面的消息可以看到，mongoDB的service已经关联到正确的Pod上了，看Endpoints是否与Pod的IP一致。

接下来我们就可以创建mongo express这个Web App相关的 Deployment， ConfigMap，还有external service了。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mongodb-configmap
data:
  database_url: mongodb-service  # 直接引用mongoDB的internal service的name
```

```bash
mathxh@MathxH:~/k8s-learn$ kubectl apply -f ./mongo-configmap.yaml
configmap/mongodb-configmap created
mathxh@MathxH:~/k8s-learn$ kubectl get configmap
NAME                DATA   AGE
kube-root-ca.crt    1      180d
mongodb-configmap   1      8s
```
这下就OK了，后面就是mongo express的Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb-express-deployment
  labels:
    app: mongodb-express
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongodb-express
  template:
    metadata:
      labels:
        app: mongodb-express
    spec:
      containers:
        - name: mongodb-express
          image: mongo-express
          ports:
            - containerPort: 8081    # 来自于Docker hub官方镜像的文档
          env:
            - name: ME_CONFIG_MONGODB_ADMINUSERNAME
              valueFrom:
                secretKeyRef:
                  name: mongodb-secret
                  key: mongo-root-username
            - name: ME_CONFIG_MONGODB_ADMINPASSWORD
              valueFrom:
                secretKeyRef:
                  name: mongodb-secret
                  key: mongo-root-password
            - name: ME_CONFIG_MONGODB_SERVER
              valueFrom:
                configMapKeyRef:
                  name: mongodb-configmap
                  key: database_url
```
```bash
mathxh@MathxH:~/k8s-learn$ kubectl apply -f ./mongo-express.yaml
deployment.apps/mongodb-express-deployment created
mathxh@MathxH:~/k8s-learn$ kubectl get pod --watch
NAME                                          READY   STATUS              RESTARTS   AGE
mongodb-deployment-6c587ddcbb-cjt4c           1/1     Running             0          50m
mongodb-express-deployment-5c4b6d7cb8-7tc65   0/1     ContainerCreating   0          9s
```
像之前一样耐心等待express的镜像拉取吧。

OK 就可以看看:

```bash
mathxh@MathxH:~/k8s-learn$ kubectl logs mongodb-express-deployment-5c4b6d7cb8-7tc65
Welcome to mongo-express
------------------------


(node:8) [MONGODB DRIVER] Warning: Current Server Discovery and Monitoring engine is deprecated, and will be removed in a future version. To use the new Server Discover and Monitoring engine, pass option { useUnifiedTopology: true } to the MongoClient constructor.
Mongo Express server listening at http://0.0.0.0:8081
Server is open to allow connections from anyone (0.0.0.0)
basicAuth credentials are "admin:pass", it is recommended you change this in your config.js!
```
mongo express的Deployment的port就监听在8081上，跟yaml里面配置的一样。然后我们需要外部浏览器来访问这个express的服务，最后我们就需要创建一个externa service了。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongodb-express-service
spec:
  selector:
    app: mongodb-express
  type: LoadBalancer # for external service， but internal service is also act as load balancer
  ports:
    - protocol: TCP
      port: 8081
      targetPort: 8081
      nodePort: 30000 #  for external service, port for external ip address ,you need put it into your browser. must be 30000 to 32767
```

```bash
mathxh@MathxH:~/k8s-learn$ kubectl apply -f ./mongo-express.yaml
deployment.apps/mongodb-express-deployment unchanged
service/mongodb-express-service created
mathxh@MathxH:~/k8s-learn$ kubectl get service
NAME                      TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
kubernetes                ClusterIP      10.96.0.1       <none>        443/TCP          180d
mongodb-express-service   LoadBalancer   10.96.244.156   <pending>     8081:30000/TCP   66s
mongodb-service           ClusterIP      10.110.27.119   <none>        27017/TCP        52m
mathxh@MathxH:~/k8s-learn$ kubectl describe service mongodb-express-service
Name:                     mongodb-express-service
Namespace:                default
Labels:                   <none>
Annotations:              <none>
Selector:                 app=mongodb-express
Type:                     LoadBalancer
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.96.244.156
IPs:                      10.96.244.156
Port:                     <unset>  8081/TCP
TargetPort:               8081/TCP
NodePort:                 <unset>  30000/TCP
Endpoints:                172.17.0.4:8081
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
mathxh@MathxH:~/k8s-learn$ kubectl get pod -o wide
NAME                                          READY   STATUS    RESTARTS   AGE   IP           NODE       NOMINATED NODE   READINESS GATES
mongodb-deployment-6c587ddcbb-cjt4c           1/1     Running   0          75m   172.17.0.3   minikube   <none>
  <none>
mongodb-express-deployment-5c4b6d7cb8-7tc65   1/1     Running   0          25m   172.17.0.4   minikube   <none>
  <none>
```

根据以上命令你会发现，interal service和external service的区别就在于，external service会多配一个external ip，这两种service都有ClusterIP这样的内部IP。

然后就可以通过minikube的命令来打开这个service，默认会用浏览器打开:

```bash
mathxh@MathxH:~/k8s-learn$ minikube service mongodb-express-service

|-----------|-------------------------|-------------|---------------------------|
| NAMESPACE |          NAME           | TARGET PORT |            URL            |
|-----------|-------------------------|-------------|---------------------------|
| default   | mongodb-express-service |        8081 | http://192.168.49.2:30000 |
|-----------|-------------------------|-------------|---------------------------|
🏃  Starting tunnel for service mongodb-express-service.
|-----------|-------------------------|-------------|------------------------|
| NAMESPACE |          NAME           | TARGET PORT |          URL           |
|-----------|-------------------------|-------------|------------------------|
| default   | mongodb-express-service |             | http://127.0.0.1:38077 |
|-----------|-------------------------|-------------|------------------------|
🎉  Opening service default/mongodb-express-service in default browser...
✋  Stopping tunnel for service mongodb-express-service.
```
![$)(MUCT9Y`NEDP$%2HP(A(3](https://user-images.githubusercontent.com/8574915/167302647-67468840-624d-4108-94da-325646144725.png)

![@O8@MMH _VG$HUM$J2O~`6T](https://user-images.githubusercontent.com/8574915/167303114-8fa82596-bb78-420a-8e31-d2daf1339f33.png)

我可以在页面上那个create database按钮那里创建一个新的DB，然后可以查看这个DB的状态和信息。也就是到此为止，我们正常使用了跑在k8s上的这个Demo工程。

有没有发现，上面的地址和端口有点奇怪，因为我用的是windows的WSL2，如果是本机就是Linux Mac什么的，访问的端口就是30000。

以上命令才会真正分配一个外部IP，不然就是pending状态。关于这个mongo的Demo项目就讲完了，完整的yaml配置可以在这里找到： https://github.com/weechien/mongo-express-k8s

### k8s namespaces的讲解

什么是namespace？

- namespace就是在k8s集群组织资源的一种方式
- 你可以把它想象成在一个k8s集群中的虚拟集群（集群中的集群）

![~66_8NXK)`6ALZAE`@(ZC(R](https://user-images.githubusercontent.com/8574915/167849806-6a935a15-cb29-41e8-9b05-a84b02e83e64.png)


默认情况下，一个k8s集群内有4个默认的namespaces。

```bash
mathxh@MathxH:~/Project/telecom-tools$ kubectl get namespace
NAME              STATUS   AGE
default           Active   183d
kube-node-lease   Active   183d
kube-public       Active   183d
kube-system       Active   183d
```

- kube-system，  这个名字空间下的东西不是直接给用户用的，所以一般不要创建或修改该名字空间。里面有一些系统进程，Master和kubectl的进程。
- kube-public,  里面有公开可访问的数据，一个ConfigMap，这一个组件里面包含这个集群的一些信息

```bash
mathxh@MathxH:~/Project/telecom-tools$ kubectl cluster-info
Kubernetes control plane is running at https://127.0.0.1:49154
CoreDNS is running at https://127.0.0.1:49154/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

比如以上命令的返回信息就是从kube-public获取的。

- kube-node-lease,  记录集群内节点的心跳，在这个namespace中，每个节点都有关联的lease对象(租约对象)，lease是分布式系统中的一种资源租赁的概念。还有决定一个节点是否可用
- default, 如果你新建的资源没有指定新的namespace，那么这些资源默认都会创建在这个namespace。

你可以用`kubectl create namespace <namespace-name>` 来新建一个namespace。

```bash
mathxh@MathxH:~/Project/telecom-tools$ kubectl create namespace mathxh
namespace/mathxh created
mathxh@MathxH:~/Project/telecom-tools$ kubectl get namespace
NAME              STATUS   AGE
default           Active   183d
kube-node-lease   Active   183d
kube-public       Active   183d
kube-system       Active   183d
mathxh            Active   13s
```

当然，最推荐的做法还是在yaml配置中指定namespace，它自己会新建。

![R3ZSF2F29KEF IW%PE}~{NI](https://user-images.githubusercontent.com/8574915/167849859-668eba37-2c42-404e-b0e2-d226e7999532.png)

为什么需要使用命名空间呢？你想象一下，如果你不指定名字空间，那么新建的这些组件（ConfigMap，Deployment，Service等）都默认会新建到default 这个namespace中，那么多么杂乱啊，不好管理，所以要分割区分。就像各种编程语言的模块一样。

正确的做法就是，比如，你有不同的系统在一个集群里面，那么数据库就可以给数据库建立一个namespace，monitor监控就可以建立一个属于监控的namespace，以此类推。

![60)QQOV2@3U~22S(740UI$U](https://user-images.githubusercontent.com/8574915/167851045-d77dcb26-837e-4a6d-9e54-bf65b8cd6966.png)

当然，如果是小项目就不需要搞那么多namespace了。

还有一种用namespace的动机是，多团队，如果多个团队在使用一个k8s集群，那么如果不小心创建相同名字的组件资源，如果在同一个namespace下那么就会冲突，后apply的资源会把之前的资源覆盖掉。而且权限等资源限制也是可以指定namespace的，比如指定团队A，只能访问自己的namespace A，它只可以在自己的namespace创建更新组件。

![JL}HHYZ B9N%XN}R`IH%9@5](https://user-images.githubusercontent.com/8574915/167852076-8e67ec7f-d451-41df-9780-bf6f29fcb8bc.png)

最后就是组件之间的复用，这样就方便做staging dev，灰度测试发布等场景。也可以在指定的namespace上面限制CPU,RAM Storage等资源的使用，跟Docker一样。这些是用Resouece Quota指定的。

#### namespace的一些特点

- 大部分情况下，其他namespace的大部分类型的resource不可以访问本namespace下的resource，比如namespace B的资源，不可以访问namespace A下的ConfigMap。但是你可以为不同的namespace分别创建相同的ConfigMap。当然，对于Secret也是一样的

![DP8~E)9%938R2O901B ICF4](https://user-images.githubusercontent.com/8574915/167854178-194ee08d-8531-4653-b485-4a797773583e.png)

- 可以访问其他namespace下的Service类型的resource

![XF89 RR9 ~449 WSY`UDAEM](https://user-images.githubusercontent.com/8574915/167854693-7671962b-6e61-466a-acaf-512a70ee3351.png)

- 还有一些resource，你不能创建在一个namespace下，因为是整个集群，全局共享。比如Volume和Node。

![C A9GH)2QT`IZW5B}NFHJS6](https://user-images.githubusercontent.com/8574915/167855063-17803d21-e031-48c6-b00d-aa25f28865b9.png)

查看特定namepsace下的resources: `kubectl get <configmap | service | deployment | ...> -n <namesapce-name>`
创建： `kubectl apply -f <yaml-file-path> --namespace=<namespace-name>`
查看特定resource属于哪个namespace: `kubectl get <configmap | service |  ....> <resouece-name> -o yaml`

如果你想把默认的这个default namespace切换成其他的namespace，那么可以用kubens这个工具来做， Active状态就是当前工作活跃的NS了，这样以后，你如果创建 修改的资源，就默认会在切换了的NS上进行。

### k8s Ingress详解

#### External Service VS. Ingress

![GA{X4MWLB6_@A_{KIR% BFU](https://user-images.githubusercontent.com/8574915/167858443-a5d7226a-c004-489a-a98e-15e37afa6ddb.png)


看上图你应该就可以理解了，考虑Web应用，如果你用https，那么肯定是域名访问，正式环境一般是这样的，这样就必须要Ingress了，External Service只可以做成http的IP address + Port的这种访问形式。

一般来说，在yaml文件中，kind为Service， Type为LoadBalancer，还有一个nodePort的Service就是External Service了。

![5(064%Y0PNVU357MM(45ANR](https://user-images.githubusercontent.com/8574915/167858833-20e2238b-feb5-4f00-8f9a-069015b0ac55.png)

![C$%82T2OVF6 XHF6W%LALEI](https://user-images.githubusercontent.com/8574915/167859897-f88d75b9-fd7d-4331-b436-b4bef7bf369b.png)

![)3DDWL$VMBK68BP(DWU$DGI](https://user-images.githubusercontent.com/8574915/167859926-f566cd3a-e102-4216-8fbf-beb350eb5ef5.png)

#### 怎样给你的集群配置Ingress ？

首先需要一个Ingress Controller:

![U4(RL( F@H)@RDD4YRJ{ 6S](https://user-images.githubusercontent.com/8574915/167860486-f05e6157-b3f4-42e7-90d1-5ee298df1167.png)

这个Controller是集群的入口点，评估所有rules，管理重定向，Controller有很多第三方实现，比如k8s Nginx Ingress Controller。如果是使用Cloud Provider的，那么就不需要关心那么多，AWS, Google Cloud， Linode都提供开箱即用的配置功能。

![D`R}X0$}}0LZLCH2`N6H)_2](https://user-images.githubusercontent.com/8574915/167861560-594f9a15-ec2e-4758-8f7c-29a0c9f03863.png)

如果你是在裸机(Bare metal)上搞k8s生产用的集群，那么你就需要配置这么一个类似的入口。比如用一个单独的Proxy server，不是任何人都可以访问k8s Cluster的。

![AT)AMM(2VZG@PV28HI Z~79](https://user-images.githubusercontent.com/8574915/167862049-7e23fc17-6050-495d-8208-bd1ab569094e.png)

![4HB DI4GJF`@UW0{@T$V1FT](https://user-images.githubusercontent.com/8574915/167862381-6e4ac30c-4422-4eee-b286-4baae9daa032.png)

#### 在minikube上配置一个Ingress Controller

首先要在minikube中安装一个Ingress Controller

```bash
mathxh@MathxH:~/Project/telecom-tools$ minikube addons enable ingress
    ▪ Using image k8s.gcr.io/ingress-nginx/kube-webhook-certgen:v1.1.1
    ▪ Using image k8s.gcr.io/ingress-nginx/kube-webhook-certgen:v1.1.1
    ▪ Using image k8s.gcr.io/ingress-nginx/controller:v1.0.4
🔎  Verifying ingress addon...
```

以上命令其实自动安装的就是k8s Nginx Ingress Controller这个实现，你可以用 `kubectl get pod -n kube-system` 看到这个Niginx Ingress Controller这个Pod。

这个Ingress Controller就会校验 ingress中的rules了。

![2MWG~ARID`QNAJG(4__P%F6](https://user-images.githubusercontent.com/8574915/167868377-b4f2f39f-ad3b-43cf-9f9d-19a9d41d66c6.png)

![4` VY4O`B6WOYY8S)KW}E H](https://user-images.githubusercontent.com/8574915/167868773-1d038f7d-7e31-4833-9aa6-e09ff0135613.png)

#### 配置TLS证书

![DJGL3X7)XS25EDE_Y(72({F](https://user-images.githubusercontent.com/8574915/167869098-9d23a3c4-4ac7-4424-bffa-f0232aaf399f.png)

注意TLS的Secret要创建在与对应的Ingress相同的namespace下， crt和key的base64编码不是File Path，而是content。

### Helm k8s的包管理器

```bash
curl https://baltocdn.com/helm/signing.asc | sudo apt-key add -
sudo apt-get install apt-transport-https --yes
echo "deb https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

这个包管理器干了什么呢？ 其实就是打包一堆yaml 组成一个类似于docker compose的k8s上面跑的软件stack，然后把打包好的东西publish 到一个公有或私有的repo或hub中，跟DockerHub  npm resgitery这样类似了。好了，这样其实就方便了。

比如你的k8s集群里面你的应用需要一个ELK的技术栈支撑logging监控，收集之类的，如果你手工创建k8s组件，你需要ConfigMap，Service，Stateful Set，Secret，还有一些权限。太耗费时间了，所以有了helm，可以直接用别人的yaml stack了。这堆打包物就叫Helm Charts。常用的Charts就是MongoDB，ELK，Promethous，监控这些常用不会变的东西了。

本质上就是一个yaml配置复用了。通过 `helm search hub <package-name>` 搜索官方的hub就行了。如果是搜索第三方的repo，那么需要添加repo `helm repo add brigade https://brigadecore.github.io/charts` 类似这样， `helm search repo <package-name>`。 

helm还有一项功能就是作为yaml的模板引擎, 也算是增加复用吧。

![ESOTH~P5{TDI}VFUHBVM8_C](https://user-images.githubusercontent.com/8574915/168087059-81ecfe84-49ba-48cb-9978-80985c74c078.png)

这样有什么用呢？ 可以用在CI/CD pipeline等工作流上，当然，还可以跨环境，也是复用吧，这样Dev， Staging，Prod的环境就是一套yaml了（Chart）。

下面我们来看看一个Chart的目录结构，以及说明:

![0KO8H@1L`1$F3ME%RUAB)_A](https://user-images.githubusercontent.com/8574915/168088266-b813b26e-ae35-448a-af24-e22216992717.png)

所以当你运行 `helm install <chart-name>` 的时候，templates目录下的模板文件就被values.yaml里面的值填充覆盖了，这个是默认行为。如果不想用默认的填充，还有一种可以指定另外的values覆盖template:  `helm install --values=<my-vaues.yaml> <chart-name>`， 也可以`helm install --set <key>=<value>`来指定覆盖的yaml配置项。

注意helm v2和v3变化有点大，原理有点不一样，这个详情看官网: https://helm.sh/docs/topics/v2_v3_migration/ https://helm.sh/docs/faq/changes_since_helm2/   比如v3中没有v2中的Tiller概念，Tiller是helm CLI的server端，tiller是在k8s集群当中的。

### k8s Volumes详解

之前的章节也说到了，k8s的Storage是脱离Pod的生命周期的，是全局的，即便集群挂了，都会存在。其实Volume本质上就是一个目录，这个目录下面你可以读写存放数据。

#### Persistent Volume（PV）

永久卷是一个集群级别的resource，无namespace（集群内所有namesapce的resource都可以访问PV）也是通过yaml来创建，它需要真实的物理存储，比如本地磁盘，NFS Server， Cloud Storage。那么怎么把这些存储挂到一个k8s集群里呢？对，就是用Persistent Volume这个组件来对接，它相当于一个存储接口一样。因为k8s不会管理存储，所以需要什么样的存储，存储的备份管理都需要你自己来维护

![O8$20IZFRMBDO673D381JV0](https://user-images.githubusercontent.com/8574915/168290364-2fdcd75d-1409-4bce-8b61-5e495d0cf4b1.png)

下面是一个NFS Server的PV例子:

![8S_2H{68G7{H@{E~LT(WCG1](https://user-images.githubusercontent.com/8574915/168290596-ad3d238c-1bbc-47e7-8d1e-810fc50771e5.png)

然后还有Cloud Storage的本地磁盘的:

![)%~9@{FJFZBI` EYYJF$F0Z](https://user-images.githubusercontent.com/8574915/168291689-eb859435-a291-44c0-84b6-7e23ef645b0b.png)

![8E(U4 H90P9 BP69E{QMWUH](https://user-images.githubusercontent.com/8574915/168291751-414be1c8-c89b-471d-97e9-ea353c0d061d.png)

不同类型的存储，yaml里面的sepc项都不同。支持的存储后端类型太多了，什么cephFS，AzureDisk啥都有，可以去官网看文档。

看到这里你会发现，上面的存储类型，大致可以分成两类: local和remote的。那么这两种大类大概的应用场景是什么呢？

local的只会作用绑定在一个物理节点上，如果Pod漂移，那么重启的Pod就利用不了之前的状态了，这样是不行的。如果这样情况，就需要用remote。比如DB就需要用remote storage，一定要用remote！！！

谁创建了PV，是什么时候创建的？答： 如果有Pod依赖，那么肯定要在Pod创建之前，先建立PV。 一般这个在正规公司里面，是k8s 管理员（运维人员）建立，因为管理员对整个集群负责，其他k8s用户部署App在Cluster上就行了。

#### Persistent Volume Claim（PVC）

![Y8_NB1%D$VR@B3ANL@5V06G](https://user-images.githubusercontent.com/8574915/168294379-7154b309-1575-4228-a0de-af2788cb1f03.png)

看上图就可以看出来，PVC是PV的selector，具体的Pod再引用PVC，通过PVC来使用PV。为了使用Storage，k8s做了两层抽象。

![D6LY1A DQ8%H{WD@_){E_X1](https://user-images.githubusercontent.com/8574915/168295073-aa3e53d5-c08d-442b-91a1-f69b346e3567.png)

注意，PVC必须与引用它的Pod在同一个namespace下。这样PVC引用的PV所代表的Storage才会被挂在进Pod里面。

为什么k8s会搞这么复杂呢？其实也是运维和开发部署分离，因为运维的k8s管理员，要经常管理Storage，所以他需要维护PV，而普通的开发人员，部署的App同时创建PVC，这样就有一个隔离，选择的余地，降低风险范围，因为PV老变动，如果没有PVC，那么Deployment这些就得跟着改。很麻烦。本质上也是解耦，k8s普通用户不需要关系实际的存储是怎么回事，嗯。

最后，还有一点要注意，ConfigMap和Secret的组件的内容，其实是存储在local volume的，不归PVC和PV管，是归k8s管。这两个组件是可以被挂在进Pod的:

![8WX_3~Y ~ {LPTQDD}PHF$W](https://user-images.githubusercontent.com/8574915/168297208-8c62ff59-ca2b-40b2-b8ba-bf58759c2d53.png)

![(P{I(103QTBR9 O~9}%T K9](https://user-images.githubusercontent.com/8574915/168297936-1324b4f8-6b44-4da4-bb82-d4750e1fe207.png)

#### Storage Class(SC)

前面不是讲到 PVC PV这个解耦问题嘛，PV一般来说都是k8s管理员手动创建维护，如果要大量创建PV，那么肯定是耗费时间的，所以SC就出来了。当PVC claim了PV， SC可以动态地提供PV。存储后端可以直接定义在SC中。

![OLBS@ZCF %I5TO)NLY(_%O2](https://user-images.githubusercontent.com/8574915/168299443-99aff429-5241-4367-b93d-f4c8dce9b9c9.png)

SC就是一个Storage Provider的抽象，PVC根据这个去按需获取Storage resource。

![{DR~N4M%%5_~B2(ANMQR@9Y](https://user-images.githubusercontent.com/8574915/168299932-65a2b076-4046-46f8-85b3-d5e4f6f6a7a9.png)

![YB3AK@0~V0}RZ{WK6F7%M I](https://user-images.githubusercontent.com/8574915/168300076-f4a3017f-98ab-4a47-8237-dbd54ac686f7.png)


### StatefulSet详解

这个就是为有状态的应用准备的了，比如数据库，还有一些有状态的应用服务都是Stateful的。有状态和无状态应用的概念我想做互联网的人都应该知道了。比如要横向扩展，必须尽量把自己的服务设计成stateless的云云，诸如此类的讨论很多。就不解释了。

stateless的应用服务部署，前面讲过了，就用Deployment这个组件部署，可以方便地replicate多个Pod副本。stateful的服务当然就是StatefulSet这个组件了。这两个组件都可以使用PVC PV这些的，都可以使用外挂存储。那么到底有什么区别呢？好像Deployment也可以部署数据库嘛，即使replica指定多个，只要remote storage是一个地方就行，状态就可以一致啊？

但是注意，数据库的数据一致性，不仅仅是storage这个Volume目录，比如MySQL在单台机器上，有多个副本，副本都可以读写，那么也会导致数据不一致的。只能做主从结构，但是MySQL的主从结构，Master workder节点两个配置是不一样的，所以不可以通过Deployment的Replica 做 Scale。还记得做replica要无状态吗？所有副本都相同，LoadBalancer都可以随机分配request，没有顺序，随机启停，随便scale。所以数据库是不可以用Deployment的！

而StatefulSet呢，每个Pod replica就不是随机的ID了，是固定的启动顺序和停止顺序（停机顺序跟启动顺序反过来），一切都是要确定的。一个Pod依赖另一个Pod。每个Pod都是独一无二的，所以对应的internal service也是独一无二的名字。即使Pod重启了，名字和endpoint也不会变！（仅仅IP变了）

![VKFDK2MG{J`J@`IIN{2EVDH](https://user-images.githubusercontent.com/8574915/168305403-977db310-725b-48ad-8883-e918e44a9f18.png)


嗯，总得看下来，其实还不太方便，数据库之间的数据同步还是要自己手动配置，remote storage这些都需要自己管理，k8s还是不会管。k8s的StatefulSet只提供基本的元素。k8s始终还是最适合无状态服务。有状态服务能尽量绕开k8s就绕开吧。

### Service详解

前面已经或多或少使用过讲解过Service了，这小节主要是更广泛深入涉及Service，讲解不同的Service的使用场景和不同。比如ClusterIP Service， Headless Service， NodePort Service， LoadBalancer Service。

再来回顾下，为什么需要Service， 因为Pod是靠大二层网络互连在一起的，每个Pod都有自己的IP，这个IP随着Pod的漂移，重启会变，所以就需要Service这个抽象，作为一组Pods的接口入口。Service的IP就是固定的。这样就方便访问Pods的服务了。Service还有一层负载均衡的功能，因为这个replica Pods，request随便分配给哪个Pod都行，因为无状态嘛，所以Service就提供分配request的负载均衡功能。也解耦了，跟PVC和PV的关系是一个道理。

#### ClusterIP Service

这个是最常用的Service之一，也就是我们通常所说的internal Service。它的类型是默认的，不需要显式指定，创建出来一般都是这种类型的Service。

![%2T%NG9AM5}AC21C1K3VQJR](https://user-images.githubusercontent.com/8574915/168405253-3ee698a0-e923-4dec-8ac9-b9dabf64db0d.png)

想象一下，在一个Pod中跑了两个container，一个container是App应用，一个contianer是log collector作为一个side-car来收集App的日志，这个两个container的port不一样。然后Pod有一个自己的集群内部IP，如果Pod有多个副本，自然副本之间的IP也不一样，如果你想查看这些Pods的IP ，可以 `kubectl get pod -o wid`。

这种Service就是Pods的抽象了，所以它需要selector选择相应的Deployment，然后配置上自己的servicePort，这样ingress就可以访问它了。selector本质上是一个KV键值对，名字你随便取

![K}TY4K8SP(PDJ97E9RJF68](https://user-images.githubusercontent.com/8574915/168405557-96d7daa1-8355-4cdd-93b3-3b98e38ddffa.png)

上面讲了，Pod里面是运行着两个container的，每个container Port不一样，那么Service是怎样知道是把Request转发到哪个container Port呢？ 答案就在Service yaml中的targetPort配置项


![D9UD3ZVSW 6KQM}S 6) 2FS](https://user-images.githubusercontent.com/8574915/168405683-f9ce3be6-2219-4e55-8b4b-ef6d2edea200.png)


![BVV9P P)0J7H(5 7@})V0(9](https://user-images.githubusercontent.com/8574915/168405895-af5330f8-2b07-4a2a-bfa7-05ab4188a2a4.png)

![50W0X`I2LP7`` WV%E`XS L](https://user-images.githubusercontent.com/8574915/168405899-dae8c516-17b0-4dd4-bd8c-833bc64826d7.png)


在Service选择了一组Pod以后，k8s会把这个Pod的IP和port注册为EndPoint对象，EndPoint对象就是为了追踪Service关联的这些Pods的。EndPoint的名字跟Service的名字是一致的。

上面你也看到了，对于多端口的ClusterIP Service就是需要暴露出多个端口来。多端口的service，需要在port 下面加入name，每个port都有自己的name。

#### Headless Service

如果你想达到下面的目的，Client 自主选择一个特定的Pod，某一个Pod想直接跟一个特定的Pod通信，或者ClusterIP Service不想做随机负载均衡到选择的Pod，那么Headless Service就是你需要的了。

![PH@L4)B3SJ1) }U@B}6PD0](https://user-images.githubusercontent.com/8574915/168406763-aa30fdf9-599c-4887-a340-e6bffc3c6e31.png)

使用在什么场景呢？想想，一般k8s都是推荐部署stateless App，所以Service随机选择Pod无所谓，但是对于之前提到的数据库这些stateful的服务就不行了，需要选择特定的Pod，比如Master是Master  workder是workder，完全不能同等看到，所以这个headless service的使用场景就是stateful App。

所以根据上面的场景，在Cluster外部的client就需要直接确定它需要的Pod的IP地址，怎么确定呢？

![6K~SY9}AQO_KO7@Y_BNU} 8](https://user-images.githubusercontent.com/8574915/168406919-b5487108-e532-484d-8e54-c74c7618fb35.png)

上图的方法其实都不行。所以直接需要Headless Service，这个Service就是直接把clusterIP改成None。

![N~~JJSFE1IYB6E 3$ZSGE}C](https://user-images.githubusercontent.com/8574915/168406969-dcd2b2e0-f299-4e7b-b56f-9e658fc8260b.png)

然后这个Service就可以直接返回Pod的地址了，不然就是ClusterIP。

#### NodePort Service

![E0@16AWWF6R)3ML0@GX} OV](https://user-images.githubusercontent.com/8574915/168407177-2b9c61d4-b9db-4836-8294-1044e3b4724b.png)

注意，nodePort的端口范围是30000 - 32767。NodePortService它不安全，所以LoadBalancer的一种External Service出来了，建议用LoadBalancer替代NodeType，虽然这两者都有nodePort这个配置项，但是不一样的。

NodePort Service不是给集群外部链接使用的，一般不用在正式的生产环境，不过可以方便地用在测试环境，看看正不正常，因为都有nodePort的功能。

#### LoadBalancer Service

![R@_LE$}`F{@XDBR{DTSE%DE](https://user-images.githubusercontent.com/8574915/168407318-c10a5b1f-32d0-4142-a808-2be7dc5d149c.png)

是NodePort Service的一种扩展。NodePort Service又是ClusterIP Service的一种扩展。但是NodePort Service是没有External IP的，跟ClusterIP Service一样。而Headless Service既没有ClusterIP又没有External IP。LoadBalancer Service既有ClusterIP又有External IP。

LoadBalancer和Ingress是用在生产环境上的。

EOF

