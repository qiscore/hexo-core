---
title: docker常用容器创建
date: 2023-12-09 15:09:56
tags: docker
categories: docker
---

# 1. MySQL


## 1. 准备mysql配置文件my.cnf
```shell
[client]
default-character-set = utf8mb4

[mysql]
default-character-set = utf8mb4

[mysqld]
character-set-client-handshake = FALSE
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci
init_connect='SET NAMES utf8mb4'

#默认时区配置
default-time_zone = '+8:00' 

#设置数据库支持分组
sql_mode=STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION
#表名不区分大小写
lower_case_table_names=1

```

## 2. 准备容器创建脚本mysql-install-docker.sh

```shell

#!/bin/bash

docker_root=/usr/local/software/docker
mysql_dir=$docker_root/mysql
config_dir=$mysql_dir/config
data_dir=$mysql_dir/data
logs_dir=$mysql_dir/logs

echo "创建数据目录"
mkdir -p $config_dir
mkdir -p $data_dir

mv ./my.cnf $config_dir
chmod -R 755 $docker_root

echo "开始拉取MySQL5.7镜像"
docker pull mysql:5.7

echo "创建容器"

docker run -d -p 3306:3306 --privileged=true \
-v $config_dir/my.cnf:/etc/mysql/my.cnf \
-v $data_dir:/var/lib/mysql \
-v $logs_dir:/logs \
-e MYSQL_ROOT_PASSWORD=root \
--name mysql mysql:5.7

echo "容器创建完成"
docker ps | grep "mysql"

mv ./mysql-install-docker.sh $mysql_dir

```

## 3. 执行脚本

```shell
sh mysql-install-docker.sh

```

