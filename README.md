
# Understanding CSRF

The Express team's [csrf](https://github.com/pillarjs/csrf) and [csurf](https://github.com/expressjs/csurf) modules
frequently have issues popping up concerned about our usage of cryptographic functions.
These concerns are unwarranted due to a misunderstanding of how CSRF tokens work.
So here's a quick run down!

Read this and still have questions? Want to tell us we're wrong? Open an issue!

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

### Use only JSON APIs

AJAX calls use JavaScript and are CORS-restricted.
There is no way for a simple `<form>` to send `JSON`,
so by accepting only JSON,
you eliminate the possibility of the above form.

### Disable CORS

The first way to mitigate CSRF attacks is to disable cross-origin requests.
If you're going to allow CORS,
only allow it on `OPTIONS, HEAD, GET` as they are not supposed to have side-effects.

Unfortunately, this does not block the above request as it does not use JavaScript (so CORS is not applicable).

### Check the referrer header

Unfortunately, checking the referrer header is a pain in the ass,
but you could always block requests whose referrer headers are not from your site.
This really isn't worth the trouble.

For example, you could not load sessions if the referrer header is not your server.

### GET should not have side effects

Make sure that none of your `GET` requests change any relevant data in your database.
This is a very novice mistake to make and makes your app susceptible to more than just CSRF attacks.

### Avoid using POST

Because `<form>`s can only `GET` and `POST`,
by using other methods like `PUT`, `PATCH`, and `DELETE`,
an attacker has fewer methods to attack your site.

### Don't use method override!

Many applications use [method-override](https://github.com/expressjs/method-override) to use
`PUT`, `PATCH`, and `DELETE` requests over a regular form.
This, however, converts requests that were previously invulnerable vulnerable!

Don't use `method-override` in your apps - just use AJAX!

### Don't support old browsers

Old browsers do not support CORS or security policies.
By disabling support for older browsers
(which more technologically-illiterate people use, who are more (easily) attacked),
you minimize CSRF attack vectors.

### CSRF Tokens

Alas, the final solution is using CSRF tokens.
How do CSRF tokens work?

1. Server sends the client a token.
2. Client submits a form with the token.
3. The server rejects the request if the token is invalid.

An attacker would have to somehow get the CSRF token from your site,
and they would have to use JavaScript to do so.
Thus, if your site does not support CORS,
then there's no way for the attacker to get the CSRF token,
eliminating the threat.

__Make sure CSRF tokens can not be accessed with AJAX!__
Don't create a `/csrf` route just to grab a token,
and especially don't support CORS on that route!

The token just needs to be "unguessable",
making it difficult for an attacker to successfully guess within a couple of tries.
It does not have to be cryptographically secure.
An attack is one or two clicks by an unbeknownst user,
not a brute force attack by a server.

## BREACH attack

This is where the salt comes along.
The BREACH attack is pretty simple: if the server sends the same or very similar response over `HTTPS+gzip` multiple times,
an attacker could guess the contents of response body (making HTTPS utterly useless).
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

Note that CSRF doesn't _solve_ the BREACH attack,
but the module simply randomizes requests to mitigate the BREACH attack for you.

## The salt doesn't have to be cryptographically secure

__Because the client knows the salt!!!__
The server will send `<salt>;<token>` and the client will return the same value to the server on a request.
The server will then check to make sure `<secret>+<salt>=<token>`.
The salt must be sent with the token,
otherwise the server can't verify the authenticity of the token.

This is the simplest cryptographic method.
There are more methods, but they are more complex and not worth the trouble.

## Creating tokens must be fast

__Because they are created on every request!__
Doing something as simple as `Math.random().toString(36).slice(2)` is sufficient as well as extremely performant!
You don't need OpenSSL to create cryptographically secure tokens on every request.

## The secret doesn't have to be secret

But it is.
If you're using a database-backed session store,
the client will never know the secret as it's stored on your DB.
If you're using cookie sessions,
the secret will be stored as a cookie and sent to the client.
Thus, __make sure cookie sessions use `httpOnly` so the client can't read the secret via client-side JavaScript!__

## When you're using CSRF tokens incorrectly

### Adding them to JSON AJAX calls

As noted above, if you don't support CORS and your APIs are strictly JSON,
there is absolutely no point in adding CSRF tokens to your AJAX calls.

### Exposing your CSRF token via AJAX

Don't ever create a `GET /csrf` route in your app
and especially don't enable CORS on it.
Don't send CSRF tokens with API response bodies.

## Conclusion

As the web moves towards JSON APIs and browsers become more secure with more security policies,
CSRF is becoming less of a concern.
Block older browsers from accessing your site and change as many of your APIs to be JSON APIs,
and you basically no longer need CSRF tokens.
But to be safe, you should still enable them whenever possible and especially when it's non-trivial to implement.
