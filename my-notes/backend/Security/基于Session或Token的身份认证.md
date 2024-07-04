
这里的身份认证就是Authentication，不是Authorization。

HTTP是无状态的。所有请求都是无状态的。然而，有些情况下我们希望能够记住我们的状态。例如，在一个在线商店中，当我们把香蕉放入购物车后，当我们去另一个页面购买苹果时，我们不希望香蕉消失。也就是说，在浏览在线商店时，我们希望记住购买状态！

为了克服HTTP请求的无状态特性，我们可以使用会话Session或令牌Token之一。

### Session based身份认证

在基于会话的身份验证中，服务器将在用户登录后为用户创建一个会话。然后，会话ID存储在用户浏览器上的cookie中。当用户保持登录状态时，每个后续请求都会发送该cookie。然后，服务器可以将存储在cookie上的会话ID与内存中存储的会话信息进行比较，以验证用户身份，并发送相应状态！

![[Pasted image 20240531142124.png]]

### Token based认证

![[Pasted image 20240531142146.png]]


> 其实在之前的有篇文章 [[我应该使用JWT来作为认证Token吗]] 已经说明了，一般不推荐JWT作为身份认证(Authentication)， JWT的无法吊销的问题，被偷了就完蛋了, 即使是在JWT的payload中加入expire的时间戳，过期时间还不能太长，也仅仅只是降低攻击面。如果是单体应用直接无脑用最安全的Session 身份认证，如果是微服务的身份认证，其实可以用JWT。详情请看 [[使用JWT作为微服务的安全身份验证的网关]]  [[在微服务中的身份认证-技术方案]]

## 评论

> For authentication and authorization, cookies are the only safest possible mechanism at present for web apps. Tokens are not safe at all. Read these articles: https://developer.okta.com/blog/2018/06/20/what-happens-if-your-jwt-is-stolen   https://www.rdegges.com/2018/please-stop-using-local-storage/

> Nice article but JWT isn't good for Authentication. Because you can't revoke JWT if it is stolen. Issue a highly random token and store it in secure cookie on user login and store the token on backend database. If the user want to logout, just delete that token from backend.