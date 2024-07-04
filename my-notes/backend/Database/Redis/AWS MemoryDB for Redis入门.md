
[Getting started with MemoryDB - Amazon MemoryDB for Redis](https://docs.aws.amazon.com/memorydb/latest/devguide/getting-started.html)

## 创建

同 [[AWS ElastiCache for redis入门#创建]]


## 创建一个集群

基本与ElastiCache for Redis一样，也是基于VPC的。

可以用AWS管理console，AWS CLI来创建

### 创建一个集群(Console)

###### To create a cluster using the MemoryDB console

1. Sign in to the AWS Management Console and open the MemoryDB for Redis console at [https://console.aws.amazon.com/memorydb/](https://console.aws.amazon.com/memorydb/).
    
2. Choose **Clusters** In the left navigation pane and then choose **Create cluster**.
    
3. Complete the **Cluster info** section.
    
    1. In **Name**, enter a name for your cluster.
        
        Cluster naming constraints are as follows:
        
        - Must contain 1–40 alphanumeric characters or hyphens.
            
        - Must begin with a letter.
            
        - Can't contain two consecutive hyphens.
            
        - Can't end with a hyphen.
            
        
    2. In the **Description** box, enter a description for this cluster.
        
4. Complete the **Subnet groups** section:
    
    1. For **Subnet groups**, create a new subnet group or choose an existing one from the available list that you want to apply to this cluster. If you are creating a new one:
        
        - Enter a **Name**
            
        - Enter a **Description**
            
        - If you enabled Multi-AZ, the subnet group must contain at least two subnets that reside in different availability zones. For more information, see [Subnets and subnet groups](https://docs.aws.amazon.com/memorydb/latest/devguide/subnetgroups.html).
            
        - If you are creating a new subnet group and do not have an existing VPC, you will be asked to create a VPC. For more information, see [What is Amazon VPC?](https://docs.aws.amazon.com/vpc/latest/userguide/what-is-amazon-vpc.html) in the _Amazon VPC User Guide._
            
        
5. Complete the **Cluster settings** section:
    
    1. For **Redis version compatibility**, accept the default `6.2`.
        
    2. For **Port**, accept the default Redis port of 6379 or, if you have a reason to use a different port, enter the port number..
        
    3. For **Parameter group**, accept the `default.memorydb-redis6` parameter group.
        
        Parameter groups control the runtime parameters of your cluster. For more information on parameter groups, see [Redis specific parameters](https://docs.aws.amazon.com/memorydb/latest/devguide/parametergroups.redis.html).
        
    4. For **Node type**, choose a value for the node type (along with its associated memory size) that you want.
        
        If you choose a node type from the r6gd family, you will automatically enable data-tiering, which splits data storage between memory and SSD. For more information, see [Data tiering](https://docs.aws.amazon.com/memorydb/latest/devguide/data-tiering.html).
        
    5. For **Number of shards**, choose the number of shards that you want for this cluster. For higher availability of your clusters, we recommend that you add at least 2 shards.
        
        You can change the number of shards in your cluster dynamically. For more information, see [Scaling MemoryDB clusters](https://docs.aws.amazon.com/memorydb/latest/devguide/scaling-cluster.html).
        
    6. For **Replicas per shard**, choose the number of read replica nodes that you want in each shard.
        
        The following restrictions exist:
        
        - If you have Multi-AZ enabled, make sure that you have at least one replica per shard.
            
        - The number of replicas is the same for each shard when creating the cluster using the console.
            
        
    7. Choose **Next**
        
    8. Complete the **Advanced settings** section:
        
        1. For **Security groups**, choose the security groups that you want for this cluster. A _security group_ acts as a firewall to control network access to your cluster. You can use the default security group for your VPC or create a new one.
            
            For more information on security groups, see [Security groups for your VPC](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_SecurityGroups.html) in the _Amazon VPC User Guide._
            
        2. To encrypt your data, you have the following options:
            
            - **Encryption at rest** – Enables encryption of data stored on disk. For more information, see [Encryption at Rest](https://docs.aws.amazon.com/memorydb/latest/devguide/at-rest-encryption.html).
                
                ###### Note
                
                You have the option to supply an encryption key other than default by choosing **Customer Managed AWS-owned KMS key** and choosing the key.
                
            - **Encryption in-transit** – Enables encryption of data on the wire. If you select no encryption, then an open Access control list called “open access” will be created with a default user. For more information, see [Authenticating users with Access Control Lists (ACLs)](https://docs.aws.amazon.com/memorydb/latest/devguide/clusters.acls.html).
                
            
        3. For **Snapshot**, optionally specify a snapshot retention period and a snapshot window. By default, **Enable automatic snapshots** is pre-selected.
            
        4. For **Maintenance window** optionally specify a maintenance window. The _maintenance window_ is the time, generally an hour in length, each week when MemoryDB schedules system maintenance for your cluster. You can allow MemoryDB to choose the day and time for your maintenance window (_No preference_), or you can choose the day, time, and duration yourself (_Specify maintenance window_). If you choose _Specify maintenance window_ from the lists, choose the _Start day_, _Start time_, and _Duration_ (in hours) for your maintenance window. All times are UCT times.
            
            For more information, see [Managing maintenance](https://docs.aws.amazon.com/memorydb/latest/devguide/maintenance-window.html).
            
        5. For **Notifications**, choose an existing Amazon Simple Notification Service (Amazon SNS) topic, or choose Manual ARN input and enter the topic's Amazon Resource Name (ARN). Amazon SNS allows you to push notifications to Internet-connected smart devices. The default is to disable notifications. For more information, see [https://aws.amazon.com/sns/](https://aws.amazon.com/sns/).
            
        6. For **Tags**, you can optionally apply tags to search and filter your clusters or track your AWS costs.
            
    9. Review all your entries and choices, then make any needed corrections. When you're ready, choose **Create cluster** to launch your cluster, or **Cancel** to cancel the operation.
        
    
    As soon as your cluster's status is _available_, you can grant EC2 access to it, connect to it, and begin using it. For more information, see [Step 2: Authorize access to the cluster](https://docs.aws.amazon.com/memorydb/latest/devguide/getting-started.authorizeaccess.html)
    
    ###### Important
    
    As soon as your cluster becomes available, you're billed for each hour or partial hour that the cluster is active, even if you're not actively using it. To stop incurring charges for this cluster, you must delete it. See [Step 4: Deleting a cluster](https://docs.aws.amazon.com/memorydb/latest/devguide/clusters.delete.html).

### 创建一个集群(CLI)

To create a cluster using the AWS CLI, see [`create-cluster`](https://docs.aws.amazon.com/cli/latest/reference/memorydb/create-cluster.html). The following is an example:

For Linux, macOS, or Unix:

```bash
aws memorydb create-cluster \
    --cluster-name my-cluster \
    --node-type db.r6g.large \
    --acl-name my-acl \
    --subnet-group my-sg

```

Windows

```bash
aws memorydb create-cluster ^
   --cluster-name my-cluster ^
   --node-type db.r6g.large ^
   --acl-name my-acl ^
   --subnet-group my-sg
```

You should get the following JSON response:

```json
{
    "Cluster": {
        "Name": "my-cluster",
        "Status": "creating",
        "NumberOfShards": 1,
        "AvailabilityMode": "MultiAZ",
        "ClusterEndpoint": {
            "Port": 6379
        },
        "NodeType": "db.r6g.large",
        "EngineVersion": "6.2",
        "EnginePatchVersion": "6.2.6",
        "ParameterGroupName": "default.memorydb-redis6",
        "ParameterGroupStatus": "in-sync",
        "SubnetGroupName": "my-sg",
        "TLSEnabled": true,
        "ARN": "arn:aws:memorydb:us-east-1:xxxxxxxxxxxxxx:cluster/my-cluster",
        "SnapshotRetentionLimit": 0,
        "MaintenanceWindow": "wed:03:00-wed:04:00",
        "SnapshotWindow": "04:30-05:30",
        "ACLName": "my-acl",
        "DataTiering": "false",
        "AutoMinorVersionUpgrade": true
    }
}
```

You can begin using the cluster once its status changes to `available`.

###### Important

As soon as your cluster becomes available, you're billed for each hour or partial hour that the cluster is active, even if you're not actively using it. To stop incurring charges for this cluster, you must delete it. See [Step 4: Deleting a cluster](https://docs.aws.amazon.com/memorydb/latest/devguide/clusters.delete.html).
## 认证对集群的访问

本节假设您已经熟悉如何启动和连接到Amazon EC2实例。

MemoryDB集群设计用于从Amazon EC2实例访问。它们也可以通过在Amazon Elastic Container Service或AWS Lambda中运行的容器化或serverless应用程序进行访问。最常见的情况是从同一Amazon Virtual Private Cloud（Amazon VPC）中的Amazon EC2实例访问MemoryDB集群，这将是本练习的情况。

在您可以从EC2实例连接到集群之前，必须授权该EC2实例访问该集群。

最常见的用例是当部署在EC2实例上的应用程序需要连接到同一VPC中的一个集群时。管理EC2实例和同一VPC中集群之间访问权限最简单的方法是执行以下操作：

1. Create a VPC security group for your cluster. This security group can be used to restrict access to the clusters. For example, you can create a custom rule for this security group that allows TCP access using the port you assigned to the cluster when you created it and an IP address you will use to access the cluster.
    
    The default port for MemoryDB clusters is `6379`.
    
2. Create a VPC security group for your EC2 instances (web and application servers). This security group can, if needed, allow access to the EC2 instance from the Internet via the VPC's routing table. For example, you can set rules on this security group to allow TCP access to the EC2 instance over port 22.
    
3. Create custom rules in the security group for your cluster that allow connections from the security group you created for your EC2 instances. This would allow any member of the security group to access the clusters.
    

###### To create a rule in a VPC security group that allows connections from another security group

1. Sign in to the AWS Management Console and open the Amazon VPC console at [https://console.aws.amazon.com/vpc](https://console.aws.amazon.com/vpc).
    
2. In the left navigation pane, choose **Security Groups**.
    
3. Select or create a security group that you will use for your clusters. Under **Inbound Rules**, select **Edit Inbound Rules** and then select **Add Rule**. This security group will allow access to members of another security group.
    
4. From **Type** choose **Custom TCP Rule**.
    
    1. For **Port Range**, specify the port you used when you created your cluster.
        
        The default port for MemoryDB clusters is `6379`.
        
    2. In the **Source** box, start typing the ID of the security group. From the list select the security group you will use for your Amazon EC2 instances.
        
5. Choose **Save** when you finish.
    

Once you have enabled access, you are now ready to connect to the cluster, as discussed in the next section.

For information on accessing your MemoryDB cluster from a different Amazon VPC, a different AWS Region, or even your corporate network, see the following:

- [Access Patterns for Accessing a MemoryDB Cluster in an Amazon VPC](https://docs.aws.amazon.com/memorydb/latest/devguide/memorydb-vpc-accessing.html)
    
- [Accessing MemoryDB resources from outside AWS](https://docs.aws.amazon.com/memorydb/latest/devguide/accessing-memorydb.html#access-from-outside-aws)

## 连接到一个集群

### 找到你的集群endpoint

当您的集群处于可用状态并且已经授权访问权限时，您可以登录到 Amazon EC2 实例并连接到集群。为此，您首先需要确定端点。

要进一步了解如何查找您的端点，请参阅以下内容：

- [Finding the Endpoint for a MemoryDB Cluster (AWS Management Console)](https://docs.aws.amazon.com/memorydb/latest/devguide/endpoints.html#endpoints.find.console)
    
- [Finding the Endpoint for a MemoryDB Cluster (AWS CLI)](https://docs.aws.amazon.com/memorydb/latest/devguide/endpoints.html#endpoints.find.cli)
    
- [Finding the Endpoint for a MemoryDB Cluster (MemoryDB API)](https://docs.aws.amazon.com/memorydb/latest/devguide/endpoints.html#endpoints.find.api)

### 连接到一个MemoryDB集群(Linux)

现在您已经拥有所需的endpoint，可以登录到一个 EC2 实例并连接到集群。在下面的示例中，您使用 cli 工具来连接到一个使用 Ubuntu 22 的集群。最新版本的 cli 还支持 SSL/TLS 来连接启用了加密/身份验证功能的集群。

#### 使用redis-cli连接到MemoryDB的节点

To access data from MemoryDB nodes, you use clients that work with Secure Socket Layer (SSL). You can also use redis-cli with TLS/SSL on Amazon Linux and Amazon Linux 2.

###### To use redis-cli to connect to a MemoryDB cluster on Amazon Linux 2 or Amazon Linux

1. Download and compile the redis-cli utility. This utility is included in the Redis software distribution.
    
2. At the command prompt of your EC2 instance, type the following commands:
    
    Amazon Linux 2
    
    `sudo yum -y install openssl-devel gcc wget http://download.redis.io/redis-stable.tar.gz tar xvzf redis-stable.tar.gz cd redis-stable make distclean make redis-cli BUILD_TLS=yes sudo install -m 755 src/redis-cli /usr/local/bin/`
    
    Amazon Linux
    
    `sudo yum install gcc jemalloc-devel openssl-devel tcl tcl-devel clang wget wget http://download.redis.io/redis-stable.tar.gz tar xvzf redis-stable.tar.gz cd redis-stable make redis-cli CC=clang BUILD_TLS=yes sudo install -m 755 src/redis-cli /usr/local/bin/`
    
3. After this, it is recommended that you run the optional `make test` command.
    
4. At the command prompt of your EC2 instance, type the following command, substituting the endpoint of your cluster and port for what is shown in this example.
    
    ``src/redis-cli -c -h `Cluster Endpoint` --tls -p 6379``

## 删除一个集群

### Console方式

The following procedure deletes a single cluster from your deployment. To delete multiple clusters, repeat the procedure for each cluster that you want to delete. You do not need to wait for one cluster to finish deleting before starting the procedure to delete another cluster.

###### To delete a cluster

1. Sign in to the AWS Management Console and open the MemoryDB for Redis console at [https://console.aws.amazon.com/memorydb/](https://console.aws.amazon.com/memorydb/).
    
2. To choose the cluster to delete, choose the radio button next to the cluster's name from the list of clusters. In this case, the name of the cluster you created at [Step 1: Create a cluster](https://docs.aws.amazon.com/memorydb/latest/devguide/getting-started.createcluster.html).
    
3. For **Actions**, choose **Delete**.
    
4. First choose whether to create a snapshot of the cluster before deleting it and then enter `delete` in the confirmation box and **Delete** to delete the cluster, or choose **Cancel** to keep the cluster.
    
    If you chose **Delete**, the status of the cluster changes to _deleting_.
    

As soon as your cluster is no longer listed in the list of clusters, you stop incurring charges for it.

### CLI方式

The following code deletes the cluster `my-cluster`. In this case, substitute `my-cluster` with the name of the cluster you created at [Step 1: Create a cluster](https://docs.aws.amazon.com/memorydb/latest/devguide/getting-started.createcluster.html).

`` aws memorydb delete-cluster --cluster-name `my-cluster` ``

The `delete-cluster` CLI operation only deletes one cluster. To delete multiple clusters, call `delete-cluster` for each cluster that you want to delete. You do not need to wait for one cluster to finish deleting before deleting another.

For Linux, macOS, or Unix:

`` aws memorydb delete-cluster \     --cluster-name `my-cluster` \     --region `us-east-1` ``

For Windows:

`` aws memorydb delete-cluster ^     --cluster-name `my-cluster` ^     --region `us-east-1` ``

For more information, see [`delete-cluster`](https://docs.aws.amazon.com/cli/latest/reference/memorydb/delete-cluster.html).