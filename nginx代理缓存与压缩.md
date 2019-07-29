## nginx代理缓存
nginx的ngx_http_proxy_module自带了缓存功能,下面介绍几个常用的指令以及如何配置。
### proxy_cache_path
nginx缓存的内容是放在磁盘中的，所以我们需要定义存放缓存的载体，proxy_cache_path设置缓存的路径和其他参数。缓存中的文件名为proxy_cache_key定义的字符串的hash结果
#### proxy_cache_path语法
|语法|proxy_cache_path path [levels=levels][use_temp_path=on丨off] keys_zone=name:size [inactive=time] [max_size=size] [manager_files=number] [manager_sleep=time] [manager_threshold=time] [loader_files=number] [loader_sleep=time] [loader_threshold=time] [purger=on丨off] [purger_files=number] [purger_sleep=time] [purger_threshold=time];|
| ------ | ----- |
|默认|-|
|应用位置| http|  

#### 参数解释：
* path：定义缓存文件的路径，目录不存在会自动创建
* levels：定义缓存目录的级别，最多为3级，每级目录名长度字符只能为1或者2个字节。
* use_temp_path：当参数为on时，会使用proxy_temp_path定义的目录，否则临时文件将直接放在缓存目录里
* keys_zone：定义共享内存名字和共享内存大小，name表示共享内存名称，size表示共享内存大小
* inactive：在inactive参数指定的时间内未被访问的缓存数据 将从缓存中删除，默认为十分钟
* max_size：定义缓存文件最大空间，超过将会被cache manager进程删除掉最近最少使用的缓存
* manager_files：定义cache manager进程执行一次所要删除的文件数，默认为100
* manager_sleep：执行一次cache manager进程后的休眠时间，默认为50毫秒
* manager_threshold：执行一次cache manager删除循环的最大耗时，默认为200毫秒。
* loader_files：cache loader进程用于将磁盘中原有的缓存数据加载到共享内存中，也是通过循环来完成的，用于定义每次载入的最大文件数，默认为100
* loader_sleep：执行一次载入缓存进程的休眠时间，默认为50毫秒
* loader_threshold：执行一次载入缓存进程所需的时间，默认为50毫秒  


### proxy_cache
用来指定使用哪个共享内存，使用proxy_cache_path中的name来引用
#### proxy_cache语法  

|语法| proxy_cache zone 丨 off;|
| ------ | ----- |
|默认|proxy_cache off;|
|应用位置| http,server,location|  

### proxy_cache_key  
定义缓存的key，将以key的hash值作为缓存文件名 
#### proxy_cache_key语法  

|语法| proxy_cache_key string;|
| ------ | ----- |
|默认| proxy_cache_key $ scheme $ proxy_host $ request_uri;|
|应用位置|http,server,location|  

### proxy_cache_valid
用于设置不同响应代码的缓存时间
#### proxy_cache_valid语法
|语法| proxy_cache_valid [code ...] time;|
| ------ | ----- |
|默认| -|
|应用位置|http,server,location|  

```
proxy_cache_valid 200 302 10m;
proxy_cache_valid 404 1m;
proxy_cache_valid 5m;
proxy_cache_valid any 1m;
```  

以上代码表示状态码为200和302的缓存有效期为10分钟，状态码为404的缓存有效期为1分钟，如果不指定状态码，那么只有缓存状态码200,301和302各五分钟，any表示缓存任何响应
#### 在响应头中设置缓存时长
* 当X-Accel-Expires为0时，禁止缓存内容，使用@可以设置一天中的某一时刻
* 当请求头中包含“Set-Cookie”字段时，则不会缓存此类响应
* 当"Vary”字段的值为"*"时，则不会缓存此类响应  


### proxy_no_cache
定义不将响应保存到缓存的条件。当字符串参数为真时，则响应不会保存到缓存
#### proxy_no_cache语法
|语法| proxy_no_cache string ...;|
| ------ | ----- |
|默认|-|
|应用位置| http,server,location|    

### proxy_cache_bypass
定义不从缓存中获取响应的条件，当字符串参数为真时，则不会从缓存中获取响应
#### proxy_cache_bypass语法
|语法| proxy_cache_bypass string ...;|
| ------ | ----- |
|默认| -|
|应用位置| http,server,location|  

### $upstream_cache_status变量
$upstream_cache_status是一个位于ngx_http_upstream_module模块来显示缓存状态的变量，可以在配置中添加一个http头来显示此变量的值
#### 变量的值  

* MISS：未命中的缓存
* HIT：命中缓存
* EXPIRED：缓存已经过期，请求将被传递到后端
* STALE：后端将得到过期的应答
* UPDATING： 正在更新缓存，将使用旧的应答
* REVALIDATED：nginx验证了旧的缓存依然有效
* BYPASS：缓存被绕过了，应答是从原始服务器获得的

