# nginx简单介绍

Nginx是一款轻量级的HTTP服务器（相比于Apache、Lighttpd而言），同时是一个高性能的HTTP和反向代理服务器，国内主流网站基本都是搭建于Nginx之上。

## 正向代理

正向代理，意思是一个位于客户端和目标服务器之间的服务器，为了从目标服务器取得内容，客户端向代理发送一个请求并指定目标服务器，然后代理向目标服务器转交请求并将获得的内容返回给客户端。
正向代理是为客户端服务的，客户端可以根据正向代理访问到它本身无法访问到的服务器资源。
对客户端是透明的，对服务端是非透明的，即服务端并不知道自己收到的是来自代理的访问还是来自真实客户端的访问

## 反向代理

反向代理是指以代理服务器来接受外网的连接请求，然后将请求转发给内部网络上的服务器，并将从服务器上得到的结果返回给外网请求连接的客户端，此时代理服务器对外就表现为一个反向代理服务器。
反向代理是为服务端服务的，反向代理可以帮助服务器接收来自客户端的请求，帮助服务器做请求转发，负载均衡等。对服务端是透明的，对客户端是非透明的，即客户端并不知道自己访问的是代理服务器，而服务器知道反向代理在为他服务。

## 常见配置

Nginx.conf (default)文件配置结构

- events:配置影响nginx服务器或与用户的网络连接。
- http：可以嵌套多个server，配置代理，缓存，日志定义等绝大多数功能和第三方模块的配置。
- server：配置虚拟主机的相关参数，一个http中可以有多个server。
- location：配置请求的路由，以及各种页面的处理情况。
- upstream：配置后端服务器具体地址，负载均衡配置不可或缺的部分。

```nginx
# user字段表明了Nginx服务是由哪个用户哪个群组来负责维护进程的，默认是nobody
# 这里用了root用户，staff组来启动并维护进程
# 查看当前用户命令： whoami
# 查看当前用户所属组命令： groups ，当前用户可能有多个所属组，选第一个即可
user root staff;
# worker_processes字段表示Nginx服务占用的内核数量
# 为了充分利用服务器性能你可以直接写你本机最高内核
# 查看本机最高内核数量命令： sysctl -n hw.ncpu
worker_processes 4;
# error_log字段表示Nginx错误日志记录的位置
# 模式选择：debug/info/notice/warn/error/crit
# 上面模式从左到右记录的信息从最详细到最少
error_log  /var/logs/nginx/error.log debug;
# Nginx执行的进程id,默认配置文件是注释了
# 如果上面worker_processes的数量大于1那Nginx就会启动多个进程
# 而发信号的时候需要知道要向哪个进程发信息，不同进程有不同的pid，所以写进文件发信号比较简单
# 你只需要手动创建，比如我下面的位置： touch /usr/local/var/run/nginx.pid

pid  /usr/local/var/run/nginx.pid;
events {    # 每一个worker进程能并发处理的最大连接数
    # 当作为反向代理服务器，计算公式为： `worker_processes * worker_connections / 4`
    # 当作为HTTP服务器时，公式是除以2
    worker_connections  2048;}
http {    # 关闭错误页面的nginx版本数字，提高安全性
    server_tokens off;    include       mime.types;    default_type  application/octet-stream;
    # 日志记录格式，如果关闭了access_log可以注释掉这段
    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                 '$status $body_bytes_sent "$http_referer" '
    #                '"$http_user_agent" "$http_x_forwarded_for"';

    # 关闭access_log可以让读取磁盘IO操作更快
    # 当然如果你在学习的过程中可以打开方便查看Nginx的访问日志
    access_log off;
    sendfile        on;
    # 在一个数据包里发送所有头文件，而不是一个接一个的发送
    tcp_nopush     on;
    # 不要缓存
    tcp_nodelay on;
    keepalive_timeout  65;        # 开启gzip
    gzip  on;    client_max_body_size 10m;    client_body_buffer_size 128k;
    # 关于下面这段在后面紧接着来谈！
    include /usr/local/etc/nginx/conf.d/*;
    
    # 服务器配置
    server {
        listen       80;
        server_name  a.tff.com *.a.tff.com;

        root /www/html/project;

        access_log  /var/log/nginx/project_access.log;
        error_log /var/log/nginx/project_error.log;

        location / {
            # index字段声明了解析的后缀名的先后顺序
            index index.html index.htm index.php;
            # 自动索引
            autoindex off;
            # 404 的时候重定向
            try_files $uri $uri/ /index.php?$query_string;
        }
        
        # 当用root配置的时候，root后面指定的目录是上级目录
        # 并且该上级目录必须含有和location后指定的名称的同名目录，否则404
        # root末尾的"/"加不加无所谓
        # 下面的配置如果访问站点http://localhost/test1访问的就是/var/www/test1目录下的站点信息
        location /test1/ {
            root /var/www/;
        }

        # 如果用alias配置，其后面跟的指定目录是准确的，并且末尾必须加"/"，否则404
        # 下面的配置如果访问站点http://localhost/test2访问的就是/var/www/目录下的站点信息
        location /test2/ {
            alias /var/www/;
        }
        
        # 静态资源分配
        location ~* \.(png|gif|jpg|jpeg)$ {
            root    /root/static/;  
            # autoindex 自动创建索引
            # on是打开 off关闭
            autoindex on;
            access_log  off;
            expires     10h;# 设置过期时间为10小时          
        }
        
        location ~* \.(eot|ttf|woff|svg|otf)$ {
            add_header Access-Control-Allow-Origin *;
        }

        #error_page  404              /404.html;

        # redirect server error pages to the static page /50x.html
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
            
        # 由于nginx调用php并不是如同调用一个静态文件那么直接简单，是需要动态执行php脚本
        # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
        location ~ \.php$ {
            fastcgi_pass   127.0.0.1:9000;
            fastcgi_index  index.php;
            fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
            include        fastcgi_params;
        }
    }
}
```

