

需要三台支持硬件虚拟化的主机，无论是AMD-V 还是Intel-VT


192.168.13.7    master
192.168.13.9    slave
192.168.13.10  slave
root  

landui@123

通过运行一下命令查看CPU是否支持Virtualization:

```bash
egrep -o '(vmx|svm)' /proc/cpuinfo
# 硬盘大小，最好500G   250G以上
df -h
```

输出vmx就是Intel-VT，输出svm就是AMD-V

cat /etc/redhat-release
192.168.13.7 的系统版本是 CentOS Linux release 7.6.1810 (Core)

[How To Install MySQL on CentOS 7 | DigitalOcean](https://www.digitalocean.com/community/tutorials/how-to-install-mysql-on-centos-7)

curl -sSLO https://dev.mysql.com/get/mysql57-community-release-el7-11.noarch.rpm

rpm -ivh ./mysql57-community-release-el7-11.noarch.rpm

```
rm -rf /var/lib/mysql/*
yum remove $(rpm -qa | grep mysql)
```

[Compatibility Matrix — Apache CloudStack 4.19.0.0 documentation](https://docs.cloudstack.apache.org/en/latest/releasenotes/compat.html)

grep 'temporary password' /var/log/mysqld.log

mysql_secure_installation

新密码 landui@123LANDUI