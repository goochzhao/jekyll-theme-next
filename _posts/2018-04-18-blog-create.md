---
title: blogs create
date: 2018-04-18 14：28
categories:
- blogs
tags:
- blogs create
description: 创建博客记录第一篇！！！
---

## 个人博客的初步搭建

>开天辟地头一篇博文
>  


这几天工作忙完之后，突然想起来之前一直想搭建的博客还没落实下来，趁着现在有时间就开始了我的blog踩坑之路！！！在网上看了有好几种方法取快速搭建，推荐比较多的就两种
- Gitpage+Hexo
- Gitpage+jekyll

之前有同事用的是Hexo 的方式搭建的个人博客并买了域名，我看了篇文章说gitpage是默认用jekyll来生成静态页面的。他们两者的主要区别在于，
- Hexo需要本地环境搭建生成html页面，再进行上传，才能在线访问
- Jekyll 则只需要上传md文件即可，完全的git操作，启动本地Jekyll后只需要输入localhost：4000/projectname/ 即可访问本地调试非常方便


所以果断选用了jekyll，其实现在大多数技术博客都会选用MarkDown去书写文章，方便快捷，不用考虑各种复杂的排版问题。
![jekyll](http://img1.imgtn.bdimg.com/it/u=3660077456,970110315&fm=27&gp=0.jpg)

搭建博客现在大多数都是模板套用，所以先在这里给大家推荐一个[Jekyll主题网站](http://jekyllthemes.org/)，选择模板后直接fork到自己的GitHub上，只需要更改_config.yml文件的配置即可。

----------

## 四步完成博客搭建
>
	
>寻找自己满意的[主题模板](http://jekyllthemes.org/ "jekyll theme")(当然可以自己慢慢研究jekyll语法，博主暂时没有去研究）

这里选模板就不仔细讲了，我选用的是[NexT主题](https://github.com/Simpleyyt/jekyll-theme-next "homepage")
github地址：[https://github.com/Simpleyyt/jekyll-theme-next](https://github.com/Simpleyyt/jekyll-theme-next "github地址")
  

![主题样式](http://iissnan.com/nexus/next/desktop-preview.png)

选择好主题样式后，直接fork到自己的GitHub中即可。

----------

>个人相关配置

首先去你的仓库下的setting中修改你的reponame
再这里主要是_config.yml文件的修改，其实如果简单修改的话也就没几处：
- url：If your site is put in a subdirectory, set url as "http://yoursite.com"
- baseurl : baseurl as '/child'就是你的仓库名字
- 再就是一些个性化配置的东西了，像什么代码高亮，markdown解析器（**gitpage 默认是kramdown**，语法和原生的不太一样）

----------

>本地jekyll安装（博主用的是window7）

#### Mac ####
使用mac的就非常简单了只需要几个命令即可由于mac自带ruby所以就简单多了。
- ruby -v
- gem install bundler jekyll
- jekyll -v

如果出现对应的jekyll版本，说明安装完成

#### windows
- 先去安装ruby 地址：https://rubyinstaller.org/downloads/ 选择下载 **WITH DEVKEI** 直接安装即可
- 安装完成后，在cmd中 ruby -v 查看版本
- 执行命令 gem update 
- 执行 gem install jekyll
- 安装完成后 执行 jekyll -v 
- 安装完成

----------

>本地调试博客

- 使用git将fork下来的项目clone到本地
- 在cmd中打开对应的clone的项目目录下执行***jekyll serve***命令（如果项目中没有 Gemfile和Gemfile.lock文件，否则执行 ***bundle exec jekyll serve***,该命令可能会报错，他会提示你先执行 ***bundle install*** 和***bundle update*** ，然后再执行 ***bundle execjekyll serve***）如果报错，可能是缺少文件，执行 ***gem install xxx***即可缺少什么安装什么即可。
- 此时本地服务就启动了，输入cmd中显示的网址**localhost：4000/xxx** 即可


----------

>发布文章

	|-- _config.yml
    |-- _includes
    |-- _layouts
    |   |-- default.html
    	|   `-- post.html
    |-- _posts
    	|   |-- 2007-10-29-why-every-programmer-should-play-nethack.textile
		|	|-- 2009-04-26-barcamp-boston-4-roundup.textile
    |-- _site
	|-- index.html

- **_config.yml**   
配置文件，用来定义你想要的效果，设置之后就不用关心了
- **_includes**  
可以用来存放一些小的可复用的模块，方便通过{ % include file.ext %}（去掉前两个{中或者{与%中的空格，下同）灵活的调用。这条命令会调用_includes/file.ext文件。
- **_layout**  
这是模板文件存放的位置。模板需要通过YAML front matter来定义，后面会讲到，{ { content }}标记用来将数据插入到这些模板中来。
- **_posts**  
你的动态内容，一般来说就是你的博客正文存放的文件夹。他的命名有严格的规定，必须是2012-02-22-artical-title.md这样的形式，MARKUP是你所使用标记语言的文件后缀名，根据_config.yml中设定的链接规则，可以根据你的文件名灵活调整，文章的日期和标记语言后缀与文章的标题的独立的。
- **_site**  
这个是Jekyll生成的最终的文档，不用去关心。最好把他放在你的.gitignore文件中忽略它。

书写的md文件都会有一个头部定义比如：

	

	---
	layout:     post
	title:      title
	category: blog
	description: description
	published: true # default true
	---

接下来直接写文章即可。  
