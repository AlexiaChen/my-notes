
以下是通过ElastiCache管理控制台创建、授予访问权限、连接和最终删除Redis（禁用集群模式）集群的主题。作为其中的一部分，本节将帮助您确定集群的要求并创建自己的AWS账户。Amazon ElastiCache通过使用Redis复制组来支持高可用性。有关Redis复制组及其创建方法的信息，请参阅使用[复制组实现高可用性]([High availability using replication groups - Amazon ElastiCache for Redis](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/Replication.html))。从Redis 3.2版本开始，ElastiCache Redis支持在多个节点组之间对数据进行分区，每个节点组都实现了一个复制组。此操作将创建一个独立的Redis集群。

## 创建

### 创建你的AWS账号

#### 注册一个AWS账号

If you do not have an AWS account, complete the following steps to create one.

###### To sign up for an AWS account

1. Open [https://portal.aws.amazon.com/billing/signup](https://portal.aws.amazon.com/billing/signup).
    
2. Follow the online instructions.
    
    Part of the sign-up procedure involves receiving a phone call and entering a verification code on the phone keypad.
    
    When you sign up for an AWS account, an _AWS account root user_ is created. The root user has access to all AWS services and resources in the account. As a security best practice, [assign administrative access to an administrative user](https://docs.aws.amazon.com/singlesignon/latest/userguide/getting-started.html), and use only the root user to perform [tasks that require root user access](https://docs.aws.amazon.com/accounts/latest/reference/root-user-tasks.html).
    

AWS sends you a confirmation email after the sign-up process is complete. At any time, you can view your current account activity and manage your account by going to [https://aws.amazon.com/](https://aws.amazon.com/) and choosing **My Account**.
#### 创建一个管理员账户

After you sign up for an AWS account, create an administrative user so that you don't use the root user for everyday tasks.

###### Secure your AWS account root user

1. Sign in to the [AWS Management Console](https://console.aws.amazon.com/) as the account owner by choosing **Root user** and entering your AWS account email address. On the next page, enter your password.
    
    For help signing in by using root user, see [Signing in as the root user](https://docs.aws.amazon.com/signin/latest/userguide/console-sign-in-tutorials.html#introduction-to-root-user-sign-in-tutorial) in the _AWS Sign-In User Guide_.
    
2. Turn on multi-factor authentication (MFA) for your root user.
    
    For instructions, see [Enable a virtual MFA device for your AWS account root user (console)](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_mfa_enable_virtual.html#enable-virt-mfa-for-root) in the _IAM User Guide_.
    

###### Create an administrative user

- For your daily administrative tasks, grant administrative access to an administrative user in AWS IAM Identity Center.
    
    For instructions, see [Getting started](https://docs.aws.amazon.com/singlesignon/latest/userguide/getting-started.html) in the _AWS IAM Identity Center User Guide_.
    

###### Sign in as the administrative user

- To sign in with your IAM Identity Center user, use the sign-in URL that was sent to your email address when you created the IAM Identity Center user.
    
    For help signing in using an IAM Identity Center user, see [Signing in to the AWS access portal](https://docs.aws.amazon.com/signin/latest/userguide/iam-id-center-sign-in-tutorial.html) in the _AWS Sign-In User Guide_.

### 授权编程访问

如果用户想要在 AWS 管理控制台之外与 AWS 进行交互，他们需要编程访问权限。授予编程访问权限的方式取决于访问 AWS 的用户类型。为了授予用户编程访问权限，请选择以下选项之一。

|Which user needs programmatic access?|To|By|
|---|---|---|
|Workforce identity<br><br>(Users managed in IAM Identity Center)|Use temporary credentials to sign programmatic requests to the AWS CLI, AWS SDKs, or AWS APIs.|Following the instructions for the interface that you want to use.<br><br>- For the AWS CLI, see [Configuring the AWS CLI to use AWS IAM Identity Center](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-sso.html) in the _AWS Command Line Interface User Guide_.<br>    <br>- For AWS SDKs, tools, and AWS APIs, see [IAM Identity Center authentication](https://docs.aws.amazon.com/sdkref/latest/guide/access-sso.html) in the _AWS SDKs and Tools Reference Guide_.|
|IAM|Use temporary credentials to sign programmatic requests to the AWS CLI, AWS SDKs, or AWS APIs.|Following the instructions in [Using temporary credentials with AWS resources](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp_use-resources.html) in the _IAM User Guide_.|
|IAM|(Not recommended)<br><br>Use long-term credentials to sign programmatic requests to the AWS CLI, AWS SDKs, or AWS APIs.|Following the instructions for the interface that you want to use.<br><br>- For the AWS CLI, see [Authenticating using IAM user credentials](https://docs.aws.amazon.com/cli/latest/userguide/cli-authentication-user.html) in the _AWS Command Line Interface User Guide_.<br>    <br>- For AWS SDKs and tools, see [Authenticate using long-term credentials](https://docs.aws.amazon.com/sdkref/latest/guide/access-iam-users.html) in the _AWS SDKs and Tools Reference Guide_.<br>    <br>- For AWS APIs, see [Managing access keys for IAM users](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html) in the _IAM User Guide_.|

###### Related topics:

- [What is IAM](https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction.html) in the _IAM User Guide_.
    
- [AWS Security Credentials](https://docs.aws.amazon.com/general/latest/gr/aws-security-credentials.html) in _AWS General Reference_.
### 设置你的权限（新的用户才需要）

To provide access, add permissions to your users, groups, or roles:

- Users and groups in AWS IAM Identity Center:
    
    Create a permission set. Follow the instructions in [Create a permission set](https://docs.aws.amazon.com/singlesignon/latest/userguide/howtocreatepermissionset.html) in the _AWS IAM Identity Center User Guide_.
    
- Users managed in IAM through an identity provider:
    
    Create a role for identity federation. Follow the instructions in [Creating a role for a third-party identity provider (federation)](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_for-idp.html) in the _IAM User Guide_.
    
- IAM users:
    
    - Create a role that your user can assume. Follow the instructions in [Creating a role for an IAM user](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_for-user.html) in the _IAM User Guide_.
        
    - (Not recommended) Attach a policy directly to a user or add a user to a user group. Follow the instructions in [Adding permissions to a user (console)](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_change-permissions.html#users_change_permissions-add-console) in the _IAM User Guide_.
        
    

Amazon ElastiCache creates and uses service-linked roles to provision resources and access other AWS resources and services on your behalf. For ElastiCache to create a service-linked role for you, use the AWS-managed policy named `AmazonElastiCacheFullAccess`. This role comes preprovisioned with permission that the service requires to create a service-linked role on your behalf.

You might decide not to use the default policy and instead to use a custom-managed policy. In this case, make sure that you have either permissions to call `iam:createServiceLinkedRole` or that you have created the ElastiCache service-linked role.

For more information, see the following:

- [Creating a New Policy](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_create.html) (IAM)
    
- [AWS-managed (predefined) policies for Amazon ElastiCache](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/IAM.IdentityBasedPolicies.html#IAM.IdentityBasedPolicies.PredefinedPolicies)
    
- [Using Service-Linked Roles for Amazon ElastiCache](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/using-service-linked-roles.html)

### 下载和配置AWS CLI

The AWS CLI is available at [http://aws.amazon.com/cli](http://aws.amazon.com/cli). It runs on Windows, MacOS and Linux. After you download the AWS CLI, follow these steps to install and configure it:

1. Go to the [AWS Command Line Interface User Guide](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-welcome.html).
    
2. Follow the instructions for [Installing the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html) and [Configuring the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html).

## 创建一个子网组(sub-net group)

在创建集群之前，您首先需要创建一个子网组。缓存子网组是一组子网，您可能希望为VPC中的缓存集群指定这些子网。在VPC中启动缓存集群时，您需要选择一个缓存子网组。然后ElastiCache使用该缓存子网组将IP地址分配给集群中的每个缓存节点。

当您创建新的子网组时，请注意可用IP地址的数量。如果该子网上有很少空闲IP地址，则可能会限制您可以添加到集群中的节点数量。为了解决此问题，您可以将一个或多个子网分配给一个子网组，以便在集群所在的可用区拥有足够数量的IP地址。之后，您可以向集群添加更多节点。

以下步骤展示了如何通过控制台、AWS CLI和ElastiCache API来创建名为mysubnetgroup的子网组。

#### Console方式

The following procedure shows how to create a subnet group (console).

###### To create a subnet group (Console)

1. Sign in to the AWS Management Console, and open the ElastiCache console at [https://console.aws.amazon.com/elasticache/](https://console.aws.amazon.com/elasticache/).
    
2. In the navigation list, choose **Subnet Groups**.
    
3. Choose **Create Subnet Group**.
    
4. In the **Create Subnet Group** wizard, do the following. When all the settings are as you want them, choose **Yes, Create**.
    
    1. In the **Name** box, type a name for your subnet group.
        
    2. In the **Description** box, type a description for your subnet group.
        
    3. In the **VPC ID** box, choose the Amazon VPC that you created.
        
    4. In the **Availability Zone** and **Subnet ID** lists, choose the Availability Zone or [Local Zone](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/Local_zones.html) and ID of your private subnet, and then choose **Add**.

![[Pasted image 20231017162341.png]]

5. In the confirmation message that appears, choose **Close**.
    
    Your new subnet group appears in the **Subnet Groups** list of the ElastiCache console. At the bottom of the window you can choose the subnet group to see details, such as all of the subnets associated with this group.


#### CLI方式

At a command prompt, use the command `create-cache-subnet-group` to create a subnet group.

For Linux, macOS, or Unix:

```bash
aws elasticache create-cache-subnet-group \
    --cache-subnet-group-name mysubnetgroup \
    --cache-subnet-group-description "Testing" \
    --subnet-ids subnet-53df9c3a
```

For Windows:

```bash
aws elasticache create-cache-subnet-group ^
    --cache-subnet-group-name mysubnetgroup ^
    --cache-subnet-group-description "Testing" ^
    --subnet-ids subnet-53df9c3a
```

This command should produce output similar to the following:

```json
{
    "CacheSubnetGroup": {
        "VpcId": "vpc-37c3cd17", 
        "CacheSubnetGroupDescription": "Testing", 
        "Subnets": [
            {
                "SubnetIdentifier": "subnet-53df9c3a", 
                "SubnetAvailabilityZone": {
                    "Name": "us-west-2a"
                }
            }
        ], 
        "CacheSubnetGroupName": "mysubnetgroup"
    }
}
```

For more information, see the AWS CLI topic create-cache-subnet-group.

#### ElastiCache API方式

sing the ElastiCache API, call `CreateCacheSubnetGroup` with the following parameters:

- `CacheSubnetGroupName= `mysubnetgroup`
    
- `CacheSubnetGroupDescription= =`Testing
    
- `SubnetIds.member.1= subnet-53df9c3a

```txt
https://elasticache.us-west-2.amazonaws.com/
    ?Action=CreateCacheSubnetGroup
    &CacheSubnetGroupDescription=Testing
    &CacheSubnetGroupName=mysubnetgroup
    &SignatureMethod=HmacSHA256
    &SignatureVersion=4
    &SubnetIds.member.1=subnet-53df9c3a 
    &Timestamp=20141201T220302Z
    &Version=2014-12-01
    &X-Amz-Algorithm=&AWS;4-HMAC-SHA256
    &X-Amz-Credential=<credential>
    &X-Amz-Date=20141201T220302Z
    &X-Amz-Expires=20141201T220302Z
    &X-Amz-Signature=<signature>
    &X-Amz-SignedHeaders=Host
```

## 创建一个集群

在为生产使用创建集群之前，您显然需要考虑如何配置集群以满足业务需求。这些问题在[“准备集群”]([Preparing a cluster - Amazon ElastiCache for Redis](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/Clusters.Prepare.html))部分中得到解决。对于此入门练习，您将创建一个禁用了集群模式的集群，并且可以接受适用的默认配置值。

您创建的集群将是实时运行的，而不是在沙盒中运行。直到删除它之前，您将承担标准ElastiCache实例使用费用。如果您完成本文所述的练习并在结束时删除自己的集群，则总费用将很小（通常少于一美元）。有关ElastiCache使用率更多信息，请参阅Amazon ElastiCache。[Redis and Memcached-Compatible Cache – Amazon ElastiCache – Amazon Web Services](https://aws.amazon.com/elasticache/) 

基于Amazon VPC服务，在虚拟私有云（VPC）中启动了您的集群。

### 创建一个Redis集群(Console)(Redis的Cluster mode关闭)

###### To create a Redis (cluster mode disabled) cluster using the ElastiCache console

1. Sign in to the AWS Management Console and open the Amazon ElastiCache console at [https://console.aws.amazon.com/elasticache/](https://console.aws.amazon.com/elasticache/).
    
2. From the list in the upper-right corner, choose the AWS Region that you want to launch this cluster in.
    
3. Choose **Get started** from the navigation pane.
    
4. Choose **Create VPC** and follow the steps outlined at [Creating a Virtual Private Cloud (VPC)](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/VPCs.CreatingVPC.html).
    
5. On the ElastiCache dashboard page, choose **Create cluster** and then choose **Create Redis cluster**.
    
6. Under **Cluster settings**, do the following:
    
    1. Choose **Configure and create a new cluster**.
        
    2. For **Cluster mode**, choose **Disabled**.
        
    3. For **Cluster info** enter a value for **Name**.
        
    4. (Optional) Enter a value for **Description**.
        
7. Under **Location**:

	1. For **AWS Cloud**, we recommend you accept the default settings for **Multi-AZ** and **Auto-failover**. For more information, see [Minimizing downtime in ElastiCache for Redis with Multi-AZ](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/AutoFailover.html).
	    
	2. Under **Cluster settings**
	    
	    1. For **Engine version**, choose an available version.
	        
	    2. For **Port**, use the default port, 6379. If you have a reason to use a different port, enter the port number.
	        
	    3. For **Parameter group**, choose a parameter group or create a new one. Parameter groups control the runtime parameters of your cluster. For more information on parameter groups, see [Redis-specific parameters](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/ParameterGroups.Redis.html) and [Creating a parameter group](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/ParameterGroups.Creating.html).
	        
	        ###### Note
	        
	        When you select a parameter group to set the engine configuration values, that parameter group is applied to all clusters in the global datastore. On the **Parameter Groups** page, the yes/no **Global** attribute indicates whether a parameter group is part of a global datastore.
	        
	    4. For **Node type**, choose the down arrow (  ![](https://docs.aws.amazon.com/images/AmazonElastiCache/latest/red-ug/images/ElastiCache-DnArrow.png)). In the **Change node type** dialog box, choose a value for **Instance family** for the node type that you want. Then choose the node type that you want to use for this cluster, and then choose **Save**.
	        
	        For more information, see [Choosing your node size](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/nodes-select-size.html#CacheNodes.SelectSize).
	        
	        If you choose an r6gd node type, data-tiering is automatically enabled. For more information, see [Data tiering](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/data-tiering.html).
	        
	    5. For **Number of replicas**, choose the number of read replicas you want. If you enabled Multi-AZ, the number must be between 1-5.
	        
	3. Under **Connectivity**
	    
	    1. For **Network type**, choose the IP version(s) this cluster will support.
	        
	    2. For **Subnet groups**, choose the subnet that you want to apply to this cluster. ElastiCache uses that subnet group to choose a subnet and IP addresses within that subnet to associate with your nodes. ElastiCache clusters require a dual-stack subnet with both IPv4 and IPv6 addresses assigned to them to operate in dual-stack mode and an IPv6-only subnet to operate as IPv6-only.
	        
	        When creating a new subnet group, enter the **VPC ID** to which it belongs.
	        
	        For more information, see:
	        
	        - [Choosing a network type](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/network-type.html).
	            
	        - [Create a subnet in your VPC](https://docs.aws.amazon.com/vpc/latest/userguide/working-with-vpcs.html#AddaSubnet).
	            
	        
	        If you are [Using local zones with ElastiCache](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/Local_zones.html) , you must create or choose a subnet that is in the local zone.
	        
	        For more information, see [Subnets and subnet groups](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/SubnetGroups.html).
	        
	4. For **Availability zone placements**, you have two options:
	    
	    - **No preference** – ElastiCache chooses the Availability Zone.
	        
	    - **Specify availability zones** – You specify the Availability Zone for each cluster.
	        
	        If you chose to specify the Availability Zones, for each cluster in each shard, choose the Availability Zone from the list.
	        
	    
	    For more information, see [Choosing regions and availability zones](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/RegionsAndAZs.html).
	    
	5. Choose **Next**
	    
	6. Under **Advanced Redis settings**
	    
	    1. For **Security**:
	        
	        1. To encrypt your data, you have the following options:
	            
	            - **Encryption at rest** – Enables encryption of data stored on disk. For more information, see [Encryption at Rest](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/at-rest-encryption.html).
	                
	                ###### Note
	                
	                You have the option to supply a different encryption key by choosing **Customer Managed AWS KMS key** and choosing the key. For more information, see [Using customer managed keys from AWS KMS](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/at-rest-encryption.html#using-customer-managed-keys-for-elasticache-security).
	                
	            - **Encryption in-transit** – Enables encryption of data on the wire. For more information, see [encryption in transit](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/in-transit-encryption.html). For Redis engine version 6.0 and above, if you enable Encryption in-transit you will be prompted to specify one of the following **Access Control** options:
	                
	                - **No Access Control** – This is the default setting. This indicates no restrictions on user access to the cluster.
	                    
	                - **User Group Access Control List** – Select a user group with a defined set of users that can access the cluster. For more information, see [Managing User Groups with the Console and CLI](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/Clusters.RBAC.html#User-Groups).
	                    
	                - **Redis AUTH Default User** – An authentication mechanism for Redis server. For more information, see [Redis AUTH](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/auth.html).
	                    
	                
	            - **Redis AUTH** – An authentication mechanism for Redis server. For more information, see [Redis AUTH](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/auth.html).
	                
	            
	            ###### Note
	            
	            For Redis versions between 3.2.6 onward, excluding version 3.2.10, Redis AUTH is the sole option.
	            
	        2. For **Security groups**, choose the security groups that you want for this cluster. A _security group_ acts as a firewall to control network access to your cluster. You can use the default security group for your VPC or create a new one.
	            
	            For more information on security groups, see [Security groups for your VPC](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_SecurityGroups.html) in the _Amazon VPC User Guide._
	            
	7. For regularly scheduled automatic backups, select **Enable automatic backups** and then enter the number of days that you want each automatic backup retained before it is automatically deleted. If you don't want regularly scheduled automatic backups, clear the **Enable automatic backups** check box. In either case, you always have the option to create manual backups.
	    
	    For more information on Redis backup and restore, see [Backup and restore for ElastiCache for Redis](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/backups.html) .
	    
	8. (Optional) Specify a maintenance window. The _maintenance window_ is the time, generally an hour in length, each week when ElastiCache schedules system maintenance for your cluster. You can allow ElastiCache to choose the day and time for your maintenance window (_No preference_), or you can choose the day, time, and duration yourself (_Specify maintenance window_). If you choose _Specify maintenance window_ from the lists, choose the _Start day_, _Start time_, and _Duration_ (in hours) for your maintenance window. All times are UCT times.
	    
	    For more information, see [Managing maintenance](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/maintenance-window.html).
	    
	9. (Optional) For **Logs**:
	    
	    - Under **Log format**, choose either **Text** or **JSON**.
	        
	    - Under **Destination Type**, choose either **CloudWatch Logs** or **Kinesis Firehose**.
	        
	    - Under **Log destination**, choose either **Create new** and enter either your CloudWatch Logs log group name or your Kinesis Data Firehose stream name, or choose **Select existing** and then choose either your CloudWatch Logs log group name or your Kinesis Data Firehose stream name,
	        
	    
	10. For **Tags**, to help you manage your clusters and other ElastiCache resources, you can assign your own metadata to each resource in the form of tags. For mor information, see [Tagging your ElastiCache resources](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/Tagging-Resources.html).
	    
	11. Choose **Next**.
	    
	12. Review all your entries and choices, then make any needed corrections. When you're ready, choose **Create**.

一旦您的集群状态可用，您可以授予 Amazon EC2 访问权限，连接到它并开始使用。有关更多信息，请参阅  [[#集群的访问认证]] 和  [[#连接集群的节点]]

#### CLI的方式

Unix Like：

```bash
aws elasticache create-cache-cluster \
--cache-cluster-id my-cluster \
--cache-node-type cache.r4.large \
--engine redis \
--num-cache-nodes 1 \
--snapshot-arns arn:aws:s3:::my_bucket/snapshot.rdb
```

windows:

```bash
aws elasticache create-cache-cluster ^
--cache-cluster-id my-cluster ^
--cache-node-type cache.r4.large ^
--engine redis ^
--num-cache-nodes 1 ^
--snapshot-arns arn:aws:s3:::my_bucket/snapshot.rdb
```

以上都是Redus Cluster mode关闭下的创建Redis集群，下面给出Cluster mode打开的:

- To use the console, see [Creating a Redis (cluster mode enabled) cluster (Console)](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/Clusters.Create.html#Clusters.Create.CON.RedisCluster).
    
- To use the AWS CLI, see [Creating a Redis (cluster mode enabled) cluster (AWS CLI)](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/Clusters.Create.html#Clusters.Create.CLI.RedisCluster).
    
- To use the ElastiCache API, see[Creating a cache cluster in Redis (cluster mode enabled) (ElastiCache API)](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/Clusters.Create.html#Clusters.Create.API.RedisCluster).

## 集群的访问认证

本节假设您已经熟悉如何启动和连接到Amazon EC2实例。更多信息，请参阅[Amazon EC2入门指南]([Tutorial: Get started with Amazon EC2 Linux instances - Amazon Elastic Compute Cloud](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EC2_GetStarted.html))。

所有的ElastiCache集群都设计为可以从Amazon EC2实例访问。最常见的情况是从同一Amazon Virtual Private Cloud（Amazon VPC）中的一个Amazon EC2实例访问ElastiCache集群，这也是本练习的情况。

默认情况下，对集群的网络访问仅限于用于创建该集群的帐户。在您可以从EC2实例连接到集群之前，必须授权该EC2实例访问该集群。所需步骤取决于您将集群启动到EC2-VPC还是EC2-Classic。

最常见的用例是当部署在EC2实例上的应用程序需要连接到同一VPC中的一个集群时。管理在同一VPC中的EC2实例和集群之间访问权限最简单方法如下：

- 为您的集群创建一个VPC安全组。该安全组可用于限制对集群实例的访问。例如，您可以为此安全组创建一个自定义规则，允许使用在创建集群时分配给它的端口进行TCP访问，并指定将用于访问集群的IP地址。Redis集群和复制组的默认端口是6379。

> 重要： Amazon ElastiCache安全组仅适用于不在Amazon Virtual Private Cloud环境（VPC）中运行的集群。如果您正在Amazon Virtual Private Cloud中运行，安全组在控制台导航窗格中不可用。
> 
> 如果您将ElastiCache节点运行在Amazon VPC中，您可以使用与ElastiCache安全组不同的Amazon VPC安全组来控制对集群的访问。有关在Amazon VPC中使用ElastiCache的更多信息，请参阅 [Amazon VPCs and ElastiCache security]([Amazon VPCs and ElastiCache security - Amazon ElastiCache for Redis](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/VPCs.html)).

- 为您的EC2实例（Web和应用服务器）创建一个VPC安全组。如果需要，此安全组可以通过VPC的路由表允许从互联网访问EC2实例。例如，您可以在此安全组上设置规则，以允许通过端口22进行TCP访问到EC2实例。
- 为您的集群创建自定义规则，允许来自为您的EC2实例创建的安全组的连接。这将允许安全组中的任何成员访问集群。

> 注意： 如果您计划使用[本地区域]([Using local zones with ElastiCache - Amazon ElastiCache for Redis](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/Local_zones.html))，请确保已启用它们。当您在该本地区域创建子网组时，您的VPC将扩展到该本地区域，并且您的VPC将把该子网视为任何其他可用区中的子网。所有相关的网关和路由表都将自动调整。

###### To create a rule in a VPC security group that allows connections from another security group

1. Sign in to the AWS Management Console and open the Amazon VPC console at [https://console.aws.amazon.com/vpc](https://console.aws.amazon.com/vpc).
    
2. In the navigation pane, choose **Security Groups**.
    
3. Select or create a security group that you will use for your Cluster instances. Under **Inbound Rules**, select **Edit Inbound Rules** and then select **Add Rule**. This security group will allow access to members of another security group.
    
4. From **Type** choose **Custom TCP Rule**.
    
    1. For **Port Range**, specify the port you used when you created your cluster.
        
        The default port for Redis clusters and replication groups is `6379`.
        
    2. In the **Source** box, start typing the ID of the security group. From the list select the security group you will use for your Amazon EC2 instances.
        
5. Choose **Save** when you finish.

![[Pasted image 20231017165404.png]]


Once you have enabled access, you are now ready to connect to the node, as discussed in the next section.

For information on accessing your ElastiCache cluster from a different Amazon VPC, a different AWS Region, or even your corporate network, see the following:

- [Access Patterns for Accessing an ElastiCache Cluster in an Amazon VPC](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/elasticache-vpc-accessing.html)
    
- [Accessing ElastiCache resources from outside AWS](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/accessing-elasticache.html#access-from-outside-aws)

## 连接集群的节点

### 找到你的节点端点

当您的集群处于可用状态并且已经授权访问权限时，您可以登录到 Amazon EC2 实例并连接到集群。为此，您首先需要确定端点。

#### 找到一个Redis集群的端点(Console)(Redis的Cluster mode关闭)

如果Redis（禁用集群模式）集群只有一个节点，则该节点的终端点用于读和写。如果集群有多个节点，则有三种类型的终端点；primary endpoint、reader endpoint和node endpoint。

primary endpoint是一个DNS名称，始终解析为集群中的主要节点。主要终端点不受您的集群更改（例如将读副本提升为主要角色）的影响。对于写入活动，我们建议您的应用程序连接到主要终端点。

reader endpoint将传入连接均匀分配给ElastiCache for Redis集群中所有读副本。其他因素，如应用程序创建连接时或应用程序如何使用连接（重新使用），将确定流量分布。reader endpoint会实时跟踪随着副本被添加或删除而发生的集群变化。您可以将ElastiCache for Redis集群的多个读副本放置在不同AWS可用区（AZ）中，以确保reader endpoint具有高可用性。

> 注意： reader endpoint不是负载均衡器。它是一个 DNS 记录，将以轮询方式解析为副本节点之一的 IP 地址。

对于读取活动，应用程序还可以连接到集群中的任何节点。与primary endpoint不同，node endpoint解析为特定的端点。如果您在集群中进行更改，例如添加或删除副本，则必须更新应用程序中的node endpoint。

###### To find a Redis (cluster mode disabled) cluster's endpoints

1. Sign in to the AWS Management Console and open the ElastiCache console at [https://console.aws.amazon.com/elasticache/](https://console.aws.amazon.com/elasticache/).
    
2. From the navigation pane, choose **Redis clusters**.
    
    The clusters screen will appear with a list of Redis (cluster mode disabled) and Redis (cluster mode enabled) clusters. Choose the cluster you created in the [Creating a Redis (cluster mode disabled) cluster (Console)](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/GettingStarted.CreateCluster.html#Clusters.Create.CON.Redis-gs) section.
    
3. To find the cluster's Primary and/or Reader endpoints, choose the cluster's name (not the radio button).

![[Pasted image 20231017171534.png]]

  Primary and Reader endpoints for a Redis (cluster mode disabled) cluster_

   If there is only one node in the cluster, there is no primary endpoint and you can continue at the next step.

4. If the Redis (cluster mode disabled) cluster has replica nodes, you can find the cluster's replica node endpoints by choosing the cluster's name and then choosing the **Nodes** tab.
    
    The nodes screen appears with each node in the cluster, primary and replicas, listed with its endpoint.

![[Pasted image 20231017171639.png]]

_Node endpoints for a Redis (cluster mode disabled) cluster_

5. To copy an endpoint to your clipboard:
    
    1. One endpoint at a time, find the endpoint you want to copy.
        
    2. Choose the copy icon directly in front of the endpoint.
        
    
    The endpoint is now copied to your clipboard. For information on using the endpoint to connect to a node, see [Connecting to nodes](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/nodes-connecting.html).

A Redis (cluster mode disabled) primary endpoint looks something like the following. There is a difference depending upon whether or not In-Transit encryption is enabled.

**In-transit encryption not enabled**

```txt
clusterName.xxxxxx`.`nodeId`.`regionAndAz.cache.amazonaws.com:`port` 			 redis-01.7abc2d.0001.usw2.cache.amazonaws.com:6379
```

**In-transit encryption enabled**

```txt
master.`clusterName`.`xxxxxx`.`regionAndAz`.cache.amazonaws.com:`port` master.ncit.ameaqx.use1.cache.amazonaws.com:6379
```

To further explore how to find your endpoints, see the relevant topics for the engine and cluster type you're running.

- [Finding connection endpoints](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/Endpoints.html)
    
- [Finding Endpoints for a Redis (Cluster Mode Enabled) Cluster (Console)](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/Endpoints.html#Endpoints.Find.RedisCluster)—You need the cluster's Configuration endpoint.
    
- [Finding Endpoints (AWS CLI)](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/Endpoints.html#Endpoints.Find.CLI)
    
- [Finding Endpoints (ElastiCache API)](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/Endpoints.html#Endpoints.Find.API)

### 连接到一个Redis集群或者副本组(Linux)

现在您已经拥有所需的endpoint，可以登录到一个 EC2 实例并连接到集群或复制组。在下面的示例中，您使用 redis-cli 工具来连接到一个集群。最新版本的 redis-cli 还支持 SSL/TLS 用于连接启用了加密/身份验证功能的集群。

以下示例使用运行 Amazon Linux 和 Amazon Linux 2 的 Amazon EC2 实例。有关在其他 Linux 发行版上安装和编译 redis-cli 的详细信息，请参阅特定操作系统的文档。

> 注意： 此过程仅涵盖使用 redis-cli 实用程序进行非计划使用的连接测试。有关支持的 Redis 客户端列表，请参阅 [Redis 文档]([Redis](https://redis.io/))。有关使用 AWS SDK 与 ElastiCache 的示例，请参阅[《ElastiCache 和 AWS SDK 入门》](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/ElastiCache-Getting-Started-Tutorials.html)。

### 下载和安装Redis-cli

1. Connect to your Amazon EC2 instance using the connection utility of your choice. For instructions on how to connect to an Amazon EC2 instance, see the [Amazon EC2 Getting Started Guide](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EC2_GetStarted.html).
    
2. Download and install redis-cli utility by running following commands:
    
    Amazon Linux 2
    
    `sudo amazon-linux-extras install epel -y sudo yum install gcc jemalloc-devel openssl-devel tcl tcl-devel -y sudo wget http://download.redis.io/redis-stable.tar.gz sudo tar xvzf redis-stable.tar.gz cd redis-stable sudo make BUILD_TLS=yes`
    
    Amazon Linux
    
    `sudo yum install gcc jemalloc-devel openssl-devel tcl tcl-devel clang wget sudo wget http://download.redis.io/redis-stable.tar.gz sudo tar xvzf redis-stable.tar.gz cd redis-stable sudo CC=clang make BUILD_TLS=yes`
    
    ###### Note
    
    If the cluster you are connecting to isn't encrypted, you don't need the `Build_TLS=yes` option.

### 连接到一个集群模式关闭下的未加密集群

1. Run the following command to connect to the cluster and replace `primary-endpoint` and `port number` with the endpoint of your cluster and your port number. (The default port for Redis is 6379.)
    
    `` src/redis-cli -h `primary-endpoint` -p `port number` ``
    
    The result in a Redis command prompt looks similar to the following:
    
    `` `primary-endpoint`:`port number` ``
    
2. You can now run Redis commands.
    
    `set x Hello OK  get x "Hello"`

### 连接到一个集群模式打开下的未加密集群

1. Run the following command to connect to the cluster and replace `configuration-endpoint` and `port number` with the endpoint of your cluster and your port number. (The default port for Redis is 6379.)
    
    `` src/redis-cli -h `configuration-endpoint` -c -p `port number` ``
    
    ###### Note
    
    In the preceding command, option -c enables cluster mode following [-ASK and -MOVED redirections](https://redis.io/topics/cluster-spec).
    
    The result in a Redis command prompt looks similar to the following:
    
    `` `configuration-endpoint`:`port number` ``
    
2. You can now run Redis commands. Note that redirection occurs because you enabled it using the -c option. If redirection isn't enabled, the command returns the MOVED error. For more information on the MOVED error, see [Redis cluster specification](https://redis.io/topics/cluster-spec).
    
    `set x Hi -> Redirected to slot [16287] located at 172.31.28.122:6379 OK set y Hello OK get y "Hello" set z Bye -> Redirected to slot [8157] located at 172.31.9.201:6379 OK get z "Bye" get x -> Redirected to slot [16287] located at 172.31.28.122:6379 "Hi"`

### 连接到一个加密和认证开启的集群

By default, redis-cli uses an unencrypted TCP connection when connecting to Redis. The option `BUILD_TLS=yes` enables SSL/TLS at the time of redis-cli compilation as shown in the preceding [Download and install redis-cli](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/GettingStarted.ConnectToCacheNode.html#Download-and-install-redis-cli) section. Enabling AUTH is optional. However, you must enable encryption in-transit in order to enable AUTH. For more details on ElastiCache encryption and authentication, see [ElastiCache in-transit encryption (TLS)](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/in-transit-encryption.html).

###### Note

You can use the option `--tls` with redis-cli to connect to both cluster mode enabled and disabled encrypted clusters. If a cluster has an AUTH token set, then you can use the option `-a` to provide an AUTH password.

In the following examples, be sure to replace `cluster-endpoint` and `port number` with the endpoint of your cluster and your port number. (The default port for Redis is 6379.)

**Connect to cluster mode disabled encrypted clusters**

The following example connects to an encryption and authentication enabled cluster:

`` src/redis-cli -h `cluster-endpoint` --tls -a `your-password` -p `port number` ``

The following example connects to a cluster that has only encryption enabled:

`` src/redis-cli -h `cluster-endpoint` --tls -p `port number` ``

**Connect to cluster mode enabled encrypted clusters**

The following example connects to an encryption and authentication enabled cluster:

`` src/redis-cli -c -h `cluster-endpoint` --tls -a `your-password` -p `port number` ``

The following example connects to a cluster that has only encryption enabled:

`` src/redis-cli -c -h `cluster-endpoint` --tls -p `port number` ``

After you connect to the cluster, you can run the Redis commands as shown in the preceding examples for unencrypted clusters.

### Redis-cli的其他选择

If the cluster isn't cluster mode enabled and you need to make a connection to the cluster for a short test but without going through the redis-cli compilation, you can use telnet or openssl. In the following example commands, be sure to replace `cluster-endpoint` and `port number` with the endpoint of your cluster and your port number. (The default port for Redis is 6379.)

The following example connects to an encryption and/or authentication enabled cluster mode disabled cluster:

`` openssl s_client -connect `cluster-endpoint`:`port number` ``

If the cluster has a password set, first connect to the cluster. After connecting, authenticate the cluster using the following command, then press the `Enter` key. In the following example, replace `your-password` with the password for your cluster.

`` Auth `your-password` ``

The following example connects to a cluster mode disabled cluster that doesn't have encryption or authentication enabled:

`` telnet `cluster-endpoint` `port number` ``

### 连接到一个Redis集群或者副本组(Windows)

In order to connect to the Redis Cluster from an EC2 Windows instance using the Redis CLI, you must download the _redis-cli_ package and use _redis-cli.exe_ to connect to the Redis Cluster from an EC2 Windows instance.

In the following example, you use the _redis-cli_ utility to connect to a cluster that is not encryption enabled and running Redis. For more information about Redis and available Redis commands, see [Redis commands](http://redis.io/commands) on the Redis website.

###### To connect to a Redis cluster that is not encryption-enabled using _redis-cli_

1. Connect to your Amazon EC2 instance using the connection utility of your choice. For instructions on how to connect to an Amazon EC2 instance, see the [Amazon EC2 Getting Started Guide](https://docs.aws.amazon.com/AWSEC2/latest/GettingStartedGuide/).
    
2. Copy and paste the link [https://github.com/microsoftarchive/redis/releases/download/win-3.0.504/Redis-x64-3.0.504.zip](https://github.com/microsoftarchive/redis/releases/download/win-3.0.504/Redis-x64-3.0.504.zip) in an Internet browser to download the zip file for the Redis client from the available release at GitHub [https://github.com/microsoftarchive/redis/releases/tag/win-3.0.504](https://github.com/microsoftarchive/redis/releases/tag/win-3.0.504)
    
    Extract the zip file to you desired folder/path.
    
    Open the Command Prompt and change to the Redis directory and run the command ``c:\Redis>redis-cli -h `Redis_Cluster_Endpoint` -p 6379``.
    
    For example:
    
    `c:\Redis>redis-cli -h cmd.xxxxxxx.ng.0001.usw2.cache.amazonaws.com -p 6379`
    
3. Run Redis commands.
    
    You are now connected to the cluster and can run Redis commands like the following.
    
    `` `set a "hello"`          // Set key "a" with a string value and no expiration OK `get a`                  // Get value for key "a" "hello" `get b`                  // Get value for key "b" results in miss (nil)				 `set b "Good-bye" EX 5`  // Set key "b" with a string value and a 5 second expiration "Good-bye" `get b`                  // Get value for key "b" "Good-bye"                        // wait >= 5 seconds `get b` (nil)                  // key has expired, nothing returned `quit`                   // Exit from redis-cli ``
## 删除一个集群

只要集群处于可用状态，无论您是否正在使用它，都会向您收取费用。要停止产生费用，请删除该集群。

> 警告: 当您删除一个ElastiCache for Redis集群时，您的手动快照将被保留。您还可以在删除集群之前创建最终快照。自动缓存快照不会被保留。有关更多信息，请参阅[ElastiCache for Redis的备份和恢复文档]([Backup and restore for ElastiCache for Redis - Amazon ElastiCache for Redis](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/backups.html))。

#### 使用AWS管理Console删除

The following procedure deletes a single cluster from your deployment. To delete multiple clusters, repeat the procedure for each cluster that you want to delete. You do not need to wait for one cluster to finish deleting before starting the procedure to delete another cluster.

###### To delete a cluster

1. Sign in to the AWS Management Console and open the Amazon ElastiCache console at [https://console.aws.amazon.com/elasticache/](https://console.aws.amazon.com/elasticache/).
    
2. In the ElastiCache console dashboard, choose Redis.
    
    A list of all clusters running Redis appears.
    
3. To choose the cluster to delete, choose the cluster's name from the list of clusters. In this case, the name of the cluster you created at [Step 2: Create a cluster](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/GettingStarted.CreateCluster.html).
    
    ###### Important
    
    You can only delete one cluster at a time from the ElastiCache console. Choosing multiple clusters disables the delete operation.
    
4. For **Actions**, choose **Delete**.
    
5. In the **Delete Cluster** confirmation screen, choose **Delete** to delete the cluster, or choose **Cancel** to keep the cluster.
    
    If you chose **Delete**, the status of the cluster changes to _deleting_.
    

As soon as your cluster is no longer listed in the list of clusters, you stop incurring charges for it.

#### 使用AWS CLI删除

The following code deletes the cache cluster `my-cluster`. In this case, substitute `my-cluster` with the name of the cluster you created at [Step 2: Create a cluster](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/GettingStarted.CreateCluster.html).

`` aws elasticache delete-cache-cluster --cache-cluster-id `my-cluster` ``

The `delete-cache-cluster` CLI action only deletes one cache cluster. To delete multiple cache clusters, call `delete-cache-cluster` for each cache cluster that you want to delete. You do not need to wait for one cache cluster to finish deleting before deleting another.

For Linux, macOS, or Unix:

`` aws elasticache delete-cache-cluster \     --cache-cluster-id `my-cluster` \     --region `us-east-2` ``

For Windows:

`` aws elasticache delete-cache-cluster ^     --cache-cluster-id `my-cluster` ^     --region `us-east-2` ``

For more information, see the AWS CLI for ElastiCache topic [`delete-cache-cluster`](https://docs.aws.amazon.com/cli/latest/reference/elasticache/delete-cache-cluster.html).

## 教程和视频


The following tutorials address tasks of interest to the Amazon ElastiCache user.

- [ElastiCache Videos](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/Tutorials.html#tutorial-videos)
    
- [Tutorial: Configuring a Lambda Function to Access Amazon ElastiCache in an Amazon VPC](https://docs.aws.amazon.com/lambda/latest/dg/vpc-ec.html)
    

## ElastiCache Videos

Following, you can find videos to help you learn basic and advanced Amazon ElastiCache concepts. For information about AWS Training, see [AWS Training & Certification](https://aws.amazon.com/training/).

###### Topics

- [Introductory Videos](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/Tutorials.html#WhatIs.Videos.Beginning)
- [Advanced Videos](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/Tutorials.html#WhatIs.Videos.Advanced)

### Introductory Videos

The following videos introduce you to Amazon ElastiCache.

###### Topics

- [AWS re:Invent 2020: What’s new in Amazon ElastiCache](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/Tutorials.html#WhatIs.Videos.Beginning.2020)
- [AWS re:Invent 2019: What’s new in Amazon ElastiCache](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/Tutorials.html#WhatIs.Videos.Beginning.2019)
- [AWS re:Invent 2017: What’s new in Amazon ElastiCache](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/Tutorials.html#WhatIs.Videos.Beginning.2017)
- [DAT204—Building Scalable Applications on AWS NoSQL Services (re:Invent 2015)](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/Tutorials.html#WhatIs.Videos.Beginning.2015.DAT204)
- [DAT207—Accelerating Application Performance with Amazon ElastiCache (AWS re:Invent 2013)](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/Tutorials.html#WhatIs.Videos.Beginning.2013.DAT207)

#### AWS re:Invent 2020: What’s new in Amazon ElastiCache

https://youtu.be/O9mqbIYJXWE

#### AWS re:Invent 2019: What’s new in Amazon ElastiCache

https://youtu.be/SaGW_Bln3qA

#### AWS re:Invent 2017: What’s new in Amazon ElastiCache

https://youtu.be/wkGn1TzCgnk

#### DAT204—Building Scalable Applications on AWS NoSQL Services (re:Invent 2015)

In this session, we discuss the benefits of NoSQL databases and take a tour of the main NoSQL services offered by AWS—Amazon DynamoDB and Amazon ElastiCache. Then, we hear from two leading customers, Expedia and Mapbox, about their use cases and architectural challenges, and how they addressed them using AWS NoSQL services, including design patterns and best practices. You should come out of this session having a better understanding of NoSQL and its powerful capabilities, ready to tackle your database challenges with confidence.

https://youtu.be/ie4dWGT76LM

#### DAT207—Accelerating Application Performance with Amazon ElastiCache (AWS re:Invent 2013)

In this video, learn how you can use Amazon ElastiCache to easily deploy an in-memory caching system to speed up your application performance. We show you how to use Amazon ElastiCache to improve your application latency and reduce the load on your database servers. We'll also show you how to build a caching layer that is easy to manage and scale as your application grows. During this session, we go over various scenarios and use cases that can benefit by enabling caching, and discuss the features provided by Amazon ElastiCache.

https://youtu.be/odMmdPBV8hM

### Advanced Videos

The following videos cover more advanced Amazon ElastiCache topics.

###### Topics

- [Design for success with Amazon ElastiCache best practices (re:Invent 2020)](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/Tutorials.html#WhatIs.Videos.Advanced.2020.DAT305)
- [Supercharge your real-time apps with Amazon ElastiCache (re:Invent 2019)](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/Tutorials.html#WhatIs.Videos.Advanced.2019.DAT305)
- [Best practices: migrating Redis clusters from Amazon EC2 to ElastiCache (re:Invent 2019)](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/Tutorials.html#WhatIs.Videos.Advanced.2019.DAT358)
- [Scaling a Fantasy Sports Platform with Amazon ElastiCache & Amazon Aurora STP11 (re:Invent 2018)](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/Tutorials.html#WhatIs.Videos.Advanced.2019)
- [Reliable & Scalable Redis in the Cloud with Amazon ElastiCache (re:Invent 2018)](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/Tutorials.html#WhatIs.Videos.AdvancedDD.2018)
- [ElastiCache Deep Dive: Design Patterns for In-Memory Data Stores (re:Invent 2018)](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/Tutorials.html#WhatIs.Videos.AdvancedDD.2019)
- [DAT305—Amazon ElastiCache Deep Dive (re:Invent 2017)](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/Tutorials.html#WhatIs.Videos.Advanced.2017.DAT305)
- [DAT306—Amazon ElastiCache Deep Dive (re:Invent 2016)](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/Tutorials.html#WhatIs.Videos.Advanced.2016.DAT306)
- [DAT317—How IFTTT uses ElastiCache for Redis to Predict Events (re:Invent 2016)](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/Tutorials.html#WhatIs.Videos.Advanced.2016.DAT317)
- [DAT407—Amazon ElastiCache Deep Dive (re:Invent 2015)](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/Tutorials.html#WhatIs.Videos.Advanced.2015.DAT407)
- [SDD402—Amazon ElastiCache Deep Dive (re:Invent 2014)](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/Tutorials.html#WhatIs.Videos.Advanced.2014.SDD402)
- [DAT307—Deep Dive into Amazon ElastiCache Architecture and Design Patterns (re:Invent 2013)](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/Tutorials.html#WhatIs.Videos.Advanced.2013.DAT307)

#### Design for success with Amazon ElastiCache best practices (re:Invent 2020)

With the explosive growth of business-critical, real-time applications built on Redis, availability, scalability, and security have become top considerations. Learn best practices for setting up Amazon ElastiCache for success with online scaling, high availability across Multi-AZ deployments, and security configurations.

https://youtu.be/_4SkEy6r-C4

#### Supercharge your real-time apps with Amazon ElastiCache (re:Invent 2019)

With the rapid growth in cloud adoption and the new scenarios that it empowers, applications need microsecond latency and high throughput to support millions of requests per second. Developers have traditionally relied on specialized hardware and workarounds, such as disk-based databases combined with data reduction techniques, to manage data for real-time applications. These approaches can be expensive and not scalable. Learn how you can boost the performance of real-time applications by using the fully managed, in-memory Amazon ElastiCache for extreme performance, high scalability, availability, and security.

https://youtu.be/v0GfpL5jfns

#### Best practices: migrating Redis clusters from Amazon EC2 to ElastiCache (re:Invent 2019)

Managing Redis clusters on your own can be hard. You have to provision hardware, patch software, back up data, and monitor workloads constantly. With the newly released Online Migration feature for Amazon ElastiCache, you can now easily move your data from self-hosted Redis on Amazon EC2 to fully managed Amazon ElastiCache, with cluster mode disabled. In this session, you learn about the new Online Migration tool, see a demo, and, more importantly, learn hands-on best practices for a smooth migration to Amazon ElastiCache.

https://youtu.be/Rpni5uPe0uI

#### Scaling a Fantasy Sports Platform with Amazon ElastiCache & Amazon Aurora STP11 (re:Invent 2018)

Dream11 is India’s leading sports-tech startup. It has a growing base of 40 million+ users playing multiple sports, including fantasy cricket, football, and basketball, and it currently serves one million concurrent users, who produce three million requests per minute under a 50-millisecond response time. In this talk, Dream11 CTO Amit Sharma explains how the company uses Amazon Aurora and Amazon ElastiCache to handle flash traffic, which can triple within a 30-second response window. Sharma also talks about scaling transactions without locking, and he shares the steps for handling flash traffic—thereby serving five million daily active users. Complete Title: AWS re:Invent 2018: Scaling a Fantasy Sports Platform with Amazon ElastiCache & Amazon Aurora (STP11)

https://youtu.be/hIPOLeEjVQY

#### Reliable & Scalable Redis in the Cloud with Amazon ElastiCache (re:Invent 2018)

This session covers the features and enhancements in our Redis-compatible service, Amazon ElastiCache for Redis. We cover key features, such as Redis 5, scalability and performance improvements, security and compliance, and much more. We also discuss upcoming features and customer case studies.

https://youtu.be/pgXEnAcTNPI

#### ElastiCache Deep Dive: Design Patterns for In-Memory Data Stores (re:Invent 2018)

In this session, we provide a behind the scenes peek to learn about the design and architecture of Amazon ElastiCache. See common design patterns with our Redis and Memcached offerings and how customers use them for in-memory data processing to reduce latency and improve application throughput. We review ElastiCache best practices, design patterns, and anti-patterns.

https://youtu.be/QxcB53mL_oA

#### DAT305—Amazon ElastiCache Deep Dive (re:Invent 2017)

Look behind the scenes to learn about Amazon ElastiCache's design and architecture. See common design patterns with our Memcached and Redis offerings and how customers have used them for in-memory operations to reduce latency and improve application throughput. During this video, we review ElastiCache best practices, design patterns, and anti-patterns.

The video introduces the following:

- ElastiCache for Redis online resharding
    
- ElastiCache security and encryption
    
- ElastiCache for Redis version 3.2.10
    
https://youtu.be/_YYBdsuUq2M

#### DAT306—Amazon ElastiCache Deep Dive (re:Invent 2016)

Look behind the scenes to learn about Amazon ElastiCache's design and architecture. See common design patterns with our Memcached and Redis offerings and how customers have used them for in-memory operations to reduce latency and improve application throughput. During this session, we review ElastiCache best practices, design patterns, and anti-patterns.

https://youtu.be/e9sN15a7utI

#### DAT317—How IFTTT uses ElastiCache for Redis to Predict Events (re:Invent 2016)

IFTTT is a free service that empowers people to do more with the services they love, from automating simple tasks to transforming how someone interacts with and controls their home. IFTTT uses ElastiCache for Redis to store transaction run history and schedule predictions as well as indexes for log documents on Amazon S3. View this session to learn how the scripting power of Lua and the data types of Redis allowed people to accomplish something they wouldn't have been able to elsewhere.

https://youtu.be/eQbsXN0kcc0

#### DAT407—Amazon ElastiCache Deep Dive (re:Invent 2015)

Peek behind the scenes to learn about Amazon ElastiCache's design and architecture. See common design patterns of our Memcached and Redis offerings and how customers have used them for in-memory operations and achieved improved latency and throughput for applications. During this session, we review best practices, design patterns, and anti-patterns related to Amazon ElastiCache.

https://youtu.be/4VfIINg9DYI

#### SDD402—Amazon ElastiCache Deep Dive (re:Invent 2014)

In this video, we examine common caching use cases, the Memcached and Redis engines, patterns that help you determine which engine is better for your needs, consistent hashing, and more as means to building fast, scalable applications. Frank Wiebe, Principal Scientist at Adobe, details how Adobe uses Amazon ElastiCache to improve customer experience and scale their business.

https://youtu.be/cEkHBqhQnog

#### DAT307—Deep Dive into Amazon ElastiCache Architecture and Design Patterns (re:Invent 2013)

In this video, we examine caching, caching strategies, scaling out, monitoring. We also compare the Memcached and Redis engines. During this session, also we review best practices and design patterns related to Amazon ElastiCache.

https://youtu.be/me0Tw13O1H4