内置的全局变量：

|变量名|功能|
| ---- | ---- |
|$host|请求信息中的Host，如果请求中没有Host行，则等于设置的服务器名|
|$request_method|客户端请求类型，如GET、POST|
|$remote_addr|客户端的IP地址|
|$args|请求中的参数|
|$content_length|请求头中的Content-length字段|
|$http_user_agent|客户端agent信息|
|$http_cookie|客户端cookie信息|
|$remote_port|客户端的端口|
|$server_protocol|请求使用的协议，如HTTP/1.0、HTTP/1.1|
|$server_addr|服务器地址|
|$server_name|服务器名称||
|$server_port|服务器的端口号

- nginx -t  检查配置是否合理正确
- nginx -s reload  配置热更新
- nginx -s stop   停止nginx

## [Nginx 之 Location 总结](https://zhuanlan.zhihu.com/p/60909782)

### 初探 Location

```nginx
location
syntax: location [=|~|~*|^~|@] /uri/ { ... }
```

其中 “~ ” 前缀表示匹配区分大小写的正则 location，“~* ” 前缀表示匹配不区分大小写的正则 location；其他前缀（包括：“=”，“^~ ”和“@ ”）和 无任何前缀的都属于普通 location，= 精确匹配，^~ 不使用正则，@ 内部重定向。

### 匹配顺序

对于一个特定的 HTTP 请求，nginx 应该匹配哪个 location 块的指令呢?
匹配规则是：先匹配普通location ，再匹配正则表达式。

> 对于匹配普通 location，有如下两点：

- 匹配 URI 的前缀部分
- 最大匹配原则

（ 因为 location 不是 “严格匹配”，而是 “前缀匹配”，就会产生一个 HTTP 请求，可以 “前缀匹配” 到多个普通 location，例如：location /prefix/mid/ {} 和 location /prefix/ {}，对于请求 /prefix/mid/t.html，前缀匹配的话两个 location 都满足，选哪个？根据最大匹配原则 ，于是选的是 location /prefix/mid/ {} ）
> 对于正则表达式的匹配：

通常的规则是匹配完了 “普通 location” 指令，还需要继续匹配 “正则 location”。
但是也可以告诉 nginx 匹配到了 “普通 location” 后，不再需要继续匹配 “正则 ” 了。
要做到这一点只要在 “普通 location” 前面加上 “^~ ” 符号（ ^ 表示 “非”，~ 表示 “正则”，意思是：不要继续匹配正则 ）。
除了 “^~ ” 可以阻止继续搜索正则 location 外，还可以加 “=”。
那么 “^~ ” 和 “=” 都能阻止继续搜索正则 location 的话，那它们之间有什么区别呢？

> 区别很简单：

