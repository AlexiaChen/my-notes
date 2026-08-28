
# 第11章 保护 Kafka · 读书笔记

> 💡 这一章表面是"如何给 Kafka 加安全"，实质是在回答分布式日志系统的**第一性安全原理**：**认证（Authentication）回答"你是谁"，授权（Authorization）回答"你能做什么"，加密（Encryption）回答"能否被窃听/篡改"**。这三件事在 Kafka 中不是全局开关，而是**per-listener（每个监听器独立）**配置的——这意味着你可以让 broker 间用 SASL_SSL、内部客户端用 mTLS、外部客户端用 SASL_SSL+OAUTHBEARER，互不影响。原书第2版写于 ZooKeeper 时代，但 Kafka 4.x 已经**移除 ZooKeeper**、Confluent 7.5+ 新部署默认 KRaft，ACL 存储从 ZooKeeper znode 迁到元数据日志（StandardAuthorizer）。所以这份笔记的做法是：**以原书目录为骨架，用现代 Kafka 安全模型重写**——你会发现，Kafka 安全不是一个"功能"，而是一套"以 listener 为边界、以插件化 Authorizer 为核心、以 KRaft 元数据日志为 ACL 存储"的完整体系。

---

## 11.1 锁住 Kafka

"锁住 Kafka"不是指加一把锁，而是指**在每一个网络入口处建立安全边界**。Kafka 的安全边界由 listener 定义：

```
外部客户端 ──► CLIENT listener (SASL_SSL + SCRAM) ──► Broker
内部 broker ─► BROKER listener (SASL_SSL + SCRAM) ──► Broker
Controller  ─► CONTROLLER listener (SSL/mTLS)    ──► Controller
```

每个 listener 在 `listener.security.protocol.map` 中独立配置安全协议：

|协议|认证|加密|
|---|---|---|
|`PLAINTEXT`|无|无|
|`SSL`|可选 mTLS|TLS|
|`SASL_PLAINTEXT`|SASL（PLAIN/SCRAM/Kerberos/OAUTHBEARER）|无|
|`SASL_SSL`|SASL|TLS|

> ⚠️ **工业铁律**：`SASL_PLAINTEXT` 绝不能暴露到公网——凭证明文传输违反 RFC 5802，且极易被嗅探。所有面向网络的 listener 都应是 `SASL_SSL` 或 `SSL`。

### 洞见：安全的"第一性边界"是 Listener

Kafka 架构的一个精妙之处：**安全不是集群级开关，而是 listener 级策略**。这意味着：

- 你可以用**不同的认证机制**保护不同的入口（broker 间用 SCRAM，客户端用 OAUTHBEARER）
- 你可以用**不同的加密强度**保护不同信任域（内部网络用 SASL_PLAINTEXT 省 CPU，外部用 SASL_SSL）
- 防火墙规则可以与 listener 端口一一对应（9092 给客户端，9091 给 broker，9094 给 controller）

这种"listener 即安全边界"的设计，让 Kafka 能够适应从完全内网到跨云混合云的任意安全场景。

---

## 11.2 安全协议

Kafka 的安全协议是**认证与加密的正交组合**：

```
无加密              有加密(TLS)
            ─────             ───────────
无认证       PLAINTEXT             SSL
有认证    SASL_PLAINTEXT        SASL_SSL
```

**核心配置**：

```
# 定义监听器
listeners=BROKER://0.0.0.0:9091,CLIENT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9094

# 映射安全协议
listener.security.protocol.map=BROKER:SASL_SSL,CLIENT:SASL_SSL,CONTROLLER:SSL

# 指定 broker 间通信使用的 listener
inter.broker.listener.name=BROKER

# KRaft 模式下的 controller listener
controller.listener.names=CONTROLLER
```

**不停机安全化**（原书 11.3.4 的核心内容）：

Kafka 支持滚动升级方式启用安全，这是生产环境的关键能力：

