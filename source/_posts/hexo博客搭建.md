---
title: Hello Hexo
date:  2016-08-26
categories:  Git
keywords:  [博客搭建, Git使用, Hexo]
tags: 
	- 博客搭建
	- Git使用
	- Hexo
---

> 准备阶段：
	
	1. 安装git客户端
	2. 安装node
	3. 准备好属于自己的域名，最好有自己的服务器
	4. 申请github账号

# 安装和配置Hexo

1. 打开Git-bash或者cmd，输入
```
npm install -g hexo-cli
```
2. 本地建站
	
	选择一个文件夹下，使用如下命令：
	`hexo init`
	`npm install`
	如果hexo安装成功，则会在该目录下生成如下的目录结构
	![](http://i.imgur.com/YxvWsWf.png)
	
3. 本地测试

	`hexo s`
	提示：
	![](http://i.imgur.com/7qp42fz.png)
	接下来就可以使用浏览器访问啦！不过现在是本地的，如果想放到万维网并使用自己的域名就需要配合github了。
# 主题更换
> clone主题代码到git上

```
git clone https://..
```

> 修改config.yml配置文件

	将里面的theme: 的值改为对应的主题

> 提交代码

```
git add .
commit -m "备注"
git push origin master
```

> 发布和测试

```
hexo d -g //会出现很多文件
hexo s
```

# 使用Github Pages

1. 在自己的github上创建名为xxx.github.io的repository
2. 设置repository的pages，具体操作去github官网查看（可以设置上自己的域名）
3. 创建CNMAE文件，写入自己的域名
4. 将本地hexo根目录下的`_config.yml`中的如下位置，填写自己的信息

	![](http://i.imgur.com/kDYceNd.png) 
5. 将上一步的hexo代码上传到repository

	`git status` 查看状态

	`git add .`

	`git commit -m "备注"`

	`git pull origin master`

	`git push origin master`
4. 发布
	
	`hexo d -g`
> 这时候，就可以用repository名，也就是xxx.github.io进行访问了。

# 域名解析设置
如下图：
![](http://i.imgur.com/tuiyoWD.png)

设置好后，大约十分钟，便可用域名访问啦！

# 发表文章

> 发表文章操作非常简单，在网站存放的根目录打开git bash，输入

	hexo n "name of the new post"

> 回车后，在source文件夹下的_post文件夹下，可以看到新建了一个name of the new post.md的文件，打开填写文件头信息
![](http://i.imgur.com/5jtBn7m.png)

> 可以给文章贴上相应的tags，如有多个则按照如下格式

`[tag1, tag2, tag3, ...]`

>　在- - -下方添加正文内容即可，注意需要使用markdown语法进行书写。在这里有关于Markdown语法的简单说明。推荐使用CMDmarkdown进行md文件的编辑工作。
文章撰写完成后保存，输入

	hexo g -d

> 即可生成新网站，并且同步Github上的网站内容。

-------------------------------------------
>**注意**：转载请标明，转自[itboy-木小草](http://muxiaocao.cn)。
>**尊重原创，尊重技术。**
