---
title: 分布式架构——Mysql实现主从同步
date:  2016-09-06
categories:  分布式架构
keywords: [Mysql,主从同步, 高可用]
tags:  [Mysql,主从同步, 高可用]
---

# mysql实现两台机器的主从同步#
## 准备工作##
> 将Master服务器上的备份数据库拷贝到Slave服务器上
> 
> **注意：**
> 使用Navicat拷贝的时候，需要在Slave上先创建数据库，然后再把数据和格式拷贝到此数据库上。


## Master配置##
修改/etc/my.cnf:
> 
	server-id=1 
	#需要备份的数据库名,如果需要备份多个数据库，重复设置这个选项即可。 
	binlog-do-db=repl
	#启动二进制日志系统  
	log-bin=mysql-bin       

查看信息
>
	show master status
	会显示log_file 与 log_pos信息，配置slave的时候需要用到
	![这里写图片描述](http://img.blog.csdn.net/20160706162518273)
## Slave配置##
登录之后，输入如下命令行：

> 
	slave stop;
	SET GLOBAL server_id=2;
>
	change master to
    master_host='120.26.203.200',
    master_user='gl',
    master_password='***',
    master_log_file='mysql-bin-log.000010',
    master_log_pos=1198;

>
	slave start;
![这里写图片描述](http://img.blog.csdn.net/20160706162702005)
**注意：**
>1. 在进行配置之前，一定要先将slave stop。配置完了，在start。
>2. 必须要设置server_id
>3. log_file与log_pos需要与master上的一致

## 双向同步实现
> 配置双向同步方法就是在单向同步的基础上略加改动，即在从机上做主机配置，在主机上做从机配置。

### 在slave上做主机配置
配置文件配置
>	
	在配置文件中添加：
	server-id=2
	binlog-do-db=bbc
	log-bin=mysql-bin

重启mysql
>
	service mysql restart
创建同步账号
>
	进入mysql命令窗口执行：
		grant replication slave on *.* to 'gl'@'120.26.203.200' identified by '***'; 
	刷新配置：
		flush privileges；

查看master状态
>
	show master status;
### 在master上做从机配置###
登录数据库执行如下命令：
>1. slave stop;
>2. change master to
    master_host='120.26.221.197',
    master_user='gl',
    master_password='***',
    master_log_file='mysql-bin-log.000002',
    master_log_pos=475;

>3. slave start;

 -------------------------------------------
  >**注意**：转载请标明，转自[itboy-木小草](http://muxiaocao.cn/2016/09/06/系统架构设计——Mysql实现主从同步/)。
>**尊重原创，尊重技术。**