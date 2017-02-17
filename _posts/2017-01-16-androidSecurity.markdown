--- 

layout:     post
title:      "基于Android的ARM指令集学习笔记"
subtitle:   " \"移动安全\""
date:       2017-01-16 19:25:35
author:     "1r0nz"
header-img: "img/post-bg-unix-linux.jpg"
tags:
    - 安全研究

---

[TOC]
---  

[TOC]
#### 前言  
&nbsp;&nbsp;&nbsp;&nbsp;传统android开发倾向于将所有代码部署在android应用层，这种方式在面对软件安全威胁的情况下时比较容易破解。将重要的代码使用android ndk原生层的方式实现，由于ndk原生层代码逆向原本难度较高，使得文件无法轻易被破解。因此，理解和掌握android原生层运行机制对逆向水平的提高有比较大的帮助。  

---  

#### 原生程序生成  
&nbsp;&nbsp;&nbsp;&nbsp;原生程序的生成需要经过一下几步：预处理——>编译——>汇编——>链接（这几个文件后缀变化依次为a.c——>a.s——>a.o——>a
），其中经过第2步编译过后c代码就成了arm汇编代码。学习的重点也就是基于arm的汇编。  

---  

#### arm的一些基本概念  
1. arm微处理器共有37个32位寄存器，其中31个为通用寄存器，6个为状态寄存器。  
2. arm处理器的七种运行模式：  
* 用户模式usr：arm处理器正常的程序执行状态。  
* 快速中断模式fiq：用于高速数据传输或通道处理。  
* 外部中断模式irq：用于通用的中断处理。  
* 管理模式svc：操作系统使用的保护模式。  
* 数据访问终止模式：当数据或指令预取终止时进入该模式，可用于虚拟存储以及存储保护。  
* 系统模式sys：运行具有特权的操作系统任务。  
* 为定义指令中止模式und：当未定义的指令执行时进入该模式。  
需要注意的是，arm运行模式可以通过软件本省改变，也可以通过外部中断或者异常处理改变，不同模式所使用的寄存器也不一样，另外，除了用户模式其他六种模式都可以访问受保护的系统资源。  
3. 当运行在用户模式中，处理器可以访问的寄存器为不分组寄存器r0-r7、分组寄存器r8-r14、程序计数器r15(PC)还有状态寄存器cpsr。  
4. arm寄存器的两装工作状态：arm状态和thumb状态，其中arm状态执行32位字对齐。thumb状态执行16位字对齐。  
5. 寄存器只是临时存放处。  

---  

#### arm的程序结构  
&nbsp;&nbsp;&nbsp;&nbsp;组成：处理器架构定义、数据段、代码段和main函数。  
* __处理器架构定义__  
	.arch  armv5te  
        .fpu  softvfp  
        .eabi_attribute  20,1  
        .eabi_attribute  21,1  
        .eabi_attribute  23,3  
        .eabi_attribute  24,1  
        .eabi_attribute  25,1  
        .eabi_attribute  26,2  
        .eabi_attribute  30,6  
        .eabi_attribute  18,4	
这些指令制定了程序使用的处理器架构、协处理器类型与接口的一些属性。  
.arch 制定了arm处理器架构  
.fpu 指定了协处理器的类型  
.eabi_attribute 指定了一些接口类型  

* __段定义__  
c语言所定义的全局变量和常量都都会编译到.data下，常量数据放在.rodata的只读数据段中，代码放到.text数据段中，这里的代码才可以执行。arm指定段的格式为：  
.section name \[,"flag"\[,%type\[,flag_specific_arguments\]\]\]

