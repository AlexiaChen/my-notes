#### Netflix消息推送到设备的系统设计

![[Pasted image 20220827221017.png]]

𝐑𝐞𝐪𝐮𝐢𝐫𝐞𝐦𝐞𝐧𝐭𝐬 & 𝐬𝐜𝐚𝐥𝐞  
- 2.2亿用户  
- 近乎实时  
- 后台系统需要向各种客户端发送通知  
- 支持的客户端：iOS、Android、智能电视、Roku、Amazon FireStick、网络浏览器  
  
1. 推送通知事件是由时钟、用户行为或系统触发的。  
2. 事件被发送至事件管理引擎。  
3.事件管理引擎监听特定的事件并将事件转发到不同的队列中。这些队列由基于优先级的事件转发规则填充。  
4. "基于事件优先级的处理集群 "处理事件并为设备生成推送通知数据。  
5. 一个Cassandra数据库被用来存储通知数据。  
6. 一个推送通知被发送到外发信息系统。  
7. 对于安卓系统，FCM被用来发送推送通知。对于苹果设备，使用的是APN。对于网络、电视和其他流媒体设备，使用Netflix自制的解决方案，称为 "Zuul Push"。  
  

[Rapid Event Notification System at Netflix | by Netflix Technology Blog | Netflix TechBlog](https://netflixtechblog.com/rapid-event-notification-system-at-netflix-6deb1d2b57d1)
