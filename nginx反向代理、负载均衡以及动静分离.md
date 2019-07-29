## 反向代理
### 正向代理与反向代理
正向代理是一个位于客户端和目标服务器之间的代理服务器(中间服务器)。为了从原始服务器取得内容，客户端向代理服务器发送一个请求，并且指定目标服务器，之后代理向目标服务器转交并且将获得的内容返回给客户端。
反向代理实际运行方式是指以代理服务器来接受internet上的连接请求，然后将请求转发给内部网络上的服务器，并将从服务器上得到的结果返回给internet上请求连接的客户端，此时代理服务器对外就表现为一个服务器。
反向代理工作在服务期的前端，作为前端服务器，正向代理工作在客户端的前端，为客户端做代理。
### 反向代理的作用
1. 保证内网的安全，大型网站，通常将反向代理作为公网访问地址，Web服务器是内网。
2. 可以通过反向代理来实现负载均衡，通过反向代理服务器来优化网站的负载

### 反向代理示例
#### 环境  
```
192.168.0.168  代理服务器(nginx)
192.168.0.52   后端服务器(httpd)  
```  

#### 修改nginx代理配置  

```
location /wanger {
            proxy_pass http://192.168.0.52;
            proxy_set_header   Host             $host;
            proxy_set_header   X-Real-IP        $remote_addr;
            proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
        }
```  

#### 添加后端192.168.0.52/wanger首页 
```
mkdir /var/www/html/wanger
echo 192.168.0.52 > /var/www/html/wanger/index.html
```  

#### 在前端服务器进行访问测试
当客户端访问192.168.0.168/wanger/时，会将请求转发给后端192.168.0.52/wanger/index.html  
```
[root@192.168.0.168]# curl 127.0.0.1/wanger/
192.168.0.52
```  

