

虽然Arch体系的Linux都推荐用MariaDB，也不知道为什么Arch Linux不大推荐MySQL，虽然二者协议兼容，但是我就想用MySQL。

我这边是中科大的源。

```bash
sudo pacman -S mysql
sudo mysqld --initialize --user=mysql --basedir=/usr --datadir=/var/lib/mysql
```

如果成功，terminal会出现：

```txt

2018-08-25T05:33:04.399546Z 0 [Warning] [MY-010915] [Server] 'NO_ZERO_DATE', 'NO_ZERO_IN_DATE' and 'ERROR_FOR_DIVISION_BY_ZERO' sql modes should be used with strict mode. They will be merged with strict mode in a future release.

2018-08-25T05:33:04.399603Z 0 [System] [MY-013169] [Server] /usr/bin/mysqld (mysqld 8.0.12) initializing of server in progress as process 20610

2018-08-25T05:33:40.201045Z 5 [Note] [MY-010454] [Server] A temporary password is generated for root@localhost: 56%ei0#nQ:/;

2018-08-25T05:34:11.553951Z 0 [System] [MY-013170] [Server] /usr/bin/mysqld (mysqld 8.0.12) initializing of server has completed
```

临时产生了一个root用户的密码，需要登录进去重新更改才可以操作数据库。

先启动mysql服务：

```bash
sudo systemctl start mysqld
```

然后登录：

```bash
mysql -u root -p[temp_password]
```

更改root临时密码：

```bash

ALTER user 'root'@'localhost' IDENTIFIED BY 'new_password';

```

然后就可以各种命令了, 建库，建用户，授权等等:

```bash

create database [new_db_name];

create user 'new_user_name'@'localhost' identified by 'user_password';

use [db_name];

grant all privileges on [db_name].* to '[user_name]'@'localhost' with grant option;

flush privileges;

```

EOF