```
阶段1: 打开安全端口（保留 PLAINTEXT 端口）
  listeners=PLAINTEXT://broker:9091,SSL://broker:9092
  滚动重启每个 broker
  
阶段2: 客户端切换到安全端口
  bootstrap.servers=broker:9092
  security.protocol=SSL
  
阶段3: 启用 broker 间安全协议
  security.inter.broker.protocol=SSL
  
阶段4: 关闭 PLAINTEXT 端口
  listeners=SSL://broker:9092
  滚动重启每个 broker
```

> 💡 这个过程体现了 Kafka 设计的成熟度——安全不是"重建集群"，而是"逐步收紧边界"。

---

## 11.3 身份验证

Kafka 支持两类认证通道：**TLS/mTLS**（基于证书）和 **SASL**（基于机制）。

### 11.3.1 SSL（实为 TLS）

这里的"SSL"在现代 Kafka 中实际指 **TLS 1.2+**。它承担两个职责：

**1. 加密传输**：防止中间人窃听和篡改

**2. 双向认证（mTLS）**：客户端与 broker 互相出示证书

**Broker 端配置**：

```
# 启用 TLS 加密
security.inter.broker.protocol=SSL

# Keystore：broker 自己的证书和私钥
ssl.keystore.type=PKCS12
ssl.keystore.location=/var/private/ssl/kafka.server.keystore.p12
ssl.keystore.password=<keystore-password>
ssl.key.password=<key-password>

# Truststore：CA 证书，用于验证客户端证书
ssl.truststore.type=PKCS12
ssl.truststore.location=/var/private/ssl/kafka.server.truststore.p12
ssl.truststore.password=<truststore-password>

# 强制 mTLS（可选：none / requested / required）
ssl.client.auth=required

# 协议与套件
ssl.enabled.protocols=TLSv1.2,TLSv1.3
ssl.cipher.suites=TLS_AES_128_GCM_SHA256,TLS_AES_256_GCM_SHA384
```

**客户端证书要求**：

```
证书 CN 或 SAN 必须包含 broker 的 advertised hostname
使用 CA 签名的证书（而非自签名）
证书有效期 ≤ 90 天（行业最佳实践）
```

**Principal 提取**：

默认 SSL 用户名格式：`CN=writeuser,OU=Unknown,O=Unknown,L=Unknown,ST=Unknown,C=Unknown`

通过 `ssl.principal.mapping.rules` 映射为短名：

```
ssl.principal.mapping.rules=RULE:^CN=([a-zA-Z0-9.]*).*/$1/,DEFAULT
```

这样 `CN=producer-app,OU=...` 会被映射为 `producer-app`，用于 ACL 匹配。

### 11.3.2 SASL

SASL（Simple Authentication and Security Layer）是**认证机制的框架**，Kafka 支持多种机制：

#### SASL/PLAIN

最简单的机制，**明文传输用户名密码**，必须配合 TLS：

```
# Broker
sasl.enabled.mechanisms=PLAIN
listener.name.sasl_ssl.plain.sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required \
  username="admin" \
  password="admin-secret" \
  user_producer-app="producer-secret" \
  user_consumer-app="consumer-secret";

# Client
security.protocol=SASL_SSL
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required \
  username="producer-app" \
  password="producer-secret";
```

> ⚠️ PLAIN 的密码是静态 JAAS 文件，旋转密码需重启 broker。**生产环境不推荐**，除非配合外部密钥管理系统。

#### SASL/SCRAM（推荐）

SCRAM（Salted Challenge Response Authentication Mechanism）**不在网络上传输密码**，而是用挑战-响应协议证明密码知识：

```
# Broker 启用 SCRAM-SHA-512
sasl.enabled.mechanisms=SCRAM-SHA-512
sasl.mechanism.inter.broker.protocol=SCRAM-SHA-512

# 创建 SCRAM 凭证（KRaft 模式）
kafka-configs --bootstrap-server localhost:9092 --alter \
  --add-config 'SCRAM-SHA-512=[password=secure-password]' \
  --entity-type users --entity-name producer-app
```

**SCRAM 的核心优势**：

- 密码不以明文传输（challenge-response）
- 凭证存储在 KRaft 元数据日志（旧版在 ZooKeeper）
- **支持动态旋转，无需重启 broker**
- 适合多租户环境集中管理凭证

