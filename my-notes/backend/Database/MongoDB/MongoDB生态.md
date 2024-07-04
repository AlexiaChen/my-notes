# 1 MongoDB on AWS

  

[How To Run MongoDB Atlas On AWS Cloud | MongoDB](https://www.mongodb.com/mongodb-on-aws)    是MongoDB官方的，托管于AWS上的。

  

当然还有AWS自己出品的DocumentDB，兼容MongoDB协议的。 [Json Document Database - Amazon DocumentDB - AWS](https://aws.amazon.com/documentdb/?nc2=h_ql_prod_db_doc)

  

# 2 MongoDB系列产品

  

MongoDB Enterprise，  MongoDB Community  MongoDB Atlas， Percona MongoDB  

  

## 2.1 MongoDB企业版

  

这个会提供企业级所需的丰富功能，高吞吐，内建的复制机制，运维工具，空闲时间加密数据，分析，索引，平台级集成，平台级证书，数据可视化，内存存储引擎，加密存储引擎，原生的数据分片能力，内建的容错(failover)， LDAP支持，低延迟，审计，灵活的查询，运维管理器，高可用。

[Datavail_WP_MongoDB_Which_Version_is_Right_for_You.pdf](https://www.datavail.com/whitepaper/Datavail_WP_MongoDB_Which_Version_is_Right_for_You.pdf)

## 2.2 MongoDB社区版

  

社区版其实就是免费的MongoDB实现，但是可能你会没有类似企业版那样的高级功能。比如LDAP。

社区版本提供： 灵活的查询，原生的数据分片能力，内建的复制机制，索引，实时聚合能力，高可用。



  

社区版选择两个版本 ：

  

5.0.23    [MongoDB Repositories](https://repo.mongodb.org/yum/redhat/7/mongodb-org/5.0/x86_64/RPMS/)

6.0.12   [MongoDB Repositories](https://repo.mongodb.org/yum/redhat/7/mongodb-org/6.0/x86_64/RPMS/)


## 2.3 MongoDB Altlas

  

直接就是一个PaaS实现，有云的所有优势。可以在AWS，Google Cloud，Azure上完美工作。可伸缩非常简单。

  

提供的功能： 实时触发器，全球级别的集群，高度安全的环境，数据库监控，完全托管的备份和恢复机制，自定义报警。Point-in-Time恢复机制，按需快照，简单的数据库迁移。

  

## 2.4 Percona MongoDB

  

这个版本是MongoDB社区版的一个平等替换版本，免费且开源，而且在性能优化和可靠性上下了大功夫，可以简单理解为是穷人版企业版。

  

提供的功能有： 数据库审计，OpenLDAP支持，Active Directory支持，数据空闲时加密，热备份机制，Percona实现的内存存储引擎，日志压缩， HashCorp Vault集成（这个Vault其实就是存放各种配置的敏感字段，比如APIKey这样的，数据仓库），可提炼的分片Keys，有付费支持的选项（毕竟Percona要赚钱）。

  

Percona 是一个有趣的选择，它想社区版到企业版的过渡版本，然后还提供了社区版没有的一些企业版的关键功能，负载也比社区版高。而且价格也便宜。

  

## 3 总结

也就是在社区版和Percona MongoDB之间选择了。