- 共同点是它们都能阻止继续搜索正则 location
- 不同点是 “^~ ” 依然遵守 “最大前缀” 匹配规则，然而 “=” 不是 “最大前缀”，而是严格匹配 ( exact match )
例如，location / {} 和 location = / {} 的区别：
- location / {} 遵守普通 location 的最大前缀匹配。
由于任何 URI 都必然以“/ ”根开头，所以对于一个 URI，如果有更精确的匹配，那自然是选这个更精确的；如果没有，“/ ” 一定能为这个 URI 垫背（ 至少能匹配到“/ ”）。
也就是说 location / {} 有点默认配置的味道，其他更精确的配置能覆盖这个默认配置（ 这也是为什么我们总能看到 location / {} 这个配置的一个很重要的原因）。
- location = / {}遵守的是 “严格精确匹配 exact match”。
也就是只能匹配对根目录的请求，同时会禁止继续搜索 正则 location。
如果我们只想对 GET / 请求配置作用指令，那么我们可以选 location = / {} 。这样能减少正则 location 的搜索，因此效率比 location / {} 高。
普通 location 匹配完后，还会继续匹配正则 location；但是 nginx 允许阻止这种行为，方法很简单，只需要在普通 location 前加 “^~ ” 或 “=”。
但其实还有一种 “隐含” 的方式来阻止正则 location 的搜索。
这种隐含的方式就是：当 “最大前缀” 匹配恰好就是一个“严格精确 ( exact match )”匹配，照样会停止后面的搜索。
原文字面意思是：只要遇到 “精确匹配 exact match”，即使普通 location 没有带 “=” 或 “^~ ” 前缀，也一样会终止后面的匹配。
假设当前配置是：location /exact/match/test.html { 配置指令块1}，location /prefix/ { 配置指令块2} 和 location ~ .html$ { 配置指令块3}
- 如果我们请求 GET /prefix/index.html ，则会被匹配到指令块3 ，因为普通 location /prefix/ 依据最大匹配原则能匹配当前请求，但是会被后面的正则 location 覆盖；
- 当请求 GET /exact/match/test.html ，会匹配到指令块1 ，因为这个是普通 location 的 exact match ，会禁止继续搜索正则 location。

最后来看一个总结的栗子：

```nginx
location   = / {
  # matches the query / only.
  [ configuration A ]
}
location   / {
  # matches any query, since all queries begin with /, but regular
  # expressions and any longer conventional blocks will be
  # matched first.
  [ configuration B ]
}
location ^~ /photos/ {
    # matches any query beginning with /images/ and halts searching,
  # so regular expressions will not be checked.
  [ configuration C ]
}
location ~* \.(gif|jpg|jpeg)$ {
  # matches any request ending in gif, jpg, or jpeg. However, all
  # requests to the /images/ directory will be handled by
  # Configuration C.  
  [ configuration D ]
}
 
Example requests:
  ● / -> configuration A
  ● /production/document.html -> configuration B
  ● /photos/1.gif -> configuration C
  ● /production/1.jpg -> configuration D
```

## [nginx如何调用php](https://www.cnblogs.com/donghui521/p/10334776.html)

### 反向代理配置

#### 跨域

利用proxy_pass反向代理配置将请求转发到目标服务器

假设

前端页面的域名为 a.tff.com ,
后端接口的域名为 b.tff.com

当页面请求接口时，浏览器会报跨域错误，这时候我们可以通过nginx配置来转发请求

```nginx
server { 
  listen       80; 
  server_name  a.tff.com; 
  location / { 
    proxy_pass b.tff.com; 
  } 
}
```

#### 去除端口号

同理，将80端口的访问转发到目标端口就可以

假设

node服务端口为3000
对外提供域名为a.tff.com

```nginx
server { 
  listen       80; 
  server_name  a.tff.com; 
  location / { 
    proxy_pass http://127.0.0.1:3000; 
  } 
   
  location /api/ { 
      # 反向代理我们通过proxy_pass字段来设置 
      # 也就是当访问http://a.tff.com/api的时候经过Nginx反向代理到服务器上的http://127.0.0.1:3000 
      # 同时由于解析到服务器上的时候api这个字段都要处理 
      # 所以通过rewrite字段来进行正则匹配替换 
      # 也就是http://a.tff.com/api/hello经过Nginx解析到服务器变成http://127.0.0.1:3000/hello 
      proxy_pass http://127.0.0.1:3000; 
      rewrite ^/api/(.*) /$1 break; 
  } 
}
```

## 301 和 302

```nginx
location /old/ { 
    # 当匹配到http://a.tff.com/old/的时候会跳转到http://a.tff.com/new 
    # 301 永久 
    rewrite ^/(.*) http://a.tff.com/new/$1 permanent; 
    # 302 临时 
    rewrite ^/(.*) http://a.tff.com/$1 redirect; 
}
```

## Nginx URL重写（rewrite）介绍

和apache等web服务软件一样，rewrite的组要功能是实现RUL地址的重定向。Nginx的rewrite功能需要PCRE软件的支持，即通过perl兼容正则表达式语句进行规则匹配的。默认参数编译nginx就会支持rewrite的模块，但是也必须要PCRE的支持
rewrite是实现URL重写的关键指令，根据regex（正则表达式）部分内容，重定向到replacement，结尾是flag标记。

