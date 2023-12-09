---
title: shell编程语法
date: 2023-12-09 14:21:07
tags: shell编程
categories: shell
---

# 1.bash文件的定义

告诉计算机使用哪种解释器，解释器有以下几种

#!/bin/bash

#!/bin/sh

#!/usr/bin/env

#!/usr/bin/python

```shell
#!/bin/bash
#/bin/bash表示使用bash解释器
#/bin/sh表示使用sh解释器
#脚本内容
echo $(cat /etc/profile)
```

# 2.shell文件的执行方式

- 文件保存为xxx.sh

- 4种执行方式

  - 文件执行,有权限限制，需要先赋予执行权限，开启子进程执行

    `./xxx.sh`

  - bash命令执行，解释器为bash，开启子进程执行

    ​	`bash ./xxx.sh`

  - sh命令执行,解释器为sh,开启子进程执行

    ​	`sh ./xxx.sh`

  - source命令(点命令)执行,不开启子进程(后台执行，终端关闭)

    ​	`source ./xxx.sh` `. ./xxx.sh`

# 3.输入与输出

> > 输出

- echo

```shell
#原始内容直接输出，输出结果为shell
echo "shell"
#转义输出,输出结果为hello world
echo -e "hello\tworld"

```



- printf

```shell
#格式化输出,输出结果为｜12｜
printf "|%d|" 12 
```

> > 输入

- read

  - -p显示提示信息
  - -t设置超时时间
  - -n允许输入的字符长度
  - -r 支持读取\
  - -s不显示输入内容

  ```shell
  #输入3组字符串
  read a b c
  #设置提示消息
  read -p "请输入内容：" a
  #不显示输入的内容
  read -s -p "请输入密码：" a
  ```

  

# 4.管道命令（中竖线） ｜

```shell
#统计命令执行结果中数据的行数
ls -la | wc -l
#查看所有服务监听的端口列表中包含sshd的
ss -nutlp | grep sshd
#将输出的字符串作为参数，修改root的密码
echo "123..." | passwd --stdin root
```

# 5.输入与输出重定向（将执行结果保存到文件中）

- 输出重定向

```shell
#将执行结果导出到文件，如果文件不存在，则创建，存在则覆盖内容
ls -la > aaa.txt
#将执行结果追加到文件的末尾，不覆盖文件
ls -la >>aaa.txt


#正确与错误信息输出的重定向
#1.正确输出重定向
ls -la >> aaa.txt
#2.错误输出重定向
ls -la 2>>aaa.txt
#3.正确与错误输出重定向到不同的文件
ls -la >>aaa.txt 2>>bbb.txt
#4.正确与错误信息同时追加重定向到同一个文件中
ls -la &>>aaa.txt
#5.将错误输出重定向到正确输出
ls -la  >aaa.txt 2>&1
#6.将正确输出重定向到错误输出
ls -la  >aaa.txt 1>>&2

```

- 输入重定向

使用<。<后面跟一个文件名。将数据内容重定向传递给前面的一个命令，作为命令的输入。

```shell
#非交互的执行命令.使用脚本自动发送邮件，从文件中读取参数
mail -s warning root@localhosts < /etc/hosts

#使用<<就可以让脚本不需要依赖文件即可独立运行
#!/bin/bash
#语法格式
#命令 << 分隔符
#内容
#分隔符

#cat通过<<读取数据，再通过输出重定向将数据导出到文件中
#!/bin/bash
cat > aaa.txt << HERE
this is content
HERE

#不能屏蔽tab键，缩进作为内容的一部分被输出，并传递给程序
#!/bin/bash
cat > aaa.txt << HERE
	this is content
HERE

#使用<<-可以忽略掉数据内容及分隔符前的tab键，不传递给程序
#!/bin/bash
cat > aaa.txt <<- HERE
	this is content
HERE

```

# 6.变量与符号的使用
更新中...