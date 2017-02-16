---  

layout:     post
title:      "apk静态分析"
subtitle:   " \"认识apk文件\""
date:       2016-07-29 19:30:00
author:     "R0nzy"
header-img: "img/post-bg-unix-linux.jpg"
tags:
    - 客户端安全
---

## 一般静态分析工具思路
* 一般的工具分析思路分为两种：  
（1）先使用dex2jar将dex文件转化成.class文件的jar包再使用jd-gui将.class文件jar转化为可以直接查看的源代码。  
（2）直接使用apktool逆向dex文件，阅读smali源码和逆向分析.so文件。 

*  