可以看到反向代理测试成功
### proxy_set_header指令解释
proxy_set_header指令设置nginx发送到后端服务器的标头
上述配置中，将请求标头的Host字段设置为$ host变量。
将X-Real-IP字段设置为$remote_adde变量，也就是客户端的真实ip
将X-Forwarded-For字段设为$proxy_add_x_forwarded_for变量，$proxy_add_x_forwarded_for是ngx_http_proxy_module内置变量
#### proxy_add_x_forwarded_for变量
proxy_add_x_forwarded_for变量包含了X-Forward-For字段和remote_adr变量，并用逗号分隔，当X-Forwarded-For字段不存在时，那么$proxy_add_x_forwarded_for变量等于$remote_addr变量，当一个web应用被两台nginx代理服务器转发的时候，第一台的nginx代理的X-Forwarded-For字段为真实的客户端ip，第二台nginx代理的X-Forwarded-For字段为客户端ip,第一台代理的ip地址，也就是说，只有当代理服务器的数量大于1的时候，proxy_add_x_forwarded_for才有效果  
![](https://s1.51cto.com/images/blog/201907/26/1c2ea511d5e1383f2bad6c624378fb05.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)    

### 修改后端web日志格式
首先查看访问后端的日志，可以看到访问ip显示的还是nginx代理的ip  
![](https://s1.51cto.com/images/blog/201907/26/171abc02bfeb4a4da6d8c1b8c28fb30a.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)  

修改httpd日志格式，将日志中的ip修改为客户端ip  
```
vim /etc/httpd/conf/httpd.conf
LogFormat "%{X-Real-IP}i %{Host}i %l %u %t \"%r\" %>s %b \"%{Referer}i\" \"%{User-Agent}i\"" combined
```  

修改后的日志格式如下：  
![](https://s1.51cto.com/images/blog/201907/26/7f4aad637dd29f8f1dabebdda0a9d326.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)  

这样就会在后端服务器中显示真实的客户端ip了
### proxy_pass加不加/的区别
当proxy_pass在后面的url加上了/，相当于是绝对根路径，则nginx不会把location中匹配的路径部分代理走;如果没有/，则会把匹配的路径部分也给代理走
#### 示例    

```
location /wanger {
            proxy_pass http://192.168.0.52;
        }
```  

此时后端的url为192.168.0.52/wanger/index.html
```
location /wanger {
            proxy_pass http://192.168.0.52;
        }
```  

此时后端的url为192.168.0.52/index.html

## 负载均衡
负载均衡可以将请求前端的请求分担到后端多个节点上，提升系统的响应和处理能力。
### 环境  

```
192.168.0.168   负载均衡服务器
192.168.0.52  上游节点1
192.168.0.84   上游节点2  
```  

### 负载均衡调度策略
#### weight轮询（默认）
将请求按时间顺序分配到后端服务器，通过weight可以定义轮询权重，权重越高，访问几率越高  
```
upstream read{
        server 192.168.0.52 weight=2 max_fails=3 fail_timeout=20s;
        server 192.168.0.84:8080 weight=1 max_fails=3 fail_timeout=20s;
        server 192.168.0.96 down;
        server 192.168.0.168  backup;
}
```  

1. max_fails，允许fail_timeout时间内请求失败的最大次数，否则后端服务器将被标记为不可用
1. fail_timeout，表示错误次数的超时时间；当被标记为不可用后，暂停服务的时间。
1. down，将当前的server标记为不可用，即不参与负载均衡。
1. backup，标记为备用服务器，当所有非backup服务器不可用或者忙时，会请求该服务器。


请求结果如下：  
![](https://s1.51cto.com/images/blog/201907/26/b10308cccbd5d46b0704d461c867125d.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)    
#### ip_hash
将请求按照访问ip的hash结果分配到后端服务器，这样同一ip每次请求都能够访问到同一台后端服务器，可以解决动态网页的session问题  
```
upstream read{
        ip_hash
        server 192.168.0.52;
        server 192.168.0.84:8080;
}
```  

请求结果如下：
![](https://s1.51cto.com/images/blog/201907/26/e12750728faf4dd7c51405075d09bca2.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)  
#### 最少连接(least_conn)
根据后端服务器当前的连接情况，将请求分配给当前连接数最少的一台后端服务器上  
```
upstream read{
        least_conn;
        server 192.168.0.52;
        server 192.168.0.84:8080;
}  
```

请求结果如下：  
![](https://s1.51cto.com/images/blog/201907/26/83d17784d4b76fa1b408a9632382718c.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)  

#### fair
将请求分配给响应时间短的后端服务器
```
upstream read{
        fair;
        server 192.168.0.52;
        server 192.168.0.84:8080;
}
```  

#### url_hash
根据访问url的hash结果来分配请求，使每个url定向到同一个后端服务器，可以进一步提高后端缓存服务器的效率  
```
upstream read{
        hash $request_uri;
        server 192.168.0.52;
        server 192.168.0.84:8080;
}
```  

## 动静分离
### 为什么要做动静分离
Nginx的静态处理能力很强，但是动态处理能力不足，动静分离之后，方便对静态资源做缓存操作，并且提高了网站的响应速度
### 动静分离配置  
```
upstream static{
        server 192.168.0.52 weight=2 max_fails=3 fail_timeout=20s;     
}
upstream dynamic{
        server 192.168.0.168:9200  weight=2 max_fails=3 fail_timeout=20s;
}
server {
     listen 80;
     server_name localhost;
    location ~* \.php$ {              
                fastcgi_pass http://dynamic;
                fastcgi_index index.php;  
                fastcgi_param  SCRIPT_FILENAME  /usr/share/nginx/html$fastcgi_script_name;
                include fastcgi_params;
        }
    location ~* \.(jpg|gif|png|css|html|htm|js)$ {  
                proxy_pass http://static; 
                expires 12h;   
}   
```  


-----
欢迎关注个人微信公众号“没有故事的陈师傅”  
![](https://s1.51cto.com/images/blog/201907/26/c689c03bd7df06c9c4b9b0a227bca226.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)
