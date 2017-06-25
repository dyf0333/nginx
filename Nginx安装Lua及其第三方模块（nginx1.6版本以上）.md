
`ngx_lua_module` 是一个nginx http模块，它把 lua 解析器内嵌到 nginx，用来解析并执行lua 语言编写的网页后台脚本

```
建议1.若服务器已经在运行，且ngxin版本低于1.6，先平滑升级到1.6及以上，再重新编译同版本的nginx，防止业务系统崩溃
建议2.若服务器还未正式运行，可使用如下方法(步骤5)，直接下载新的高版本nginx进行编译安装
```

## 1.下载安装LuaJIT 2.1（2.0或者2.1都是支持的，官方推荐2.1）
    
    http://luajit.org/download.html
```
cd /usr/local/src
wget http://luajit.org/download/LuaJIT-2.1.0-beta2.tar.gz
tar zxf LuaJIT-2.1.0-beta2.tar.gz
cd LuaJIT-2.1.0-beta2
make PREFIX=/usr/local/luajit
make install PREFIX=/usr/local/luajit
```

## 2.下载ngx_devel_kit（NDK）模块 ：https://github.com/simpl/ngx_devel_kit/tags，不需要安装

```
cd /usr/local/src
wget https://github.com/simpl/ngx_devel_kit/archive/v0.2.19.tar.gz
tar -xzvf v0.2.19.tar.gz
```

## 3.下载最新的lua-nginx-module 模块 ：https://github.com/openresty/lua-nginx-module/tags，不需要安装

```
cd /usr/local/src
wget https://github.com/openresty/lua-nginx-module/archive/v0.10.2.tar.gz
tar -xzvf v0.10.2.tar.gz
```


## 4.nginx -V查看已经编译的配置

```
nginx -V
或者
/usr/local/nginx/sbin/nginx -V
```

笔者的配置如下：
```
./configure --user=www --group=www --prefix=/usr/local/nginx --with-http_stub_status_module --with-http_ssl_module --with-http_v2_module --with-http_gzip_static_module --with-ipv6 --with-http_sub_module
```

## 5.进入之前安装nginx的解压目录，重新编译安装(在nginx -V得到的配置下，加入ngx_devel_kit-0.2.19和lua-nginx-module-0.10.2的目录)：

（重新编译nginx）
注：如果是nginx1.6以下是版本先看下面升级nginx的方法先升级，否则安装不了
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

## 6.编译安装
```
make -j2
make install
```

## 7.查看是否编译成功

在/usr/local/nginx/conf/nginx.conf中加入如下代码：
```
location /hello_lua {
      default_type 'text/plain';
      content_by_lua 'ngx.say("hello, lua")';
}
```
重启nginx:
```
service nginx restart
```
访问localhost/hello_lua会出现”hello, lua”表示安装成功

hello, lua

#### 注意事项：

#### 1.下载nginx的版本，1.6以上的，否则安装完lua，nginx重新编译不了

#### 2.安装完成后，重启或者查看版本，都提示找不到 LuaJIT的 so 静态包，

    error while loading shared libraries

原因是nginx找不到 luajit的lib的路径，解决办法：

    vim /etc/ld.so.conf

加入/usr/local/luajit/lib这一行，

/usr/local/luajit/lib

保存之后，再运行：/sbin/ldconfig –v更新一下配置即可。

/sbin/ldconfig –v

#### 3.在配置./configure 时，会出现

    ./configure: error: ngx_http_lua_module requires the Lua library.

找不到lua的静态库，解决办法就是下载：

    yum install lua-devel

#### 4.在配置./configure时

提示 ./configure: error: the HTTP rewrite module requires the PCRE library.

    yum install pcre-devel

#### 5.缺少echo、set_unescape_uri功能

原因：缺少echo-nginx-module、set-misc-nginx-module。
解决办法：
```
git clone https://github.com/openresty/echo-nginx-module.git
git clone https://github.com/openresty/set-misc-nginx-module.git
```
//重新编译nginx
./configure （之前的配置，通过 nginx -V获取）--add-module=/usr/local/src/ngx_devel_kit-0.2.19 --add-module=/usr/local/src/lua-nginx-module-0.10.2  --add-module=/usr/local/src/echo-nginx-module  --add-module=/usr/local/src/set-misc-nginx-module

#### 6.也可直接安装OpenResty的源码包（包含了lua及其第三方库）

OpenResty ™ 是一个基于 Nginx 与 Lua 的高性能 Web 平台，其内部集成了大量精良的 Lua 库、第三方模块以及大多数的依赖项。用于方便地搭建能够处理超高并发、扩展性极高的动态 Web 应用、Web 服务和动态网关。

    http://openresty.org/cn/

安装步骤：
````
yum install readline-devel pcre-devel openssl-devel
wget http://openresty.org/download/ngx_openresty-1.5.8.1.tar.gz
tar xzvf ngx_openresty-1.5.8.1.tar.gz
cd ngx_openresty-1.5.8.1/
./configure --with-luajit
make
make install
````
