---
title: 妙用travis-ci实现gitee和github代码同步
date: 2019-11-26 18:47:47
tags: [gitee,oschina,github,travis-ci]
categories: [tool,ci]
cover: http://public-img.forest10.com/oschina/gitee-log.png
---



7月底，GitHub 断供伊朗、克里米亚.
11月伊始，GitLab 公开表示拒绝为中国&俄国人提供 offer.
.....

种种迹象在悄无声息的告诉我们是时候找到一个合适的国产化git工具了，在寻找国产化git时候,入选的有阿里git和腾讯git.毕竟都是大厂.后来发现他们不支持github项目迁移.所以就放弃了.
这时候,**gitee** 是oschina给出的答案 —— 为记录自由思想和分享知识提供更专业的工具。 您可以使用gitee：

> * 支持 Git 和 SVN
> * 免费的私有仓库
> * 已有超过 350 万的开发者选择码云

码云提供了同步GitHub的选项,但是又带来了另一个问题.码云的同步需要手工动作才可以.并且每次gitHub有更新时候还需要手工同步一次.......

正所谓懒人改变世界.是时候请出今天的大杀器了--
## travis-ci
------

## 什么是 travis-ci

Travis CI是国外新兴的开源持续集成构建项目，支持Github项目。使用十分方便。

 1. 使用Github账号登录[Travis CI][1]；
 2. 登录之后会自动同步Github项目，选择需要使用Travis CI的项目
 3. 在项目的根目录新增.travis.yml文件，内容如下：
```
#指定运行环境
language: node_js
#指定nodejs版本，可以指定多个
node_js:
  - 0.12.5

#运行的脚本命令
script:
  - npm run ci

#指定分支，只有指定的分支提交时才会运行脚本
branches:
  only:
    - master
```
更多语法请看[这里][2]。使用起来非常方便，这样当你每次向github push代码的时候，Travis CI就会自动运行.travis.yml里面的script。自动进行编译以及运行单测。

### 1. 首先创建一个github工程,一个Gitee工程

- [ ] 名称任意--比如travis-ci
- [x] 必须是public的,因为org后缀的travis只支持public.如需private需到com


### 2. 开始操作

 1. 登录之后选择头像的settings
![][3]

 2. 找到自己创建的测试工程,打开按钮
 ![][4]

 3. 进入设置,加入一些私密值设置,比如access_token(gitee的个人令牌需要提前创建,在设置里面的私人令牌)
![][5]
 
### 3. 将测试github工程clone至本地,在跟路径创建 .travis.yml

```
install:
  - git clone https://${GITHUB_REF} githubTmp
  # GITEE,需要注意的是gitee的对于command进行Pull和Push的处理跟GitHub 不一样.需要在token前加入userName作为命名空间限制.
  # userName需要置换成自己的github用户名
  - git clone https://userName:${GITEE_TOKEN}@${GITEE_REF}  giteeTmp
script:
  - cd ./githubTmp
  - git pull
  - cd ../giteeTmp
  - git pull
  # 把github的文件全量复制到gitee中
  - cp -R  ../githubTmp/* ./
  # 设置用户名mail
  - git config user.name "userName"
  - git config user.email "userName@gmail.com"
  # 进入gitee 开始操作
  - git add .
  - git commit -m "Sync From GitHub By TravisCI With Build $TRAVIS_BUILD_NUMBER"
  # GITEE
  - git push

env:
  global:
    - GITHUB_REF: github.com/userName/Travis-CI-.git
    # gitee
    - GITEE_REF: gitee.com/userName/Travis-CI.git

```

### 4. 可以更改下ReadMe等文件.完毕之后push


### 5. 点击history,会看到最近的构建
![][6]

### 6. 查看详情
![][7]

### 7. 去码云看最新的push
![妈妈再也不用担心我的github哪天被墙掉啦][8]

### 8. 更详细的说明

 - 坚信懒人改变世界.能用计算机完成的任务.坚决不要手工搞定.
 - 有兴趣的朋友欢迎关注我的[Blog][9],还在奋笔疾书中,只分享实验过能用的经验.
 - 以上过程只是简单的使用了Travis-CI进行了自动化同步github和gitee代码---只针对master.全分支同步还在探索中.相信也太难不到哪儿去.
 - gitee对于command access_token并没有详细说明 (我一直在模拟github的提交模式: https://access_token@git_url), 导致前几次的同步一直失败.这点希望码云早点出关于access_token详细文档
 - 感谢[七牛云](https://portal.qiniu.com/signup?code=1hkqx38g57yvm)提供的[免费图床](https://portal.qiniu.com/signup?code=1hkqx38g57yvm)以及[CDN](https://portal.qiniu.com/signup?code=1hkqx38g57yvm)支持
 - 感谢[码云](https://gitee.com/Forest10)提供的[免费git](https://gitee.com/Forest10)以及[博客收录](https://gitee.com/Forest10)支持


------
  


  [1]: https://travis-ci.org/
  [2]: https://docs.travis-ci.org/
  [3]: http://public-img.forest10.com/oschina/travis-ci-settings-01.png
  [4]: http://public-img.forest10.com/oschina/travis-ci-settings-02.png
  [5]: http://public-img.forest10.com/oschina/travis-ci-settings-03.png
  [6]: http://public-img.forest10.com/oschina/travis-ci-build-01.png
  [7]: http://public-img.forest10.com/oschina/travis-ci-build-02.png
  [8]: http://public-img.forest10.com/oschina/travis-ci-build-04.png
  [9]: http://blog.forest10.com/