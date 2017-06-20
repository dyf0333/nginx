
# 基于nginx的日志收集系统设计与实现


日志收集统计系统以网站数据统计为基础，进行数据收集，形成日志，后续展示查询、进行分析。


## 1.数据收集原理分析

　　网站统计分析需要收集到用户浏览目标网站的行为（如打开某网页、点击某按钮等）及行为附加数据（如某按钮产生的行为数据等）。早期的网站统计往往只收集一种用户行为：页面的打开。而后用户在页面中的行为均无法收集。
　　这种收集策略能满足基本的流量分析、来源分析、内容分析及访客属性等常用分析视角，后来，Google在其产品谷歌分析中创新性的引入了可定制的数据收集脚本，用户通过谷歌分析定义好的可扩展接口，只需编写少量的javascript代码就可以实现自定义事件和自定义指标的跟踪和分析。目前百度统计、搜狗分析等产品均照搬了谷歌分析的模式。
### 1.1流程概览

![@图1. 网站统计数据收集基本流程|center](../master/src/1487845617089.png)


　　首先，用户的行为会触发浏览器对被统计页面的一个http请求，页面中的埋点javascript片段会被执行，这小段被加在网页中的javascript代码，会动态创建一个script标签，并将src指向一个单独的js文件，此时这个单独的js文件（图1绿色节点）会被浏览器请求到并执行，这个js往往就是真正的数据收集脚本。
　　
  数据收集完成后，js会请求一个后端的数据收集脚本（图1中的backend），一般伪装成图片的动态脚本程序，可能由php、python或其它服务端语言编写，js会将收集到的数据通过http参数的方式传递给后端脚本，后端脚本解析参数并按固定格式记录到访问日志，同时在http响应中给客户端cookie中种植一些用于追踪的唯一标识。

## 2.系统的设计实现

　　以上述原理，搭建一个访问日志收集系统工作如下：
  
![@图2. 访问数据收集系统工作分解|center](../master/src/1487845657758.png)

### 2.1前端开发
   
#### 2.1.1确定收集的信息

| name |    名称	| 途径	 |备注|
| :-------- | :--------| :------ |:----|
||网站标识	|javascript	|自定义对象|
|$u_utrace|用户id标识|web server|自定义
|$u_domain|域名	|javascript	|document.domain|
|$u_url|URL	|javascript	|document.URL|
|$u_title|页面标题|	javascript	|document.title|
|$u_sw|分辨率.宽度|javascript	|window.screen.width|
|$u_sh|分辨率.高度|javascript	|window.screen.height|
|$u_cd|颜色深度	|javascript	|window.screen.colorDepth|
|$u_referrer|Referrer	|javascript	|document.referrer|
|$u_lang|客户端语言	|javascript	|navigator.language|
| $msec  | 访问时间 |  web server   | Nginx |
|$http_user_agent|浏览客户端|	web server	|Nginx |
|$msec|当前的Unix时间戳 |	web server	|Nginx |
|$time_local	|访问时间和时区|web server	|Nginx |
|$request	|请求的URI和HTTP协议|web server	|Nginx |
|$http_host	|浏览器中输入的地址|web server	|Nginx |
|$status	|HTTP请求状态|web server	|Nginx |
|$http_referer	|url跳转来源|web server	|Nginx |
|$upstream_addr	|后台upstream的地址，<br>即真正提供服务的主机地址|web server	|Nginx |
|$request_time	|整个请求的总时间|web server	|Nginx |
|$upstream_response_time	|请求过程中，upstream响应时间|web server	|Nginx |
|$request_body	|post 数据内容|web server	|Nginx |
|$upstream_status	|upstream状态|web server	|Nginx |
|$body_bytes_sent	|发送给客户端文件内容大小|web server	|Nginx |
|$http_x_forwarded_for	|客户端真实IP|web server	|Nginx |
|$remote_addr |客户端IP	|web server	|Nginx |

前端js可记录：
- Document对象数据
-  Window对象数据
-  navigator对象数据
- 自定义业务数据
- 自定义用户数据