### 代理缓存配置示例
为验证缓存，这里我将缓存超时时间设为1分钟  
```
proxy_cache_path /data/cache levels=1:2 keys_zone=cache:10m max_size=100m inactive=1m use_temp_path=off;
    server {
        listen       80;
        server_name  wanger.com;
        include /etc/nginx/default.d/*.conf;
        location /wanger {
            proxy_pass http://192.168.0.52;
            proxy_cache cache;
            proxy_cache_valid 200 301 1m;
            add_header X-Cache $upstream_cache_status;
            proxy_cache_key $host$uri;
        }
   }     
```  


可以看到第一次访问并没有命中缓存，而第二次访问的时候已经建立好缓存了  
![](https://s1.51cto.com/images/blog/201907/26/2183caf0bf82c919c6ef8fcdf9f8ec08.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)  

查看缓存目录，可以看到缓存文件已经存在，并且前两级的目录名是缓存文件名的后三个字符  
![](https://s1.51cto.com/images/blog/201907/26/f5c2152cb2021b78ddbdbe48708e2023.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)  

进入后端服务器，添加响应头字段X-Accel-Expires，并将值设置为3
```
server {
        listen       80;
        server_name  localhost;
        add_header X-Accel-Expires 3;
location / {
                root    html;
                index   index.html;
        }
    }
```  

再次进行测试，可以看到第一次请求显示缓存已经过期
![](https://s1.51cto.com/images/blog/201907/26/214afde7a217954dd25c4ce9e7bbcd21.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)  

再次进入后端服务器，添加响应头字段Vary，并将其值设置为"*"，并进行测试
```
server {
        listen       80;
        server_name  localhost;
        add_header X-Accel-Expires 3;
        add_header Vary *;
location / {
                root    html;
                index   index.html;
        }
    }
```  

可以看到index.htm文件一直没有被缓存  
![](https://s1.51cto.com/images/blog/201907/26/d511f82cbd9b558c9f435932cf49ce25.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)  

### 代理缓存清理
nginx提供了一个第三方模块ngx_cache_purge，GitHub地址：https://github.com/FRiCKLE/ngx_cache_purge
安装ngx_cache_purge模块需要重新编译nginx，并使用--add-module=模块位置参数添加模块到nginx里
#### proxy_cache_purge语法
这里需要用到proxy_cache_purge指令  

|语法|proxy_cache_purge zone_name key|
| ------- | ------- |
|默认| -|
|应用位置|location|
#### 配置示例
location ~ /purge(/.*) {
                proxy_cache_purge cache $host$1;
}
        location /wanger {
            proxy_pass http://192.168.0.52;
            proxy_cache cache;
            proxy_cache_valid 200 301 1m;
            add_header X-Cache $upstream_cache_status;
            proxy_cache_key $host$uri;

#### 代理缓存清理测试
由于我缓存过期时间设置的是1分钟，当我命中缓存之后，就开始进行缓存清理测试，之后在一分钟内再次访问同一个URL，就发现缓存命中失败了  
![](https://s1.51cto.com/images/blog/201907/26/239b6d45ff033a776d035cf93f89f323.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)  

## gzip压缩
gzip压缩模块提供了压缩文件内容的功能，通过压缩可以使服务器与浏览器之间传输的数据量更小，提高了客户端的响应速度，但压缩也会消耗nginx性能
### 指令解释
* gzip on|off：是否开启gzip，默认为off
* gzip_buffers number size： number指服务器向系统申请缓存空间的个数，size指每个缓存空间的大小
* gzip_comp_level [1-9]： 设置压缩级别，默认为1(级别越高,压的越小,越浪费CPU计算资源)
* gzip_disable regex：使用正则匹配某种浏览器类型不进行压缩
* gzip_min_length length：设置将被压缩的响应的最小长度，默认为20
* gzip_types text/plain application/xml：对哪些类型的文件启用压缩，文件类型可以在nginx/mime.types文件里查看
* gzip_vary on|off：启用或禁用插入“Vary：Accept-Encoding”响应头字段  


#### 配置示例  
```
server {
        listen       80;
        server_name  192.168.0.168;  
        gzip        on;
        gzip_types  image/jpeg;
        gzip_buffers 32 4K;
        gzip_min_length 100;
        gzip_comp_level 6;
        gzip_vary on;
}
```  

#### 测试压缩配置
浏览器要禁用浏览器缓存，可以看到压缩之后图片大小是75.9KB，响应头中也会多一个字段Content-Encoding: gzip  
![](https://s1.51cto.com/images/blog/201907/26/69d09b46656cec36fe24100dc2c997d7.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)

将gzip参数修改为off，重载nginx，再来看图片大小为76.4KB  
![](https://s1.51cto.com/images/blog/201907/26/1fb08ebde01e49d5e988fa32e1684ef8.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)  


-----

欢迎各位关注我的个人公众号“没有故事的陈师傅”