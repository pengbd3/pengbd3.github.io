---
title: 利用mysql做GEARMAN的持久化
date: 2017-11-09 15:06:01
tags:
  - PHP学习
  - gearman
categories: "PHP学习"
---
##### 一、为什么要持久化? #####

gearman的job server中的工作队列存储在内存中，一旦服务器有未处理的任务时重启或者宕机，那么这些任务就会丢失。  
持久化存储队列可以允许添加后台任务，并将其存储在外部的持久型队列里（比如MySQL数据库）。  
##### 二、关于gearman的持久化的文章，建议可以看官方文档 #####
``	
http://gearman.org/manual/job_server/#persistent_queues``  
##### 三、创建用于持久化的数据库和表 #####
登录mysql ``mysql -uroot -p`` ，创建表
![安装成功后的进程展示](http://oih4t7o53.bkt.clouddn.com/gearman2/1.jpg)
```mysql
CREATE DATABASE gearman;
 
CREATE TABLE `gearman_queue` (
`unique_key` varchar(64) NOT NULL,
`function_name` varchar(255) NOT NULL,
`priority` int(11) NOT NULL,
`data` longblob NOT NULL,
`when_to_run` int(11),
PRIMARY KEY (`unique_key`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```
创建mysql中的连接gearman用户  
``` mysql
grant all privileges on gearman.* to gearman@'%' identified by '123456';
flush privileges;
exit;
```
![新建gearman用户](http://oih4t7o53.bkt.clouddn.com/gearman2/2.jpg)  
确保3306的端口已经打开，测试远程是否能够正常访问  
![新建gearman用户](http://oih4t7o53.bkt.clouddn.com/gearman2/3.jpg)

```shell
gearmand -q mysql --mysql-host=192.168.18.16 --mysql-port=3306 --mysql-user=gearman --mysql-password=123456 --mysql-db=gearman --mysql-table=gearman_queue
```
