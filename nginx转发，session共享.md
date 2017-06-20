
# nginx转发，session共享


在nginx转发后，通常会带来cookie和session的一些问题。本文实现的是一个在nginx转发后并保持session，让两个服务器公用一个session，从而让服务见进行一些数据通信、免去一些多余权限判断。

![|center](../master/src/1487846181327.png)



## 实现条件
>A服务器，负责nginx转发。

>B服务器，负责被转发后的代码实现

>Redis服务器，存储所有服务器产生的session（依据是本地浏览器cookie里的sessionId，由thinkphp产生）
    

## 过程：
1.在A服务器上的其他项目或其他部分代码进行一些操作，产生session并存储在redis里。

>	这里产生的session最好是thinkphp框架产生，因为tp框架会以其自身生成的sessionId为key将信息存储在本地浏览器cookie和redis服务器里。

> 去redis获取session需要根据sessionId、

> sessionId是在session_start()时，就会自动产生session_id()


2.在A服务器上访问nginx正则并转发的目录，会被nginx的 proxy_pass 转发到目标服务器（B服务器）的对应文件夹
3.B服务器上的代码实现一个功能，展示当前session的信息，便会拿当前服务器（是转发，不是跳转，所以虽然当前代码是以转发后的B服务器，但是“当前服务器”依旧是转发前的A服务器）的cookie里的sessionId去redis查询，会获得转发前就已经存储的session



## 简单实现部分：

A服务器的nginx的配置文件 /usr/local/nginx/conf/nginx.conf 里的server里添加以下代码，进行nginx转发
```
   location /ops/ {
                proxy_pass http://B服务器IP/think/public/;
        }
```

在A、B服务器的thinkphp框架的 /application/config.php 文件添加以下代码，实现session的redis化
```
    // +----------------------------------------------------------------------
    // | 会话设置
    // +----------------------------------------------------------------------

    'session'                => [
        'id'             => '',
        // SESSION_ID的提交变量,解决flash上传跨域
        'var_session_id' => '',
        // SESSION 前缀
        'prefix'         => 'think',
        // 驱动方式 支持redis memcache memcached
        'type'           => 'redis',
        // 是否自动开启 SESSION
        'auto_start'     => true,
        // redis主机
        'host'       => '192.168.9.142', //redis服务器的IP
        // redis端口
        'port'       => 6379,
        // 密码
        'password'   => '',
    ],
```

## 例：

A服务器的session写入代码
>http://192.168.9.142/think/public/?s=/index/index/set&key=user&value=20170227
index文件的set方法，实现session写入
```
  public function set()
    {
        $key = $_GET['key'];
        $vlaue = $_GET['value'];
        Session::set($key,$vlaue);
        echo "session的set函数,写入成功。<hr>获取".$key."，得到";
        return '';
    }

```

A服务器的nginx正则ops，并转发到B服务器进行解析
>http://192.168.9.142/ops/?s=/index/index/index
index文件的index方法
```
public  function  index(){
        Session::set('name','hahahahahhp');
        echo 'B服务器获取到的session  user ：' . Session::get('user');
    }
```