**SCRAM-SHA-256 vs SCRAM-SHA-512**：

> 💡 新部署推荐 **SCRAM-SHA-512**，加密属性更强。

#### SASL/GSSAPI（Kerberos）

企业环境首选，与 Active Directory 集成：

```
# Broker
sasl.enabled.mechanisms=GSSAPI
sasl.kerberos.principal.to.local.rules=DEFAULT

# 需要 JAAS 配置和 keytab 文件
```

**适用场景**：

- 已有 Kerberos/KDC 基础设施的大型组织
- 需要单点登录（SSO）
- 金融行业、大型企业

**代价**：需要 KDC、DNS 同步、时钟同步，运维复杂。

#### SASL/OAUTHBEARER

现代云原生身份集成的未来方向：

```
# Client
security.protocol=SASL_SSL
sasl.mechanism=OAUTHBEARER
sasl.jaas.config=org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required \
  clientId="kafka-client" \
  clientSecret="client-secret" \
  scope="kafka.cluster" \
  tokenEndpointUri="https://identity-provider.com/oauth2/token";

# Broker 端验证
sasl.enabled.mechanisms=OAUTHBEARER
listener.name.sasl_ssl.oauthbearer.sasl.server.callback.handler.class=org.apache.kafka.common.security.oauthbearer.secured.OAuthBearerValidatorCallbackHandler
listener.name.sasl_ssl.oauthbearer.sasl.jwks.endpoint.url=https://identity-provider.com/.well-known/jwks.json
```

**优势**：与 Okta、Azure AD、Auth0 等 IdP 集成，支持短期令牌自动旋转。

### 11.3.3 重新认证（KIP-368）

长连接面临一个问题：**认证状态会过期**（如 Kerberos ticket 或 OAuth token 的 TTL）。KIP-368 引入 `connections.max.reauth.ms`：

```
# 启用 SASL 重新认证
connections.max.reauth.ms=3600000  # 1 小时后强制重认证
```

**工作机制**：

- 连接在达到 `connections.max.reauth.ms` 阈值时，broker 强制客户端重新认证
- 若客户端未及时成功重认证，连接被终止
- 防止"认证后永不重新验证"的安全漏洞

> ⚠️ **历史教训**：KAFKA-9241 漏洞显示，若客户端不使用 `SaslAuthenticateRequest` API，即使启用了重认证，连接也不会被 kill。该漏洞在 Kafka 2.5.0 修复。**生产环境务必使用 Kafka 2.5+**。

### 11.3.4 安全更新不停机

这是工业界最关心的运维能力。三种常见场景：

**场景1：旋转 SCRAM 密码**

```
# 1. 添加新凭证（旧凭证仍有效）
kafka-configs --bootstrap-server localhost:9092 --alter \
  --add-config 'SCRAM-SHA-512=[password=new-secret]' \
  --entity-type users --entity-name producer-app

# 2. 客户端滚动重启使用新密码
# 3. 删除旧凭证
kafka-configs --bootstrap-server localhost:9092 --alter \
  --delete-config 'SCRAM-SHA-512' \
  --entity-type users --entity-name producer-app
```

**场景2：旋转 TLS 证书**

```
# 1. 将新 CA 证书添加到 truststore（broker 在握手时读取，无需重启）
keytool -importcert -keystore server.truststore.p12 \
  -alias new-ca -file new-ca.crt

# 2. 滚动重启 broker，逐个更新 keystore
# 3. 验证新证书生效
```

**场景3：切换 SASL 机制**

通过**分阶段 bounce**实现（见 11.2 节），先开新机制的 listener，客户端切换，最后关闭旧机制。

> 💡 **核心原则**：Kafka 的所有安全更新都可以通过"多 listener + 滚动重启"实现零停机。这是 Kafka 架构"listener 即边界"设计的直接红利。

---

## 11.4 加密

加密在 Kafka 中有三个层次：

**1. 传输加密（TLS）**：防止网络嗅探，配置见 11.3.1

**2. 存储加密**：依赖底层存储系统（LUKS、云平台 KMS）

