## 什么是nginx？
NGINX是一个免费的，开源的，高性能的HTTP服务器和反向代理，以及IMAP / POP3代理服务器。NGINX以其高性能，稳定性，丰富的功能集，简单的配置和低资源消耗而闻名。
## nginx能做什么？
NGINX是用于Web服务，反向代理，缓存，负载平衡，媒体流等的开源软件。它最初是一个旨在实现最高性能和稳定性的Web服务器。除了HTTP服务器功能外，NGINX还可以用作电子邮件（IMAP，POP3和SMTP）的代理服务器以及HTTP，TCP和UDP服务器的反向代理和负载平衡器。
## nginx安装方法
### yum安装
#### 1.安装utils
yum install yum-utils
#### 2.配置yum仓库  
```
vim /etc/yum.repos.d/nginx.repo
[nginx-stable]
name=nginx stable repo
baseurl=http://nginx.org/packages/centos/$releasever/$basearch/
gpgcheck=1
enabled=1
gpgkey=https://nginx.org/keys/nginx_signing.key

[nginx-mainline]
name=nginx mainline repo
baseurl=http://nginx.org/packages/mainline/centos/$releasever/$basearch/
gpgcheck=1
enabled=0
gpgkey=https://nginx.org/keys/nginx_signing.key
```  


#### 3. 安装nginx
`yum install nginx`
### 编译安装
#### 1.下载并解压源码
```
wget http://nginx.org/download/nginx-1.9.4.tar.gz
tar -xzf nginx-1.9.4.tar.gz
cd nginx-1.9.4
```
#### 2.安装编译环境
```  
yum update
yum -y install gcc pcre pcre-devel zlib zlib-devel openssl openssl-devel  
```
#### 3.编译安装
添加用户和组  
```
groupadd www
useradd -g www www
配置
./configure \
--user=www \
--group=www \
--prefix=/usr/local/nginx \
--with-http_ssl_module \
--with-http_stub_status_module \
--with-http_realip_module \
--with-threads
编译
make
安装
make install
```

#### 4.创建软链接
`ln -s /usr/local/nginx/sbin/nginx /usr/bin/nginx`
#### 5.启动程序
`nginx/sbin/nginx`

## nginx命令
* nginx 				#打开 nginx，
* nginx -t   				#测试配置文件是否有语法错误
* nginx -s reopen		#重新打开nginx日志，对应USR1信号，即kil -9 USR1 pid
* nginx -s reload			#在不重启的情况下重新加载Nginx配置文件，对应HUP信号，即kill -9 HUP pid
* nginx -s stop  			#强制停止Nginx服务，对应TERM信号或者INT信号,即kill -9 TERM|INT pid
* nginx -s quit  			#优雅地停止Nginx服务（即处理完所有请求后再停止服务），对应QUIT信号，即kill -9 QUIT pid

