# 用nginx访问日志记录mysql数据库中的用户id



nginx有很强大的日志功能，但是在缺省状态下，它只能记录用户的IP地址以及浏览器信息。
	
如果我们有用户登录注册系统，在用户已登录的情况下，想记录访问某一个网页的到底是哪一个用户，怎么办呢？因为我们不只想知道到底是哪一个IP地址访问了哪一个网页，并且还想知道到底是哪一个登录用户访问了哪一个网页，

这对于我们日后有针对性地向他/她推荐信息甚至推送广告都是非常有用的。

### nginx缺省的日志格式

```
127.0.0.1 - - [20/Jul/2017:22:04:08 +0800] "GET /news/index HTTP/1.1" 200 22262 "-""Mozilla/5.0 (Macintosh; Intel Mac OS X 10_12_5) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/60.0.3112.66 Safari/537.36"
```

在这里，我们看到，虽然用户已经登录，但是日志里没有任何与用户相关的信息，只有ip地址。如果我们想记录用户的id等信息，怎么办呢？

### 在PHP端输出特殊的header


我们想到，既然用户已登录了，则它肯定有cookie或者session或者token信息，不管是哪种方式，我们的php一定是可以有效地获取到这个用户的信息的。

在这里举例我们通过session获取到了用户的id信息：


```
$user_id = Yii::$app->session['user_id'];
if (empty($user_id)) {
    header('X-UID: 0');
} else {
    header('X-UID: ' . $user_id);
}
```

如果session里没有用户id，则说明用户还没有登录，则输出X-UID: 0(或者也可以干脆什么也不输出)。

如果获取到了session，说明用户已登录，则我们把他的user_id输出给nginx: X-UID: 12345这样的形式。

在这里，你不止可以输出一个信息，你可以输出好几个不同的字段，包括他的姓名、性别、年龄等等都可以。

### 创建一种新的日志格式

`log_format`只能被存储在http段里，所以我们需要找到nginx.conf文件。

nginx缺省的日志格式第二部分就是用户信息，但通常什么也没有，只是一个-，这里我们它改造成我们从后端传进来的header信息。

由上文我们创造的特殊header是X-UID，这里需要先做一个小的转换，把大写字母全部改为小写，把所有的-改为下划线，就变成了x_uid，然后在前面拼接上$upstream_http_，就得到了最终的结果

`$upstream_http_x_uid`，然后把它插入到日志格式任何你想让它出现的地方：

```
log_format front '$remote_addr - $upstream_http_x_uid [$time_local] "$request" $status$body_bytes_sent "$http_referer" "$http_user_agent"';
```

### 在server里引用这种日志格式

在server相关的设置里，因为我们上面给日志格式起名为front，所以在这里我们引用它时，需要指明用front日志格式：

	access_log /var/log/nginx/front-access.log front;

新的日志结果

```
127.0.0.1 - 52248 [20/Jul/2017:22:35:40 +0800] "GET /news/view?id=56 HTTP/1.1" 200 19455 "-""Mozilla/5.0 (Macintosh; Intel Mac OS X 10_12_5) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/60.0.3112.66 Safari/537.36"
```

注意上面第2个数字52248，这就是我们登录用户的个人ID。

我这里的例子比较简单，如果你不嫌麻烦，甚至可以把登录用户的所有个人信息，包括手机号、邮箱全部打印在日志里，就看你是否顾虑安全问题了。

### 对用户隐藏id

在上面的第一步，我们用php输出了一个特殊header，本来我们这个header只是供nginx消费用的，但是这个header会被nginx原封不动地显示给前端，可能会有细心的用户感到不安。

为此我们可以在nginx的server设置里再加一个小开关，隐藏掉这个头部：

	proxy_hide_header X-UID;

这样用户从浏览器端就看不到这个特殊头部了，而并不影响nginx记录它。

### 最终处理

那么我们费这么大力气，记录下来一个ID有什么用呢？

这个用处可就大了。大家都知道我们有一个日志分析的利器`logstash(整套的话就是ELK)`，通过它结合上ELK组件可以分析处理Apache或者nginx日志。

如果我们没有这个ID信息的话，最多也只能分析出来哪一个网页经常被用户访问，仅此而已。

但现在我们有了用户ID，我们甚至可以连接mysql数据库表进行分析，研究哪一个年龄段的，哪一个性别的，或者哪一个城市的用户喜欢访问什么网页，甚至有针对性地了解具体某一个用户，他喜欢在什么时间段访问什么网页，进而有针对性地为他提供定制化的服务。