后端nginx可记录：
- web server相关数据
- IP相关数据
- 交互数据相关数据

<hr>
请求总时间 & 响应时间 （`$request_time` & `$upstream_response_time`）

![|center](../master/src/1487846181326.png)

`request_time`
官网描述：request processing time in seconds with a milliseconds resolution; time elapsed between the first bytes were read from the client and the log write after the last bytes were sent to the client 。
指的就是从接受用户请求的第一个字节到发送完响应数据的时间，即包括接收请求数据时间、程序响应时间、输出响应数据时间。

`upstream_response_time`
官网描述：keeps times of responses obtained from upstream servers; times are kept in seconds with a milliseconds resolution. Several response times are separated by commas and colons like addresses in the $upstream_addr variable
是指从Nginx向后端（php-fpm)建立连接开始到接受完数据然后关闭连接为止的时间。

<br>

从上面的描述可以看出，`request_time`肯定比`upstream_response_time`值大，特别是使用POST方式传递参数时，因为Nginx会把request body缓存住，接受完毕后才会把数据一起发给后端。所以如果用户网络较差，或者传递数据较大时，`request_time`会比`upstream_response_time`大很多。
所以如果使用nginx的accesslog查看php程序中哪些接口比较慢的话，可对比log_format中加入`upstream_response_time` 和 `request_time` 。

<hr>

nginx中`remote_addr` & `x_forwarded_for`参数区别

![Alt text|center](../master/src/1487846394808.png)

`remote_addr`
remote_addr代表客户端的IP，但它的值不是由客户端提供的，而是服务端根据客户端的ip指定的，当你的浏览器访问某个网站时，假设中间没有任何代理，那么网站的web服务器（Nginx，Apache等）就会把remote_addr设为你的机器IP，如果你用了某个代理，那么你的浏览器会先访问这个代理，然后再由这个代理转发到网站，这样web服务器就会把remote_addr设为这台代理机器的IP
`x_forwarded_for`
正如上面所述，当使用了代理，web服务器就不知道真实IP，为了避免这个情况，代理服务器通常会增加一个叫做x_forwarded_for的头信息，把连接它的客户端IP加到这个头信息里，这样就能保证网站的web服务器能获取到真实IP，一般只有在通过了HTTP 代理或者负载均衡服务器时才会添加该项。

`x_forwarded_for` 的HTTP头一般格式如下:
>X-Forwarded-For: client1, proxy1, proxy2, proxy3

其中的值通过一个 逗号+空格 把多个IP地址区分开, 最左边(client1)是最原始客户端的IP地址, 代理服务器每成功收到一个请求，就把请求来源IP地址添加到右边。 在上面这个例子中，这个请求成功通过了三台代理服务器：proxy1, proxy2 及 proxy3。请求由client1发出，到达了proxy3(proxy3可能是请求的终点)。请求刚从client1中发出时，XFF是空的，请求被发往proxy1；通过proxy1的时候，client1被添加到XFF中，之后请求被发往proxy2;通过proxy2的时候，proxy1被添加到XFF中，之后请求被发往proxy3；通过proxy3时，proxy2被添加到XFF中，之后请求的的去向不明，如果proxy3不是请求终点，请求会被继续转发。
正常情况下XFF中最后一个IP地址是最后一个代理服务器的IP地址, 这通常是一个比较可靠的信息来源。

<hr>
               	
#### 2.1.2埋点代码
在目标页面中插入一段javascript片段，这个片段往往被称为埋点代码。
```vbscript-html
<html>
<title>dyf web</title>
<body>
html内容
<script type="text/javascript">
    var _maq = _maq || [];
    _maq.push(['_setAccount', 'dyf_account']);

    (function() {
        var ma = document.createElement('script');
        ma.type = 'text/javascript';
        ma.async = true;
        ma.src = ('https:' == document.location.protocol ? 'https://' : 'http://') + '192.168.9.140/ma.js';
        var s = document.getElementsByTagName('script')[0];
        s.parentNode.insertBefore(ma, s);
    })();
</script>
</body>
</html>
```
这段代码的主要目的就是引入一个外部的js文件，方式是通过document.createElement方法创建一个script并根据协议（http或https）将src指向对应的ma.js，最后将这个element插入页面的DOM树上。注意ga.async = true的意思是异步调用外部js文件，即不阻塞浏览器的解析，待外部js下载完成后异步执行。

