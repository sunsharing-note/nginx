## 读写分离
### 环境
这里的服务器地址为虚拟ip，因为我是在我的三台云主机上操作的
192.168.0.10   nginx前端
192.168.0.20   httpd（用于读）
192.168.0.30   httpd（用于写）
### 什么是WebDAV？
Web分布式创作和版本控制（WebDAV）是超文本传输​​协议（HTTP）的扩展，允许客户端执行远程Web内容创作操作。实质上，它使Web服务器可以充当文件服务器，允许作者在Web内容上进行协作。使应用程序可直接对Web Server直接读写，并支持写文件锁定(Locking)及解锁(Unlock)，还可以支持文件的版本控制。各种服务器都支持WebDAV，包括Apache，Microsoft的Internet信息系统，SabreDAV，Nginx，ownCloud和Nextcloud。
### 在写服务器上开启WebDAV功能  
```
vim /etc/httpd/conf/httpd.conf

<Directory "/var/www/html">
    Dav on
</Directory>
```  

#### 重启httpd服务
`systemctl restart httpd`

### 授予读写服务器网站的访问权限
为防止网站访问权限不足，需要分别为读写服务器的网站目录授予访问权限
```
setfacl -m u:apache:rwx /var/www/html/  
```

### 添加读写服务器的首页文件  
```
[root@192.168.0.20]# echo 192.168.0.20 > /var/www/html/index.html
[root@192.168.0.30]# echo 192.168.0.30 > /var/www/html/index.html
```  

### 进行上传文件测试
分别往读写服务器上上传文件，写服务器成功上传，由于读服务器没有开启WebDAV功能，所以读服务器报405错误。  
```
curl -T zz.txt http://192.168.0.20 （读服务器）
curl -T zz.txt http://192.168.0.30 （写服务器）  
```   

![](https://s1.51cto.com/images/blog/201907/26/f41ea73b05c2b7d1eb81fce29d5bf394.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)  

在写服务器的网站根目录也可以看到上传的文件  
![](https://s1.51cto.com/images/blog/201907/26/2f0a4c85151eba946fb27d295cb946e3.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)  
### 在nginx前端配置读写分离  
#### 编辑nginx配置文件  

```
location /wanger {
            root   html;
            index  index.html index.htm;
            proxy_pass  http://192.168.0.20/;
            if ($request_method = "PUT"){
                      proxy_pass  http://192.168.0.30;
            }
        }
```  

### 重载nginx  
```
nginx -s reload
```  

### 进行访问测试  
```
[root@192.168.0.10 ~]# curl 127.0.0.1/
192.168.0.20
[root@192.168.0.10 ~]# curl -T sh.txt 127.0.0.1/
```   

![](https://s1.51cto.com/images/blog/201907/26/b241dd3243736c0bf60477d7569b666b.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)  

可以看到写服务器已经存在我们刚上传的文件了，读写分离测试完成
## 防盗链  
### 什么是盗链
盗链指的是通过一些技术手段来获取他人服务器上的资源来展示在自己的网站上，而在自己的服务器并没有存储这个资源，通过盗链，使他人的的网站服务器压力负担变大。通过借助nginx的ngx_http_referer_module模块的valid_referers指令可以防止盗链，但是referer是可以伪造的，因此也只能防住一部分的盗链。
### valid_referers语法  

|语法| valid_referers none丨 blocked丨 server_names 丨 string ...;|
| ----- | ------ |
|默认|-|
|应用位置| server，location|  

### 参数说明
* none：referer字段为空
* blocked：Referer的值被防火墙或者代理服务器删除或伪装
* server_names：一个或多个服务器列表，检测Referer的值是否是列表中的某个
* $invalid_referer：内置变量，如果来源域名不在这个列表中，那么$invalid_referer变量的值为0，否则为1
### 盗链演示
首先准备两台服务器，地址分别为172.17.51.80，172.17.51.90，其中172.17.51.80为被盗链服务器，我们首先配置一下172.17.51.80的图片地址，将图片放入网站根目录  
![](https://s1.51cto.com/images/blog/201907/26/bf0d1dc36a4d95ad199e65b6b95444aa.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)   

然后配置172.17.51.90服务器进行盗链HTML的配置，在网站根目录下编辑index.html，并添加172.17.51.80的图片链接
html代码如下：  
```html
<a href="www.baidu.com">
<img src="http://172.17.51.80/a.jpg" /></a>
```  

然后访问172.17.51.90查看盗链效果，盗链成功，这里我使用的是我的两台云主机  
![](https://s1.51cto.com/images/blog/201907/26/2c00b1ff1ee7e4f29549253e5525e2e3.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)    

### 防盗链演示
修改172.17.51.80配置，对referer头进行过滤  
```
location ~ .*\.(jpg|gif|png)$ {
    valid_referers none blocked .*wanger.com;
    if ($invalid_referer) {
        #rewrite ^/ http://$host/403.png;
        return 403;
    }
}
```  

修改完成后重载nginx，并清除浏览器缓存，再次访问可以看到图片挂了  
![](https://s1.51cto.com/images/blog/201907/26/1dd324fa8f93b7e79c64c476e9e895cf.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)  


-----
欢迎各位关注我的个人公众号“没有故事的陈师傅”
![](https://s1.51cto.com/images/blog/201907/26/781425673712ab8f12bb98a3a1ef52db.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)
