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

###### 实现原理

![image](https://raw.githubusercontent.com/nanphonfy/note-images/master/promote-2019/distributed/04/HTTPS.png)

- 1 客户端发起请求
>a) 三次握手，建立TCP连接;  
b) 支持的协议版本(TLS/SSL);  
c) 客户端生成随机数client.random——生成"对话密钥";  
d) 客户端支持的加密算法;  
e) sessionid：保持同一会话（避免费尽周折建立HTTPS链接，刚建完就断了）。

- 2 服务端收到请求、响应
>a) 确认加密通道协议版本；
b) 服务端生成随机数erver.random——生成"对话密钥";  
c) 确认使用的加密算法（后续签名防篡改）；  
d) 服务器证书（CA机构颁发给服务端的证书）。

- 3 客户端验证收到的证书
>a) 验证证书是否是上级CA签发的。浏览器会对证书路径中的所有证书一级一级验证，只有路径中所有证书都受信，整个验证结果才受信；   
b) 服务端返回的证书中包含证书有效期，验证是否过期；  
c) 验证证书是否被吊销；  
d) CA机构签发证书时，会用自己的私钥对证书签名，证书里的签名算法字段sha256RSA表示CA机构使用sha256对证书进行摘要，然后使用RSA算法对摘要进行私钥签名（RSA算法中，用私钥签名后，只有公钥才能验签）；  
e) 浏览器用内置在操作系统上的CA机构的公钥对服务器证书验签。确定证书是否由正规机构颁发。验签后得知CA机构用sha256证书摘要，客户端再用
sha256对证书内容进行一次摘要，若得到的值和服务端返回证书验签后的摘要相同，表示证书没被修改过；  
f) 验证通过后，显示绿色安全字样；  
g) 验证通过后，客户端生成一个随机数pre-master secret，与之前的：client.random +sever.random +pre-master生成对称密钥，然后使用证书中的公
钥加密，同时用前面协商好的加密算法，将握手消息取HASH值，用"随机数加密"握手消息+握手消息 HASH值（签名），传递给服务器端。
>>之所以要取握手消息的HASH值，是把握手消息做一个签名，用于验证握手消息在传输过程中没被篡改过。 

- 4 服务端接收随机数  
>a) 服务端收到客户端的加密数据后，用自己的私钥对密文解密，得到client.random/server.random/pre-master secret，再用随机数密码解密握手消息与HASH 值，与传过来的HASH值，对比确认是否一致；  
b) 用随机密码加密一段握手消息(握手消息+握手消息的HASH值 )给客户端。 

- 5 客户端接收消息
>a) 客户端用随机数解密并计算握手消息的HASH， 若与服务端发来的HASH一致，握手过程结束；  
b) 之后所有的通信数据将由之前交互过程中生成的pre master secret /client.random/server.random通过算法得出 session Key，作为后续交互过程中的对称密钥。