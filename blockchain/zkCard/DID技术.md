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

DID Doc可以被看作是一个身份信息地图，如图6所示由两部分组成，第一部分被称为标签，在DID Doc中可以查询到并可以直接阅读的内容，包括三部分：核心标签（如id,controller,authentication等）、拓展标签（如以太坊地址等）、以及一些未在W3C DID规范登记的标签；第二部分是未在DID Doc列出，而是借助URL等特定形式，链接到第三方平台或网站系统查询相关身份信息；为了保证最大程度的互操作性和信息兼容性，W3C建立了DID Specification Registry,保证特定形式的内容在DID Doc中是可以被识别和解析的。当有新的标签出现，相关平台或系统需要向DID Specification Registry登记。

![[Pasted image 20220904221740.png]]


不同DID之间可能存在信息交互关系，如图7所示，在W3C中提出了Production & Consumption概念：创建一个DID Document的过程是Production,而将创建的这条Document引用至该DID Subject其他DID创建过程则是Consumption。在验证过程中，每个DID对应的DID Document是独立的，相当于对每个DID做了信息隔离。在验证过程中，DID持有人可以根据需要对不同DID授权，验证人只能阅读到被授权的DID Doc，而无法获得更多信息，从而达到DID Subject的信息保护目的。

![[Pasted image 20220904221756.png]]

-   **Verifiable Data Registries (VDR)**

DID的初衷是将用户身份信息管理权从平台交回用户自己，这过程中用户必须解决的问题是信息存储在哪里？以及需要验证的时候去哪里找到这些数据？怎么保证数据的真实性？VDR讨论的就是怎么解决这些问题，我们将支持记录DID数据且能够在生成DID Doc时提供相关数据的系统称为Verifiable Data Registry（VDR），这种系统包括分布式账本、分布式文件系统、P2P网络或其他可被信任的渠道；而且VDR与DID Method有着直接的关联性，一般每个VDR都会基于W3C DID规范提出自己的DID Method。目前市场主推DID储存媒介是钱包，分为托管钱包（如Coinbase）、普通钱包（如imtoken），以及智能钱包（Gnosis Safe, Dappe，Argent），具体哪种媒介能更有效的存储DID信息，暂不在本文详细讨论。

**DID的实现：“身份-身份证明-身份验证”**

我们基于第一部分的“身份-身份证明-身份验证“，简单讨论DID如何实现这些功能的。

**身份**

在DID方案中，每个人可以在不同场景、不同时间，因为不同目的，在任意可信的第三平台登记不同的DID，相关权益和资产所属权与不同的DID直接绑定，而身份主体通过持有DID来证明其对资产的所有权或具体权益。DID没有直接与物理世界身份生成映射关系，且DID信息维护也是由身份主体或可信第三方来维护，保证了信息的安全性。对于身份主体而言，需要做好DID的安全持有工作，同时维护好与DID对应的身份文件(DID Doc)。

**身份证明**

DID只是一串带有密钥的随机数值，在具体使用中第三方机构根据DID信息将身份证明写入DID Doc，同时第三方机构会将自己的数字签名加入文件中，方便后期身份验证。例如，张三，需要证明自己具有驾驶能力，此时无需像传统的中心化方法，由权威机构签发一张驾驶证给张三，具体个人信息也无需存储在权威机构的数据库里；通过DID技术给出的解决方案是[1]：张三向车管所提供自己准备的DID或使用车管所提供的DID，车管所按照DID Doc的JSON-LD数据结构写入相关信息（包括但不限于id，type,有效期，controller，验证方法等），同时加入车管所的数字签名。DID Doc可以储存在车管所，可以储存在张三的智能钱包里，或者其他存储媒介。需要注意的是，此处DID并没有泄露张三的身份特征，没有映射物理世界的其他身份证明，这个DID只是张三持有的众多DID中的一个。因此，只要张三本人不出示DID证明，就没有人能知道这份DID Doc是张三的，从而保护了张三的个人隐私。

**身份验证**

验证的主要目的是为了证明目标主体可以合规的或有权限的进行某项程序，在W3C《DID V1.0白皮书》中从验证目的进行了梳理，分为五类，分别是：验证、申明、重要协议、性能调用和性能授权。根据五种不同目的，在DID Method中可以设计不同的方案。验证信息的来源分为两类，一类是DID Doc中列举的数据；另一类是需要借助于外部系统或平台的数据，对于材料的格式W3C做了要求（主要包括publicKeyJwk和publickeymultibase），以方便解析及识别。下面以“申明”为例说明，根据国家最新青少年网络游戏规定中要求每天限时一小时，传统的方法则需要上传身份证信息，但分布式标识符解决方案中，只需要提供自己持有的DID，通过零知识证明验证用户是否超过18岁即可，而无需告知平台方用户具体年龄。这只是众多验证方法中的一种。

### A16z的DID

[Anonymous Credentials and Decentralizing ID Systems with Foteini Baldimtsi | a16z crypto research - YouTube](https://www.youtube.com/watch?v=kgNcxQH-hMQ)

