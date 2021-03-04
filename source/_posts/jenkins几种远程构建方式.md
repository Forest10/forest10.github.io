---
title: jenkins几种远程构建方式
date: 2018-09-18 18:47:47
tags: [ci,jenkins]
categories: [tool,ci]
---


# 一 : 纯粹的WEB-URL调用(适合 github,阿里git)
------
### 1、首先去系统管理->管理插件里边，搜索并安装插件 Build Authorization Token Root Plugin

 
### 2、然后点击右上角，你登录的用户名，再点击设置，找到API Token，复制下来你这个用户的Token，用于远程访问Job用。

![avatar](http://image.forest10.com/pic/hexo/jenkins%E8%BF%9C%E7%A8%8B%E6%9E%84%E5%BB%BA%E7%9A%84Job%E7%9A%84Token.png)
 
### 3、找到你要触发远程构建的Job，把这个复制的Token粘贴进去，点击保存
![avatar](http://image.forest10.com/pic/hexo/jenkins%E8%BF%9C%E7%A8%8B%E6%9E%84%E5%BB%BA%E7%9A%84Job%E7%9A%84Token%E7%B2%98%E8%B4%B4.png)
 
### 4、这样你就可以用如下地址来远程触发这个Job执行了，并且不用登录系统就可以触发
http://192.168.3.11:8848/buildByToken/build?job=FlashRegistration&token=6f8ab858888888f844ab5e27a206692
http://{IP}:{端口号}/buildByToken/build?job={Job名称}&token={Token}


Job有参数，怎么在调用Job时传参数，好办，用下边的地址
http://192.168.3.11:8848/buildByToken/buildWithParameters?job=FlashRegistration&token=6f8ab85afbda2f8f844ab5e27a206692&branch=master
http://{IP}:{端口号}/buildByToken/buildWithParameters?job={Job名称}&token={Token}&{参数名}={参数值}

# 二 : WEB_HOOK(适合 github,阿里git)
### 1.阿里git使用 webhook


#### step 1
![avatar](http://image.forest10.com/pic/hexo/ali-git-webhook-first-setting.png)
#### step 2
![avatar](http://image.forest10.com/pic/hexo/ali-git-webhook-second-setting.png)

### 2.github 使用 webhook



#### step 1
![avatar](http://image.forest10.com/pic/hexo/github-webhook-first-settings.png)

#### step 2
![avatar](http://image.forest10.com/pic/hexo/github-webhook-second-settings.png)

------