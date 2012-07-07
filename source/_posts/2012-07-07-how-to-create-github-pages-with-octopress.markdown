---
layout: post
title: "搭建基于octopress的GitHub博客"
date: 2012-07-07 22:47
comments: true
categories:
---
一直对[GitHub](https://github.com/)上[markdown](http://en.wikipedia.org/wiki/Markdown)语法快速成文的写作方式很感兴趣，这种方式入门略有门槛，但是习惯了写作效率会很高。项目稍微闲了下来，抽点空在GitHub上搭个基于[octopress](http://octopress.org/)的博客空间，以后一些开发上的心得会在这里分享。

这篇博客先记录一下搭建GitHub博客的流程。

##配置GitHub账号:
首先你当然需要一个[GitHub](https://github.com/)的账号，如果你没有，先去申请一个。

需要在本地生成一个密钥，然后上传到GitHub。

首先在终端中输入
	[[ -f ~/.ssh/id_rsa.pub ]] || ssh-keygen -t rsa
生成密钥之后，将生成的信息复制下来
	[[ -f ~/.ssh/id_rsa.pub ]] && cat ~/.ssh/id_rsa.pub
在浏览器中打开页面 [https://github.com/account/ssh](https://github.com/account/ssh)，点击“Add another public key” 添加密钥，粘贴之前复制的信息，然后点击“Add Key”即可。这里需要注意的是：Title不需要填写内容。

##创建GitHub Pages Repo
如果你的GitHub用户名是username，那个就创建一个名称为”username.github.com”的repo，这个repo就是你的GitHub Pages Repo，更多信息可以参考[这里](http://pages.github.com/)。

##配置Octopress个人博客
首先你需要Git克隆一个octopress的仓库<br>
 	git clone git://github.com/imathis/octopress.git octopress
 	cd octopress
 	
安装相应的gem
	bundle update
在此之前，如果你没有安装RVM，请参照octopress官方的[指引](http://octopress.org/docs/setup/rvm/)安装。
生成模版文件
	rake install
 现在就可以开始分发到GitHub上了。这里要先保证你已经配置好了GitHub账号，以及创建了GitHub Pages Repo
	cd octopress
	git remove add jackyche git@github.com:jackyche/jackyche.github.com.git
	
##编写测试博客
你可以用以下命令新增博客
	rake new_post["new test blog"]
这里"new test blog"是博客的默认标题，它其实干的活是在source/_posts/目录下生成一个名字叫2012-07-07-new-test-blog的markdown模版文件（这里的时间是可变的）。所以你大概可以猜到不能用中文来做new_post的参数了，显示的标题可以在markdown文件里面修改。

你需要稍微了解一下markdown的语法，然后编写完成你的博客，接下来就可以生成静态站点了。
	rake generate
然后配置octopress与GitHub的连接
	rake setup_github_pages
这里需要输入你的GitHub Pages Repo地址，格式如下
	git@github.com:jackyche/jackyche.github.com.git
最后把你的博客分发到GitHub上去
	rake deploy
嗯嗯~~现在就可以尝试浏览你的博客了
	http://jackyche.github.com

##保存写作的markdown源文件
你的写作源当然很重要啊，所以需要创建一个新的GitHub的source分支，来保存你的创作源
	git add .
	git commit -m "save markdown source"
	git push jackyche source
	
OK，到此为止，一个基于octopress的GitHub博客空间就创建完毕了。