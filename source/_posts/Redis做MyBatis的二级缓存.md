---
title: 使用Redis做MyBatis的二级缓存
date:  2016-08-15
categories:  框架学习
keywords:  [锁,ReentrantLock, AQS, 并发包]
tags: 
	- redis
	- myBatis
---

# 使用Redis做MyBatis的二级缓存
>　通常为了减轻数据库的压力，我们会引入缓存。在Dao查询数据库之前，先去缓存中找是否有要找的数据，如果有则用缓存中的数据即可，就不用查询数据库了。如果没有才去数据库中查找。这样就能分担一下数据库的压力。另外，为了让缓存中的数据与数据库同步，我们应该在该数据发生变化的地方加入更新缓存的逻辑代码。这样无形之中增加了工作量，同时也是一种对原有代码的入侵。这对于有着代码洁癖的程序员来说，无疑是一种伤害。

> MyBatis框架早就考虑到了这些问题，因此MyBatis提供了自定义的二级缓存概念，方便引入我们自己的缓存机制，而不用更改原有的业务逻辑。下面就让我们了解一下MyBatis的缓存机制。

## 一、缓存概述


> 正如大多数持久层框架一样，MyBatis 同样提供了一级缓存和二级缓存的支持；


- 一级缓存基于 PerpetualCache 的 HashMap 本地缓存，其存储作用域为 Session，当 Session flush 或 close 之后，该Session中的所有 Cache 就将清空。

- 二级缓存与一级缓存其机制相同，默认也是采用 PerpetualCache，HashMap存储，不同在于其存储作用域为 Mapper(Namespace)，并且可自定义存储源，如 Ehcache、Hazelcast等。
- 对于缓存数据更新机制，当某一个作用域(一级缓存Session/二级缓存Namespaces)的进行了 C/U/D 操作后，默认该作用域下所有 select 中的缓存将被clear。

- MyBatis 的缓存采用了delegate机制 及 装饰器模式设计，当put、get、remove时，其中会经过多层 delegate cache 处理，其Cache类别有：BaseCache(基础缓存)、EvictionCache(排除算法缓存) 、DecoratorCache(装饰器缓存)：
	1. BaseCache         ：为缓存数据最终存储的处理类，默认为 PerpetualCache，基于Map存储；可自定义存储处理，如基于EhCache、Memcached等； 
	2. EvictionCache    ：当缓存数量达到一定大小后，将通过算法对缓存数据进行清除。默认采用 Lru 算法(LruCache)，提供有 fifo 算法(FifoCache)等； 
	3. DecoratorCache：缓存put/get处理前后的装饰器，如使用 LoggingCache 输出缓存命中日志信息、使用 SerializedCache 对 Cache的数据 put或get 进行序列化及反序列化处理、当设置flushInterval(默认1/h)后，则使用 ScheduledCache 对缓存数据进行定时刷新等。

- 一般缓存框架的数据结构基本上都是 Key-Value 方式存储，MyBatis 对于其 Key 的生成采取规则为：
	
	[hashcode : checksum : mappedStementId : offset : limit : executeSql : queryParams]。

- 对于并发 Read/Write 时缓存数据的同步问题，MyBatis 默认基于 JDK/concurrent中的ReadWriteLock，使用 ReentrantReadWriteLock 的实现，从而通过 Lock 机制防止在并发 Write Cache 过程中线程安全问题。

## 二、源码剖析
### 2.1 执行流程分析
> 接下来将结合 MyBatis 序列图进行源码分析。在分析其Cache前，先看看其整个处理过程。

