
## 安装


[Download MongoDB Community Server | MongoDB](https://www.mongodb.com/try/download/community)

[Install MongoDB Community Edition on Linux — MongoDB Manual](https://www.mongodb.com/docs/manual/administration/install-on-linux/#std-label-install-mdb-community-edition-linux)

```bash
sudo yum install -y mongodb-org-7.0.3 mongodb-org-database-7.0.3 mongodb-org-server-7.0.3 mongodb-mongosh-7.0.3 mongodb-org-mongos-7.0.3 mongodb-org-tools-7.0.3
```
### 卸载

To uninstall and delete MongoDB completely on CentOS, you need to stop the MongoDB process, remove the installed MongoDB packages, and delete the data directories, MongoDB database(s), and log files. You must have sudo access to do this. Here are the steps to follow:

1. Stop the MongoDB process: `sudo service mongod stop`
2. Completely remove the installed MongoDB packages: `sudo yum remove mongodb-org*`
3. Remove the data directories, MongoDB database(s), and log files: `sudo rm -r /var/log/mongodb /var/lib/mongo`

After completing these steps, you can check if MongoDB is successfully uninstalled by typing: `service mongod status`. If you wish to reinstall MongoDB on Linux, follow the links provided in the search results.

Citations:
[1] https://www.youtube.com/watch?v=mDvqUIETkoI
[2] https://www.mongodb.com/basics/uninstall-mongodb
[3] https://www.mongodb.com/community/forums/t/mongodb-shell-is-not-uninstalled/194750
[4] https://gist.github.com/median-man/ea9e307d842c5f32adf20d383f9e6293
[5] https://stackoverflow.com/questions/8766579/uninstalling-mongo

## 检查是否安装mongo

To check if MongoDB is installed on CentOS, you can use the command line and check the version of MongoDB installed. Open the terminal and type the following command: `mongod --version`. If MongoDB is installed, it will display the version of MongoDB installed. If it is not installed, it will display a message saying "command not found". Alternatively, you can also use the `mongo --version` command to check the version of the MongoDB shell installed. If you want to check the version of MongoDB server, you need to connect to the MongoDB server first and then use the `db.version()` method. [1][5]

Citations:
[1] https://www.configserverfirewall.com/mongodb/check-mongodb-version/
[2] https://www.mongodb.com/docs/manual/tutorial/install-mongodb-on-red-hat/
[3] https://www.youtube.com/watch?v=mDvqUIETkoI
[4] https://stackoverflow.com/questions/19405791/what-version-of-mongodb-is-installed-on-ubuntu
[5] https://database.guide/7-ways-to-check-your-mongodb-version/


## 容器化安装

```bash
chown 999:999 /mnt/rasp_data/mongo/
docker run -d --name mongo --privileged=true -p 27017:27017 -v /mnt/rasp_data/mongo:/data/db mongo:4.2
```