---
layout: post
title: 搭建github博客和jekyll本地环境 
categories: Tools
description: 网上的教程不靠谱,东拼西凑总算搞出来了,总结一下
keywords: 教程
---

　　折腾了老大功夫总算搭建了github博客还有本地的jekyll(windows),在这里总结一下

## github博客

　　github的博客搭建一开始挺懵逼的,现在弄好了感觉还是不难的,主要来说就是分成两步

 
```
1. 找一个好看的jekyll主题

2. 在你的github下创建一个repo作为博客
```


　　概括起来就是这样啦,具体再细说说

#### 找个好看的主题

　　这个东西就自行google,百度就可以了,找到了别的jekyll网页以后,去他的github把这主题fork过来,我自己是直接download下载下来贴到我的repo里面去然后commit,简直粗暴:joy::joy::joy::joy:

#### 在github下创建一个repo作为博客

　　这个有两种办法

　　一种是创建一个username.github.io的repo,这个username就是你的github用户名,这样就可以了,在master下push就好了

　　还有一种是随便建一个repo,然后在这个下面建一个分支,名为**gh-pages**,在这个分支下push就好了

#### chrome访问提示不安全

　　这个问题我遇到了,修改了一下强制使用https就可以了,具体我还不知道原因

　　在_config.yml里面url: 里的网址改成https

　　找到header.html文件,或者应该是类似名字吧,里面有一堆css就对了,在head里加上

 
```
<link rel="canonical" href="{ { site.url } }{ { page.url } }" />
```

　　这样应该就可以了,还不可以的话,google一下吧

## jekyll

　　然后是jekyll的搭建,遇到了些问题,不过后来都靠google解决了,教程真的是没讲清楚啊,当然也可能环境不一样了,在这里记录一下呗

　　首先就是下载安装一下ruby,windows下直接下载[RubyInstaller](http://railsinstaller.org/en),安装好以后好像是都好了

　　然后就是在命令行里

```
gem install jekyll github-pages
```

　　这样都安装好以后就可以去你的工程目录下开启server了,指令是


```
jekyll server
```

　　如果成功那就最好啦

　　万一失败了也不打紧,我遇到说kramdom的版本太高的提示,查了半天,原来卸载就可以了,其他同样问题的一样解决


```
gem uninstall kramdom
```

　　嗯,没有一句卸载不能解决的问题,如果有,那就两句吧

　　现在你以为就可以开起来了吗,太天真了,如果你跟我一样,会提示说SSL什么的有问题,真是扎心了啊:joy::joy::joy::joy:

　　这个也能解决,哈哈哈:laughing::laughing::laughing:

　　下个证书就好了[cacert.pem](https://curl.haxx.se/ca/cacert.pem),然后要设置一下环境变量,就在命令行下


```
set SSL_CERT_FILE=C:\...\cacert.pem
```

　　这下不出意外的话应该好了吧,可以通过localhost访问了


```
http://localhost:4000
```

