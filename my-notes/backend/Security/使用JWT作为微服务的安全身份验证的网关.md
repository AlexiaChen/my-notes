
## 简介

本博客深入探讨了使用身份验证(Authentication)网关为微服务提供安全访问的方法。它描述了网关如何使用JSON Web Token（JWT）对希望访问由不同微服务托管的Web服务端点(API endpoint)的客户进行身份验证(Authentication)。
## JWT

分为三个部分

- Header，标识了采用的签名算法和Token类型，Token类型就是JWT。  用Based64Url编码的JSON
- Payload，又叫Claims，存储在JWT内的数据。一个实体信息。比如用户和其他额外信息

```json
{
 "sub": "1234567890",
 "name": "Mark",
 "admin": true,
 "iss": http://abc.com,
 "iat": 1472033308,
 "exp": 1472034208
}
```

- Signature  签名是用SecretKey对编码后的Header和Payload签名，比如用HS256:

```java
HMACSHA256( 
base64UrlEncode(header) + "." +_
base64UrlEncode(payload),  secret)_
```

JWT令牌应该以Bearer模式发送到Http Authorization头部，用于访问下面展示的受保护资源。

```txt
Authorization: Bearer <JWT token>
```
## 系统架构

该系统是由一组相互通信的Spring Boot应用程序(分布式微服务)实现的。Apache Maven被用作应用程序的依赖和构建工具。从下面的工作流程图中可以最好地理解系统组件。

![[Pasted image 20240531145229.png]]

### 系统组件和描述

#### 客户端Client

它可以是任何平台上的任何Web服务消费者。简单来说，它可以是另一个Webservice、UI应用程序或移动平台，希望以安全方式与具有各种微服务的应用程序进行数据读写交互。通常情况下，上述工作流程图中显示的客户端请求可以是从任何REST客户端调用的REST API请求。

#### Eureka服务注册表

Eureka服务注册表是另一个主要处理不同微服务之间通信的微服务。每个微服务在Eureka服务器中注册自己时，必须具备高可用性，因为每个服务都与它通信以发现其他服务。

#### 微服务A和B

这些是托管应用程序不同资源的各种REST API端点，即GET/POST/PUT/DELETE的微服务。每个微服务都是一个Eureka客户端应用程序，并且必须向Eureka服务器注册自己。

#### 身份验证网关

网关是使用Spring Cloud Zuul Proxy和Spring Security API实现的微服务。它处理集中式身份验证(Authentication)并将客户端请求路由到各种微服务，使用Eureka服务注册表路由请求到微服务。它充当客户端的代理，抽象出微服务架构，并且必须具有高可用性，因为它作为不同操作的单一交互点工作，无论是用户注册、用户凭据验证、生成和验证JWT令牌还是通过与相关微服务端点通信来处理客户端业务请求。简而言之，它是一个请求路由器，同时也兼作认证微服务。

或者，包括不同身份验证机制（JWT、OAuth）在内的用户管理功能也可以作为独立的“用户管理”微服务托管。在这种情况下，网关只需作为轻量级请求路由器工作，并通过JWT令牌与其进行用户身份验证通信。

### JWT认证流程

- 客户通过POST URI /users/signup向身份验证网关提供用户名和密码进行注册（该URI允许公开访问，没有任何安全性限制）。
- 客户通过发送凭据，从身份验证网关通过POST URI /token/generate-token请求“JWT访问令牌”。
- 认证网关验证凭据，并在成功认证后生成包含用户详细信息和权限permission的JWT访问令牌。该令牌作为响应发送给客户端。
- 客户然后在每个REST API请求中的授权头部(Authorization Header)发送JWT访问令牌到认证网关。
- 认证网关从客户端请求的授权头(Authorization Header)中检索JWT访问令牌并验证签名。如果签名有效，则根据在application.yml或application.properties中配置的路由将请求路由到匹配的端点（微服务）。也可以通过JwtAuthenticationFilter处理微服务级别的授权。
- 网关还可以通过自定义的Zuul过滤器在请求头中发送额外的参数（JWT令牌、其他用户信息等）。
- 微服务可以选择性地对请求进行授权，并向身份验证网关提供响应。对于授权，微服务需要传递JWT访问令牌。然后，它可以验证JWT令牌并从声明中提取用户角色，并根据情况允许/拒绝有关API端点的请求。
## 结论

JWT身份验证网关为保护微服务应用程序提供了一种非常有用的方法，对微服务代码的影响很小。因此，应用程序开发人员可以专注于核心业务逻辑，而不必担心保护应用程序的安全机制。它可以独立扩展和部署以进行负载测试和维持高可用性。它还可以作为其他横切关注点（如微服务监控、路由规则等）的集中实体。