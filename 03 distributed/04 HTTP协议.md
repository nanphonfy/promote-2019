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