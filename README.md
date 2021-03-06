# Stormpath Express.js SDK [BETA]

[![NPM Version](https://img.shields.io/npm/v/stormpath-sdk-express.svg?style=flat)](https://npmjs.org/package/stormpath-sdk-express)
[![NPM Downloads](http://img.shields.io/npm/dm/stormpath-sdk-express.svg?style=flat)](https://npmjs.org/package/stormpath-sdk-express)
[![Build Status](https://img.shields.io/travis/stormpath/stormpath-sdk-express.svg?style=flat)](https://travis-ci.org/stormpath/stormpath-sdk-express)

*A simple user authentication library for Express.js.*


[Stormpath](https://stormpath.com) is a User Management API that reduces
development time with instant-on, scalable user infrastructure.  Stormpath's
intuitive API and expert support make it easy for developers to authenticate,
manage, and secure users and roles in any application.

This library provides middleware for [Express.js] that will allow you to implement
token authentication strategies in your Express application.  This allows you
to authenticate an account with [Username and Password authentication] or
[Api Key Authentication] and return an access token with configurable scope
and TTL.  This token can be supplied by the client on subsequent requests and
can be used as an authentication mechanism.

This Beta
release is focused particularly on token authentication, other Stormpath
features may be expressed in the future.

If you are building an Angular application you can use [stormpath-angular] with
this library to create a robust API and Single Page App which leverages Stormpath
for user data.

If you are looking for a comprehensive web-application solution which includes a
server-side page templating system (but does not include token authentication
at this time) please visit our [stormpath-express] integration.

At the moment this library is focusing purely on authentication.  Authorization
(aka access control) is left in your control.  A common use case with Stormpath
is to use Groups as an access control feature.  You can leverage this by
working with the [`Account`](#Account) object that this library will assign to
`req.user`

If you have feedback about this library, please get in touch and share your
thoughts! support@stormpath.com




## Table of Contents

* [Usage](#usage)
 * [Installation](#usage-installation)
 * [Use with all routes](#usage-all)
 * [Use with some routes](#usage-some)
 * [Complex usage](#usage-complex)

* [`spConfig` Options](#spConfig)
  * [accessTokenCookieName](#access-token-cookie)
  * [accessTokenTtl](#accessTokenTtl)
  * [allowedOrigins](#allowedOrigins)
  * [endOnError](#error-handlers)
  * [forceHttps](#forceHttps)
  * [scopeFactory](#scopeFactory)
  * [tokenEndpoint](#tokenEndpoint)
  * [writeAccessTokenToCookie](#access-token-cookie)
  * [writeAccessTokenResponse](#access-token-response)
  * [xsrf](#XSRF)

* [Custom Token Strategies](#custom-token-strategies)

* [Middleware API](#middleware-api)
 * [authenticate](#authenticate)
 * [authenticateApiKeyForToken](#authenticateApiKeyForToken)
 * [authenticateBearerAuthorizationHeader](#authenticateBearerAuthorizationHeader)
 * [authenticateCookie](#authenticateCookie)
 * [authenticateForToken](#authenticateForToken)
 * [authenticateUsernamePasswordForToken](#authenticateUsernamePasswordForToken)
 * [logout](#logout)
 * [writeToken](#writeToken)

* [Middleware Factory](#middleware-factory)

* [Types](#types)
 * [Account](#Account)
 * [AuthenticationRequest](#AuthenticationRequest)
 * [Jwt](#Jwt)
 * [TokenResponse](#TokenResponse)




## <a name="usage"></a> Usage

#### <a name="usage-installation"></a> Install via NPM

```bash
npm install --save stormpath-sdk-express
```

The features in this library depend on cookies and POST body
data.  For that reason you should use [cookie-parser] or [body-parser] or
any other library which sets `req.cookies` and `req.body`.  If you need
to install them:

```bash
npm install --save cookie-parser body-parser
```




#### <a name="usage-all"></a> Use with all routes


To use the library with default options simply require it, create a default
middleware set and pass it to your app:

```javascript
var cookieParser = reqiure('cookie-parser');
var bodyParser = reqiure('body-parser');
var express = reqiure('express');
var stormpathExpressSdk = require('stormpath-express-sdk');

var spConfig = {
  appHref: 'YOUR_STORMPATH_APP_HREF',
  apiKeyId: 'YOUR_STORMPATH_API_KEY_ID',
  apiKeySecret: 'YOUR_STORMPATH_API_KEY_SECRET'
}

var spMiddleware = stormpathExpressSdk.createMiddleware(spConfig);

var app = express();

app.use(cookieParser());
app.use(bodyParser.json());
app.use(spMiddleware);
```

Doing this will enable the following functionality:

* All endpoints in your Express application will first be routed through the
[`authenticate`](#authenticate) middleware, which will assert that the user has
an existing access token.  If this is not true an error response will be sent.

* The endpoint `/oauth/token` will accept Client Password or Client Credential grant
requests and respond with an access token if authentication is successful.
Depending on the grant type (`password` or `client_credentials`) it delegates to
[`authenticateUsernamePasswordForToken`](#authenticateUsernamePasswordForToken)
or [`authenticateApiKeyForToken`](#authenticateApiKeyForToken).

* The `/logout` endpoint will be provided for ending cookie-based sessions.


* The [XSRF Protection](#XSRF) feature will be enabled.  After authentication,
all POST requests will validated with an XSRF token as well.


#### <a name="usage-some"></a> Use with some routes

If you dont need to authenticate all routes, but still want to use token
authentication, you can break it up like this:

```javascript
var spMiddleware = stormpathExpressSdk.createMiddleware(spConfig);

var app = express();

// Manually define the credential exchange endpoint

app.post('/tokens',spMiddleware.authenticateForToken);

// Enforce authentication on the API

app.get('/api/*',spMiddleware.authenticate,function(req,res,next){
  // If we get here, the user has been authentiated
  // The account object is available at req.user
});

// Allow anyone to use the public site

app.get('/public/*',function(){
  res.send('This is the public site, authentication not required');
});
```

#### <a name="usage-complex"></a> More complex usage

If you have a more complex use case, we suggest:

* Using the [`spConfig`](#spConfig) options to control features
* Custom chaining of specific middleware from the [Middleware API](#middleware-api)
* Create custom middleware using the [Middleware Factory Methods](#middleware-factory)
* Creating multiple instances of `spMiddleware`, with different [`spConfig`](#spConfig) options
* Using the Router object in Express 4.x to organize your routes


## <a name="spConfig"></a> `spConfig` options

The default beahvior of the middleware can be modified via the `spCpnfig` object
that you pass to `createMiddleware()`.  This section describes all the
available options and which middleware functions they affect.




#### <a name="accessTokenTtl"></a> Access Token TTL (seconds)

The default duration of access tokens is one hour.  Use `accessTokenTtl` to
set a different value, in seconds.  Once the token expires the client
will need to obtain a new token.

```javascript
var spConfig = {
  accessTokenTtl: 60 * 60 * 24 // One day (24 hours)
}
```

Used by [`authenticateUsernamePasswordForToken`](#authenticateUsernamePasswordForToken), [`authenticateApiKeyForToken`](#authenticateApiKeyForToken)




#### <a name="access-token-cookie"></a> Access Token Cookie

The default response to a successful password grant request is to create an
access token and write it to a secure cookie.  The name of the cookie will be
`access_token` and the value will be a JWT token.

Set `{ writeAccessTokenToCookie: false }` if you do not want to write the access
token to a cookie.

Use `accessTokenCookieName` to change the default name of the cookie.

**Example: disable access token cookies**
```javascript
var spConfig = {
  writeAccessTokenToCookie: false
}
```

**Example: custom access token cookie name**
```javascript
var spConfig = {
  accessTokenCookieName: 'custom_cookie_name'
}
```

Used by [`authenticateUsernamePasswordForToken`](#authenticateUsernamePasswordForToken), [`authenticateCookie`](#authenticateCookie)




#### <a name="access-token-response"></a> Access Token Response

By default this library will not send the access token in the
body of the response if the grant type is `password`.  Instead it will
write the access token to an HTTPS-only cookie and send 201 for the response status.

This is for security purposes: by exposing it in the body you expose it
to the javascript environment on the client - which is vulnerable to XSS
attacks.

You can disable this security feature by enabling the access token response writer:

```javascript
var spConfig = {
  writeAccessTokenResponse: true
}
```

When enabled the response body will be a [`TokenResponse`](#TokenResponse)

Used by [`authenticateUsernamePasswordForToken`](#authenticateUsernamePasswordForToken)




#### <a name="allowedOrigins"></a> Allowed Origins

This is a whitelist of domains that are allowed to make requests of your API.
If you are making cross-origin requests (CORS) to your server you will need to
whitelist the origin domains with this option.  This library will automatically
respond with the relevant response headers that are required by CORS.

```javascript
var spConfig = {
  allowedOrigins: ['example.com']
}
```

Used by `autoRouteHandler` via `app.use(stormpathMiddleware)`

#### <a name="error-handlers"></a> Error Handlers

By default the library will respond to failed authentication by ending the
response and sending an appropriate error code and message.

Set `endOnError` to false to disable this default behaviour.

In this case the library will assign the error to `req.authenticationError`
and continue the middleware chain.

```javascript
var spConfig = {
  endOnError: false
}
```

Used by `handleAuthenticationError`

**Example: custom error handling**

```javascript
app.get('/secrets',spMiddleware.authenticate,function(req,res,next){
  if(req.authenticationError){
    res.json(401,{
      error: req.authenticationError
    });
  }else{
    res.json(200,{
      message: 'Hello, ' + req.user.fullName
    });
  }
});
```



#### <a name="forceHttps"></a> HTTPS

This library auto-negotiates HTTP vs HTTPS.  If your server is accepting
HTTPS requests it will automatically add the `Secure` option to the
access token cookie.  This relies on [`req.protocol`][express.req.protocol] from the Express
framework.

If your server is behind a load balancer you should use the [`"trust proxy"`][trust-proxy-option]
option for Express.

If you cannot trust that your forward proxy is doing the right thing then you
should force this library to always use HTTPS

**Example: force HTTPS always**

```javascript
var spConfig = {
  forceHttps: true
}
```
Used by [`writeToken`](#writeToken)

#### <a name="scopeFactory"></a> Scope Factory

By default the library will not add scope to access tokens, we leave this
in your control.
Implement a scope factory if you wish to respond to requested scope by granting
specific scopes.  It will be called before the access token
is written.  You MUST call the done callback with the granted scope.  If
your logic determines that no scope should be given, simply call back
with a false-ey value.

If you need more control over the token creation, see
[Custom Token Strategies](#custom-token-strategies)

```javascript

var spConfig = {
  // ...
  scopeFactory: function(req,res,authenticationResult,account,requestedScope,done){

    // requestedScope is a string - passed in from the request

    var grantedScope = '';

    // do your logic, then..

    done(grantedScope)

  }
}
```




#### <a name="tokenEndpoint"></a> Token Endpoint

Defines the endpoint that is auto-bound to [`authenticateForToken`](#authenticateForToken).
The binding only happens if you call `app.use(spMiddleware)`.

```javascript
var spConfig = {
  tokenEndpoint: '/my/alt/token/endpoint'
}
```




#### <a name="XSRF"></a> XSRF Protection

This library combats [Cross-Site Request Forgery] by issuing XSRF tokens.
They are issued at the time an access token is created.  The value of the XSRF
token is written to the `XSRF-TOKEN` cookie and stored as the `xsrfToken` claim
within the access token.  Doing this allows the library to do a stateless
check when validating XSRF tokens.

Your client application, for any POST request, should supply the XSRF token
via the `X-XSRF-TOKEN: <token>` HTTP Header.  If you are using Angular.js this
will happen for you automatically.

If you do not want to issue or validate XSRF tokens, disable the feature:

```javascript
var spConfig = {
  xsrf: false
}
```




## <a name="custom-token-strategies"></a> Custom Token Strategies

This library will issue JWT access tokens with a configurable TTL and scope.
All other values (`sub`, `iat`, etc) are set automatically and
used for verifying the integrity of the token during authentication.

If you want to implement your own token responder (not recommended!) you can
do so by setting the `{ writeTokens: false }` option in [`spConfig`](#spConfig)

Doing this will prevent the library from automatically generating tokens and sending
them in responses.  Instead it will provide an [`AuthenticationRequest`](#AuthenticationRequest)
object at `req.authenticationRequest` and call `next()`. It is
up to you to generate a token and end the response.

**NOTE:** Disabling token creation will also disable the creation and
validation of XSRF tokens.


## <a name="middleware-api"></a> Middleware API

This section is a list of of middleware functions that are available
on the object that is returned by `createMiddleware(spConfig)`

When you call `createMiddleware(spConfig)` you are simply creating a set
of middleware functions which are pre-bound to the Stormpath Application that
you define with the `spConfig` options.  You can then mix-and-match these
middleware functions to create authentication logic which suits your
application.

**WARNING**: We have put a lot of thought into the default decisions of this
library.  While we provide this API for developer use it is required that
you understand the security trade-offs that you may be making by customizing
this library.  If you need any assistance please contact support@stormpath.com

### <a name="authenticate"></a> authenticate

This is a convenience middleware that will inspect the request and call
[`authenticateCookie`](#authenticateCookie) or
[`authenticateBearerAuthorizationHeader`](#authenticateBearerAuthorizationHeader) for you

**Example usage: authenticate specific routes**

````javascript
var app = express();

var spMiddleware = stormpathExpressSdk.createMiddleware({/* options */})

app.get('/something',spMiddleware.authenticate,function(req,res,next){
  res.json({message: 'Hello, ' + req.user.fullName});
});
````




### <a name="authenticateCookie"></a> authenticateCookie

Looks for an access token in the request cookies.

If authenticated, assigns an [`Account`](#Account) to `req.user` and provides
the unpacked access token at `req.accessToken` (an instance of [`Jwt`](#Jwt))

If an error is encountered it ends the response with an error.  If
using the option `{ endOnError: false}` it sets
`req.authenticationError` and continues the chain

**Example: use cookie authentication for a specific endpoint**
````javascript
var spMiddleware = stormpathExpressSdk.createMiddleware({/* options */})

var app = express();

app.get('/something',spMiddleware.authenticateCookie,function(req,res,next){
  res.json({
    message: 'Hello, ' + req.user.fullName + ', ' +
      'you have a valid access token in a cookie. ' +
      'Your token exires in: ' + req.accessToken.body.exp
    });
});
````




### <a name="authenticateBearerAuthorizationHeader"></a> authenticateBearerAuthorizationHeader

Looks for an access token in the `Authorization: Bearer <Base64(accesToken)>` header
on the request

If authenticated, assigns an [`Account`](#Account) to `req.user` and provides
the unpacked access token at `req.accessToken` (an instance of [`Jwt`](#Jwt))

If an error is encountered it ends the response with an error.  If
using the option `{ endOnError: false}` it sets
`req.authenticationError` and continues the chain

**Example: use bearer authentication for a specific endpoint**
````javascript
var spMiddleware = stormpathExpressSdk.createMiddleware({/* options */})
var app = express();

app.use(spMiddleware)

app.get('/something',spMiddleware.authenticateCookie,function(req,res,next){
  res.json({
    message: 'Hello, ' + req.user.fullName + ', ' +
      'you have a valid access token in your Authorization header. ' +
      'Your token exires in: ' + req.accessToken.body.exp
    });
});
````




### <a name="authenticateApiKeyForToken"></a> authenticateApiKeyForToken

This is used with the [Stormpath API Key Authentication Feature][Api Key Authentication].  It expects an
account's API Key and Secret to be supplied in HTTP headers via the HTTP
Basic scheme.


**Example: posting api key ID and secret to the token endpoint**
```
POST /oauth/tokens?grant_type=client_credentials
Authoriation: Basic <Base64(apiKeyId:apiKeySecret)>
```

If the supplied credentials are a valid API Key for an account, this function will respond
by writing a [`TokenResponse`](#TokenResponse) and ending the request.

**Example: accept only api keys for token exchange**
````javascript
var spMiddleware = stormpathExpressSdk.createMiddleware({/* options */})

var app = express();

app.post('/oauth/token',spMiddleware.authenticateApiKeyForToken);
````




### <a name="authenticateUsernamePasswordForToken"></a> authenticateUsernamePasswordForToken

Expects a JSON POST body which has a `username` and `password` field, and a
grant type request of `password`.

**Example: posting username and password to the token endpoint**
```
POST /oauth/tokens?grant_type=password

{
  "username": "foo@bar.com",
  "password": "aSuperSecurePassword"
}
```

If the supplied password is valid for the username, this function will respond
with an access token cookie or access token response, depending on the configuration
set by [`writeAccessTokenResponse`](#access-token-response) and
[`writeAccessTokenToCookie`](#access-token-cookie)

**Example: accept only username & password for token exchange**
````javascript
var spMiddleware = stormpathExpressSdk.createMiddleware({/* options */})

var app = express();

app.post('/login',spMiddleware.authenticateUsernamePasswordForToken);
````




#### <a name="authenticateForToken"></a> authenticateForToken

This is a convenience middleware that will inspect the request
and delegate to [`authenticateApiKeyForToken`](#authenticateApiKeyForToken) or
[`authenticateUsernamePasswordForToken`](#authenticateUsernamePasswordForToken)

**Example: manually defining the token endpoint**
````javascript
var spMiddleware = stormpathExpressSdk.createMiddleware({/* options */})

var app = express();

app.post('/tokens-r-us',spMiddleware.authenticateForToken);
````
Note: this example can also be accomplished with the `tokenEndpoint` option in
`spConfig`




#### <a name="writeToken"></a> writeToken

This method will write an access token cookie or access token body response,
as configured by the [`writeAccessTokenResponse`](#access-token-response) and
[`writeAccessTokenToCookie`](#access-token-cookie) options.

It expects that `req.jwt` has already been populated by one of the authentication
functions.

**Example: manually defining the token endpoint**
````
var spMiddleware = stormpathExpressSdk.createMiddleware({/* options */})

var app = express();

app.post('/tokens-r-us',spMiddleware.writeToken);
````
Note: example can also be accomplished with the `tokenEndpoint` option in `spConfig`




#### <a name="logout"></a> logout

This method will delete the XSRF and access token cookies on the client

**Example: manually defining a logout endpoint**
````
var spMiddleware = stormpathExpressSdk.createMiddleware({/* options */})

var app = express();

app.get('/logout',spMiddleware.logout);
````




## <a name="middleware-factory"></a> Middleware Factory Methods

If you only need certain features or middleware functions you can construct
the middleware functions manually.  All middleware functions can be constructed
by calling their constructor directly off the library export.  Simply
use a capital letter to access the constructor.  When calling the constructor
you must provide an `spConfig` object with relevent options.

**WARNING**: We have put a lot of thought into the default decisions of this
library.  While we provide this API for developer use it is required that
you understand the security trade-offs that you may be making but customizing
this library.  If you need any assistance please contact support@stormpath.com

For example: say you want one web service to issue access tokens for API
Authentication.  You then want all your other web services to read
these tokens and use them for authentication.

**Example: a token generation webservice**
```javascript
var express = require('express');
var stormpathSdkExpress = require('stormpath-sdk-express');

var spConfig = {
  appHref: '',
  apiKeyId: '',
  apiKeySecret: '',
}

var tokenExchanger = stormpathSdkExpress.AuthenticateApiKeyForToken(spConfig);

var app = express();

app.post('/oauth/tokens',tokenExchanger);
```

**Example: authenticate tokens in your other web services**
```javascript
var express = require('express');
var stormpathSdkExpress = require('stormpath-sdk-express');

var spConfig = {
  appHref: '',
  apiKeyId: '',
  apiKeySecret: '',
}

var authenticate = stormpathSdkExpress.Authenticate(spConfig);

var app = express();

app.use('/api/*',authenticate,function(req,res,next){
  // handle api request.  account will be available at req.user
});
```



## <a name="types"></a> Types

These are the object types that you will  find in this library.

### <a name="Account"></a> Account

This object is provided by the underlying [Stormpath Node SDK][stormpath-node-sdk],
it is documented here:

http://docs.stormpath.com/nodejs/api/account

### <a name="AuthenticationRequest"></a> AuthenticationRequest

This object is provided by the underlying [Stormpath Node SDK][stormpath-node-sdk],
it is documented here:

http://docs.stormpath.com/nodejs/api/authenticationResult

### <a name="Jwt"></a> Jwt

These are objects which represent a JWT token.  They have methods for manipulating
the token and compacting it to an encoded string.  These instaces are provided by
the [nJwt Library][nJwt].

### <a name="TokenResponse"></a> TokenResponse

This object encapsulates the compacted JWT, exposes the scope of the token,
and declares the expiration time as seconds from now.

**Example: token response format**
```json
{
  "access_token": "eyJ0eXAiOiJKV1QiLCJhbGc ...",
  "expires_in": 3600,
  "token_type": "Bearer",
  "scope": "given-scope"
}
```

## Contributing

In lieu of a formal style guide, take care to maintain the existing coding
style.  Add unit tests for any new or changed functionality.  Lint and test
your code using [Grunt](http://gruntjs.com/).

You can make your own contributions by forking the develop branch of this
repository, making your changes, and issuing pull request on the develop branch.

We regularly maintain this repository, and are quick to review pull requests and
accept changes!

We <333 contributions!


## Copyright

Copyright &copy; 2015 Stormpath, Inc. and contributors.

This project is open-source via the [Apache 2.0 License](http://www.apache.org/licenses/LICENSE-2.0).

[support@stormpth.com]: mailto:support@stormpath.com "Email Stormpath Support"
[Express.js]: http://expressjs.com/ "Express.js"
[cookie-parser]: https://github.com/expressjs/cookie-parser "Cookie Parser"
[body-parser]: https://github.com/expressjs/body-parser "Body Parser"
[nJwt]: https://github.com/jwtk/njwt "nJWt"
[stormpath]: https://stormpath.com "Stormpath"
[stormpath-angular]: https://github.com/stormpath/stormpath-angular "Stormpath Angular"
[stormpath-express]: https://github.com/stormpath/stormpath-express "Stormpath"
[stormpath-node-sdk]: https://github.com/stormpath/stormpath-sdk-node "Stormpath Node SDK"
[stormpath-api-key-docs]: http://docs.stormpath.com/guides/api-key-management/ "Stormpath Api Key Management"
[Username and Password authentication]: http://docs.stormpath.com/rest/product-guide/#authenticate-an-account "Username and Password authentication"
[Api Key Authentication]: http://docs.stormpath.com/guides/api-key-management/ "Api Key Authentication"
[Cross-Site Request Forgery]: https://www.owasp.org/index.php/Cross-Site_Request_Forgery "Cross-Site Request Forgery (CSRF)"
[trust-proxy-option]: http://expressjs.com/4x/api.html#trust.proxy.options.table
[express.req.protocol]: http://expressjs.com/4x/api.html#req.protocol