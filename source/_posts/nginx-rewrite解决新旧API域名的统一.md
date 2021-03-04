---
title: nginx rewrite解决新旧API域名的统一
date: 2018-07-08 12:00:29
tags: [backend,Nginx]
---

大多数开发者在接手新公司以前业务之后,会有这样的困扰,比如原先公司的后端API接口为old.haha.com

然后请求url为:http://old.haha.com/getUserName?userId=13, 这是一个标准的Api后端接口,如果有3-5个这样零散的接口,我相信大多数人都会疯掉.

下面说下我的最佳实践(仅限于我自己的,我经过搜索之后没有得到我想要的文章,故写此文章)

1.原始域名

old.haha.com

原始请求url

http://old.haha.com/getUserName?userId=13

2.新域名

api.haha.com

nginx统一配置
```
  server {
    listen       80;
    server_name  old.haha.com;//这里是以前的老Api域名
    rewrite ^/ http://api.haha.com$request_uri? permanent;
}
```
配置生效之后,以后请求都会被转入api.haha.com
比如

针对原始请求:http://old.haha.com/getUserName?userId=13
都会被请求转发到api.haha.com,二次请求域名为:http://api.haha.com/getUserName?userId=13