js统计脚本
整个脚本放在匿名函数里，确保不会污染全局环境。
```
(function () {
    var params = {};
    //Document对象数据
    if(document) {
        params.domain = document.domain || '';
        params.url = document.URL || '';
        params.title = document.title || '';
        params.referrer = document.referrer || '';
    }  
    //Window对象数据
    if(window && window.screen) {
        params.sh = window.screen.height || 0;
        params.sw = window.screen.width || 0;
        params.cd = window.screen.colorDepth || 0;
    }  
    //navigator对象数据
    if(navigator) {
        params.lang = navigator.language || '';
    }  
    //解析_maq配置
    if(_maq) {
        for(var i in _maq) {
            switch(_maq[i][0]) {
                case '_setAccount':
                    params.account = _maq[i][1];
                    break;
                default:
                    break;
            }  
        }  
    }  
    //拼接参数串
    var args = '';
    for(var i in params) {
        if(args != '') {
            args += '&';
        }  
        args += i + '=' + encodeURIComponent(params[i]);
    }  
	console.log(args);
    //通过Image对象请求后端脚本
    var img = new Image(1, 1);
    img.src = 'http://192.168.9.141/gif?' + args;
})();
```
　数据收集脚本被请求后会被执行，这个脚本一般要做如下几件事：
　　
  1、通过浏览器内置javascript对象收集信息，如页面title（通过document.title）、referrer（上一跳url，通过document.referrer）、用户显示器分辨率（通过windows.screen）、cookie信息（通过document.cookie）等等一些信息。
　　
  2、解析自定义数据，可能会包括用户自定义的事件跟踪、业务数据等。
　　
  3、将上面两步收集的数据按预定义格式解析并拼接。
　　
  4、请求一个后端脚本，将信息放在http request参数中携带给后端脚本。
　　
  javascript请求后端脚本常用的方法是ajax，但是ajax是不能跨域请求的。这里ma.js在被统计网站的域内执行，而后端脚本在另外的域，ajax行不通。一种通用的方法是js脚本创建一个Image对象，将Image对象的src属性指向后端脚本并携带参数，此时即实现了跨域请求后端。

### 2.2后端开发

#### 2.2.1设计日志格式
nginx配置文件中定义日志格式如下
>log_format logName "\$tempA $tempB ...";

>log_format log_dyf '{"msec":"$msec","remote_addr":"$remote_addr","domain":"$u_domain","url":"$u_url","title":"$u_title","referrer":"$u_referrer","sh":"$u_sh","sw":"$u_sw","cd":"$u_cd","lang":"$u_lang","http_user_agent":"$http_user_agent","trace":"$u_utrace","time_local":"$time_local","request":"$request","http_host":"$http_host","status":"$status","http_referer":"$http_referer","upstream_addr":"$upstream_addr","request_time":"$request_time","upstream_reponse_time":"$upstream_response_time","request_body":"$request_body","upstream_status":"$upstream_status","body_bytes_sent":"$body_bytes_sent","upstream_response_time":"$upstream_response_time","http_x_forwarded_for":"$http_x_forwarded_for"}';

　　注意这里以$u_开头的是我们系统自定义的变量，其它的是nginx内置变量。


#### 2.2.2 编写后端脚本
　　我们使用nginx的access_log做日志收集，不过有个问题就是nginx配置本身的逻辑表达能力有限，所以选用了`OpenResty`的lua扩展。  
　　`OpenResty ™` 是一个基于 Nginx 与 Lua 的高性能 Web 平台，其内部集成了大量精良的 Lua 库、第三方模块以及大多数的依赖项。用于方便地搭建能够处理超高并发、扩展性极高的动态 Web 应用、Web 服务和动态网关。其中的核心是通过ngx_lua模块集成了Lua，从而在nginx配置文件中可以通过Lua来表述业务。

