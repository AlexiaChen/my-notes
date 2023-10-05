说是ShadowSocks过时了，现在都是v2ray（可是我记得北理工出了个专利论文，墙可以检测v2ray了? https://www.zhihu.com/question/347846360 ），所以在搬瓦工上买了台VPS，ubuntu 18.04。

SS只是简单的代理软件，v2ray说是被定位成一个平台，更加强大，也更加复杂，产业链也不成熟(可以由第三方开发代理接入v2ray):

- V2Ray 使用了新的自行研发的 VMess 协议，改正了 Shadowsocks 一些已有的缺点，更难被墙检测到(VMess 是一个基于 TCP 的协议，所有数据使用 TCP 传输)
- 网络性能更好
-支持各种协议


由于v2ray的官网被墙，所以下面还是摘录一些官网内容吧：

## 安装

```bash
bash <(curl -L https://raw.githubusercontent.com/v2fly/fhs-install-v2ray/master/install-release.sh)
```

此脚本会自动安装以下文件：

-  /usr/bin/v2ray/v2ray：V2Ray 程序；
- /usr/bin/v2ray/v2ctl：V2Ray 工具；
-  /etc/v2ray/config.json：配置文件；
- /usr/bin/v2ray/geoip.dat：IP 数据文件
- /usr/bin/v2ray/geosite.dat：域名数据文件

此脚本会配置自动运行脚本。自动运行脚本会在系统重启之后，自动运行 V2Ray。目前自动运行脚本只支持带有 Systemd 的系统，以及 Debian / Ubuntu 全系列。

脚本运行完成后，你需要：

- 编辑 /etc/v2ray/config.json 文件来配置你需要的代理方式；
- 运行 service v2ray start 来启动 V2Ray 进程；
- 之后可以使用 service v2ray start|stop|status|reload|restart|force-reload 控制 V2Ray 的运行。

go.sh的参数：

- --local 使用一个本地文件进行安装。如果你已经下载了某个版本的 V2Ray，则可通过这个参数指定一个文件路径来进行安装。比如，./go.sh --version v1.13 --local /path/to/v2ray.zip 

## 新手上路

下载并安装了 V2Ray 之后，你需要对它进行一下配置。这里介绍一下简单的配置方式，只是为了演示，如需配置更复杂的功能，请参考配置文件说明，后面有教程链接

### 服务器端

你需要一台防火墙外的服务器，来运行服务器端的 V2Ray。配置如下：

```json
{
  "inbounds": [{
    "port": 10086, // 服务器监听端口，必须和上面的一样
    "protocol": "vmess",
    "settings": {
      "clients": [{ "id": "b831381d-6324-4d53-ad4f-8cda48b30811" }]
    }
  }],
  "outbounds": [{
    "protocol": "freedom",
    "settings": {}
  }]
}
```

服务器的配置中需要确保 id 和端口与客户端一致，就可以正常连接了。

### 客户端

在你的 PC （或手机）中，你需要运行 V2Ray 并使用下面的配置：

```json
{
  "inbounds": [{
    "port": 1080,  // SOCKS 代理端口，在浏览器中需配置代理并指向这个端口
    "listen": "127.0.0.1",
    "protocol": "socks",
    "settings": {
      "udp": true
    }
  }],
  "outbounds": [{
    "protocol": "vmess",
    "settings": {
      "vnext": [{
        "address": "server", // 服务器地址，请修改为你自己的服务器 ip 或域名
        "port": 10086,  // 服务器端口
        "users": [{ "id": "b831381d-6324-4d53-ad4f-8cda48b30811" }]
      }]
    }
  },{
    "protocol": "freedom",
    "tag": "direct",
    "settings": {}
  }],
  "routing": {
    "domainStrategy": "IPOnDemand",
    "rules": [{
      "type": "field",
      "ip": ["geoip:private"],
      "outboundTag": "direct"
    }]
  }
}
```

上述配置唯一要改的地方就是你的服务器 IP，配置中已注明。上述配置会把除了局域网（比如访问路由器）之外的所有流量转发到你的服务器。

## 白话文教程

更加详细的说明请参考白话文教程 https://toutyrater.github.io/ （估计该地址被墙了），等之后我搬运过来。

## 我的服务端配置

