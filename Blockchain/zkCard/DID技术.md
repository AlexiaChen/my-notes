#### 简介

DID是去中心ID的一种规范描述。是全网唯一的。类似于URI这样的描述。

#### 实现

DID技术的核心构成要素包括三个：DID、DID Document和Verifiable Data Registry

##### DID

DID属于统一资源标识符URI的一种，是一个永久不可变的字符串，它存在的意义有两点，第一，标记任何目标对象(DID Subject)，可以是一个人、一件商品、一台机器或者一只动物等等；第二，DID是通过DID URL关联到描述目标对象的文件（DID Document,简称DID Doc）唯一标识符，即通过DID能够在数据库中搜索到具体的DID Doc。

##### DID Scheme

DID分为三个部分，如图所示，第一部分是DID Scheme（类似URL中的http,https,ftp等协议）；第二部分是DID方法标识符（一般是DID方法的名称）；第三部分是DID方法中特定的标识符：在整个DID方法命名空间是唯一的。W3C只规范了DID的表示结构，即<did:+DID method:+DID Method-Specific Identifier>,但没有规范三部分内容的具体标准，具体内容与DID Method有关。

![[Pasted image 20220823154943.png]]

##### DID Method

DID Method是一组公开的操作标准，定义了DID的创建、解析、更新和删除，并涵盖了DID在身份系统中注册、替换、轮换、恢复和到期等。目前没有统一的操作标准，各个公司可以根据场景特征自行设计，由W3C CCG工作组统一维护。截至2021年8月3日发布《DID V1.0》,在W3C登记的DID Method高达103项，均有不同的名称和特定的标识符表示方法。

##### DID URL

为融合现有URI网络位置标识方法，DID使用了DID URL表示资源的位置（如路径、查询和片段）。W3C对DID URL的语法描述ABNF规定如下：<did-url = did path-abempty \[“?”query\]\[“#”fragment\]>。

##### DID Document

DID Document(DID Doc)包含着所有与DID subject有关的信息，在Doc中有身份信息验证方法（包括加密公钥，相关地址等）。DID Doc是一个通用数据结构，通常是由DID controller负责数据写入和更改，文件里包含与DID验证相关的密钥信息和验证方法，提供了一组使DID控制者能够证明其对应DID控制的机制。需要说明的是，这里的管理DID Doc的DID Controller可能是DID subject本人，也有可能是第三方机构，不同DID Method对DID Doc的权限管理有所区别。

下图是之前上一幅图片对应的URL 的Doc的文件。存储在所有人能控制的位置（可以是中心化的，也可以去中心化的），以便轻松查找。


![[Pasted image 20220823155331.png]]






### A16z的DID

[Anonymous Credentials and Decentralizing ID Systems with Foteini Baldimtsi | a16z crypto research - YouTube](https://www.youtube.com/watch?v=kgNcxQH-hMQ)

