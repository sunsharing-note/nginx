该ngx_http_rewrite_module模块用于使用PCRE正则表达式更改请求URI，返回重定向，以及有条件地选择配置。rewrite指令的功能就是，使用nginx提供的全局变量或自己设置的变量，然后结合正则表达式和标志位实现url重写以及重定向。
因此需要检查pcre是否安装  
```
[root@]# rpm -q pcre
pcre-8.32-17.el7.x86_64
```  

### break
#### break语法  


| 语法 | break; |
| -------- | -------- | 
| 默认| - |
|应用位置| server，location，if|


停止处理任何rewrite的相关指令。如果出现在location里面，那么所有后面的rewrite模块指令都不会再执行，也不发起内部重定向，而是直接用新的URI进一步处理请求。
#### 例子  
```
location = /testbreak {
        break;
        return 200 $request_uri;
        }
```
当uri中包含testbreak时，那么会停止执行后面的rewrite模块的命令，return属于rewrite模块。
### if
#### if语法

| 语法 | if (condition) { ... }|
| -------- | -------- | 
| 默认| - |
|应用位置| server，location|

#### if中的几种判断条件
* 当变量的值为空字符串或“ 0”时，变量为false ;在1.0.1版之前，任何以“ 0” 开头的字符串都被视为false值。
* 使用“ =”和“ !=”运算符比较变量和字符串;
* 变量使用“ ~”（对于区分大小写的匹配）和“ ~*”（对于不区分大小写的匹配）运算符与正则表达式进行匹配。正则表达式可以包含可供以后在$1.. $9变量中重用的捕获。也可以使用!取反。
* 使用“ -f”和“ !-f”运算符检查文件是否存在;
* 使用“ -d”和“ !-d”运算符检查目录是否存在;
* 使用“ -e”和“ !-e”运算符检查文件，目录或符号链接是否存在;
* 使用“ -x”和“ !-x”运算符检查可执行文件。

#### 例子：  
```
if ($request_method = POST ) {
  return 405;
}

if ( !-f $filename ) {
    break;            
}
```  

### return
停止处理并为客户端返回状态码，没有状态码的URL将被视为一个302状态码。
return语法

|语法|return code [text];return code URL;return URL;|
| -------- | -------- | 
|默认| -|
|应用位置| server，location|
#### 例子：  
```
# return code [text]; 
location = /1 {
    return 200 "this is 1";
}
# return code URL;
location = / {
    return 302 http://www.wanger.com;
}
# return URL;
location = / {
    return http://www.wanger.com;
}
```  

### rewrite
#### 应用场景
1. URL访问跳转，支持开发设计。 页面跳转、兼容性支持(新旧版本更迭)、展示效果(网址精简)等。
2. SEO优化(Nginx伪静态的支持)
3. 后台维护、流量转发等。
4. 安全(动态界面进行伪装)。

#### rewrite语法

|语法|rewrite regex replacement [flag];|
| -------- | -------- | 
|默认| -|
|应用位置| server，location，if|
#### 例子：
```
rewrite ^(.*) http://wanger.com/$1 permanent;  
```  

如果()里的正则表达式与请求的URI匹配，那么URI将根据replacement字符串中的指定进行更改，匹配成功将跳转到http://wanger.com/$1 ，$1的值是前面()里的正则匹配到的值，而后面的permanent是永久重定向301的标志，当rewrite 后面没有任何 flag 时就顺序执行 
#### 可选flag参数可以是以下之一：
|flag标记 |说明|
| -------- | -------- | 
|last| 本条规则匹配完成后继续匹配新的URI规则|
|break |本条规则匹配完成后不在进行新的URI匹配|
|redirect| 302临时重定向，浏览器会显示跳转后的URL地址，当nginx 服务关闭的时候，将无法定向到特定的网站|
|permanent |301永久重定向，浏览器会显示跳转后的URL地址，除非客户端清理浏览器缓存|
#### last与break的区别
last 和 break一样 它们都会终止此 location 中其他它rewrite模块指令的执行，last会重新将rewrite后的地址作为一个新的URI在server块中请求，而break会直接请求重写后的地址，并不会再进行新的请求
#### 举个例子：  
```
location ~ ^/break {
                rewrite ^/break /test/ break;
        }
location ~ ^/last {
                rewrite ^/last /test/ last;
        }
location /test/ {
                default_type application/json;
                return 200 '{"status":"success"}';
        }
```  

当我请求127.0.0.1/break时，浏览器返回的是404，因为break不会去请求/test/块，而网站根目录下test目录根本不存在，当我请求127.0.0.1/last时，浏览器返回的是{"status":"success"}，因为last将地址重写后生成了新的请求，新的请求地址为/test/，然后与/test/块进行匹配，返回200状态码以及{"status":"success"}
### set
用于定义一个变量，变量的值可以包含文本，变量或者是它们的组合形式。
#### set语法
|语法| set $variable value;|
| -------- | -------- | 
|默认| -|
|应用位置| server，location，if|
#### 例子  
```
location /wanger {
        #       return 302 http://60.205.177.168/huazai;
                root    html;
                index   index.html;
                set $var1 "client address is ";
                set $var2 $remote_addr;
                return 200 "$var1$var2";
}
```
#### 效果如下：
```  
curl 127.0.0.1/wanger
client address is 127.0.0.1
```  

### rewrite的一些实例
```
if ($host = "wanger.com"){
    rewrite ^/(.*)$ http://www.wanger.com/$1 permanent;
}
if (!-f $request_filename) {
    rewrite ^/(.*)  http://www.wanger.com/index.html;
}
if (!-e $request_filename) {
             rewrite ^/(.*)$ /index.php?$1 last;
    }  
```  


-----

欢迎关注个人微信公众号“没有故事的陈师傅”  
![](https://s1.51cto.com/images/blog/201907/26/31026d9e98abfbddfa6c33bf845c66cf.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)