**3. 应用层加密**：敏感字段在 producer 端加密，consumer 端解密

**工业实践**：

```
传输层：TLS 1.3（SASL_SSL 或 SSL）
存储层：云盘加密（AWS KMS / GCP KMS / Azure Key Vault）
应用层：PCI-DSS 字段用 AES-256 加密后再 produce
```

**TLS 性能考量**：

- TLS 握手有 CPU 开销，但现代 JVM + AES-NI 指令集下开销 < 5%
- 启用会话复用减少握手
- 内部 broker 间通信若在网络隔离环境，可用 SASL_PLAINTEXT 省 CPU

> 💡 **加密的第一性原理**：加密解决的不是"谁能连上来"（那是认证的事），而是"连上来之后传的数据能否被第三方读懂"。两者必须配合使用——**只加密不认证 = 可能被中间人冒充；只认证不加密 = 凭证和数据被嗅探**。

---

## 11.5 授权

### 11.5.1 AclAuthorizer

Kafka 的授权是**插件化的**，默认实现是 ACL（Access Control List）。

**ZooKeeper 集群（旧版）**：

```
authorizer.class.name=kafka.security.authorizer.AclAuthorizer
```

**KRaft 集群（新版，Kafka 3.3+）**：

```
authorizer.class.name=org.apache.kafka.metadata.authorizer.StandardAuthorizer
```

**ACL 的结构**：

```
Principal P 在 Host H 上对 Resource R 执行 Operation O 是 [Allow|Deny]
```

**五个维度**：

|维度|说明|示例|
|---|---|---|
|**Principal**​|用户或组|`User:alice`, `User:*`|
|**Operation**​|操作类型|Read, Write, Create, Describe, Delete, Alter|
|**Resource**​|资源类型|Topic, Group, Cluster, TransactionalId|
|**Resource Name**​|资源名称|`orders`, `Prefix-orders-*`, `*`|
|**Host**​|源 IP|`192.168.1.1`, `*`|

**创建 ACL 示例**：

```
# 允许 alice 和 fred 读写 finance topic
kafka-acls --bootstrap-server localhost:9092 \
  --add \
  --allow-principal User:alice \
  --allow-principal User:fred \
  --operation Read --operation Write \
  --topic finance

# 允许所有用户在任意主机消费以 "orders-" 为前缀的 topic
kafka-acls --bootstrap-server localhost:9092 \
  --add \
  --allow-principal User:* \
  --operation Read \
  --topic orders- --resource-pattern-type prefixed
```

**ACL 决策逻辑**：

```
1. 收集所有匹配的资源模式（Literal 精确匹配 + Prefixed 前缀匹配）
2. DENY 规则优先：任何 DENY 匹配 → 拒绝
3. 若无 DENY，检查 ALLOW：任何 ALLOW 匹配 → 允许
4. 既无 DENY 也无 ALLOW → 拒绝（默认拒绝原则）
```

> 📌 **默认拒绝**是 Kafka ACL 的核心安全原则——没有显式 ALLOW 的 principal 无法访问任何资源。

**超级用户**：

```
super.users=User:admin;User:kafka
```

**KRaft 模式的特殊之处**：

在 KRaft 中，CreateTopic 等管理请求由 broker 接收后**转发给 controller**，通过 Envelope 请求包装：

```
Client → Broker: CreateTopic 请求（携带 client principal）
Broker → Controller: Envelope{principal=client, request=CreateTopic}
Controller: 
  1. 先验证 broker 的 principal（broker 间认证）
  2. 再验证 envelope 中的 client principal
  3. 执行 CreateTopic 操作
```

这意味着自定义 principal 必须实现 `KafkaPrincipalSerde` 接口以支持序列化。

### 11.5.2 自定义授权

当 ACL 不足以表达复杂的授权逻辑时，可以实现自定义 Authorizer：

```
public class CustomAuthorizer implements Authorizer {
    @Override
    public List<AuthorizationResult> authorize(
        AuthorizableRequestContext requestContext,
        List<Action> actions
    ) {
        // 查询 LDAP/数据库/外部策略引擎
        // 返回 ALLOWED 或 DENIED
    }
}
```