>作者：章亦春（agentzh）

     http://openresty.org/
     
     关于ngx_lua可以参考：
     
     https://github.com/chaoslawful/lua-nginx-module。

　nginx.conf文件添加以下功能
```
location /gif{
	  #伪装成gif文件
      default_type 'image/gif';
     
      #本身关闭access_log，通过subrequest记录log
      access_log off;

	  access_by_lua '
      #用户跟踪cookie名为__utrace
      local uid = ngx.var.cookie__utrace;
      if not uid then
           #如果没有则生成一个跟踪cookie，算法为md5(时间戳+IP+客户端信息)
           uid = ngx.md5(ngx.now() .. ngx.var.remote_addr .. ngx.var.http_user_agent)
      end

      ngx.header["Set-Cookie"] = {"_utrace=" .. uid .. ""}
      
      if ngx.var.arg_domain then
         #通过subrequest到/i-log记录日志，将参数和用户跟踪cookie带过去
         ngx.location.capture("/i-log",{args = {
                           domain = ngx.var.arg_domain,
                           url = ngx.var.arg_url,
                           title = ngx.var.arg_title,
                           referrer = ngx.var.arg_referrer,
                           sh = ngx.var.arg_sh,
                           sw = ngx.var.arg_sw,
                           cd = ngx.var.arg_cd,
                           lang = ngx.var.arg_lang,
                           utrace = uid,
                           account = ngx.var.arg_account
                                }
                        })
       end
       ';

	   #此请求不缓存
       add_header Expires "Fri, 01 Jan 1980 00:00:00 GMT";
       add_header Pragma "no-cache";
       add_header Cache-Control "no-cache, max-age=0, must-revalidate";

       #返回一个1×1的空gif图片
       empty_gif;
}

location  /i-log{
	   #内部location，不允许外部直接访问
       internal;
       
       set_unescape_uri $u_domain $arg_domain;
       set_unescape_uri $u_url $arg_url;
       set_unescape_uri $u_title $arg_title;
       set_unescape_uri $u_referrer $arg_referrer;
       set_unescape_uri $u_sh $arg_sh;
       set_unescape_uri $u_sw $arg_sw;
       set_unescape_uri $u_cd $arg_cd;
       set_unescape_uri $u_lang $arg_lang;
       set_unescape_uri $u_utrace $arg_utrace;
       set_unescape_uri $u_acctount $arg_account;

	   #打开日志
       log_subrequest on;
       
       #记录日志
       access_log /home/wwwlogs/userInfo.log log_dyf;
		
	   #输出空字符串
       echo '';
        }
```
1.gif是一个伪装成gif的脚本。这种后端脚本一般要完成以下几件事情：
　　
  1、解析http请求参数的到信息。
　　
  2、从服务器（WebServer）中获取一些客户端无法获取的信息，如访客ip等。
　　
  3、将信息按格式写入log。
　　
  4、生成一副1×1的空gif图片作为响应内容并将响应头的Content-type设为image/gif。
　　
  5、在响应头中通过Set-cookie设置一些需要的cookie信息。
　　
  之所以要设置cookie是因为如果要跟踪唯一访客，通常做法是如果在请求时发现客户端没有指定的跟踪cookie，则根据规则生成一个全局唯一的cookie并种植给用户，否则Set-cookie中放置获取到的跟踪cookie以保持同一用户cookie不变（见图4）。 这种做法虽然不是完美的（例如用户清掉cookie或更换浏览器会被认为是两个用户），但是是目前被广泛使用的手段。
  
![Alt text|center](../master/src/1487847573762.png)

#### 2.2.3日志轮转
　　真正的日志收集系统访问日志会非常多，时间一长文件变得很大，而且日志放在一个文件不便于管理。所以通常要按时间段将日志切分，例如每天或每小时切分一个日志。我这里为了效果明显，每一小时切分一个日志。我是通过crontab定时调用一个shell脚本实现的，shell脚本如下：

