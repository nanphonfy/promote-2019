### HTTP协议组成
>抓包工具：Fillder抓取请求，可得到如下请求（客户端）、响应数据（服务端）。 

- request
```java 
POST https://re.csdn.net/csdnbi HTTP/1.1
方法 url/uri 协议版本号 1.1

Host: re.csdn.net
Connection: keep-alive
Content-Length: 167
Accept: */*
Origin: https://www.csdn.net
User-Agent: Mozilla/5.0 (Windows NT 10.0; WOW64)
AppleWebKit/537.36 (KHTML, like Gecko)
Chrome/66.0.3359.23 Safari/537.36
Content-Type: text/plain;charset=UTF-8
Referer: https://www.csdn.net/
Accept-Encoding: gzip, deflate, br
Accept-Language: zh-CN,zh;q=0.9
Cookie: ......
--------------
[{"headers":{"component":"enterprise","datatype":"re","version":"v1"},"body":"{\"re\":\"ref=-&mtp=4&mod=ad_popu_131&con=ad_content_2961%2Cad_order_731&uid=-&ck=-\"}"}]
```
- response
```java 
HTTP/1.1 200 OK
协议版本号 响应状态码 状态码对应原因

Server: openresty
Date: Sun, 27 May 2018 12:08:44 GMT
Transfer-Encoding: chunked
Connection: keep-alive
Keep-Alive: timeout=20
Access-Control-Allow-Origin:https://www.csdn.net
Access-Control-Allow-Methods: GET, POST, OPTIONS
Access-Control-Allow-Credentials: true
Access-Control-Allow-Headers: DNT,X-CustomHeader,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,body

2
ok
0
```
- URL
>URL(Uniform Resource >Locator)：描述一个网络上的资源，基
本格式：http://www.xxx.com:80/xxx/index.html?name=xxx#head  
>>schema://host[:port#]/path/.../?[url-params]#[query-string]  
>>>scheme：指定应用层协议(eg.http、https、ftp)；  
host：HTTP服务器IP或域名；  
port#：HTTP 服务器默认端口80（可省略）；  
path：资源路径
query-string：查询字符串；  
#：片段标识符（可标记出已获取资源中的子资源——文档内某位置。

- URI
>Uniform Resource Identifier（统一资源标识符）：每个服务器的资源都有一个名字，可根据该名字找到对应资源。
URI用一个字符串，表示互联网上的某一资源。而 URL表示资源地点——互联网所在位置。

- 方法
>HTTP请求都需告诉服务器执行的动作（报文中的method），提供的常用方法如下：  
>GET：发送一个URI获取服务端资源（查询）；  
POST： 一般给服务端传输实体，让其保存（创建）；  
PUT：向服务器发送数据（更新）；  
HEAD：查询获取head信息，eg.index.html的有效性、最近更新时间等；  
DELETE：（删除）。

- HTTP协议特点
>无状态（服务器不知道请求访问的山上个方法），即不会保存请求和响应间的通信状态。

- 实现有状态的协议
>HTTP协议引入cookie技术——解决无状态问题。  
请求和响应报文写入Cookie来控制客户端的状态； Cookie根据响应报文的Set-Cookie首部字段信息，通知客户端保存Cookie。  
当下次发送请求时，客户端自动在请求报文加入 Cookie值。  
基于tomcat的jsp/servlet容器中，会提供session机制保存服务端的对象状态。
- 整个状态协议的流程  
......

- HTTP缺陷
>通信过程使用明文，内容可能被窃听；  
②不验证通信双方的身份；  
③无法验证报文完整性，可能被篡改。

#### HTTPS原理
- 简介
>由于HTTP的不安全性，为防止信息遭泄漏或篡改，对传输通道进行加密：HTTPS。  
它是加密的超文本传输协议，与HTTP差异：数
据传输过程，对数据做了完全加密。
......

>HTTP、HTTPS都处于TCP传输层上，故在TCP协议层之上增加一层 SSL（Secure Socket Layer，安全层或TLS（Transport Layer Security）安全层传输协议组合使用，构
造加密通道。

