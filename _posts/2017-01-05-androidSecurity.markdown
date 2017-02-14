--- 

layout:     post
title:      "根据smali源码静态分析android程序"
subtitle:   " \"移动安全\""
date:       2017-01-05 17:07:35
author:     "1r0nz"
header-img: "img/post-bg-2015.jpg"
tags:
    - 安全研究

---


--- 

#### 一、定位关键代码的六种方法  
1. 信息反馈法:说白了就是根据程序运行的反馈信息查找突破口（比如登录中验证账户使用的一些警告提示一般会放在string.xml文件中或者硬编码到程序中）  
2. 特征函数法:有些函数比较常见，容易根据关键函数寻找突破口，在android sdk中有很多api函数完成这种工作，如Toast等  
3. 顺序查看法:少废话直接通读代码  
4. 代码注入法:在需要调试的地方家Log输出日志，在ddms中查看程序运行。有点想java中的print暴力调试。
5. 栈跟踪法:输出运行时的栈跟踪信息，然后查看栈上的函数调用序列来理解方法的执行流程。  
6. Method Profiling: 主要用于热点分析与性能优化。

---  

#### 二、smali文件格式  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在用apktool反编译apk文件后，会在反编译工程目录下生成一个smali文件夹，里面有很多smali文件了。下面是一些smali的常用语法格式  
.class<访问权限>[修饰关键字]<类名>:指定当前的类名  
.super<父类名>:指定了当前类的父类  
.source<源文件名>:指定了当前类的源文件名  
#####静态字段声明#####   
\#static fields   
<font size=2 color=#000000>静态字段声明</font>  
\#static fields  
>>>>>>> f27b3b5d641baf7c24c899676cb7212e850042e4
格式:.field<访问权限>static[修饰关键字]<字段名>:<字段类型>：字段  
<font size=2 color=#000000>实例字段声明</font>  
\#instance fields  
.field private btnAnno:Landroid/widget/Button;&nbsp;&nbsp;  
<font size=2 color=#000000>方法的声明</font>  <br>
\#direct methods  
.method<访问权限>[修饰关键字]<方法原型>  
&nbsp;&nbsp;&nbsp;&nbsp;<.locals>&nbsp;&nbsp;//指定使用局部变量的个数  
&nbsp;&nbsp;&nbsp;&nbsp;[parameter]&nbsp;&nbsp;//指定了方法的参数  
&nbsp;&nbsp;&nbsp;&nbsp;[.prologue]&nbsp;&nbsp;//指定了代码的开始处  
&nbsp;&nbsp;&nbsp;&nbsp;[.line]&nbsp;&nbsp;//该处指令在源代码中的行号  
&nbsp;&nbsp;&nbsp;&nbsp;<代码体>  
.end method  
<font size=2 color=#000000>接口的声明</font>  
\#interface  
.implements<接口名>  
<font size=2 color=#000000>注释的声明</font>  
\#annotations
.annotation[注释属性]<注解类名>  
&nbsp;&nbsp;&nbsp;&nbsp;[注解字段=值]  
.end annotation  

---  

