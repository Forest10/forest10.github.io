---
title: Linux下为Tomcat安装APR
date: 2019-12-10 22:24:35
tags: [LINUX,TOMCAT,APR]
---






**一、简介** 

----------

`APR：Apache Portable  Run-timelibraries，Apache可移植运行库。`
在早期的Apache版本中，应用程序本身必须能够处理各种具体操作系统平台的细节，并针对不同的平台调用不同的处理函数。随着Apache的进一步开发，Apache组织决定将这些通用的函数独立出来并发展成为一个新的项目。这样，APR的开发就从Apache中独立出来，Apache仅仅是使用APR而已。 
`Tomcat Native`：这个项目可以让 Tomcat 使用 Apache 的 apr 包来处理包括文件和网络IO操作，以提升性能。 

**二、需要安装的程序** 


----------


 1. 最新版的apr 
 2. 最新版的apr-util 
 3. tomcat-native.tar.gz 
 
前两个可以从http://apr.apache.org/下载，最后一个位于tomcat的bin目录下。 

**三、安装** 


----------

1. 安装apr 

    将最新的apr安装程序apr-1.5.2.tar.gz下载到任意一个目录下，比如/root/目录下。
     
```
    cd /root/ 
    wget http://apr.apache.org/apr-1.5.2.tar.gz 
    tar zxvf apr-1.5.2.tar.gz 
    cd apr-1.5.2/ 
    ./configure --prefix=/usr/local/apr 
    make 
    make install 
```

    
   注意，这里的prefix参数用于指定安装路径。 

2. 安装apr-util 

```
    cd /root/ 
    wget http://apr.apache.org/apr-util-1.5.4.tar.gz 
    tar zxvf apr-util-1.5.4.tar.gz 
    cd apr-util-1.5.4/ 
    ./configure --prefix=/usr/local/apr-util --with-apr=/usr/local/apr 
    make 
    make install 
    ```

3. 安装tomcat-native 

    我的tomcat目录为/usr/local/apache-tomcat-7.0.63 
    
```
    cd /usr/local/apache-tomcat-7.0.63/bin/ 
    tar zxvf tomcat-native.tar.gz 
    cd tomcat-native-1.1.33-src/jni/native/ 
    ./configure --with-apr=/usr/local/apr --with-java-home=/etc/alternatives/java_sdk_1.7.0 
    make 
    make install
```

 

**四、设置apr的环境变量** 


----------


在/etc/profile中添加以下内容 

```
    export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/apr/lib 
```

保存后使profile生效 
```
    source /etc/profile 
```

**五、验证** 


----------

```
    cd /usr/local/apache-tomcat-7.0.63/bin/ 
    ./catalina.sh run 
    ```

在第35行附近若看到如下的日志输出则表示安装成功 
```
    INFO: Loaded APR based Apache Tomcat Native library 1.1.33 using APR version 1.5.2. 
    Jan 30, 2016 4:46:57 PM org.apache.catalina.core.AprLifecycleListener lifecycleEvent 
    INFO: APR capabilities: IPv6 [true], sendfile [true], accept filters [false], random [true]. 
```

