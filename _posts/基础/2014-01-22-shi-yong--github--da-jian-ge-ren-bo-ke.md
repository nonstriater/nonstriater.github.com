---
layout: post
title: "使用 github 搭建个人博客"
description: ""
category: "基础"
tags: iOS github jekyll
---


>by @nonstriater  nonstriater.github.com

![head photo](http://dc625.4shared.com/img/yZ94xUyZ/s7/matrix-geek-blue-binary-code-4.jpg)  

### 环境

mac OSX mavericks    
git + github + markdown + jekyll

<br/>
### jekyll-bootstrap 

jekyll-bootstrap是一个搭建好的模板框架，可以用来快速搭建个人博客。

```sh
    git clone https://github.com/plusjade/jekyll-bootstrap.git USERNAME.github.com
    cd USERNAME.github.com
    git remote set-url origin git@github.com:USERNAME/USERNAME.github.com.git
    git push origin master
```

需要等一会儿，github才能配置好你的博客环境，这时候你可以先去新建一篇文章（你可以访问USERNAME.github.com 这个域名看你的博客环境是否搭建完成）。


<br/>
### jekyll 本地安装 & 配置

Jekyll是用Ruby编写的,静态网页生成引擎。通过jekyll生成静态站点后，就可以提交到github，通过github pages生成博客。

```sh
cd USERNAME.github.com
sudo gem install jekyll
```

过程有点慢，没有log，需要等一会儿

```sh
jekyll server -w
```

本地开启jekyll，可以本地预览markdown文章。


<br/>
### 创建post

```sh
rake post title="Hello World"

// 创建文章并添加tags
rake post title="github 搭建个人博客" tags="github jekyll-bootstrap"
```

rake是用ruby写的构建语言,类似与linux中得make工具。

使用rake post创建一篇文章，然后用git来add,commit,push文章，以代码的方式来管理你的文章。

jekyll-bootstrap 标题不支持中文，分类也不支持

在本机安装Hz2Py gem install Hz2Py


#### 如何查看创建的所有文章
在Rakefile添加这一功能：[TODO:]

#### 如何修改文章的标题

#### 删除一篇文章

直接删掉_posts目录下得文章即可。
```sh
    git rm _posts/xxx.md
```



<br/>
### 创建page
```sh
    $ rake page name="about.md"     or
    $ rake page name="pages/about.md"
```



<br/>  
### 个性化定制

<br/>
#### 定制域名

1）  在主目录下，新建一个CNAME文件，里面写你的域名.如：“http://blog.xxx.com”  
2)   在DNS服务商后台将你的域名指向 **204.232.175.78** ，可能要过一段时间，就可以通过域名访问你的github博客了。

<br/>
#### 个性化标签图标

主目录放一个favicon.ico图标就可以了。

<br/>
####主题themes

```sh
    $ rake theme:install git="https://github.com/jekyllbootstrap/theme-the-program.git"
    $ rake theme:switch name="the-program"
```

<http://themes.jekyllbootstrap.com/> 提供了几个模板，你可以选择一个喜欢的安装。



<br/>
####安装各种插件
* 代码高亮
* 分享 jiathis
* 统计CNZZ
* 搜索

<br/>
### 问题

1） push文章失败后，会收到github发来的邮件，如何解决markdown error问题？

有语法错误。


2）"Liquid Exception: Included file 'JB/setup' not found in '_includes' directory in 2013-12-31-github-.md"

要在根目录下运行以下命令：
```sh
    jekyll server -w
```


"incompatible character encodings: UTF-8 and ASCII-8BIT. Use --trace to view backtrace"


3）jekyll bootstrap标题不支持中文，并且也不支持文章分类

修改Rakefile文件。参考<http://yochuo.com/tools/2013/10/24/githubjekyll-da-jian-bo-ke-liu-shui-zhang/>


<br/>
### 参考

[markdown语法](https://github.com/LearnShare/Learning-Markdown)  
[jekyll-bootstrap官方安装手册](http://jekyllbootstrap.com/usage/jekyll-quick-start.html)      
[jekyll-bootstrap使用主题](http://jekyllbootstrap.com/usage/jekyll-theming.html)

