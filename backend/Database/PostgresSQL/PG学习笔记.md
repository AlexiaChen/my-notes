#### PG内部原理

[The Internals of PostgreSQL : Introduction (interdb.jp)](https://www.interdb.jp/pg/)

### PG Web互动教学

- [Postgres Tutorials | Crunchy Data](https://www.crunchydata.com/developers/tutorials)



[How To Install PostgreSQL on Ubuntu 20.04 [Quickstart] | DigitalOcean](https://www.digitalocean.com/community/tutorials/how-to-install-postgresql-on-ubuntu-20-04-quickstart)


### 创建用户和数据库


```bash
# need you enter password with super user
psql --host=<hostname | ip | aws-rds endpoint> --port=5432 --username=<username> --password  --dbname=<database-name>
```

```sql
# 其中，`username` 是您想要创建的新用户的用户名，`password` 是您想要设置的密码。
CREATE USER username WITH PASSWORD 'password';
# 其中，`databasename` 是您想要创建的新数据库的名称。
CREATE DATABASE databasename;
# 授予新用户对新数据库的访问权限
GRANT ALL PRIVILEGES ON DATABASE databasename TO username;
```

在生产环境中，为了安全考虑，不建议在明文中指定密码，可以考虑使用加密方法来设置密码

```sql
SELECT md5('yourpassword');
CREATE USER username WITH ENCRYPTED PASSWORD 'md5encryptedpassword';
```

修改密码

```sql
SELECT md5('yourpassword');
ALTER USER username WITH ENCRYPTED PASSWORD 'md5encryptedpassword';
```