**典型场景**：

- 与企业 LDAP/AD 组集成（如 `Group:Finance` → 允许操作 `Topic:Finance`）
- 基于属性的访问控制（ABAC）
- 与 OPA（Open Policy Agent）集成
- Confluent RBAC（角色基于访问控制）

**Confluent RBAC 的角色模型**：

```
Role: Developer → 允许读写 dev-* topics
Role: Operator → 允许集群管理操作
Role: Auditor  → 只读访问所有资源
```

### 11.5.3 安全方面的考虑

**1. ACL 爆炸问题**：

当 topic 数量达到数千个时，逐 topic 配置 ACL 会变成运维噩梦。解决方案：

- 使用 **Prefixed 模式**：`orders-*` 匹配所有订单相关 topic
- 使用 **自定义 Authorizer**​ 接入 LDAP 组
- 使用 **Confluent RBAC**​ 角色模型

**2. 最小权限原则**：

```
Producer 只需要：WRITE on Topic + CREATE on Topic（如允许自动创建）
Consumer 只需要：READ on Topic + READ on Group
Admin 需要：Cluster Action + 所有 Topic/Group 操作
```

**3. 网络安全加固**：

```
- 用防火墙限制 listener 端口的访问
- BROKER listener（9091）仅允许 broker 子网
- CONTROLLER listener（9094）仅允许 controller 子网
- CLIENT listener（9092）允许客户端子网
```

**4. 凭据管理**：

```
- 使用 Vault/AWS Secrets Manager 管理 JAAS 密码
- SCRAM 密码通过 kafka-configs 动态更新
- TLS 证书自动化轮换（90 天或更短）
```

---

## 11.6 审计

Kafka 的审计通过 authorizer 日志实现：

```
# log4j.properties
log4j.logger.kafka.authorizer.logger=INFO
```

每条 ACL 检查都会产生日志：

```
[2024-01-15 10:23:45] INFO Principal=User:alice is Allowed operation=Write on Resource=Topic:orders from Host=192.168.1.100
[2024-01-15 10:23:46] INFO Principal=User:bob is Denied operation=Read on Resource=Topic:finance from Host=192.168.1.101
```

**审计最佳实践**：

1. **日志转发到 SIEM**：Splunk、ELK、Datadog
2. **告警规则**：
    - 敏感 topic 的 DENY 事件
    - 未知 consumer group 消费 PII topic
    - 消费者 principal 执行 produce 操作（异常行为）
3. **Confluent Audit Log**：提供结构化 JSON 审计事件，包含完整 principal 和操作上下文

> 💡 **审计的第一性原理**：审计不是"记录日志"，而是"提供可追溯性"。当安全事件发生时，你必须能回答：谁、在什么时间、从哪个 IP、对什么资源、执行了什么操作、结果是允许还是拒绝。

---

## 11.7 保护 ZooKeeper（遗留）

> ⚠️ **重要前提**：Kafka 4.x 已移除 ZooKeeper，Confluent 7.5+ 新部署默认 KRaft 模式。本节内容仅适用于**仍在运行 ZooKeeper 的旧版集群**。新集群应使用 KRaft，其安全模型见 11.8 节。

### 11.7.1 SASL

ZooKeeper 3.5+ 支持 SASL 认证：

```
# zookeeper.sasl.client=true
# 配置 JAAS 文件供 ZooKeeper 客户端使用
```

### 11.7.2 SSL

ZooKeeper 3.5+ 支持 mTLS：

```
# zookeeper.client.secure=true
# zookeeper.server.secure=true
# 配置 keystore 和 truststore
```

### 11.7.3 授权

ZooKeeper 使用 ACL 限制 znode 访问：

```
# Kafka 自动设置 ZooKeeper ACL
zookeeper.set.acl=true
```

---

## 11.8 保护平台（KRaft 时代）

在 KRaft 模式下，安全边界发生了变化：

**1. 无 ZooKeeper 攻击面**：

