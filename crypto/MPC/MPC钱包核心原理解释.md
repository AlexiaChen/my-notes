原理及实现简单介绍

# 一. 首次创建钱包

## 原理解释

创建钱包时，手机端与服务器之间通过安全多方计算（MPC），为每一个参与方生成一个密钥（私钥分片），并共同计算出一个公钥用于对应这一个私钥序列。

在创建钱包和后面签名的过程中，不会构建出完整的私钥，也就没有私钥泄漏的风险。

## 实现步骤

1. 用户通过滑块验证后，得到临时session；

2. 用临时session，与服务器之间建立联系，进行MPC过程；

3. MPC结束，手机端得到私钥分片（keyshare-u），并保存在本地（Android：从keystore中获取密钥，用密钥对keyshare-u加密，把加密后的keyshare-u保存到应用沙盒；Ios：从安全飞地中获取密钥，用密钥对keyshare-u加密并保存到keychain）。过程中，服务器也会给手机端分配一个UID，与此keyshare-u关联。

# 二. 备份钱包

## 原理解释

生成并保存备份文件的过程，是先随机生成一个AES密钥，用AES密钥对keyshare-u进行加密。然后生成一对RSA公钥和私钥，用RSA公钥对AES密钥加密。完成加密后，将加密后的keyshare-u和加密后的AES密钥保存到服务器，将RSA私钥保存在用户个人云盘。（可忽略上面实现，本质是把加密后的keyshare-u放到了服务器，把加密的密钥放在用户个人云盘）

## 实现步骤

整体的过程，是注册+备份。注册包括邮箱/手机号注册和人脸信息注册，备份包括备份至服务器和用户个人云盘。详细的实现步骤如下：

1. 邮箱/手机号验证，验证通过后得到临时session；

2. 用临时session，与服务器之间建立联系，并调用Facetec SDK进行人脸特征采集（采集的过程不会直接保存用户面部的图像）；

3. 人脸特征采集成功后，得到正式的session，并保存在本地；

4. 从本地取出要备份钱包的keyshare-u，过程中需要解锁设备；

5. 对keyshare-u进行加密，将加密后的keyshare-u保存在服务器，将加密的密钥保存在用户选择的个人云盘；

# 三. 非首次创建钱包

## 实现步骤

非首次创建钱包，是先创建钱包，然后自动对此钱包进行备份。非首次创建钱包，需要钱包处于已备份的状态。详细实现步骤如下：

1. 输入钱包备注名，开始创建新钱包；

2. 与服务器之间通过MPC过程，得到keyshare-u，并保存在本地，钱包创建完成；

3. 从本地取出新创建的keyshare-u，过程中需要解锁设备；

4. 用同样的密钥对keyshare-u进行加密，将加密后的keyshare-u保存到服务器。

# 四. 恢复钱包

## 实现步骤

恢复钱包，是登录+恢复的过程。登录时，需要验证备份时的注册邮箱/手机号，以及人脸信息。验证通过后，从服务器和云盘获取到要恢复的文件及密钥，进行恢复。

1. 输入邮箱/手机号，申请随机验证码进行验证，验证通过后得到临时session；

2. 用临时session，与服务器进行Facemap的验证，验证通过后，得到正式的session，并保存在本地；

3. 用户选择个人云盘，从服务器获取加密后的keyshare-u，并从用户个人云盘获取密钥，然后进行解密；

4. 将得到的keyshare-u保存在本地；

5. 从服务器获取加密后的配置文件，并用同样的密钥解密；

6. 将恢复得到的配置文件，保存在本地。恢复结束。

# 五. 转账

## 原理解释

签名过程，是手机终端和服务器，分别将各自的私钥分片作为隐私输入，将需要签名的信息作为公共输入，进行一次联合签名运算。在此过程中，参与方并不能获得其他方的私钥信息，但都可以得到签名结果。

## 实现步骤

1. 输入收款人地址、转账金额、矿工费信息后，提交转账；

2. 先从节点获取当前的nonce值，在已确认交易的nonce值基础上+1，然后与用户输入拼接出要签名的交易信息，包括Transaction Hash；

3. 从本地取出当前钱包地址的keyshare-u，过程中需要解锁设备；

4. 与服务器之间进行MPC运算，对此交易信息进行签名；

5. 签名完成后，将签名后的交易信息，提交到RPC节点，交易状态设置为pending；

6. 对pending中的交易，通过Transaction Hash，到RPC节点轮询此交易的状态，并在本地更新；

# 六. 校验和同步

## 原理解释

用户将密钥备份至个人云盘时，同时基于密钥的文件名、修改时间、文件大小等内容，生成此密钥的“指纹”信息，并保存在本地。App定期用此“指纹”信息，与个人云盘的文件进行比对。当校验发现异常时，提醒用户通过将本地的密钥同步至个人云盘进行修复。

## 实现步骤

1. App切换至前台（包括打开App），都会校验距离上次备份时否超过5min，若超过5min，启动校验流程；

2. 用本地保存的“指纹”信息，与云盘的密钥进行比对；

3. 若比对结果一致，则备份正常。若比对结果不一致，则备份结果异常，提醒用户修复；

4. 用户修复时，先从本地取出加密的密钥，此过程需要解锁设备；

5. 将加密的密钥同步到所选的云盘，完成备份修复。

# 七. 钱包管理

在用户管理钱包时，涉及钱包账户、钱包地址、链这些概念：

- 钱包账户：即UID，与用户的身份相对应，在创建钱包时生成，一个UID下可关联多个钱包地址；

- 钱包地址：每一个钱包地址对应唯一的公钥和私钥分片，同一钱包地址可存在于多条兼容链（当前钱包只支持EVM兼容链，这样每创建一个钱包地址，在所有的EVM兼容链上，都会生成一个彼此独立，但地址相同的钱包地址）；

- 链：也称为网络，对应一个公链。每个链下，可存在多个不同的钱包地址；