```json
{
 "outbound": {
  "streamSettings": null, 
  "tag": null, 
  "protocol": "freedom", 
  "mux": null, 
  "settings": null
 }, 
 "log": {
  "access": "/var/log/v2ray/access.log", 
  "loglevel": "info", 
  "error": "/var/log/v2ray/error.log"
 }, 
 "outboundDetour": [
  {
   "tag": "blocked", 
   "protocol": "blackhole", 
   "settings": null
  }
 ], 
 "inbound": {
  "streamSettings": {
   "network": "tcp", 
   "kcpSettings": null, 
   "wsSettings": null, 
   "tcpSettings": null, 
   "tlsSettings": {}, 
   "security": ""
  }, 
  "settings": {
   "ip": "v2ray server ip", 
   "udp": true, 
   "clients": [
    {
     "alterId": 100, 
     "security": "aes-128-gcm", 
     "id": "uuid"
    }
   ], 
   "auth": null
  }, 
  "protocol": "vmess", 
  "port": [v2ray server port], 
  "listen": "v2ray server ip"
 }, 
 "inboundDetour": null, 
 "routing": {
  "settings": {
   "rules": [
    {
     "ip": [
      "0.0.0.0/8", 
      "10.0.0.0/8", 
      "100.64.0.0/10", 
      "127.0.0.0/8", 
      "169.254.0.0/16", 
      "172.16.0.0/12", 
      "192.0.0.0/24", 
      "192.0.2.0/24", 
      "192.168.0.0/16", 
      "198.18.0.0/15", 
      "198.51.100.0/24", 
      "203.0.113.0/24", 
      "::1/128", 
      "fc00::/7", 
      "fe80::/10"
     ], 
     "domain": null, 
     "type": "field", 
     "port": null, 
     "outboundTag": "blocked"
    }
   ], 
   "domainStrategy": null
  }, 
  "strategy": "rules"
 }, 
 "dns": null
}
```

## 我的客户端配置

```json
{
 "outbound": {
  "streamSettings": {
   "network": "tcp", 
   "kcpSettings": null, 
   "wsSettings": null, 
   "tcpSettings": null, 
   "tlsSettings": {}, 
   "security": ""
  }, 
  "tag": "agentout", 
  "protocol": "vmess", 
  "mux": {
   "enabled": true
  }, 
  "settings": {
   "vnext": [
    {
     "users": [
      {
       "alterId": 100, 
       "security": "aes-128-gcm", 
       "id": "uuid"
      }
     ], 
     "port": [v2ray server port], 
     "address": "v2ray server ip"
    }
   ]
  }
 }, 
 "log": {
  "access": "", 
  "loglevel": "info", 
  "error": ""
 }, 
 "outboundDetour": [
  {
   "tag": "direct", 
   "protocol": "freedom", 
   "settings": {
    "response": null
   }
  }, 
  {
   "tag": "blockout", 
   "protocol": "blackhole", 
   "settings": {
    "response": {
     "type": "http"
    }
   }
  }
 ], 
 "inbound": {
  "streamSettings": null, 
  "settings": {
   "ip": "127.0.0.1", 
   "udp": true, 
   "clients": null, 
   "auth": "noauth"
  }, 
  "protocol": "http", 
  "port": 1080, 
  "listen": "127.0.0.1"
 }, 
 "inboundDetour": null, 
 "routing": {
  "settings": {
   "rules": [
    {
     "ip": [
      "0.0.0.0/8", 
      "10.0.0.0/8", 
      "100.64.0.0/10", 
      "127.0.0.0/8", 
      "169.254.0.0/16", 
      "172.16.0.0/12", 
      "192.0.0.0/24", 
      "192.0.2.0/24", 
      "192.168.0.0/16", 
      "198.18.0.0/15", 
      "198.51.100.0/24", 
      "203.0.113.0/24", 
      "::1/128", 
      "fc00::/7", 
      "fe80::/10"
     ], 
     "domain": null, 
     "type": "field", 
     "port": null, 
     "outboundTag": "direct"
    }
   ], 
   "domainStrategy": "IPIfNonMatch"
  }, 
  "strategy": "rules"
 }, 
 "dns": {
  "servers": [
   "8.8.8.8", 
   "8.8.4.4", 
   "localhost"
  ]
 }
}
```

服务端和客户端的UUID和alterid要保持一致。

如果修改完配置，记得要重启v2ray服务:

```bash
sudo service v2ray restart
```