```
ZooKeeper 模式：Kafka + ZooKeeper + ZK ACL + ZK TLS = 4 层安全配置
KRaft 模式：Kafka（含 Controller）= 1 层安全配置
```

**2. Controller 通信安全**：

```
# Controller listener 必须使用 TLS
controller.listener.names=CONTROLLER
listener.security.protocol.map=CONTROLLER:SSL

# Controller 间用 mTLS
controller.quorum.voters=1@controller1:9094,2@controller2:9094,3@controller3:9094
ssl.client.auth=required
```

**3. KRaft 元数据日志保护**：

`__cluster_metadata` topic 存储所有 ACL 和配置，必须：

```
# 限制对 __cluster_metadata 的访问
kafka-acls --bootstrap-server localhost:9092 \
  --add \
  --allow-principal User:admin \
  --operation All \
  --topic __cluster_metadata
```

**4. 凭据存储**：

```
ZooKeeper 模式：SCRAM 凭证存在 ZooKeeper znode
KRaft 模式：SCRAM 凭证存在元数据日志（自动管理）
```

**5. 网络安全**：

```
Controller 节点应位于独立网段
Controller listener 端口仅允许 broker IP 访问
使用网络策略（Kubernetes NetworkPolicy）隔离
```

### 工业界 Kafka 安全基线配置

```
# ========== Listener 配置 ==========
listeners=BROKER://0.0.0.0:9091,CLIENT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9094
listener.security.protocol.map=BROKER:SASL_SSL,CLIENT:SASL_SSL,CONTROLLER:SSL
inter.broker.listener.name=BROKER
controller.listener.names=CONTROLLER

# ========== SASL 配置 ==========
sasl.enabled.mechanisms=SCRAM-SHA-512
sasl.mechanism.inter.broker.protocol=SCRAM-SHA-512

# ========== TLS 配置 ==========
ssl.enabled.protocols=TLSv1.2,TLSv1.3
ssl.cipher.suites=TLS_AES_128_GCM_SHA256,TLS_AES_256_GCM_SHA384
ssl.truststore.type=PKCS12
ssl.truststore.location=/var/private/ssl/kafka.truststore.p12
ssl.truststore.password=${Vault_TLS_TRUSTSTORE_PASSWORD}
ssl.keystore.type=PKCS12
ssl.keystore.location=/var/private/ssl/kafka.keystore.p12
ssl.keystore.password=${Vault_TLS_KEYSTORE_PASSWORD}
ssl.key.password=${Vault_TLS_KEY_PASSWORD}

# ========== 授权配置 ==========
authorizer.class.name=org.apache.kafka.metadata.authorizer.StandardAuthorizer
super.users=User:admin

# ========== 重认证 ==========
connections.max.reauth.ms=3600000

# ========== 审计 ==========
log4j.logger.kafka.authorizer.logger=INFO
```

---

## 11.9 小结

把全章串起来，Kafka 安全是一个**四层纵深防御体系**：

```
┌─────────────────────────────────────────────────────────┐
│ 第一层：网络边界（Listener）                               │
│  • 每个 listener 独立安全协议                              │
│  • 防火墙限制端口访问                                      │
│  • 不同信任域用不同 listener                               │
├─────────────────────────────────────────────────────────┤
│ 第二层：认证（Authentication）                             │
│  • TLS/mTLS：基于证书的相互认证                            │
│  • SASL/PLAIN：简单但需 TLS 保护                          │
│  • SASL/SCRAM：推荐，支持动态凭证旋转                     │
│  • SASL/Kerberos：企业 SSO                               │
│  • SASL/OAUTHBEARER：云原生 IdP 集成                      │
├─────────────────────────────────────────────────────────┤
│ 第三层：授权（Authorization）                              │
│  • ACL：per-principal 的 Allow/Deny 规则                  │
│  • 默认拒绝原则                                           │
│  • 自定义 Authorizer：LDAP/RBAC/OPA                       │
│  • KRaft 用 StandardAuthorizer，ACL 存元数据日志           │
├─────────────────────────────────────────────────────────┤
│ 第四层：审计与加密                                         │
│  • 传输加密：TLS 1.3                                     │
│  • 存储加密：云平台 KMS                                   │
│  • 审计日志：SIEM 集成 + 告警                             │
└─────────────────────────────────────────────────────────┘
```

