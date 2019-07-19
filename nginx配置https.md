### 什么是https
HTTPS代表超文本传输协议安全。它是用于保护两个系统（例如浏览器和Web服务器）之间的通信的协议。
下图说明了通过http和https进行通信的区别：
![](https://s1.51cto.com/images/blog/201907/19/b90cb25fe3eec4d6119de6597738e03a.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)
如上图所示，http以超文本格式在浏览器和Web服务器之间传输数据，而https以加密格式传输数据。因此，https可防止hacker在浏览器和Web服务器之间传输期间读取和修改数据。即使hacker设法拦截通信，他们也无法使用它，因为消息是加密的。
HTTPS使用安全套接字层（SSL）或传输层安全性（TLS）协议在浏览器和Web服务器之间建立加密链接。TLS是SSL的新版本。
### 什么是SSL
SSL是用于在两个系统之间建立加密链接的标准安全技术。这些可以是浏览器到服务器，服务器到服务器或客户端到服务器。基本上，SSL确保两个系统之间的数据传输保持加密和私密。
https本质上是http over SSL。SSL使用SSL证书建立加密链接，SSL证书也称为数字证书。
HTTP协议以明文方式发送内容，不提供任何方式的数据加密。为了数据传输的安全，HTTPS在HTTP的基础上加入了SSL协议，SSL依靠证书来验证服务器的身份，并为浏览器和服务器之间的通信加密。
### HTTP与HTTPS比较


| HTTP | HTTPS |
| -------- | -------- | 
| 以超文本（结构化文本）格式传输数据	  |以加密格式传输数据|
|默认使用端口80 |默认使用端口443|
|不安全|使用SSL技术保护安全|
|以 http://开始	|以 https://开始|

### https的优势
- 安全通信： https通过在浏览器和服务器或任何两个系统之间建立加密链接来建立安全连接。
- 数据完整性： https通过加密数据提供数据完整性，因此，即使hacker设法捕获数据，他们也无法读取或修改数据。
- 隐私和安全： https通过防止hacker被动地监听浏览器和服务器之间的通信来保护网站用户的隐私和安全。
- 更快的性能： https通过加密和减小数据的大小来提高数据传输的速度。
- SEO：使用https增加SEO排名。在谷歌浏览器中，如果用户的数据是通过http收集的，Google会在浏览器中显示“ 不安全”标签。  


### 使用OpenSSL生成证书
OpenSSL是一种功能强大的商用级全功能工具包，适用于传输层安全性（TLS）和安全套接字层（SSL）协议。它也是一个通用的加密库。
#### 检查nginx ssl模块
nginx配置SSL需要依赖http_ssl_module模块
使用nginx -V查看是否安装
#### 使用OpenSSL生成私钥文件和CA自签证书
/etc/pki/tls/openssl.cnf文件中包含着CA的证书文件
```
>> cd /etc/pki/CA
>> (umask 077; openssl genrsa 2048 > private/cakey.pem)
>> openssl req -new -x509 -key private/cakey.pem -out cacert.pem
Country Name (2 letter code) [XX]:CN
State or Province Name (full name) []:bj
Locality Name (eg, city) [Default City]:bj
Organization Name (eg, company) [Default Company Ltd]:wanger
Organizational Unit Name (eg, section) []:wanger
Common Name (eg, your name or your server's hostname) []:CA.wanger.com
Email Address []:wanger@admin.com
>> touch serial
>> echo 01 >serial
>> touch index.txt
```  

#### 创建nginx证书申请  
```
>> mkdir /etc/nginx/ssl
>> cd mkdir /etc/nginx/ssl
>> (umask 077; openssl genrsa 1024 > nginx.key)
>> openssl req -new -key nginx.key -out nginx.csr
Country Name (2 letter code) [XX]:CN
State or Province Name (full name) []:bj
Locality Name (eg, city) [Default City]:bj
Organization Name (eg, company) [Default Company Ltd]:wanger
Organizational Unit Name (eg, section) []:wanger
Common Name (eg, your name or your server's hostname) []:www.wanger.com
Email Address []:wanger@admin.com
>> openssl ca -in nginx.csr -out nginx.crt -days 3650
```  

#### 修改nginx配置文件
```
server {
        listen  443 ssl;
        server_name     localhost;
        ssl_certificate /etc/nginx/ssl/nginx.crt;
        ssl_certificate_key     /etc/nginx/ssl/nginx.key;
        ssl_session_cache       shared:SSL:1m;
        ssl_session_timeout     5m;
        ssl_ciphers     HIGH:!aNULL:!MD5;
        ssl_prefer_server_ciphers       on;
        location / {
                root    html;
                index   index.html;
        }
    }
```


#### 配置80端口重定向到https  
```
server {
        listen       80;
        server_name  wanger.com;
        rewrite ^(.*) https://$server_name$1 permanent;
}
``` 

#### 测试并重载nginx
```
nginx -t
nginx -s reload
```  

---
欢迎各位关注我的个人公众号“没有故事的陈师傅”
![](https://s1.51cto.com/images/blog/201907/19/fcf0111ec4c55fb9d1149f07a6d8a825.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)