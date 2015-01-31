
There's been some concern around the Express team's [csrf](https://github.com/pillarjs/csurf) module,
specifically around how cryptographically secure the tokens are created.
This concern is unwarranted is due to a misunderstanding of how CSRF tokens work.
So here's a quick run down!

## How does a CSRF attack work?

On their own (phishing site), an attacker could create an AJAX button or form that creates a request against your site:

```html
<form action="https://my.site.com/me/something-destructive" method="POST">
  <button type="submit">Click here for free money!</button>
</form>
```

This is worse with AJAX as the attacker could use other methods like `DELETE` as well as read the result.
This is particularly important when the user has some sort of session with very personal details on your site.
If this is in the context of a technologically illiterate user,
there might be some inputs for credit card and social security info.

## How to mitigate CSRF attacks?

### Disable CORS

The first way to mitigate CSRF attacks is to disable cross-origin requests.
Don't allow cross-origin requests!
If you're going to allow CORS,
only allow it on `OPTIONS, HEAD, GET` and don't support sessions or cookies!
There goes the majority of the attack vector!

One recommendation is to always delete cookies from cross-origin requests before you do any business logic with these requests.
Another is to check the `referrer` header.

### Don't support old browsers

But remember, CSRF was an issue before CORS was ever implemented
and is a big issue for browsers that don't support CORS.
As far as I know, this pretty much means anything older than IE10.

Thus, you could just not support old browsers!
This means some special user-agent sniffing.
If you simply do not allow users on old browsers to login or save any personal information,
then you're good!

### CSRF Tokens

Alas, the final solution is using CSRF tokens.
How do CSRF tokens work?

1. Server sends the client a token.
2. Client submits a form with the token.
3. The server rejects the request if the token is invalid.

Basically, a attacker would need to get a valid token somehow by making an AJAX request,
then add the token to the CSRF token.
If you allow CORS, this is easy for a attacker to do if you expose the CSRF token to the client in any way (which you must do).
__DISABLE CORS!!!__
But for older browsers, this is pretty simple as long as the attack knows where to get a CSRF token.

The token just needs to be "unguessable",
making it difficult for a attacker to successful within a couple of tries.

The last thing you want is an easy way for an attacker to get a CSRF token.
Don't create a `/csrf` route just to grab a token.

## BREACH attack

This is where the salt comes along.
The BREACH attack is pretty simple: if the server sends the same or very similar response over HTTPS+gzip multiple times,
an attacker could guess the contents of response body.
Solution? Make each response a tiny bit different.

Thus, CSRF tokens are generated on a per-request basis and different every time.
But the server needs to know that any token included with a request is valid.
Thus:

1. Crytographically secure CSRF tokens are now the CSRF "secret", (supposedly) only known by the server.
2. CSRF tokens are now a hash of the secret and a salt.

Read more here:

- [BREACH][1]
- [CRIME](http://en.wikipedia.org/wiki/CRIME)
- [Defending against the BREACH Attack](https://community.qualys.com/blogs/securitylabs/2013/08/07/defending-against-the-breach-attack)

[1]: http://en.wikipedia.org/wiki/BREACH_(security_exploit)

## The salt doesn't have to be cryptographically secure

__Because the client knows the salt!!!__
The server will send `<salt>;<token>` and the client will return the same value to the server on a request.
The server will then check to make sure `<secret>+<salt>=<token>`.
The salt must be sent with the token,
otherwise the server can't verify the authenticity of the token.

This is the simplest cryptographical method.
There are more methods, but they are more complex and not worth the trouble.

## Creating tokens must be fast

__Because they are created on every request!__
Some people want the salts to be generated from `crypto.randomBytes()`.
The problem with this is that if the server runs out of entropy,
then requests will be unnecessarily slow.
So some suggest `crypto.pseudRandomBytes()`,
but this is unnecessarily slow as well.

> Note: this is no longer the case with the latest version of io.js.

Thus, doing something as simple as `Math.random().toString(36).slice(2)` is sufficient as well as extremely performant!

## The secret doesn't have to be secret

But it is.
If you're using a database-backed session store,
the client will never know the secret as it's stored on your DB.
If you're using cookie sessions,
the secret will be stored as a cookie and sent to the client.
Thus, __make sure cookie sessions use `httpOnly` so the client can't read the secret via client-side JavaScript!__

Even if an attack could somehow grab the `secret`,
it would have to go through CORS requests as well.

## CSRF attacks are only relevant in the context of browsers

If you're using the browser's development tools or any other tools plebeians don't use,
then your concern isn't really valid.
There are a billion ways to attack sites through various tools,
but the main concern of CSRF is to stop these types of attacks targeting the technologically illiterate using older browsers.
A major assumption of CSRF tokens is that the attack doesn't have direct access to the victim's computer,
otherwise s/he has better ways to do damage than through CSRF.

## CSRF tokens will eventually be unnecessary

With modern browsers supporting CORS and security policies,
CSRF tokens will eventually goes away.
Don't worry too much about CSRF tokens.
For now, disallow IE9 and lower browsers from storing confidential information to your site.
But to be safe, you should still enable them whenever possible and especially when it's non-trivial to implement.
