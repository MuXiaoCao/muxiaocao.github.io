---
title: 分布式架构——dubbo与SSM整合
date:  2016-06-02
categories:  分布式架构
keywords:  [dubbo,SOA, SSM, 服务化,RPC]
tags: 
	- dubbo
	- SOA
	- SSM
	- 服务化
	- RPC
---

>随着互联网的发展，网站应用的规模不断扩大，常规的垂直应用架构已无法应对，分布式服务架构以及流动计算架构势在必行，亟需一个治理系统确保架构有条不紊的演进。其中阿里的dubbo就是一款分布式服务框架,致力于提供高性能和透明化的RPC远程服务调用方案。
## 一、Dubbo简介
### 1.1 背景与需求
>随着互联网的发展，上网的人越来越多。加之移动互联的普及，更加快了互联网的脚步。随之而来的，便是对网站应用的强大挑战，和飞速的成长。下面我们就简单总结一下网站系统的成长史。

![这里写图片描述](http://img.blog.csdn.net/20160602201614567)
>1. 单一应用框架
-  当网站流量很小时，只需一个应用，将所有的功能都不熟在一起，以减少不熟节点和成本。
- 此时，用于监护增删改查工作量的数据访问框架（ORM）是关键。
2. 垂直应用框架
- 当访问量逐渐增大，单一应用增加机器带来的加速度越来越小，将应用拆成互不相干的几个应用，以提升效率。
- 此时，用于加速前端页面开发的 Web框架(MVC) 是关键。
3. 分布式服务架构
- 当垂直应用越来越多，应用之间交互不可避免，将核心业务抽取出来，作为独立的服务，逐渐形成稳定的服务中心，使前端应用能更快速的响应多变的市场需求。
- 此时，用于提高业务复用及整合的 分布式服务框架(RPC) 是关键。
4. 流动计算架构
- 当服务越来越多，容量的评估，小服务资源的浪费等问题逐渐显现，此时需增加一个调度中心基于访问压力实时管理集群容量，提高集群利用率。
- 此时，用于提高机器利用率的 资源调度和治理中心(SOA) 是关键。

### 1.2 什么是Dubbo
>Dubbo是阿里巴巴公司开源的一个高性能优秀的服务框架，使得应用可通过高性能的 RPC 实现服务的输出和输入功能，可以和 Spring框架无缝集成。
>主要核心部件有：
>1. **Remoting**: 网络通信框架，实现了 sync-over-async 和 request-response 消息机制.
>2. **RPC**: 一个远程过程调用的抽象，支持负载均衡、容灾和集群功能
>3. **Registry**: 服务目录框架用于服务的注册和服务事件发布和订阅.
### 1.3 Dubbo架构
![这里写图片描述](http://img.blog.csdn.net/20160602202934007)
>#### **节点角色说明**:
>1. **Provider**: 暴露服务的服务提供方。
>2. **Consumer**:调用远程服务的服务消费方。
>3. **Registry**:服务注册与发现的注册中心。
>4. **Monitor**:统计服务的调用次调和调用时间的监控中心。
>5. **Container**:服务运行容器。
>#### **调用关系说明**:
>0. 服务容器负责启动，加载，运行服务提供者。
1. 服务提供者在启动时，向注册中心注册自己提供的服务。
2. 服务消费者在启动时，向注册中心订阅自己所需的服务。
3. 注册中心返回服务提供者地址列表给消费者，如果有变更，注册中心将基于长连接推送变更数据给消费者。
4. 服务消费者，从提供者地址列表中，基于软负载均衡算法，选一台提供者进行调用，如果调用失败，再选另一台调用。
5. 服务消费者和提供者，在内存中累计调用次数和调用时间，定时每分钟发送一次统计数据到监控中心。
## 二、Dubbo的简单使用
###2.1 准备工作
>使用**maven**配置dubbo
```
<dependency>
	<groupId>com.alibaba</groupId>	
	<artifactId>dubbo</artifactId>
	<version>2.4.10</version>
</dependency>
```
### 2.1 服务提供端(provider)
#### **第一步：配置provider.xml**
```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
    xsi:schemaLocation="http://www.springframework.org/schema/beans        
    http://www.springframework.org/schema/beans/spring-beans.xsd        
    http://code.alibabatech.com/schema/dubbo        
    http://code.alibabatech.com/schema/dubbo/dubbo.xsd">
    
    <!-- 提供方应用信息，用于计算依赖关系 -->
    <dubbo:application name="hello-world-app"  />
    <!-- 使用multicast广播注册中心暴露服务地址 -->
    <dubbo:registry address="multicast://224.5.6.7:1234" />
    <!-- 用dubbo协议在20880端口暴露服务 -->
    <dubbo:protocol name="dubbo" port="20880" />
    <!-- 声明需要暴露的服务接口 -->
    <dubbo:service interface="com.xiaocao.mydubbo.DemoService" ref="demoService" />
    <!-- 和本地bean一样实现服务 -->
    <bean id="demoService" class="com.xiaocao.mydubbo.impl.DemoServiceImpl" />
</beans>
```
####**第二步：设计接口与实现类**
**接口：**
```
package com.xiaocao.mydubbo;
public interface DemoService {
	public String sayHello(String name);
}
```
**实现类:**
```
package com.xiaocao.mydubbo.impl;

import com.xiaocao.mydubbo.DemoService;

public class DemoServiceImpl implements DemoService{
	@Override
	public String sayHello(String name) {
		return "hello" + name;
	}
}

```
### 2.2 服务使用端(consumer)
#### **第一步：配置consumer.xml**
```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
    xsi:schemaLocation="http://www.springframework.org/schema/beans        
    http://www.springframework.org/schema/beans/spring-beans.xsd        
    http://code.alibabatech.com/schema/dubbo        
    http://code.alibabatech.com/schema/dubbo/dubbo.xsd">
 
     <!-- 消费方应用名，用于计算依赖关系，不是匹配条件，不要与提供方一样 -->
    <dubbo:application name="consumer-of-helloworld-app"  />
    <!-- 使用multicast广播注册中心暴露发现服务地址 -->
    <dubbo:registry address="multicast://224.5.6.7:1234" />
    <!-- 生成远程服务代理，可以和本地bean一样使用demoService -->
    <dubbo:reference id="demoService" interface="com.xiaocao.mydubbo.DemoService" />
</beans>
```
#### **第二步：将provider的接口生成jar包引入，或者之际拷贝到consumer项目（包名必须一样）**
### 2.3 测试
**privider端：**
```
package com.xiaocao.mydubbo.main;

import org.springframework.context.support.ClassPathXmlApplicationContext;

public class Provider {
	public static void main(String[] args) throws Exception {
		ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext(new String[] { "provider.xml" });
		context.start();
		System.in.read(); // 按任意键退出
	}
}
```
**consumer端：**
```
package com.xiaocao.mydubbo.main;

import org.springframework.context.support.ClassPathXmlApplicationContext;

import com.xiaocao.mydubbo.DemoService;

public class Consumer {
	public static void main(String[] args) throws Exception {
		ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext(new String[] { "consumer.xml" });
		context.start();
		DemoService demoService = (DemoService)context.getBean("demoService"); // 获取远程服务代理
	    String hello = demoService.sayHello("  dubbo "); // 执行远程方法
	    context.close();
		System.out.println(hello);
	}
}
```
**结果：**
![这里写图片描述](http://img.blog.csdn.net/20160602205145241)
## **三、Dubbo与SSM框架整合实现分布式系统**
> **整合的话需要注意两个问题：**
> 1. 使用spring的时候，他总是加载不了dubbo下的xsd文件，而是去http://code.alibabatech.com/schema/dubbo/dubbo.xsd下载。而阿里的网站已经停了好几个月了，所以命名空间不能使用，需要手动配置。
> **解决办法：**取下载好的dubbo的jar包META-INF下的xsd文件，配置到eclipse的xml配置中，配置方法见：http://blog.sina.com.cn/s/blog_7750745b0101ijne.html
> 2.  Dubbo缺省依赖以下三方库：
> ![这里写图片描述](http://img.blog.csdn.net/20160602210239487)
>我们在使用mybatis与spring整合的时候，使用spring至少是3.0版本。而Dubbo引用的是2.*版本。会引起版本冲突。
>**解决办法：**修改pom文件中的dubbo
```
<!-- 分布式服务框架,致力于提供高性能和透明化的RPC远程服务调用方案 -->
<dependency>
	<groupId>com.alibaba</groupId>
	<artifactId>dubbo</artifactId>
	<version>${dubbo.version}</version>
	<exclusions>
		<exclusion>
			<groupId>org.springframework</groupId>
			<artifactId>spring</artifactId>
			<version>4.1.2<version>
		</exclusion>
	</exclusions>
</dependency>
```
### 整合步骤一：在web.xml配置加载provider.xml
>使用spring容器加载写好的provider.xml文件
```
<!-- 初始化spring容器 -->
	<context-param>
		<param-name>contextConfigLocation</param-name>
		<param-value>classpath:spring/applicationContext-*.xml</param-value>
	</context-param>
	<listener>
		<listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
	</listener>
```
**注：我的provider配置文件命名的是applicationContext-privider.xml**
### 整合步骤二：共享接口文件
![这里写图片描述](http://img.blog.csdn.net/20160602211723782)
### 整合步骤三：在consumer端使用Ioc注解，像本地一样调用即可
![这里写图片描述](http://img.blog.csdn.net/20160602213307669)
## **四、感想**
>dubbo其实是使用配置文件的方式包装了[RPC](http://blog.csdn.net/qq_25689397/article/details/51051945)远程过程调用，而RPC的底层实现，其实是使用了[动态代理](http://blog.csdn.net/qq_25689397/article/details/51427164)和Socket通信。所以乍一看dubbo的功能很神奇，可以让使用本地方法一样实现远程调用，其实内在的机制只是简单的技术进行组合。
>**总结：**我们在学习的过程中，总是会听到前辈们给我们强调**基础的重要性**。这确实是**很简单但是很重要的真理**。技术不断的推陈出新，我们只是学习使用技术是永远赶不上的，更别说有所创新和发展了。其实**新技术的产生，往往都是一些你我都知道的技术经过重新组合而产生的**，只要我们能**理解了基本的技术和知识**，应对任何新技术的产生都能很快的理解和应用。
>再说一遍我之前博客说过的话吧：**技术其实就在那里，多不多少不少，只要我们思维足够开阔，就能够把一些简单的技术组合成一种让人耳目一新的技术。而在中国其实从不缺乏技术人才，而是缺乏技术的想象力，只有我们将已有的技术进行大胆的想象和组合，就能让我们的技术流行起来，让别人追赶我们的思维，而不是我们一直再用双手追赶别人的思维，这样是永远赶不上的。**

**文章中源码下载地址：[请点击](https://github.com/MuXiaoCao/DubboDemo)**
**有关dubbo的独立服务的文章：**[请点击](http://blog.csdn.net/wilsonke/article/details/39935801)
  -------------------------------------------
  >**注意**：转载请标明，转自[itboy-木小草](http://muxiaocao.cn/2016/06/02/分布式架构——dubbo与SSM整合/)。
>**尊重原创，尊重技术。**