---
layout: post
title: Mysql 常用命令
category: default
---

***mysql为了passwork和Port简写的冲突，端口号是P，passwd是小p example:***

> $ mysql -utest -ptest123 -h127.0.0.1 -P3306 -Dimcloud -e'select * from log limit 10'

同时需要注意的： -p 中间是不能有空格的，其他是可以的，看下面的manual pages:

> The password to use when connecting to the server. If you use the short option form (-p), you cannot have a space between the option and the password. If you omit the password value following the --password or -p option on the command line, mysql prompts for one.


