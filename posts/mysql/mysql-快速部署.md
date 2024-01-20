---
title: mysql-快速部署
description: mysql-快速部署
date: 2024-01-20
tags:
  - mysql
---



怎么快速部署并配置一个可远程访问的mysql实例？

演示环境：

- 操作系统：CentOS7
- MySQL：8.0



### 1. 部署脚本

```shell
#!/bin/bash

NEW_PASSWORD="Root!123"

# 配置yum仓库
yum install -y https://repo.mysql.com//mysql80-community-release-el7-11.noarch.rpm

# 安装mysql
yum install -y mysql-community-server

# 启动mysql
systemctl start mysqld
systemctl enable mysqld

# 关闭防火墙
systemctl stop firewalld
systemctl disable firewalld

TEMP_PASSWORD=$(grep 'temporary password' /var/log/mysqld.log | awk 'END {print $NF}')

# 修改密码并开启远程连接
mysql --connect-expired-password -u root -p"$TEMP_PASSWORD" <<-EOF
alter user 'root'@'localhost' identified BY '$NEW_PASSWORD';
create user 'root'@'%' identified by '$NEW_PASSWORD';
grant all privileges on *.* to 'root'@'%' with grant option;
FLUSH PRIVILEGES;
EOF

if [ $? -eq 0 ]; then
    echo "success,password:$NEW_PASSWORD"
else
    echo "failure"
fi
```



### 2. 参考资料

- https://dev.mysql.com/doc/refman/8.3/en/linux-installation-yum-repo.html
- https://dev.mysql.com/downloads/repo/yum/