![](http://i.imgur.com/jhUJB5i.png)
1. 通常我们在service层最终都会调用Mapper的接口方法，实现对数据库的操作，本例中是通过id查询product对象。
2. 我们知道Mapper是一个接口，接口是没有对象的，更不能调用方法了，而我们调用的其实是mybatis框架的mapper动态代理对象MapperProxy，而MapperProxy中有封装了配置信息的DefaultSqlSession中的Configuration。动态代理的具体实现请戳[这里](http://blog.csdn.net/qq_25689397/article/details/51427164)。调用mapper方法的具体代码如下。
![](http://i.imgur.com/gPBHI1C.png)
3. 在执行mapperMethod的execute的时候，不仅传递了方法参数，还传递了sqlSession。在执行execute，其实是通过判断配置文件的操作类型，来调用sqlSession的对应方法的。本例中，由于是select，而返回值不是list，所以下一步执行的是sqlSession的selectOne方法具体代码如下：
![](http://i.imgur.com/hNFhXVk.png)
4. selectOne其实调用了selectList，只不过是取了第一个。具体代码如下：
![](http://i.imgur.com/DiGx23x.png)
5. selectList经过层层调用，最终交给执行器执行。具体执行器的结构待会我们会分析。注意这里的ms参数，其实就是从Configration中得到的一些配置信息，包括mapper文件里的sql语句。具体代码如下：
![](http://i.imgur.com/AvViaoY.png)
6. 这里的执行器execute，其实是spring注入的。excute是一个接口，而到时候具体是哪个execute执行，是看配置文件的。而我们的一级缓存和二级缓存其实都是execute中的一种。接下来，我们遍分析一下执行器（Executor）。

### 2.2 执行器（Executor）
> Executor:执行器接口。也是最终执行数据获取及更新的实例。其结构如下：

![](http://i.imgur.com/oVQugSs.png)

1. BaseExecutor:基础执行器抽象类。实现一些通用方法，如createCacheKey之类的。并采用[模板模式](http://blog.csdn.net/qq_25689397/article/details/52014681)将具体的数据库操作逻辑交由子类实现。另外，可以看到变量localCache:PerpetualCache,在该类采用perpetualCache实现基于map存储的一级缓存，其query方法如下：
![](http://i.imgur.com/ZyW2aKW.png)
一级缓存和二级缓存很相似，都是实现Cache缓存接口，然后等待调用。其中的一级缓存具体实现其实使用了Map存储，原理非常简单。PerPetualCache具体结构如下：
![](http://i.imgur.com/jTOgnio.png)

2. BatchExcutor、ReuseExcutor、SimpleExcutor:这三个就是简单的继承了BaseExcutor，实现了doQuery、doUpdate等发放，同样都是采用了JDBC对数据库进行操作；三者的区别在于，批量执行、重用Statement执行、普通方法执行。具体应用及长江在mybatis文档上都有详细的说明。

3. CachingExecutor:二级缓存执行器。其中使用了静态代理模式，当二级缓存中没有数据的时候，就使用BaseExecutor做代理，进行下一步执行。具体代码如下：
![](http://i.imgur.com/osVOwL4.png)

### 2.3 Cache的设计
> 像之前所说，Cache是一个缓存接口，运行时用到的其实是在解析mapper文件的时候根据配置文件生成的对应Cahce实现类。另外这个实现类的构造过程使用了建造者（Builder）模式。在build的过程中，将所有设计到的cache放入基础缓存中，并使用装饰器模式将cache进行装饰。具体设计如下：
	
	1. 从配置文件中获取节点，将配置信息提取出来初始化生成Cache
![](http://i.imgur.com/iCOeO7L.png)
	2. useNewCache方法中使用了建造者（Builder）模式，将从配置文件中读取出来的各个元素组装起来。其中最主要的是build方法。
![](http://i.imgur.com/GELQkAa.png)
	3. 在build方法中，值得注意的是使用了装饰器模式，将几个基本的Cache装饰了一下。因为我们的Cache只是加入了自定义的缓存功能和逻辑，日志功能、同步功能等其实并没有。所以需要装饰一下，具体代码如下：
![](http://i.imgur.com/dnY4SRI.png)
![](http://i.imgur.com/wSGE6zE.png)
	4. 最终的缓存实例对象结构：
![](http://i.imgur.com/ruBltZg.png)

### 2.4 总结
>　总体上看，我们可以把MyBatis关于缓存的这一部分分为三个部分：

1. 解析器：结合mybatis-spring框架，读取spring关于mybatis的配置文件。具体看是否开启缓存（这里指二级缓存），如果开启，生成的执行器为CachingExecutor。
![](http://i.imgur.com/Is1lFdr.png)
![](http://i.imgur.com/l6DoYFu.png)
2. 动态代理：实现调用mapper接口的时候执行mybatis逻辑	
3. 执行器：执行缓存处理逻辑。在这里二级缓存和一级缓存有所区别。

## 三、具体实现

### 3.1 配置文件
1. 开启缓存：修改spring中关于mybatis的配置文件，将cacheEnabled设置为true。
![](http://i.imgur.com/08pXmlO.png)
2. 添加实现Cache接口的实现类。重写方法会在查询数据库前后调用，查询、更新、删除、创建缓存需要在这几个方法中实现。值得注意的是，getObject方法，当返回的是null时，就会接着查询。如果不为null，则返回，不再查询了。
```
/**
 * 使用redis做mybatis二级缓存
 * @Description
 * @file_name MyBatisRedisCache.java
 * @time 2016-07-26 下午4:49:13
 * @author muxiaocao
 */
public class MyBatisRedisCache implements Cache{
	
	@Value("#{config['redis.ip']}")
	protected String redisIp;

	@Value("#{config['redis.port']}")
	protected Integer redisPort;
	
	private static Log logger =  LogFactory.getLog(MyBatisRedisCache.class);
  	
	private Jedis redisClient = createClient();
	
	private final ReadWriteLock readWriteLock = new ReentrantReadWriteLock();
	
	private String id;
	
	public MyBatisRedisCache(final String id) {
		if (id == null) {
			throw new IllegalArgumentException("缓存没有初始化id");
		}
		logger.debug("==================MyBatisRedisCache:id=" + id);
		this.id = id;
	}
	
	@Override
	public String getId() {
		return this.id;
	}

	@Override
	public int getSize() {
		return Integer.valueOf(redisClient.dbSize().toString());
	}

	@Override
	public void putObject(Object key, Object value) {
		logger.debug("==================pubObject:" + key + "=" + value);
		redisClient.set(SerializeUtil.serialize(key),SerializeUtil.serialize(value));
	}
	@Override
	public Object getObject(Object key) {
		Object value = SerializeUtil.unserialize(redisClient.get(SerializeUtil.serialize(key.toString())));
		logger.debug("==================getObjec:" + key + "=" + value);
		return value;
	}

	@Override
	public Object removeObject(Object key) {
		return redisClient.expire(SerializeUtil.serialize(key.toString()), 0);
	}

	@Override
	public void clear() {
		redisClient.flushDB();
	}

	@Override
	public ReadWriteLock getReadWriteLock() {
		return readWriteLock;
	}
	
	private Jedis createClient() {
		try {
			RedisUtil.initRedis(redisIp);
			return RedisUtil.getRedis();
		} catch (Exception e) {
			e.printStackTrace();
		}
		throw new RuntimeException("初始化redis错误");
	}
}

```
3. 在相应的Mapper文件中，加入Cache节点，将自定义的cache实现类加进去。
![](http://i.imgur.com/roH1eZf.png)

## 四、总结
>　MyBatis的整体思路其实很简单，但是跟着源码发现，一个好的框架需要考虑的问题很多，从可扩展性、功能维护等角度考虑，如果没有一个好的设计思路会把代码设计的很乱很乱。**MyBatis充分利用了动态代理、建造模式、装饰器模式，似的他们结合在一起，让整个框架变得简单易用，其实是很难得的**。

>　**这就好比读一本书，需要先读后再度薄一样，框架的设计最开始需要考虑到各种各样的问题，然后把一个简单的思路变得复杂。然后通过合理的设计，将复杂的问题简单的设计出来，使得代码很整洁，易于维护和读，这才是一个好的框架应该有的样子**。

>　真的很感谢能有这么一个机会研究一下mybatis，并从中学到了许多。希望有朝一日，也能写出想mybatis一样的框架。
　
> 在这里很非常感谢[http://denger.iteye.com/blog/1126423/](http://denger.iteye.com/blog/1126423/)。

-------------------------------------------
>**注意**：转载请标明，转自[itboy-木小草](http://blog.csdn.net/qq_25689397/article/details/52014681)。
>**尊重原创，尊重技术。**