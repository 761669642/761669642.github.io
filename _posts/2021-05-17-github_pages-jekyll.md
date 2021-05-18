---
layout: post
title: "Github Pages + Jekyll搭建个人博客"
date: 2021-05-17 
description: "Jekyll+Github，搭建自己的博客"
#tag: 
---  
一直想搭个博客记录一下自己的一些学习与总结，折腾了两天终于搭起来了，当然也遇到了不少的坑，主要是jekyll环境不太好配置，版本冲突和依赖关系等等，下面就总结一下搭建的过程，作为搭建成功后的第一篇博客试试水。

## Github Pages
github pages其实就是github的一个网站托管的功能，在自己的github账号上新建一个xxx.github.io的repository(xxx一定要是你自己的github账户名)，然后将网站的相关代码和文件传上去，就可以在浏览器中通过https://xxx.github.io这个网址进行访问了。而一般我们都不会自己去写html网页，所以可以通过一些工具和模板进行自动构建，因此我们选择jekyll来进行网站的构建，有了模板之后，我们写博客时只需要专注于markdown文件的撰写，写完之后可以在本地用jelly将其依照模板构建成网页进行预览。而github pages也是支持jekyll自动构建的，我们只需要将jekyll模板以及markdown文件push到github仓库，github就会自动构建，然后发布。

## jekyll环境搭建
在windows上的jekyll环境安装还是有点复杂的，要依赖于ruby，而且后面你选择的主题可能也依赖于特定版本的软件包。我是参考了[这篇文章](https://www.jianshu.com/p/9f71e260925d)搭建jekyll环境的，但是并没有一次成功，很多步骤我在这里还是重复一遍，尽量还原我自己的安装过程。
1. 首先是安装[ruby](https://rubyinstaller.org/downloads/),因为看到其他教程有提到要安装Devkit，所以我选择的是Ruby+Devkit 2.7.3.1(x64)版本的，他会默认将ruby的可执行命令加到环境变量PATH里，安装后再**新开**一个cmd或者powershell，敲```ruby -v```验证是否装成功，如果不新开一个cmd有可能敲不出来，当时就是在旧的cmd上死活敲不出来，也为没装成功。
2. 下载[RubyGems](https://rubygems.org/pages/download),解压到自己选定的一个目录，打开cmd进入该目录，然后执行```ruby setup.rb```，我的理解就是在ruby环境下对rubygem进行编译，并安装到ruby的目录下，成功后，就能敲出```gem```命令，然后```gem install jekyll```安装jekyll。
3. 选择合适的jekyll模板，我是看了B站博主[酒石酸菌](https://www.bilibili.com/video/BV14x411t7ZU?t=1176)的教学，采用了[潘柏信](https://github.com/leopardpan/leopardpan.github.io)的模板，跟着视频里一样删除了Rakefile, Gemfile以及Gemfile.lock这三个文件，然后执行```jekyll server```在本地构建成功，就可以在本地4000端口即http://127.0.0.1:4000访问到页面，但是当我push到github上之后却发现github总是构建失败。然后我又把这三个文件加回来了，但是在本地却构建失败了。
4. 照着失败的log搜了一下，搜到了[这篇文章](https://www.5616760.com/jekyll/2020/10/10/Jekyll.html)，然后敲了下面几个命令，然后重新在本地构建成功，push到github上后也成功了，具体是哪个命令起了作用以及起了什么作用，我也搞不明白深层的原理，毕竟术业有专攻，不干前端，只是拿来用一下，就不纠结了。
   
```
$ gem install bundler jekyll

$ bundle update
```
然后后续我还换了个头像，push到github，却发现在github.io上却始终刷不出来，后来发现是浏览器自动把图像缓存了，要清除浏览器缓存才能刷出来。