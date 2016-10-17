---
title: 大数据——Hadoop中的RPC
date:  2016-04-03
categories:  大数据
keywords:  [Hadoop,RPC]
tags: 
	- Hadoop
	- RPC
---

> 前几天在学习Hadoop的时候，学到了他的RPC机制，即RPC（Remote Procedure Call Protocol）——远程过程调用协议。以便日后的学习，在此记录一下学习心得。

---------
  在我看来，RPC的本质其实和JMS消息队列，WebService一样。都是两个主机，两个线程之间通信问题，不一样的就是通信的内容不同而已。
  RPC是通过动态代理，利用socket通信，实现方法体的异地调用。而JMS和WebService一个是实现消息的异地发送，一个是实现服务的异地发送。其实这些都是异地信息，只是信息的内容不一样而已。我们完全可以用JMS发送要调用打方法，返回方法的调用结果，实现和RPC同样的效果。为什么不这么做，只是他们都在各自领域有所长而已。

##hadoop中的简单应用：
```
	/**
	 * 利用RPC，实现hadoop的下载
	 * @throws IllegalArgumentException
	 * @throws IOException
	 */
	@Test
	public void fileDownLoad() throws IllegalArgumentException, IOException {
		// 设置配置信息
		Configuration conf = new Configuration();
		conf.set("fs.defaultFS", "hdfs://muxiaocao:9000/");
		conf.set("dfs.replication", "1");
		/**
		 *其实在得到Filesystem的时候就用到了RPC，
		 * 这里的fs其实是FileSystem的实现类DistributedFileSystem，其中有个很重要的成员变量是DFSClient
		 * 
		 * 而DFSClient中有个成员变量就是ClientProtocol namenode。
		 * 这就是RPC的协议类接口，远程过程调用也是调用了他的远程实现方法。
		 * 
		 * DFSClient中还有个SocketFactory，实现Socket传输信息。
		 */
		FileSystem fs = FileSystem.get(conf);
		/**
		 * 这里调用的FileSystem中的open方法，其实就是利用RPC调用的hdfs中的open实现方法，
		 * 从而实现将hdfs中namenode的元数据，发送给客户端，
		 * 客户端利用元数据中的信息，在到hdfs中的datanode中真正的下载所要的数据
		 */
		FSDataInputStream is = fs.open(new Path("hdfs://muxiaocao:9000/filefromlocal.xml"));
		FileOutputStream os = new FileOutputStream(new File("/home/muxiaocao/myFile/filefromhdfs.xml"));
		IOUtils.copy(is, os);
	}
```

  这样做的好处在于，可以高内聚、低耦合，完全把不相干的方法抽取出来放在其他主机上，而我们要用的时候，只是通知，然后接收结果信息。具体的方法实现，其实是在其他主机上的。RPC更是用上了动态代理，利用RpcProtocol起到通信协议的作用，在面向接口编程的方式下，让开发者也感觉是在本地调用方法一样，更不用说是使用者了。这样就近乎完美的达到了高内聚、低耦合的效果。
  我不禁回想自己学习IT技术的过程中，先是面向过程打C语言，把一行行代码，一个个方法堆砌成程序。再到基于过程的C++，利用结构体将一个个结构类似的代码块抽取出来。再到后来面向对象的Java，将对象的思想引用进来，把高内聚的变量和方法放在一起，实现对内隐藏细节，对外暴露接口。似乎语言或者IT的发展演进，就是朝着这个方向发展的。而RPC的思想，更是把这个思想提升到了分布式上，让低耦合的方法变成了异地的了。
  
  我不禁大胆的想象了下，我们在做Web应用的时候，如果访问量很大，就像淘宝的双十一一样，我们完全可以利用这种思想，实现分布式构建的Web应用。具体来说，如果考虑到下单可能会并发度很高，服务器压力过大。我们就可以将下单模块集成出来放在多个服务器上，用一个或者几个服务器做controller，当下单请求过来的时候，考虑到service服务器的负载均衡进行任务的分派，当服务器压力到达预订阀值也可以开启其他服务器参与下单操作，实现服务器的横向扩展。而服务器间的通讯，就可以采用RPC的模式。

##RPC的web应用
![RPC的web应用](http://img.blog.csdn.net/20160403211108782)
  另外，我们现在很多的应用都支持第三方登录，其实这就应该是RPC思想的应用，只需要第三方给出对应的接口协议和访问服务器，我们就可以自己实现第三方的登录。
###代码简单实现###

**登录提供方服务器代码**
```
package com.nanda.hadoop.muxiaocao.RPC;

/**
 * 登录接口
 * @author muxiaocao
 *
 */
public interface LoginServiceInterface {
	public static final long versionID = 1L;
	public String login(String uname,String passwd);
}
```

```
package com.nanda.hadoop.muxiaocao.RPC;
/**
 * 登录实现类
 * @author muxiaocao
 *
 */
public class LoginServiceImpl implements LoginServiceInterface{

	public String login(String uname, String passwd) {
		
		return uname + " 登陆成功！！！ ";
	}
	
}
```

```
package com.nanda.hadoop.muxiaocao.RPC;

import java.io.IOException;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.ipc.RPC;
import org.apache.hadoop.ipc.RPC.Builder;
import org.apache.hadoop.ipc.RPC.Server;
/**
 * RPC服务
 * @author muxiaocao
 *
 */
public class MyRPCLoginService {
	public static void main(String[] args) throws Exception, IOException {
		Builder builder = new RPC.Builder(new Configuration());
		builder.setPort(10000);
		builder.setBindAddress("muxiaocao");
		builder.setInstance(new LoginServiceImpl());
		builder.setProtocol(LoginServiceInterface.class);
		Server server = builder.build();
		server.start();
	}
}
```
**登录调用方服务器代码**

```
package com.nanda.hadoop.muxiaocao.RPC.controller;

import java.io.IOException;
import java.net.InetSocketAddress;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.ipc.RPC;

import com.nanda.hadoop.muxiaocao.RPC.LoginServiceInterface;

public class MyRPCController {
	public static void main(String[] args) throws IOException {
		
		LoginServiceInterface proxy = RPC.getProxy(LoginServiceInterface.class, 1L, new InetSocketAddress(
				"muxiaocao", 10000), new Configuration());
		
		System.out.println(proxy.login("muxiaocao", "123"));
	}
}
```
###运行效果
![这里写图片描述](http://img.blog.csdn.net/20160503102221966)
>右边是被调用端，开启服务后，为阻塞状态，一直监听10000端口。
>左边是调用端，通过10000端口，调用接口的同时实现动态代理加socket网络通信，实现访问被调用端，并得到调用结果。整个过程丝毫没有远程调用的感觉，就像调用本地方法一样。

  **总结**：技术其实就在那里，多不多少不少，只要我们思维足够开阔，就能够把一些简单的技术组合成一种让人耳目一新的技术。而在中国其实从不缺乏技术人才，而是缺乏技术的想象力，只有我们将已有的技术进行大胆的想象和组合，就能让我们的技术流行起来，让别人追赶我们的思维，而不是我们一直再用双手追赶别人的思维，这样是永远赶不上的。
  -------------------------------------------
  >**注意**：转载请标明，转自[itboy-木小草](http://muxiaocao.cn)。
>**尊重原创，尊重技术。**