>_prefix="/path/to/nginx"

>time=`date +%Y%m%d%H`

>mv \${_prefix}/logs/ma.log  \${_prefix}/logs/ma/ma-${time}.log

>kill -USR1 `cat ${_prefix}/logs/nginx.pid`


　　这个脚本将ma.log移动到指定文件夹并重命名为ma-{yyyymmddhh}.log，然后向nginx发送USR1信号令其重新打开日志文件。
　　然后再/etc/crontab里加入一行：
>59  *  *  *  * root /path/to/directory/rotatelog.sh

在每个小时的59分启动这个脚本进行日志轮转操作。
### 2.3测试
### 2.4展示、分析（todo）

 > 访问量（PV），访客数（UV）和独立IP数（IP），分不同业务数据分析。

　　通过上面的分析和开发可以大致理解一个网站统计的日志收集系统是如何工作的。有了这些日志，就可以进行后续的分析了。
　　
  注意，原始日志最好尽量多的保留信息而不要做过多过滤和处理。后面的系统根据原始日志可以分析出很多东西，例如通过IP库可以定位访问者的地域、user agent中可以得到访问者的操作系统、浏览器等信息，再结合复杂的分析模型，就可以做流量、来源、访客、地域、路径等分析了。当然，一般不会直接对原始日志分析，而是会将其清洗格式化后转存到其它地方，如MySQL或HBase中再做分析。
　　
  分析部分的工作有很多开源的基础设施可以使用，例如实时分析可以使用Storm，而离线分析可以使用Hadoop。



## 3日志系统流程实现
### 3.1安装nginx
>建议安装1.6以上
### 3.2安装Lua扩展
`ngx_lua_module` 是一个nginx http模块，它把 lua 解析器内嵌到 nginx，用来解析并执行lua 语言编写的网页后台脚本
>建议1.若服务器已经在运行，且ngxin版本低于1.6，先平滑升级到1.6及以上，再重新编译同版本的nginx，防止业务系统崩溃
>建议2.若服务器还未正式运行，可使用如下方法，直接下载新的高版本nginx进行编译安装

1.下载安装LuaJIT 2.1（2.0或者2.1都是支持的，官方推荐2.1）
http://luajit.org/download.html
```
cd /usr/local/src
wget http://luajit.org/download/LuaJIT-2.1.0-beta2.tar.gz
tar zxf LuaJIT-2.1.0-beta2.tar.gz
cd LuaJIT-2.1.0-beta2
make PREFIX=/usr/local/luajit
make install PREFIX=/usr/local/luajit
```
2.下载`ngx_devel_kit`（NDK）模块 ：
https://github.com/simpl/ngx_devel_kit/tags
```
cd /usr/local/src
wget https://github.com/simpl/ngx_devel_kit/archive/v0.2.19.tar.gz
tar -xzvf v0.2.19.tar.gz
```
3.下载最新的`lua-nginx-module` 模块 ：
https://github.com/openresty/lua-nginx-module/tags
```
cd /usr/local/src
wget https://github.com/openresty/lua-nginx-module/archive/v0.10.2.tar.gz
tar -xzvf v0.10.2.tar.gz
```
4.nginx -V查看已经编译的配置
```
nginx -V
```    
或者
```
/usr/local/nginx/sbin/nginx -V
```
笔者的配置如下：
```
./configure --user=www --group=www --prefix=/usr/local/nginx --with-http_stub_status_module --with-http_ssl_module --with-http_v2_module --with-http_gzip_static_module --with-ipv6 --with-http_sub_module
```

5.进入之前安装nginx的解压目录，重新编译安装(在nginx -V得到的配置下，加入`ngx_devel_kit-0.2.19`和`lua-nginx-module-0.10.2`的目录)：（重新编译nginx）
>注：如果是nginx1.6以下是版本先看下面升级nginx的方法先升级，否则安装不了

