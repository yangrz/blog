---
layout:     post
title:      "Java反序列化漏洞复现及利用方法总结"
subtitle:   " \"安全研究\""
date:       2017-12-20 15:11:25
author:     "y1r0nz"
header-img: "img/hack.jpg"
tags:
    - 安全研究
    - Java
---



#### 0x01 前言

​	java反序列漏洞被称为2016年最被低估的漏洞，其破坏力远远超过了漏洞本身造成的影响&nbsp;，自己在工作之余总结了下在手工渗透过程中验证服务是否存在java反序列化漏洞的方式。



#### 0x02 受影响的服务

首先我们需要了解java反序列化漏洞主要存在哪些服务组件中。下面列出了常见的一些服务端组件。

*  JBOSS
*  WebSphere
*  Jenkins
*  WebLogic 
*  OpenNMS

下面依次对这几种服务的进行poc验证

