
# 理解CSRF(跨站请求伪造)
>原文出处[Understanding CSRF](https://github.com/pillarjs/understanding-csrf)

对于Express团队的[csrf](https://github.com/pillarjs/csrf)模块和[csurf](https://github.com/pillarjs/csurf)模块的加密函数的用法我们经常有一些在意。
这些在意是莫须有的，因为他们不了解CSRF token是如何工作的。
下面快速过一遍！

读过后还有疑问？希望告诉我们错误？请开一个issue！

## 一个CSRF攻击是如何工作的？

在他们的钓鱼站点，攻击者可以通过创建一个AJAX按钮或者表单来针对你的网站创建一个请求：

```html
<form action="https://my.site.com/me/something-destructive" method="POST">
  <button type="submit">Click here for free money!</button>
</form>
```

这是很危险的，因为攻击者可以使用其他http方法例如 `delete` 来获取结果。
这在用户的session中有很多关于你的网站的详细信息时是相当危险的。
如果一个不懂技术的用户遇到了，他们就有可能会输入信用卡号或者个人安全信息。

## 如何减轻CSRF攻击？

### 只使用JSON api

使用JavaScript发起AJAX请求是限制跨域的。
不能通过一个简单的`<form>`来发送`JSON`，
所以，通过只接收JSON，你可以降低发生上面那种情况的可能性。

### 禁用CORS

第一种减轻CSRF攻击的方法是禁用cross-origin requests(跨域请求)。
如果你希望允许跨域请求，那么请只允许 `OPTIONS, HEAD, GET` 方法，因为他们没有副作用。

不幸的是，这不会阻止上面的请求由于它没有使用JavaScript(因此CORS不适用)。

### 检验referrer头部

不幸的是，检验referrer头部很麻烦，
但是你可以阻止那些referrer头部不是来自你的页面的请求。
这实在不值得麻烦。

举个例子，你不能加载session如果这个请求的referrer头部不是你的服务器。

### GET总是幂等的

确保你的`GET`请求不会修改你数据库中的相关数据。
这是一个初学者常犯的错误，使得你的应用不仅是易于遭受CSRF攻击。

### 避免使用POST

因为`<form>`只能用`GET`或是`POST`,
而不能使用别的方法，例如`PUT`, `PATCH`, `DELETE`，
攻击者很难有方法攻击你的网站。

### 不要复写方法

许多应用程序使用[复写方法](https://github.com/expressjs/method-override)来在一个常规表单中使用`PUT`, `PATCH`, 和`DELETE`请求。
这会使得原先不易受攻击的方法变得易受攻击。

### 不要兼容旧浏览器

旧的浏览器不支持CORS或是其他安全政策。
通过不兼容旧浏览器
(那些不懂技术的人用的越多，我们越容易被攻击),
你可以最小化受到攻击的可能性。

### CSRF Tokens

最终的解决办法是使用CSRF tokens。
CSRF tokens是如何工作的呢？

1. 服务器发送给客户端一个token。
2. 客户端提交的表单中带着这个token。
3. 如果这个token不合法，那么服务器拒绝这个请求。

攻击者需要通过某种手段获取你站点的CSRF token，
他们只能使用JavaScript来做。
所以，如果你的站点不支持CORS，
那么他们就没有办法来获取CSRF token，
降低了威胁。

__确保CSRF token不能通过AJAX访问到!__
不要创建一个`/CSRF`路由来获取一个token，
尤其不要在这个路由上支持CORS!

token需要是不容易被猜到的，
让它很难被攻击者尝试几次得到。
它不需要是密码安全的。
攻击来自从一个未知的用户的一次或者两次的点击，
而不是来自一台服务器的暴力攻击。

## BREACH攻击

这也就是salt(加盐)出现的原因。
Breach攻击相当简单：如果服务器通过`HTTPS+gzip`多次发送相同或者相似的响应，攻击者就可以猜测响应的内容(使得HTTPS完全无用)。
解决办法？让每一个响应都有那么一点不同。
于是，CSRF tokens依据每一个不同的请求还有不同的时间来生成。
但是服务器需要知道客户端请求中带的token是否是合法的。
因此：

1. 一般认为安全加密的CSRF tokens是防护CSRF的关键
2. CSRF tokens现在通常是一个秘钥或者salt的hash


了解更多:

- [BREACH][1]
- [CRIME](http://en.wikipedia.org/wiki/CRIME)
- [Defending against the BREACH Attack](https://community.qualys.com/blogs/securitylabs/2013/08/07/defending-against-the-breach-attack)

[1]: http://en.wikipedia.org/wiki/BREACH_(security_exploit)

注意，CSRF没有_解决_BREACH攻击，
但是这个模块通过随机化请求来为你减轻BREACH攻击。

## salt不需要加密

__因为客户端知道salt!!!__
服务器会发送 `<salt>;<token>` ，然后客户端会通过请求返回相同的值给服务器。服务器然后会检验 `<secret>+<salt>=<token>` 。
salt必须跟token一起被发送给服务器，否则服务器不能验证这个token。
这是最简单的加密方式。
还有很多方法，不过他们更加复杂，犯不着那么麻烦。

## 创建tokens必须要快

__因为每当进来一个请求他们就会被创建!__
像`Math.random().toString(36).slice(2)`这么做也是性能足够好的!
你不需要OpenSSL来为每一个请求创建一个密码安全的token。

## 秘钥不需要是加密的，但需要是安全的

如果你正在使用一个数据库后端来存储session，客户端是不会知道秘钥的，因为它被存储在数据库中。
如果你正在使用cookie来存储session，那么秘钥就会被存储在cookie中发送给客户端。
因此， __确保cookie sessions 使用 `httpOnly` 那样客户端就不能通过客户端JavaScript来读取到秘钥!__

## 当你不正确的使用CSRF token

### 把它们加到JSON AJAX调用中

正如上面提到的，如果你不支持CORS并且你的API是传输的严格的JSON，
绝没可能在你的AJAX 调用中加入CSRF token。

### 通过AJAX暴露你的CSRF token

不要创建一个`GET /csrf`路由
并且尤其不要在这个路由上支持CORS。
不要发送CSRF token在API响应的body中。

## 结论

因为web正在向JSON API转移，并且浏览器变得更安全，有更多的安全策略，
CSRF正在变得不那么值得关注。
阻止旧的浏览器访问你的站点，并尽可能的将你的API变成JSON API，
然后你将不再需要CSRF token。
但是为了安全起见，你还是应该尽量允许他们尤其是当难以实现的时候。