```
cd /usr/local/nginx/sbin
nginx -v
查看当前版本
 cd /usr/local/src/
 //下载大于1.6版本的nginx
 wget http://nginx.org/download/nginx-1.7.2.tar.gz
 tar -xzvf nginx-1.7.2.tar.gz
 cd nginx-1.7.2
//先导入环境变量,告诉nginx去哪里找luajit
export LUAJIT_LIB=/usr/local/luajit/lib
export LUAJIT_INC=/usr/local/luajit/include/luajit-2.1
 在本机的ngxin配置项后，添加 
    --add-module=/usr/local/src/ngx_devel_kit-0.2.19
    --add-module=/usr/local/src/lua-nginx-module-0.10.2
这两项进入配置并重新编译
 ./configure  （原本配置 + ndk + lua）。。。

例子（具体看当前服务器的nginx配置）：
./configure --user=www --group=www --prefix=/usr/local/nginx --with-http_stub_status_module --with-http_ssl_module --with-http_v2_module --with-http_gzip_static_module --with-ipv6 --with-http_sub_module --add-module=/usr/local/src/ngx_devel_kit-0.2.19 --add-module=/usr/local/src/lua-nginx-module-0.10.2
```
6.编译安装
>make -j2
>make install

7.查看是否编译成功在/usr/local/nginx/conf/nginx.conf中加入如下代码：
>location /hello_lua { 
>      default_type 'text/plain'; 
>      content_by_lua 'ngx.say("hello, lua")'; 
>}

重启nginx:
>service nginx restart

访问localhost/hello_lua会出现”hello, lua”表示安装成功

<br>
**注意事项：**
	1.下载nginx的版本，1.6以上的，否则安装完lua，nginx重新编译不了
	2.安装完成后，重启或者查看版本，都提示找不到 LuaJIT的 so 静态包，`error while loading shared libraries`原因是nginx找不到 luajit的lib的路径，解决办法:
>vim /etc/ld.so.conf
>加入/usr/local/luajit/lib这一行，
>/usr/local/luajit/lib

保存之后，再运行：/sbin/ldconfig –v更新一下配置即可。

>/sbin/ldconfig –v

3.在配置./configure 时，会出现     `./configure: error: ngx_http_lua_module requires the Lua library.`

找不到lua的静态库，解决办法就是下载：
>yum install lua-devel

4.在配置./configure时提示 `./configure: error: the HTTP rewrite module requires the PCRE library.`
>    yum install pcre-devel

5.缺少echo、set_unescape_uri功能原因：缺少`echo-nginx-module`、`set-misc-nginx-module`。
解决办法：
>git clone https://github.com/openresty/echo-nginx-module.git
>git clone https://github.com/openresty/set-misc-nginx-module.git

重新编译nginx
>./configure （之前的配置，通过 nginx -V获取）--add-module=/usr/local/src/ngx_devel_kit-0.2.19 --add-module=/usr/local/src/lua-nginx-module-0.10.2  --add-module=/usr/local/src/echo-nginx-module   --add-module=/usr/local/src/set-misc-nginx-module

6.也可直接安装`OpenResty`的源码包（包含了lua及其第三方库）OpenResty ™ 是一个基于 Nginx 与 Lua 的高性能 Web 平台，其内部集成了大量精良的 Lua 库、第三方模块以及大多数的依赖项。用于方便地搭建能够处理超高并发、扩展性极高的动态 Web 应用、Web 服务和动态网关。
    http://openresty.org/cn/

安装步骤：
>yum install readline-devel pcre-devel openssl-devel
>wget http://openresty.org/download/ngx_openresty-1.5.8.1.tar.gz
>tar xzvf ngx_openresty-1.5.8.1.tar.gz
>cd ngx_openresty-1.5.8.1/
>./configure --with-luajit
>make
>make install

###3.3编写前段代码

