---
layout: post
title: SublimeText3 支持执行运行JS，记录下过程；
category: default
---

#MAC SublimeText3 run JavaScript

SublimeText3 支持执行运行JS，记录下过程；

1. MAC JS解析引擎
--

***MAC已经自带js的控制台:***

> /System/Library/Frameworks/JavaScriptCore.framework/Versions/A/Resources/jsc

***MAC安装node***
   
> sudo brew install node

备注：这个地方突然发一个大坑，执行brew报权限问题，改为root用户执行依旧不行；

解决办法：
http://stackoverflow.com/questions/10424872/brew-install-mongodb-error-cowardly-refusing-to-sudo-brew-install-mac-osx-lio

***验证环境***
  node test.js 查看执行结果；

2. Sublime支持
--

参考文档：
http://www.wikihow.com/Create-a-Javascript-Console-in-Sublime-Text
http://www.sublimetext.com/docs/build

***1. 创建build sytem配置文件***

> Tools->Build System->New Build System

创建一个JSC.sublime-build文件，文件内容如下：

```
{
	"cmd": ["/System/Library/Frameworks/JavaScriptCore.framework/Versions/A/Resources/jsc", "$file"],
	"selector": "source.js"
}
```

***2. 保存JSC.sublime-build***

放到Sublime 的安装包的User中，如：
/Users/tianshan/Library/Application Support/Sublime Text 3/Packages/User

因为自己的sublime默认保存路径不是这，这个问题困扰了半天.....

***3。 在tools BuildSystem中选择建立好的JSC***



