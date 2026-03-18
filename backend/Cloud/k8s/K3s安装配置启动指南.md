
# K3s 安装配置指南

## 目录
1. [K3s 简介](#k3s-简介)
2. [安装 K3s](#安装-k3s)
3. [配置代理（国内网络环境）](#配置代理国内网络环境)
4. [验证 K3s 运行状态](#验证-k3s-运行状态)
5. [K3s 内置组件详解](#k3s-内置组件详解)
6. [Kubernetes 核心概念回顾](#kubernetes-核心概念回顾)
7. [常用运维命令](#常用运维命令)
8. [快速开始示例](#快速开始示例)

---

## K3s 简介

**K3s** 是 Rancher Labs 推出的轻量级 Kubernetes 发行版，专为边缘计算、物联网设备等资源受限场景设计。

### K3s vs K8s
| 特性 | K8s (原版) | K3s (轻量版) |
|------|-----------|-------------|
| 内存占用 | ~1-2 GB | ~512 MB |
| 二进制大小 | ~1 GB | ~100 MB |
| 数据库 | etcd (需要 3 节点) | SQLite/MySQL/PostgreSQL |
| 组件 | 所有云厂商附加功能 | 仅核心功能，可裁剪 |
| 适用场景 | 数据中心、云环境 | 边缘设备、ARM 设备、CI/CD |

### 为什么叫 K3s？
K3s = K8s 减去一半（"5" 去掉两个变成 "3"），寓意精简了一半以上的内存和体积。

---

## 安装 K3s

### 快速安装

```bash
# 使用官方脚本安装（会自动启动服务）
curl -sfL https://get.k3s.io | sh -
```

### 安装后的文件结构

```
/usr/local/bin/k3s              # 主程序
/usr/local/bin/kubectl          # kubectl 链接到 k3s
/usr/local/bin/crictl           # 容器运行时 CLI
/etc/systemd/system/k3s.service # systemd 服务配置
/etc/rancher/k3s/               # 配置和证书目录
/var/lib/rancher/k3s/           # 数据目录
```

---

## 配置代理（国内网络环境）

### 问题背景

由于 Docker Hub (`registry-1.docker.io`) 在中国大陆访问受限，k3s 默认无法拉取镜像。症状如下：

```
Warning  FailedCreatePodSandBox  ... dial tcp [2a03:2880:f10c:83:face:b00c:0:25de]:443: i/o timeout
```

### 解决方案：为 containerd 和 k3s 配置代理

假设你的本地代理地址是 `http://127.0.0.1:7897`（与 Docker 配置一致）：

#### 步骤 1：创建 containerd 代理配置

```bash
# 创建配置目录
sudo mkdir -p /etc/systemd/system/containerd.service.d

# 创建代理配置文件
sudo tee /etc/systemd/system/containerd.service.d/http-proxy.conf > /dev/null << 'EOF'
[Service]
Environment="HTTP_PROXY=http://127.0.0.1:7897"
Environment="HTTPS_PROXY=http://127.0.0.1:7897"
Environment="NO_PROXY=localhost,127.0.0.1"
EOF
```

#### 步骤 2：创建 k3s 代理配置

```bash
# 创建配置目录
sudo mkdir -p /etc/systemd/system/k3s.service.d

# 创建代理配置文件
sudo tee /etc/systemd/system/k3s.service.d/http-proxy.conf > /dev/null << 'EOF'
[Service]
Environment="HTTP_PROXY=http://127.0.0.1:7897"
Environment="HTTPS_PROXY=http://127.0.0.1:7897"
Environment="NO_PROXY=localhost,127.0.0.1"
EOF
```

#### 步骤 3：重载并重启服务

```bash
# 重载 systemd 配置
sudo systemctl daemon-reload

# 重启 containerd
sudo systemctl restart containerd

# 重启 k3s
sudo systemctl restart k3s
```

#### 验证配置生效

```bash
# 检查环境变量是否加载
sudo systemctl show --property=Environment containerd
sudo systemctl show --property=Environment k3s

# 应该看到类似输出：
# Environment=HTTP_PROXY=http://127.0.0.1:7897 HTTPS_PROXY=http://127.0.0.1:7897 NO_PROXY=localhost,127.0.0.1
```

---

## 验证 K3s 运行状态

### 1. 检查服务状态

```bash
# 查看 k3s 服务状态
sudo systemctl status k3s

# 输出示例：
# ● k3s.service - Lightweight Kubernetes
#      Loaded: loaded (/etc/systemd/system/k3s.service; enabled)
#      Active: active (running) since Sun 2026-02-08 16:27:00 CST
```

**命令用途**：检查 k3s 守护进程是否正常运行。

### 2. 检查 Kubernetes 版本

```bash
# 查看 client 和 server 版本
sudo kubectl version --output=yaml

# 简短输出
sudo kubectl version
```

**命令用途**：确认 kubectl 与 k3s API server 的通信正常，以及版本信息。

### 3. 检查节点状态

```bash
# 查看集群节点
sudo kubectl get nodes

# 输出示例：
# NAME        STATUS   ROLES           AGE   VERSION
# mathxh-pc   Ready    control-plane   74s   v1.34.3+k3s1
```

**命令用途**：查看集群中的节点及其健康状态。

**关键字段**：
- `STATUS`: Ready 表示节点正常，NotReady/Unknown 表示有问题
- `ROLES`: control-plane 表示这是管理节点（k3s 默认单节点既是 control-plane 也是 worker）
- `VERSION`: Kubernetes 版本号

### 4. 检查系统 Pods 状态

```bash
# 查看所有命名空间的 pods
sudo kubectl get pods --all-namespaces

# 输出示例：
# NAMESPACE     NAME                                      READY   STATUS      RESTARTS   AGE
# kube-system   coredns-7f496c8d7d-7dp9c                  1/1     Running     0          9m
# kube-system   metrics-server-7b9c9c4b9c-dpxfl           1/1     Running     0          9m
# kube-system   traefik-6f5f87584-hfknb                   1/1     Running     0          80s
```

**命令用途**：检查所有系统组件是否正常运行。

**关键字段**：
- `READY`: 如 `1/1` 表示容器已就绪，`0/1` 表示正在启动
- `STATUS`: Running/Completed 是正常，ContainerCreating/ImagePullBackOff 表示有问题
- `RESTARTS`: 重启次数，过多可能表示有 crash

### 5. 查看详细状态（问题排查）

```bash
# 查看某个 pod 的详细信息
sudo kubectl describe pod -n kube-system coredns-xxxxx

# 查看 pod 日志
sudo kubectl logs -n kube-system coredns-xxxxx

# 查看最近的集群事件
sudo kubectl get events --all-namespaces --sort-by='.lastTimestamp'
```

**命令用途**：当 pod 无法正常启动时，用于排查问题原因（如镜像拉取失败、配置错误等）。

---

## K3s 内置组件详解

K3s 默认安装以下核心组件：

### 1. CoreDNS (`coredns`)

**作用**：Kubernetes 集群的 DNS 服务器，提供服务发现功能。

**功能**：
- 为 Pod 提供 DNS 解析
- 服务名到 ClusterIP 的解析
- 支持外部 DNS 查询转发

**使用场景**：
```bash
# 在 Pod 中通过服务名访问其他服务
curl http://my-service:8080
# 而不是需要知道 IP
curl http://10.43.0.15:8080
```

### 2. Traefik (`traefik`)

**作用**：Kubernetes Ingress 控制器，处理进入集群的 HTTP/HTTPS 流量。

**功能**：
- 自动从 Kubernetes Ingress 资源读取配置
- SSL/TLS 终止
- 负载均衡
- 自动服务发现

**使用场景**：
```yaml
# 创建 Ingress 暴露服务到外网
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-service
            port:
              number: 80
```

### 3. Metrics Server (`metrics-server`)

**作用**：收集集群资源使用指标（CPU、内存）。

**功能**：
- 为 `kubectl top` 命令提供数据
- 水平自动扩缩容（HPA）的数据来源
- 不存储历史数据，仅提供实时指标

**使用场景**：
```bash
# 查看节点资源使用
kubectl top nodes

# 查看 pod 资源使用
kubectl top pods
```

### 4. Local Path Provisioner (`local-path-provisioner`)

**作用**：默认的存储类，为 Pod 提供本地存储。

**功能**：
- 自动在节点上创建 `/var/lib/rancher/k3s/storage/` 下的目录
- 适合单节点开发和测试
- 生产环境建议使用 Longhorn 或其他分布式存储

### 5. Service Load Balancer (`svclb-traefik`)

**作用**：K3s 特有的组件，为 LoadBalancer 类型的 Service 创建节点端口映射。

**功能**：
- 当创建 `type: LoadBalancer` 的 Service 时自动创建
- 使用 hostPort 暴露服务到节点网络
- 简化了在单节点环境的服务暴露

---

## Kubernetes 核心概念回顾

### 架构概览

```
┌─────────────────────────────────────────────────────────┐
│                    kubectl (CLI)                        │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│              API Server (k3s server)                    │
│  - 认证/授权  - API 入口  - 数据持久化                  │
└──────┬───────────────┬──────────────────────────────────┘
       │               │
       ▼               ▼
┌──────────────┐  ┌──────────────────────────────────┐
│  Controller  │  │         Scheduler                │
│  Manager     │  │  - 监控未调度的 Pod              │
│  - 维护状态  │  │  - 选择合适的节点运行 Pod        │
└──────────────┘  └──────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────┐
│                   kubelet                               │
│  - 与容器运行时通信  - 管理 Pod 生命周期               │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│              containerd (容器运行时)                    │
│  - 拉取镜像  - 管理容器  - 存储卷                       │
└─────────────────────────────────────────────────────────┘
```

### 核心概念

#### 1. Pod (最小部署单元)

**Pod** 是 Kubernetes 中最小的可部署单元，包含一个或多个共享网络/存储的容器。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
```

**特点**：
- 同一 Pod 内的容器共享网络命名空间（localhost 通信）
- 同一 Pod 内的容器共享存储卷
- Pod 是临时的，重启后 IP 会变化

#### 2. Deployment（声明式部署）

**Deployment** 管理 Pod 的副本和更新策略。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3              # 运行 3 个副本
  selector:
    matchLabels:
      app: nginx
  template:                # Pod 模板
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
```

**功能**：
- 确保指定数量的 Pod 副本在运行
- 滚动更新（Rolling Update）
- 回滚到之前版本
- 扩缩容（Scale Up/Down）

#### 3. Service（服务发现）

**Service** 为一组 Pod 提供稳定的网络端点。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx              # 选择带有此标签的 Pod
  ports:
  - port: 80                # Service 端口
    targetPort: 80          # Pod 端口
  type: ClusterIP           # 类型：ClusterIP/NodePort/LoadBalancer
```

**Service 类型**：

| 类型 | 访问范围 | 说明 |
|------|----------|------|
| `ClusterIP` | 集群内部 | 默认类型，仅集群内可访问 |
| `NodePort` | 节点 IP + 端口 | 在每个节点开放端口 |
| `LoadBalancer` | 外部 IP | 需要云厂商支持（k3s 用 svclb 模拟） |

#### 4. Namespace（命名空间）

**Namespace** 用于隔离集群资源。

```bash
# 查看所有命名空间
kubectl get namespaces

# 在特定命名空间操作
kubectl get pods -n kube-system
kubectl get pods -n default
```

**常见命名空间**：
- `default`: 默认命名空间
- `kube-system`: Kubernetes 系统组件
- `kube-public`: 公共资源

#### 5. ConfigMap & Secret（配置管理）

**ConfigMap**: 存储非敏感配置数据

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  database_url: "postgres://localhost:5432/mydb"
  debug_mode: "true"
```

**Secret**: 存储敏感数据（base64 编码）

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
data:
  password: cGFzc3dvcmQxMjM=  # echo -n 'password123' | base64
```

---

## 常用运维命令

### 服务管理

```bash
# 启动 k3s
sudo systemctl start k3s

# 停止 k3s
sudo systemctl stop k3s

# 重启 k3s
sudo systemctl restart k3s

# 开机自启动
sudo systemctl enable k3s

# 查看服务日志
sudo journalctl -u k3s -f

# 卸载 k3s
/usr/local/bin/k3s-uninstall.sh
```

### 集群诊断

```bash
# 查看集群信息
kubectl cluster-info

# 查看节点详细信息
kubectl describe nodes

# 查看所有资源
kubectl get all --all-namespaces

# 查看某个命名空间的所有资源
kubectl get all -n default

# 查看持久卷
kubectl get pv

# 查看持久卷声明
kubectl get pvc
```

### 日志查看

```bash
# 查看 Pod 日志
kubectl logs <pod-name>

# 实时跟踪日志
kubectl logs -f <pod-name>

# 查看包含多个容器的 Pod
kubectl logs <pod-name> -c <container-name>

# 查看之前实例的日志（Pod 重启后）
kubectl logs <pod-name> --previous
```

### 资源操作

```bash
# 创建资源（从文件）
kubectl apply -f deployment.yaml

# 创建资源（从目录）
kubectl apply -f ./k8s/

# 删除资源
kubectl delete -f deployment.yaml
kubectl delete pod <pod-name>
kubectl delete deployment <deployment-name>

# 编辑资源
kubectl edit deployment <deployment-name>

# 导出资源配置
kubectl get deployment <name> -o yaml > export.yaml
```

### 故障排查

```bash
# 查看 Pod 事件（帮助排查启动失败原因）
kubectl describe pod <pod-name>

# 查看 Pod 的 YAML 配置
kubectl get pod <pod-name> -o yaml

# 进入 Pod 容器执行命令
kubectl exec -it <pod-name> -- /bin/sh

# 端口转发（本地访问集群服务）
kubectl port-forward svc/<service-name> 8080:80

# 查看资源使用情况
kubectl top nodes
kubectl top pods
```

### 关机后下次重启k3s

```bash
# 启动 k3s
sudo systemctl start k3s

# 检查状态
sudo systemctl status k3s

# 验证集群可用
sudo kubectl get nodes
sudo kubectl get pods --all-namespaces
```

---

## 快速开始示例

### 部署一个 Nginx 应用

```bash
# 1. 创建 Deployment
kubectl create deployment nginx --image=nginx:latest

# 2. 查看 Deployment 状态
kubectl get deployments

# 3. 查看 Pod 状态
kubectl get pods

# 4. 暴露服务（NodePort 类型）
kubectl expose deployment nginx --port=80 --type=NodePort

# 5. 查看服务
kubectl get services

# 6. 获取 NodePort 端口
NODE_PORT=$(kubectl get svc nginx -o jsonpath='{.spec.ports[0].nodePort}')
echo "Access nginx at http://localhost:$NODE_PORT"

# 7. 扩容到 3 个副本
kubectl scale deployment nginx --replicas=3

# 8. 清理
kubectl delete deployment nginx
kubectl delete service nginx
```

### 使用 YAML 文件部署完整应用

```yaml
# nginx-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app
spec:
  replicas: 2
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
        image: nginx:1.25
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
```

```bash
# 部署
kubectl apply -f nginx-app.yaml

# 验证
kubectl get pods,svc

# 测试访问
kubectl port-forward svc/nginx-service 8080:80
# 然后在另一个终端访问 curl http://localhost:8080
```

---

## 附录：网络架构

### K3s 网络模型

```
                    ┌─────────────────────┐
                    │   Ingress (Traefik)  │
                    │   NodePort: 80/443   │
                    └──────────┬───────────┘
                               │
                    ┌──────────▼───────────┐
                    │  Service (ClusterIP) │
                    │   10.43.0.0/16       │
                    └──────────┬───────────┘
                               │
                    ┌──────────▼───────────┐
                    │      Pod (CNI)       │
                    │   10.42.0.0/16       │
                    │  - nginx: 10.42.0.x  │
                    │  - redis: 10.42.0.y  │
                    └──────────────────────┘
```

### 网络段说明

| 组件 | 网段 | 说明 |
|------|------|------|
| Service CIDR | `10.43.0.0/16` | Service 的虚拟 IP |
| Pod CIDR | `10.42.0.0/16` | Pod 的实际 IP |
| Cluster DNS | `10.43.0.10` | CoreDNS 服务地址 |

---

## 参考资源

- [K3s 官方文档](https://docs.k3s.io/)
- [Kubernetes 官方文档](https://kubernetes.io/docs/)
- [kubectl 命令参考](https://kubernetes.io/docs/reference/kubectl/)