嵌入到html页面的埋点代码
```
<html>
<title>dyf web</title>
<body>
html内容
<script type="text/javascript">
    var _maq = _maq || [];
    _maq.push(['_setAccount', 'dyf_account']);

    (function() {
        var ma = document.createElement('script');
        ma.type = 'text/javascript';
        ma.async = true;
        ma.src = ('https:' == document.location.protocol ? 'https://' : 'http://') + '192.168.9.140/ma.js';
        var s = document.getElementsByTagName('script')[0];
        s.parentNode.insertBefore(ma, s);
    })();
</script>
</body>
</html>                    
```
收集数据的js代码

```
(function () {
    var params = {};
    //Document对象数据
    if(document) {
        params.domain = document.domain || '';
        params.url = document.URL || '';
        params.title = document.title || '';
        params.referrer = document.referrer || '';
    }  
    //Window对象数据
    if(window && window.screen) {
        params.sh = window.screen.height || 0;
        params.sw = window.screen.width || 0;
        params.cd = window.screen.colorDepth || 0;
    }  
    //navigator对象数据
    if(navigator) {
        params.lang = navigator.language || '';
    }  
    //解析_maq配置
    if(_maq) {
        for(var i in _maq) {
            switch(_maq[i][0]) {
                case '_setAccount':
                    params.account = _maq[i][1];
                    break;
                default:
                    break;
            }  
        }  
    }  
    //拼接参数串
    var args = '';
    for(var i in params) {
        if(args != '') {
            args += '&';
        }  
        args += i + '=' + encodeURIComponent(params[i]);
    }  
	console.log(args);
    //通过Image对象请求后端脚本
    var img = new Image(1, 1);
    img.src = 'http://192.168.9.139/gif?' + args;
})();

```

### 3.4编写后端（nginx.conf）代码

```
#普通日志记录
log_format srcache_log '$remote_addr - $remote_user [$time_local] "$request" "$status" $body_bytes_sent $request_time $bytes_sent $request_length '
                        '[$upstream_response_time] [$srcache_fetch_status] [$srcache_store_status] [$srcache_expire]';

server {
        access_log logs/access.log main;
        error_log logs/error.log;

        location / srcache {
                access_log logs/access-srcache.log srcache_log;
        }
}

#本文的日志记录
log_format log_dyf '{"msec":"$msec","remote_addr":"$remote_addr","domain":"$u_domain","url":"$u_url","title":"$u_title","referrer":"$u_referrer","sh":"$u_sh","sw":"$u_sw","cd":"$u_cd","lang":"$u_lang","http_user_agent":"$http_user_agent","trace":"$u_utrace","time_local":"$time_local","request":"$request","http_host":"$http_host","status":"$status","http_referer":"$http_referer","upstream_addr":"$upstream_addr","request_time":"$request_time","upstream_reponse_time":"$upstream_response_time","request_body":"$request_body","upstream_status":"$upstream_status","body_bytes_sent":"$body_bytes_sent","upstream_response_time":"$upstream_response_time","http_x_forwarded_for":"$http_x_forwarded_for"}';

location /gif{
    #伪装成gif文件
	default_type 'image/gif';
	
	#本身关闭access_log，通过subrequest记录log
	access_log off;
	
	access_by_lua '
	local uid = ngx.var.cookie__utrace;
	if not uid then
	    #如果没有则生成一个跟踪cookie，算法为md5(时间戳+IP+客户端信息)
		uid = ngx.md5(ngx.now() .. ngx.var.remote_addr .. ngx.var.http_user_agent)
	end

	ngx.header["Set-Cookie"] = {"_utrace=" .. uid .. ""}

	if ngx.var.arg_domain then
	    #通过subrequest到/i-log记录日志，将参数和用户跟踪cookie带过去
		ngx.location.capture("/i-log",{args = {
			domain = ngx.var.arg_domain,
			url = ngx.var.arg_url,
			title = ngx.var.arg_title,
			referrer = ngx.var.arg_referrer,
			sh = ngx.var.arg_sh,
			sw = ngx.var.arg_sw,
			cd = ngx.var.arg_cd,
			lang = ngx.var.arg_lang,
			utrace = uid,
			account = ngx.var.arg_account
			}
		})
	end
	';
	
	#此请求不缓存
	add_header Expires "Fri, 01 Jan 1980 00:00:00 GMT";
	add_header Pragma "no-cache";
	add_header Cache-Control "no-cache, max-age=0, must-revalidate";
	
	#返回一个1×1的空gif图片
	empty_gif;
}

location /i-log{
	#内部location，不允许外部直接访问
	internal;
	
	set_unescape_uri $u_domain $arg_domain;
	set_unescape_uri $u_url $arg_url;
	set_unescape_uri $u_title $arg_title;
	set_unescape_uri $u_referrer $arg_referrer;
	set_unescape_uri $u_sh $arg_sh;
	set_unescape_uri $u_sw $arg_sw;
	set_unescape_uri $u_cd $arg_cd;
	set_unescape_uri $u_lang $arg_lang;
	set_unescape_uri $u_utrace $arg_utrace;
	set_unescape_uri $u_acctount $arg_account;

	#打开日志
	log_subrequest on;
	
	#记录日志
	access_log /home/wwwlogs/userInfo.log log_dyf;
	
	echo '';
}

```