### 生产环境安全决策框架

```
新部署？
├─ 是 → KRaft 模式 + StandardAuthorizer
│       Controller 用 mTLS
│       Broker 间用 SASL_SSL + SCRAM-SHA-512
│      客户端用 SASL_SSL + SCRAM 或 OAUTHBEARER
│
└─ 否（ZooKeeper 集群）→ 计划迁移到 KRaft
        过渡期：AclAuthorizer + ZooKeeper SASL/SSL

认证机制选型？
├─ 多租户 SaaS → SASL/SCRAM-SHA-512（集中管理）
├─ 企业内部 → SASL/GSSAPI（Kerberos SSO）
├─ 云原生 → SASL/OAUTHBEARER（OIDC 集成）
└─ 开发测试 → SASL/PLAIN + TLS（最简单）

授权复杂度？
├─ 简单 → ACL（Prefixed 模式）
├─ 中等 → ACL + LDAP 自定义 Authorizer
└─ 复杂 → Confluent RBAC
```

### 给读者的"Kafka 安全心智模型"

> 📌 **全章最核心的一句话**：Kafka 安全的本质是"以 listener 为边界、以插件化 Authorizer 为核心、以 KRaft 元数据日志为 ACL 存储"的三层架构——认证回答"你是谁"（TLS/mTLS 或 SASL），授权回答"你能做什么"（ACL 或 RBAC），加密回答"数据能否被窃听"（TLS）。理解 listener 是安全边界这一第一性原理，你就能理解为什么 Kafka 可以做到"broker 间用 SCRAM、客户端用 OAUTHBEARER、Controller 用 mTLS"的精细化控制，以及为什么安全更新可以零停机进行。

**三个必须刻进肌肉记忆的认知**：

1. **listener 即安全边界**。Kafka 不是"一个集群一个安全配置"，而是"每个 listener 一个安全配置"。这是 Kafka 安全架构最精妙的设计——它让安全策略可以精确到端口级别。
2. **默认拒绝是金科玉律**。启用 AclAuthorizer 后，没有显式 ALLOW 的 principal 无法访问任何资源。这意味着：启用 ACL 前必须预先创建好所有必要的 ACL，否则会锁死所有客户端。
3. **KRaft 改变了安全拓扑**。ZooKeeper 模式下你需要保护 Kafka + ZooKeeper + ZK ACL + ZK TLS（4 层）；KRaft 模式下只有 Kafka 自身（1 层），但 Controller 通信安全成为新焦点。SCRAM 凭证从 ZooKeeper znode 迁到元数据日志，自动化程度更高。

**工业界真相**：

> 绝大多数 Kafka 安全事件不是因为 Kafka 本身不安全，而是因为：
> 
> - 使用 `SASL_PLAINTEXT` 暴露在公网（凭证被嗅探）
> - 证书过期未自动轮换（导致大规模断连）
> - 启用 ACL 但未预先配置（锁死所有客户端）
> - Controller 端口暴露给客户端网络（集群被接管）
> - 审计日志未接入 SIEM（安全事件不可追溯）
> 
> Kafka 的安全模型是完整的，但安全的 Kafka 集群需要**运维纪律**：证书 90 天自动轮换、凭据用 Vault 管理、ACL 用 Prefixed 模式避免爆炸、审计日志实时告警。工具只是基础，运维才是关键。

下一章我们将进入"流式处理"（Streams），你会看到：当数据在安全、可靠的 Kafka 集群中流动时，如何用 Kafka Streams 做有状态的流处理。带着"listener 是安全边界"这条主线读下去，Kafka Streams 应用的 SASL_SSL 配置、Streams 内部 topic 的 ACL 设置、exactly-once 的事务隔离级别，都会变得清晰——它们本质上是把"安全边界"从 broker 延伸到了流处理应用：你的 Streams 应用也是一个"客户端"，需要被认证、被授权、被加密。