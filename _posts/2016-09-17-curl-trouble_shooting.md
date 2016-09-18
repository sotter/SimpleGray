---
layout: post
title: curl
category: default
---

####问题

执行curl，报这个
> curl: (2) Failed Initialization

####参考文章
> http://stackoverflow.com/questions/20331418/curl-2-failed-initialization

####解决方法
> wget https://curl.haxx.se/download/curl-7.50.3.tar.gz --no-check-certificate
> tar zxvf curl-7.50.3.tar.gz; cd cur-7.50.3
> ./configure --disable-shared; make ; make install;