## 配置文件的结构
nginx由模块组成，这些模块由配置文件中指定的指令控制。指令分为简单指令和块指令。一个简单的指令由名称和参数组成，用空格分隔，以分号（;）结尾。块指令与简单指令具有相同的结构，但它不是以分号结尾，而是以大括号（{和}）包围的一组附加指令结束。如果块指令在大括号内可以有其他指令，则称为上下文（示例： events， http， server和 location）。
http块包含处理Web流量的指令。这些指令通常被称为通用指令，因为它们被传递给NGINX服务的所有网站配置。http中可以配置多个server，一个server中可以配置多个location，除了http块、server块和location块之外，还有events块、stream块等
块指令和简单指令是有一定的对应关系的，比如，有些简单指令只能在http块中配置，有些简单指令只能在server块中配置，有些简单指令只能在location块中配置，有些简单指令既能在server块中配置又能在http块中配置，可以在官网中(http://nginx.org/en/docs/)查看指令存在的位置,而最上方不属于任何块的配置指令的区域属于主配置区，用于定义网站的全局配置  
```
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;
events {
    ...
}
http {
...	
	server {
		...	
		location ... {
				...		...
		}
	}
	server {
		...
	}
}  
```

## nginx事件驱动模型
Nginx是以事件的触发来驱动的web服务器，Nginx服务器响应和处理Web请求的过程，就是基于事件驱动模型的，事件驱动模型包括事件收集器，事件发送器，事件处理器，事件收集器负责收集所有事件，事件包括来自软件、硬件以及用户的，事件发送器负责将收集器收集到的事件发送到目标对象，事件处理器负责具体事件的响应，事件包括读事件，写事件以及异常事件。
### 事件驱动模型中事件处理器的实现方式
* 事件发送器每传递一个请求，目标对象就创建一个进程，调用事件处理器处理该请求。
* 事件发送器每传递一个请求，目标对象就创建一个线程，调用事件处理器来处理该请求。
* 事件发送器每传递一个请求，目标对象就将其放入一个待处理事件的列表（请求队列），使用非阻塞I/O方式调用事件处理器来处理该请求。
### nginx事件驱动模型库
#### select
为三类事件分别创建一个事件描述符集合，分别用来收集读事件的描述符、写事件的描述符和异常事件的描述符，调用底层select()函数，等待事件发生。然后遍历三个集合中的事件描述符，当检测到事件发生时就处理该事件，select受最大文件描述符的限制
#### poll
为三类事件创建一个集合，最后轮询的时候，可以同时检查这三种事件是否发生
#### epoll
把描述符列表的管理交给内核负责，一旦有某种事件发生，内核把发生事件的描述符列表通知给进程，这样就避免了轮询整个描述符列表。epoll库通过相关调用通知内核创建一个待处理的事件列表，当某一事件发生后，内核将发生事件的描述符列表上报给epoll库，得到事件列表的epoll库，就可以进行事件处理了。
## nginx全局变量
* nginx变量索引：http://nginx.org/en/docs/varindex.html
* $args               #这个变量等于请求行中的参数，同$query_string;
* $content_length     #请求头中的Content-length字段;
* $content_type       #请求头中的Content-Type字段;
* $document_root      #当前请求在root指令中指定的值，如:root /var/www/html;
* $host               #请求主机头字段，否则为服务器名称;
* $http_user_agent    #客户端agent信息;
* $http_cookie        #客户端cookie信息;
* $limit_rate         #这个变量可以限制连接速率;
* $request_method     #客户端请求的动作，通常为GET或POST;
* $remote_addr        #客户端的IP地址;
* $remote_port        #客户端的端口;
* $remote_user        #已经经过Auth Basic Module验证的用户名;
* $request_filename   #当前请求的文件路径，由root或alias指令与URI请求生成;
* $scheme             #HTTP方法（如http，https）;
* $server_protocol    #请求使用的协议，通常是HTTP/1.0或HTTP/1.1;
* $server_addr        #服务器地址，在完成一次系统调用后可以确定这个值;
* $server_name        #服务器名称;
* $server_port        #请求到达服务器的端口号;
* $request_uri        #包含请求参数的原始URI，不包含主机名，如：”/foo/bar.php?arg=baz”;
* $uri                #不带请求参数的当前URI，$uri不包含主机名，如”/foo/bar.html”;
* $document_uri       #与$uri相同,例：http://localhost:88/test1/test2/test.php;  

## nginx访问认证
nginx访问认证需要用到auth_basic模块，此模块使用的是HTTP Basic Authentication协议来对用户进行访问控制，但此模块并不保证安全性，因为浏览器是以明文方式将用户名和密码传给Web服务器的
### 指令解释
#### auth_basic语法

| 语法 | auth_basic string 丨 off; | 
| -------- | -------- | 
| 默认   | auth_basic off;  | 
|应用位置 | http，server，location，limit_except|


string字符会在用户认证的弹窗中显示
#### auth_basic_user_file语法
| 语法 | auth_basic_user_file file; | 
| -------- | -------- | 
| 默认   | - | 
|应用位置 | http，server，location，limit_except|
指定保存用户名和密码的文件
### 使用htpasswd创建密码文件
#### htpasswd命令语法  
```
htpasswd [-cimBdpsDv] [-C cost] passwordfile username
htpasswd -b[cmBdpsDv] [-C cost] passwordfile username password
htpasswd -n[imBdps] [-C cost] username
htpasswd -nb[mBdps] [-C cost] username password
```  

#### htpasswd命令参数  
```
-c 创建密码文件
-n 将加密后的内容输出在屏幕上；
-m 默认采用MD5算法对密码进行加密
-d 采用CRYPT算法对密码进行加密
-p 不对密码文件中的密码进行加密，即使用普通文本格式的密码
-s 采用SHA算法对密码进行加密
-b 命令行中一并输入用户名和密码而不是根据提示输入密码，可以看见明文，不需要交互
-D 从密码文件中删除指定的用户  
```

### 访问认证实例
下面我们通过auth认证来对kibana进行用户登录认证
#### 修改nginx配置文件  
```
location /kibana/ {
            auth_basic "kibana";
            auth_basic_user_file /etc/nginx/kibanauser;
            proxy_pass http://127.0.0.1:5601/;
            proxy_set_header   Host             $host:5601;
            proxy_set_header   X-Real-IP        $remote_addr;
            proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
            rewrite ^/kibana/(.*)$ /$1 break;
        }  
```

#### 使用htpasswd生成密码文件
`htpasswd -c /etc/nginx/kibanauser admin`

#### 进行访问测试  
![](https://s1.51cto.com/images/blog/201907/26/09d6cd8e8d5039e7312000fad9c23b2a.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)  

## nginx虚拟机主机配置
### 1.配置基于域名的虚拟主机
#### 编辑nginx配置文件  

```
vim /etc/nginx/nginx.conf
server {
        listen       80 default_server;
        server_name  huazai.com;
        location / {
                root    html/huazai;
                index   index.html;
        }
        error_page 404 /404.html;
            location = /40x.html {
        }
        error_page 500 502 503 504 /50x.html;
            location = /50x.html {
        }
    }
server {
        listen       80;
        server_name  wanger.com;
        location / {
                root    html/wanger;
                index   index.html;
        }
        error_page 404 /404.html;
            location = /40x.html {
        }
        error_page 500 502 503 504 /50x.html;
            location = /50x.html {
        }
    }
```  


#### 配置首页文件并重载nginx  

```
cd /usr/share/nginx/html/
mkdir wanger
mkdir huazai
echo "I'm huazai" >huazai/index.html
echo "I'm wanger" >wanger/index.html
chmod -R 777 wanger/
chmod -R 777 huazai/
nginx -s reload 
```  

#### 访问虚拟主机  
```
curl -xlocalhost:80 huazai.com
I'm huazai
curl -xlocalhost:80 wanger.com
I'm wanger
```  

### 2.配置基于端口的虚拟主机
#### 编辑配置文件  
```
server {
        listen       80 default_server;
        server_name  huazai.com;
        location / {
                root    html/huazai;
                index   index.html;
        }
        error_page 404 /404.html;
            location = /40x.html {
        }
        error_page 500 502 503 504 /50x.html;
            location = /50x.html {
        }
    }

server {
        listen       800;
        server_name  huazai.com;
        location / {
                root    html/wanger;
                index   index.html;
        }
        error_page 404 /404.html;
            location = /40x.html {
        }
        error_page 500 502 503 504 /50x.html;
            location = /50x.html {
        }
    }
```  

#### 测试并重启nginx
```
nginx -t
systemctl restart nginx
```  

#### 访问虚拟主机  
```
[root@ html]# curl 127.0.0.1:80
I'm huazai
[root@ html]# curl 127.0.0.1:800
I'm wanger  
```  
## nginx location块详解
location指令的作用是根据用户请求的URI来执行不同的应用。
### location语法  
```
location [=|~|~*|^~|@] uri {
    ……
}
```  

### 匹配模式及顺序
#### 匹配模式
uri可以是普通的字符串地址，也可以是正则表达式，其中 ~ 和 ~* 用于正则表达式， 其他前缀和无任何前缀都用于普通字符，而~是区分大小写的匹配，~*用于不区分大小写的匹配，还可以使用“!”对匹配进行取反，^~表示如果与特定的字符串进行匹配，那么不在进行正则搜索， =表示精确前缀匹配，只有完全匹配才能生效，使用完全匹配可以略微加快请求时间，@定义命名location区段，这些区段客户端不能访问，只可以由内部产生的请求来访问，如try_files或error_page等

#### 匹配顺序  

1. ：“=“的精确匹配优先级最高，将会最先匹配
2. ：带有“^~”修饰符的前缀匹配，并返回最长前缀的匹配结果
3. ：处理具有正则表达式（〜和〜 *）的所有位置指令。如果正则表达式与请求匹配，则nginx停止搜索并完成请求
4. ：如果以上都没有匹配到，则匹配最长的前缀


### location匹配实例  
```
location = / {
  return "规则A";
}
location ^~ /static/ {
  return "规则B";
}
location ^~ /static/files {
  return "规则C";
}
location ~ .*\.(gif|jpg|png|js|css)$ {
  return "规则D";
}
location ~* \.png$ {
  return "规则E";
}
location /img {
  return "规则F";
}
location / {
  return "规则G";
```   

1. 访问根目录/时将匹配规则A
1. 访问/static/index.html时将匹配规则B，访问/static/files/1.html时则匹配规则C
1. 访问/a.png时, 将匹配规则D和规则E，但是规则D顺序优先， 规则E不起作用，而/static/c.png则优先匹配到规则B
1. 访问/a.PNG时则匹配 规则E，而不会匹配规则D，因为规则E不区分大小写。
1. 访问/img/a.gif时会匹配上规则D ,虽然规则F也可以匹配上，但是因为正则匹配优先，而忽略了规则F。
1. 访问/img/a.txt时会匹配上规则F 。
1. 访问/dev/test.py时会匹配规则G。
### location的root与alias
root和alias都可以定义在location模块中，都是用来指定请求资源的真实路径  

``` 
location /wanger {
                root    html;
                index   index.html;
        }

curl 127.0.0.1/wanger/index.html
I'm wanger
```

客户端请求http://127.0.0.1/wanger/index.html 地址时，在服务器的资源是/html/wanger/index.html，真实路径是root加上location指定的值
```
location /wanger {
                alias    html/;
                index   index.html;
        }
```  

而alias是location指定的值的别名，也就是当客户端请求http://127.0.0.1/wanger/index.html 时，在服务器的资源时/html/index.html，真实路径是alias的路径，此时访问结果如下：
```
curl 127.0.0.1/wanger/index.html
I'm huazai
```  

#### 其他区别：
1. alias 只能作用在location中，而root可以存在server、http和location中。
1. alias 后面必须要用 “/” 结束，否则会找不到文件，而 root 则对 ”/” 可有可无。



-----

欢迎关注个人微信公众号“没有故事的陈师傅”
![](https://s1.51cto.com/images/blog/201907/26/26f916d2f6b6b269436909d9c7dc089c.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)