---
layout: post
title: macchanger生成随机mac地址
categories: Linux
description: macchanger生成随机mac地址
keywords: Linux 
---

　　又有好一段时间没有更新了，之前因为很多琐事忙一下，后来因为懒惰，唉。。。

　　这里更新一个挺好的工具，macchanger，可以修改Linux的mac地址，挺好使的一个工具，至于说应用场景，这个就自由发挥了

　　学到的方法来自这里[Random Mac Address on Start Up with Ubuntu](http://sumo.ly/hcLa)

　　首先是apt-get获取macchanger，其实安装完以后就会提示你是否要每次联网时候都更换mac地址，我使用时遇到问题，我的机器是用固定IP的，自动更换地址以后固定IP不见了，所以就没有使用这个自动更换，然后就是按照上面的方法，开机时自启动修改mac地址。

　　新建一个启机脚本`sudo gedit /etc/init.d/macchanger`，内容如下：

```
#!/bin/bash

# Spoof the mac addresses
/usr/bin/macchanger -r eth0

```

　　这个`-r`是随机生成一个mac地址，也可以用`-e`，是保留厂家序列号

　　然后就是修改一下脚本的权限，加个执行权限，直接上777就好了

```
sudo chmod 777 /etc/init.d/macchanger
```

　　最后让这个脚本开机执行

```
sudo update-rc.d macchanger defaults 10
```

　　好啦，重启机器就可以看到新地址啦~

　　许久没更新，今天先这样意思一下，近期看看陆续补充做些更新吧，暂时都是一些小工具的使用先