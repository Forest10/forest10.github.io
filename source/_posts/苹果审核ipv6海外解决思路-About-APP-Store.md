---
title: 苹果审核ipv6海外解决思路-About-APP-Store
date: 2035-07-06 23:16:14
tags: [backend,Nginx,App-Store]
---


[原始简书文章地址](https://www.jianshu.com/p/ef45cd73a08e)


_**首先声明,一我不负责涉及你们内部服务器. 二是好好读文章,别人能过,你们也能过**_

苹果6月1日出的`IPV6`协议阻碍了国内大多数积极开发者,我司也不外乎,经过三次被拒后,遂在网上查找关于`IPV6`审核的相关事宜,怪我年少无知以为这种开源协议的东西应该是免费的,当然,我说的免费是想着看几篇成熟的`IPV6`审核文章然后自己实践,奈何几乎所有关于苹果`IPV6`审核的文章到最后不是推荐买教育网转发要不就是直接把钱交给个人然后让第三方来协助通过.

_**我实在无法想象一个仅仅靠着linux服务器外加nginx转发就能赚大钱的畸形小社会是怎样形成的,linux市值多少钱恐怕无人能说出.**_

所有文章内说的苹果`IPV6`和后台服务器没关系是错误的,至少在请求转发层面是错误的.苹果使用`IPV6-ONLY`网络进行APP测试,如果服务器端支持`IPV6`的话则可以直接请求`IPV6`所对应的服务器进而使用nginx转发至相应的API接口.如果没有`IPV6`地址的话则直接通过NAT64转化为相应的IPV4进行请求相应API. 请注意这里的重点是这个`IPV6`,,服务器不能单单支持`IPV6`即可,所谓的支持不能仅仅是打开linux服务器内相应被封印的`IPV6`相关设置然后加一个`HE`隧道(当然这么着也有通过的,但是`HE`也是基于`IPV4`,最好还是不要走这条道)而是寻找一台_**真正有全球`IPV6`地址**_的服务器,_**这才是关键中的关键.**_至于其他文章所推崇的教育网转发,一是价格太贵,二是转发这事情由他人掌控多少有点看不起自己公司后端的意思(毕竟大多数不从事后端的人的想法就是感觉后端有毛事可干,喝喝喝)..

经过第四次的痛苦实践,现将解决方案贴于文章下.希望能帮助广大开发者早日审核通过.

##### 一、购买一台海外服务器,本人使用的是搬瓦工,直通车:
[https://www.bwh1.net/aff.php?aff=10004](https://www.bwh1.net/aff.php?aff=10004)
### 近些天有朋友反映搬瓦工部分ip被墙,所以如果想要过的可能性大一点可以直接选择阿里的海外版.###
#####现在搬瓦工的区分openVz和KVM,OPENVZ支持IPV6

#####到达购买页面之后买一台差不多配置的服务器即可.洛杉矶或者弗罗里达的都行.  

购买之后:

* 点击
![](http://image.forest10.com/pic/hexo/banwagong-ipv6-addresses.png)

* 获取全球唯一的IPV6,此IPV6为真实IPV6

![](http://image.forest10.com/hexo/banwagong-true-ipv6-addresse.png)

##### 二、海外服务器端安装nginx然后配置好转发至国内自己APP及API使用的服务器端口.
1. 普通http
```
server{
listen     你的海外服务器IPV4地址:80;
listen    你的海外服务器IPV6地址 :80;
server_name  你的域名;
location /{
proxy_pass http://你的国内服务器IPV4地址:端口/;
proxy_set_header HOST $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
}
}
```

2. https
```
server{
listen    你的海外服务器IPV4地址:80;
listen    你的海外服务器IPV6地址 :80;
listen      你的海外服务器IPV4地址:443 ssl;
listen      你的海外服务器IPV6地址:443 ssl;
server_name  你的域名;
ssl_certificate /usr/develop/nginx/sslkey/XX.crt;  #(证书公钥）
ssl_certificate_key /usr/develop/nginx/sslkey/XX.key;  #(证书私钥）
ssl_session_timeout 5m;
ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
ssl_ciphers AESGCM:ALL:!DH:!EXPORT:!RC4:+HIGH:!MEDIUM:!LOW:!aNULL:!eNULL;
ssl_prefer_server_ciphers on;
add_header Strict-Transport-Security "max-age=63072000; includeSubdomains; preload";
add_header Content-Security-Policy upgrade-insecure-requests;
if ( $scheme = http ) {
rewrite ^/(.*) https://$server_name/ permanent;
}
location / {
proxy_pass http://你的国内服务器IPV4地址:端口/;
proxy_set_header HOST $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
}
}
```

##### 三、以万网为例,修改域名解析至海外服务器,我直接把IPV4和IPV6都指向了海外,后来想想直接把IPV6指向海外服务器即可,IPV4不用变,这样可以在保证APP正常使用的情况下通过审核(不过还木有测试)  

***

##### 最后,对于你们那些利用信息不对称收钱的不要误会，我不是针对谁，我是说收钱的各位都是垃圾。

![](http://image.forest10.com/common/%E5%9C%A8%E5%BA%A7%E7%9A%84%E5%90%84%E4%BD%8D%E9%83%BD%E6%98%AF%E5%9E%83%E5%9C%BE.jpg)

![](http://image.forest10.com/common/%E6%9D%8E%E7%BA%B3%E6%96%AF%E7%AB%96%E4%B8%AD%E6%8C%87.jpg)

对了,我不是前端,因为苹果说的只需要前端API层面支持而不需要后端服务器支持的狗屁话让我们前三次的审核浪费了大量时间,原先我一直没有改动后端后来在广大收钱者的感召下开始进行后端大改造.祝各位早日通过审核.

_致敬李纳斯:_
>“Software is like sex: it"s better when it"s free.”
软件就像性,免费的比花钱的好得多.                    --*Linus Torvalds*


流程图:
![1529943565971.jpg](http://image.forest10.com/common/hexo/ipv6-apple-%E5%AE%A1%E6%A0%B8%E6%B5%81%E7%A8%8B%E5%9B%BE.jpg)



如果您renwei我的文章对于您苹果审核做出了贡献,多谢支持,金额随意.不强制.

还有就是强调一下,这种知识确实不值几个钱,但是亲自动手操刀还是需要TIME的.都是混口饭吃,给点饭钱不多.


![1475036463795.jpg](http://image.forest10.com/common/money/forest10-zfb-pay_code.jpg)
