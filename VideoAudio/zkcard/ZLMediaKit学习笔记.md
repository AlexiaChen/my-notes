它既是一个独立的Media Server服务器，又是一个Server和Client的框架，即支持推流，也支持拉流。

### ZLMediaKit的大致原理

![[Pasted image 20220728161159.png]]

[ZLMediaKit高并发实现原理 · ZLMediaKit/ZLMediaKit Wiki (github.com)](https://github.com/zlmediakit/ZLMediaKit/wiki/ZLMediaKit%E9%AB%98%E5%B9%B6%E5%8F%91%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%86)



### Http Restful 接口

[MediaServer支持的HTTP API · ZLMediaKit/ZLMediaKit Wiki (github.com)](https://github.com/zlmediakit/ZLMediaKit/wiki/MediaServer%E6%94%AF%E6%8C%81%E7%9A%84HTTP-API)


如果不是127.0.0.1的地址API请求，那么就需要带上config.ini中的secret参数

![[ed74a6c160c0971f4bb370776611b15.png]]

以上请求是拉取当前Media Server的配置。