### rewrite语法格式及参数语法说明如下
  
|rewrite|regex|replacement|flag|
|---- |----|---- |----|
|关键字|正则|替代内容|flag标记|

- 关键字：其中关键字error_log不能改变
- 正则：perl兼容正则表达式语句进行规则匹配
- 替代内容：将正则匹配的内容替换成replacement
- flag标记：rewrite支持的flag标记

> flag标记说明：

last #本条规则匹配完成后，继续向下匹配新的location URI规则

break #本条规则匹配完成即终止，不再匹配后面的任何规则

redirect #返回302临时重定向，浏览器地址会显示跳转后的URL地址

permanent #返回301永久重定向，浏览器地址栏会显示跳转后的URL地址

## gzip 和 缓存

GZIP是规定的三种标准HTTP压缩格式(Zlib、Gzip、Deflate)之一。目前绝大多数的网站都在使用GZIP传输 HTML、CSS、JavaScript 等资源文件

对于文本文件，GZip 的效果非常明显，开启后传输所需流量大约会降至 1/4 ~ 1/3，。
并不是每个浏览器都支持gzip的，如何知道客户端是否支持gzip呢，请求头中的Accept-Encoding来标识对压缩的支持。


启用gzip同时需要客户端和服务端的支持，如果客户端支持gzip的解析，那么只要服务端能够返回gzip的文件就可以启用gzip了,我们可以通过nginx的配置来让服务端支持gzip。下面的respone中content-encoding:gzip，指服务端开启了gzip的压缩方式。

```nginx
http { 
    # 开启或者关闭gzip模块, 默认值为off 
    gzip                    on; 
    # 启用 GZip 所需的HTTP 最低版本, 默认值为HTTP/1.1 
    gzip_http_version       1.1;    
    # 压缩级别，级别越高压缩率越大，当然压缩时间也就越长（传输快但比较消耗cpu） 
    # 默认值1，压缩级别取值为1-9 
    gzip_comp_level         5; 
    # 默认值: gzip_buffers 4 4k/8k
    # 设置系统获取几个单位的缓存用于存储gzip的压缩结果数据流。 例如 4 4k 代表以4k为单位，按照原始数据大小以4k为单位的4倍申请内存。 4 8k 代表以8k为单位，按照原始数据大小以8k为单位的4倍申请内存。
    # 如果没有设置，默认值是申请跟原始数据相同大小的内存空间去存储gzip压缩结果
    gzip_buffers 4 16k;
 
    # 设置允许压缩的页面最小字节数，Content-Length小于该值的请求将不会被压缩 
    # 默认值:0，当设置的值较小时，压缩后的长度可能比原文件大，建议设置1000以上 
    gzip_min_length         1024; 
    
    # 要采用gzip压缩的文件类型(MIME类型) 
    # 默认值:text/html(默认不压缩js/css) 
    gzip_types text/csv text/xml text/css text/plain text/javascript application/javascript application/x-javascript application/json application/xml; 
    
    # 和http头有关系，加个vary头，给代理服务器用的，有的浏览器支持压缩，有的不支持，所以避免浪费不支持的也压缩，所以根据客户端的HTTP头来判断，是否需要压缩
    gzip_vary on
    
        
     # Nginx作为反向代理的时候启用，开启或者关闭后端服务器返回的结果，匹配的前提是后端服务器必须要返回包含"Via"的 header头。
     # off - 关闭所有的代理结果数据的压缩
     # expired - 启用压缩，如果header头中包含 "Expires" 头信息
     # no-cache - 启用压缩，如果header头中包含 "Cache-Control:no-cache" 头信息
     # no-store - 启用压缩，如果header头中包含 "Cache-Control:no-store" 头信息
     # private - 启用压缩，如果header头中包含 "Cache-Control:private" 头信息
     # no_last_modified - 启用压缩,如果header头中不包含 "Last-Modified" 头信息
     # no_etag - 启用压缩 ,如果header头中不包含 "ETag" 头信息
     # auth - 启用压缩 , 如果header头中包含 "Authorization" 头信息
     # any - 无条件启用压缩
     # 默认值：off
    gzip_proxied [off|expired|nocache|nostore|private|no_last_modified|no_etag|auth|any] ...
}
```

## 负载均衡

```nginx
  upstream upstream_test{   
     server 192.168.0.1:8080; 
     server 192.168.0.2:8080; 
 
     #ip_hash;  固定访问ip 
     keepalive 30; 
 } 
 
server { 
  listen       80; 
  server_name  a.tff.com; 
  location / { 
    proxy_pass http://upstream_test; 
  } 
}
```
