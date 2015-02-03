
# 理解CSRF(跨站请求伪造)
>原文出处[Understanding CSRF](https://github.com/pillarjs/understanding-csrf)

对于Express团队的[csrf](https://github.com/pillarjs/csurf)模块有一些关注，尤其是围绕着如何安全的生成token。
这些关注是莫须有的，因为他们不了解CSRF token是如何工作的。
下面快速过一遍！

读过后还有疑问？希望告诉我们错误？请开一个issue！


## 一个CSRF攻击是如何工作的？
在他们的钓鱼站点，攻击者可以针对你的网站创建一个请求通过创建一个AJAX按钮或者表单：

```html
<form action="https://my.site.com/me/something-destructive" method="POST">
  <button type="submit">Click here for free money!</button>
</form>
```

这是很危险的，因为攻击者可以使用其他http方法例如 `delete` 来获取结果。
这在用户的session中有很多关于你的网站的详细信息时是相当危险的。如果一个不懂技术的用户遇到了，他们就有可能会输入信用卡号或者个人安全信息。



## 如果减轻CSRF攻击？
### 禁用CORS
第一种减轻CSRF攻击的方法是禁用cross-origin requests(跨域请求)。
不要允许跨域请求!
如果你希望允许跨域请求，那么请只允许 `OPTIONS, HEAD, GET` 方法，并且不要支持sessions或者cookies!
他们是攻击者的主要目标。

一种推荐的做法是总是删除来自跨域请求的cookies在你处理来自这些请求的任何业务逻辑之前。
另外一种是检验header中的带的 `referrer` 。


### 不要兼容旧浏览器
但是请记住，CSRF在CORS实现之前就有。
并且对于不支持CORS的浏览器来说，这是一个大问题。
就我所知道的，这意味着IE10以前的所有浏览器。

因此，你可以不支持那些旧的浏览器！
这意味着查看那些特别的 `user-agent` 。
如果你只是不允许那些使用旧浏览器的用户登录或者存储任何个人信息，那也是可以的！

### CSRF Tokens

最终的解决办法是使用CSRF tokens。
CSRF tokens是如何工作的呢？

1. 服务器发送给客户端一个token。
2. 客户端提交的表单中带着这个token。
3. 如果这个token不合法，那么服务器拒绝这个请求。

因此，攻击者需要通过谋者办法获得一个合法的token来构造一个AJAX请求。
如果你允许了CORS，那么如果攻击者做起来就很简单，如果你暴露了token给客户端通过某种方式(显然你会把token发给客户端)。

__禁用 CORS!!!__
但是对于旧的浏览器，这相当简单，只要攻击者知道可以从哪里获取一个CSRF token。

这个token需要是不可猜测的，是它不会被攻击者尝试几次就猜出来。

最后一件事是不要创建一个类似 `/csrf` 之类的路由来获取token，那么攻击者就不会很容易获取一个CSRF token。


## BREACH攻击
这也就是salt(加盐)出现的原因。
Breach攻击相当简单：如果服务器通过HTTPS+gzip多次发送相同或者相似的响应，攻击者就可以猜测响应的内容。解决办法？让每一个响应都有那么一点不同。
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

## salt不需要加密
__因为客户端知道salt!!!__
服务器会发送 `<salt>;<token>` ，然后客户端会通过请求返回相同的值给服务器。服务器然后会检验 `<secret>+<salt>=<token>` 。
salt必须跟token一起被发送给服务器，否则服务器不能验证这个token。
这是最简单的加密方式。
还有很多方法，不过他们更加复杂，犯不着那么麻烦。

## 创建tokens必须要快
__因为每当进来一个请求他们就会被创建!__
一些人希望salt通过 `crypto.randomBytes()` 来创建。
这个方法的问题在于，服务器会消耗很多资源，那么，响应速度就会不必要的变慢。
因此，一些人建议用 `crypto.pseudRandomBytes()` ,但是这也同样会导致不必要的变慢。

>注意: 在最新版本的io.js中不再有这个问题

因此，类似 `Math.random().toString(36).slice(2)` 这种简单操作是相当高效的。

## 秘钥不需要是加密的，但需要是安全的
如果你正在使用一个数据库后端来存储session，客户端是不会知道秘钥的，因为它被存储在数据库中。
如果你正在使用cookie来存储session，那么秘钥就会被存储在cookie中发送给客户端。
因此， __确保cookie sessions 使用 `httpOnly` 那样客户端就不能通过客户端JavaScript来读取到秘钥!__

即使攻击者可以通过某种手段获取到`secret`，他也必须要通过CORS请求。

## CSRF攻击只和浏览器有关
如果你正在使用浏览器的开发工具或者其他一般人不会使用到的工具，那你的担心是没有道理的。
使用不同的工具有很多种攻击网站的办法，但是CSRF的主要关注点是阻止那些对于不懂技术又使用旧浏览器的使用者的攻击。对于CSRF tokens的一个主要假设是，攻击者不会接触到的受害者的电脑，否则他有比CSRF更好的办法去攻击受害者的电脑。


## CSRF tokens最终会变得不是那么必要
随着现代浏览器支持CORS以及安全策略，CSRF tokens最终会消失。
不用太担心CSRF tokens。
对于现在来说，禁止IE9及其以下的浏览器存储你网站的机密信息。
但是为了安全起见，你还是应该尽量允许他们尤其是当难以实现的时候。