#### 三、android程序中的类  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;介绍完了smali文件格式，再来看看android程序反汇编后生成了哪些smali文件。  
<font size=2 color=#000000>内部类</font>  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;java语言允许在一个类的内部定义另一个类，这种情况称为内部类。内部类也分很多情况，如成员内部类、静态内部类、方法内部类、匿名内部类。baksmali在反编译dex文件的时候，会为每个类单独生成一个smali文件，内部类作为一个独立的类，也拥有自己独立的smali文件，文件形式为“[外部类]$[内部类].smali”,例如下面的类。  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;class&nbsp;&nbsp;Outer{  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;class Inner{}  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}  
baksmali反编译后会生成Outer.smali和Outer$Inner.smali两个文件。  
注：针对this$X，是内部类自动保留的一个指向所在外部类的引用。左边的this表示为父类的引用，右边的数值0表示引用的层数。如下面的类。  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;public&nbsp;&nbsp;class&nbsp;&nbsp;Outer{  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;public class FirstInner{  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;public class SecondInner{  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}  
每往里面一层右边的数值就加1,如SecondInner类访问FirstInner类的引用为this$1,另外，synthetic属性表明它们是被编译器合成的、虚构的，并没有声明该字段。  
在Dalvik虚拟机中，对于一个非静态的方法而言，会隐含的使用p0寄存器当作类的this引用。  
<font size=2 color=#000000>监听器</font>  
监听器在android应用开发中大量使用，如Button点击事件响应onClickListener、Button长按事件响应OnLongClickListener、ListView列表项点击事件OnItemSelectedListener等。监听器只使用一次，所以多用匿名内部类的形式来实现。  
btn.setOnClickListener(new android.view.View.OnClickListener(){  
&nbsp;&nbsp;&nbsp;&nbsp;@Override  
&nbsp;&nbsp;&nbsp;&nbsp;public void onClick(View v){  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;......  
&nbsp;&nbsp;&nbsp;&nbsp;}  
});  
监听器的实质是接口，只需实现接口中相应的方法即可，如上例中实现的onClick方法。  
<font size=2 color=#000000>注解类</font>  
/#annotations  
.annotation system Ldalvik/annotation/MemberClasses;  
&nbsp;&nbsp;value={  
&nbsp;&nbsp;&nbsp;&nbsp;Lcom/droider/crackme/MainActivity$SNCChecker;  
&nbsp;&nbsp;}  
.end annotation  
MemberClasses注解是编译时自动加上的。这是一个系统注解，作用是为父类提供一个MemberClasses列表。MemberClasses即子类成员集合，通俗的讲就是一个内部类列表。  
EnclosingMethod注解用来说明整个MainActivity$1类的作用范围，其中的Method表明它作用于一个方法，而注解的value表明它位于MainActivity的onCreate()方法中。
EnclosingClass注解表明MainActivity$SNChecker作用于一个类，注解的value表明这个类是MainActivity。在EnclosingClass注解的下面是InnerClass，它表明自身是一个内部类  
如果注解类在声明时使用了默认值，那么程序中会使用到AnnotationDefault注解。  
Signature注解用于验证方法的签名。  
方法的声明中使用throws关键字抛出异常，则会生成相应的Throws注解。  
<font size=2 color=#000000>自动生成的类</font>  
使用AndroidSDK默认生成的工程会自动添加一些类。这些类在程序发布后会仍然保留在apk文件中。  
R类：R类中资源类都会独立生成一个类文件，在反编译出的代码中，可以发现R.smali、R$attr.smali、R$dimen.smali、R$drawable.smali、R$id.smali等  

---  

#### 阅读smali代码结构  
<font size=2 color=#000000>循环语句</font>  
首先要知道常见的四种循环语句：迭代其循环、for循环、while循环、do while循环。  
:goto_0表示循环开始，goto:goto_0表示这一轮循环结束跳转到goto_0开始的地方(这里的:goto_0表示的代码段标号)  
for循环特点：在进入循环前，需要先初始化循环计数器变量，且它的值需要在循环体中更改。循环条件判断可以是条件跳转指令组成的合法语句。循环中使用goto指令来控制代码的流程。  
while循环和do while循环，两者结构差异不大，只是循环条件判断的位置有所不同。  
<font size=2 color=#000000>switch语句</font>  
有规律递增switch语句:packed-switch分支，pswitch_data_0指定case区域。有规律递增switch语句case区域指定初值并依次递增。  
无规律递增switch语句:sparse_switch分支，sswitch_data_0制定case区域。case区域每个case 值——>case 标号的形式给出。  
<font size=2 color=#000000>trycatch语句</font>  
代码中的try语句块使用try_start_开头的标号注明，以try_end_开头的标号结束。第一个try语句开头标号是try_start_0，结束标号为try_end_0。使用多个try语句块时标号名称后面的数值依次递增。  
try_end_0下面使用".catch"指令制定处理到的异常类型与catch的标号，格式为:`.catch<异常类型>{<try 起始标号>..<try 结束标号>}<catch 标号>`