**internal**
`虽然禁止外部直接访问，并且会返回404，但是依旧会执行完代码`
语法：internal 
默认值：no 
使用字段： location 
internal指令指定某个location只能被“内部的”请求调用，外部的调用请求会返回”Not found” (404)
“内部的”是指下列类型：
指令error_page重定向的请求。 
一个防止错误页面被用户直接访问的例子：
>error_page 404 /404.html;
>location  /404.html {
> internal;
>}


### 3.5日志轮转
#### 3.5.1编写日志存储sh文件
```
#! /bin/bash
_prefix="/home/wwwlogs"
time=`date +%Y%m%d%H%M%S`
mv ${_prefix}/userInfo.log ${_prefix}/userInfoLogs/userInfo-${time}.log
kill -USR1 `cat /usr/local/nginx/logs/nginx.pid`
```
####3.5.2把sh文件加入crontab队列
```
     crontab -e
	  添加
      59 * * * * sh /home/logsh.sh
     
     service crond restart      启动
     service crond reload      启动
     或
     /bin/systemctl reload crond.service
     /bin/systemctl restart crond.service
```


#### 3.5.3定时任务的php实现
##### 3.5.3.1 workerman-crontab

https://github.com/shuiguang/workerman-crontab
>运行环境要求(php>=5.3.3)
>PHP必须支持exec函数

安装：

git clone https://github.com/shuiguang/workerman-crontab.git
启动守护进程
>php ./start.php start -d

创建新的定时任务组如果需要添加一组定时任务，组名为job1，那么可以在`./Applications/Crontab/cron_dir/`下创建`job1.crontab`，内容如下：
```
#案例1：每天22:00执行一次shell脚本
00 22 * * * www /www/cut-logs
#案例2：每分钟执行一次php脚本
* * * * * www /usr/local/php/bin/php /www/test.php
```
**备注：**
1.设置定时任务时间到秒级
     `./Applications/Crontab/Bootstrap/CrontabWorker.php`
>设置
>private static \$cron_standard = 'Y-m-d H:i';
>->
>private static \$cron_standard = 'Y-m-d H:i:s';

2.还有windows版(此处不做介绍 )
https://github.com/shuiguang/windows-crontab


##### 3.5.3.2 timephp
https://github.com/qq8044023/timePHP
>php版本要求5.6及以上

![Alt text|center](../master/src/1487905631941.png)


例：
1.在/task/crontab/clearroom/init.php 的_init函数，添加
>\$command="touch ".date("Ymd",time());
>shell_exec($command);

2.在/task/config.php文件修改clearroom的项
>"clearroom"=>[
>    "time"=>5, //间隔时间
>    "number"=>10,
>    "name"=>"clearroom"


## 4Todo
谷歌分析
http://www.google.com/analytics/
百度统计
http://tongji.baidu.com/
腾讯分析
http://ta.qq.com/

按照不同业务收集对应日志信息

日志展示系统

逻辑与业务大数据分析，反向指导业务


