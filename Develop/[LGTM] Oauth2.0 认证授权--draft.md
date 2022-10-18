## [LGTM] Oauth2.0 认证授权--draft

### JWT： only  token format

**JWT (JSON Web Tokens)**- It is just a token format. JWT tokens are JSON encoded data structures contains information about issuer, subject (claims), expiration time etc. 

 the encoding rules of a JWT also make these tokens very easy to use within the context of HTTP.

https://jwt.io/   测试JWT数据

JSON Web Token (JWT) is a compact, URL-safe means of representing claims to be transferred between two parties.  The claims in a JWT are encoded as a JSON object that is used as the payload of a JSON Web Signature (JWS) structure or as the plaintext of a JSON Web Encryption (JWE) structure, enabling the claims to be digitally signed or integrity protected with a Message Authentication Code (MAC) and/or encrypted.

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/jwt-arch.png)

#### 数据结构

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/Structure of JWT.jpg)



#### 应用场景

If you have very simple scenarios, like a single client application, a single API then it might not pay off to go OAuth 2.0, on the other hand, lots of different clients (browser-based, native mobile, server-side, etc) then sticking to OAuth 2.0 rules might make it more manageable than trying to roll your own system.

### OAuth 2.0：a protocol

OAuth 不是一个API或者服务，而是一个验证授权(Authorization)的开放标准，所有人都有基于这个标准实现自己的OAuth。

在OAuth之前，HTTP Basic Authentication, 即用户输入用户名，密码的形式进行验证, 这种形式是不安全的。OAuth的出现就是为了解决访问资源的安全性以及灵活性。OAuth使得第三方应用对资源的访问更加安全。

#### workflow

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/oauth-2-workflow.jpg)

1. Authorization Request（Client to User）：用户打开client后，client要求用户给予授权。eg: 微信上任意一个小程序的登陆
2. Authorization Grant（User to Client）：小程序client请求用户同意给予客户端授权。eg：用户可以选择微信小程序读取用户头像、昵称、性别等信息。
3. Authorization Grant（Client to Auth Server）： 用户同意小程序client的全部授权请求或者自定义授权选项，微信小程序client根据用户最终给予的授权信息向认证服务申请令牌Tokens
4. Access Token（Auth Server to Client）:认证服务器对client进行认证，确认无误后，同意发放一定权限的令牌
5. Access Token（client to Resource server）：client使用令牌向资源服务器申请相关客户已经许可的资源
6. Protected Resource（ Resource server to client）：资源服务器确认Token，同意向client开发相关资源

好理解的图

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/oauth-2-workflow-2.jpg)



#### Tokens

OAuth or its v2.0 is all about tokens. Hence, it’s crucial to understand what the term means. In OAuth, two token kinds exist.  

An **access token** is shared as a request header or parameter by the client. It can permit the 3rd party application to approach user data present on the resource server. The time-constraint feature of the token helps the client (app) define an app usage/access limit for 3rd party resources. While one tries to use it, it’s important to define its scope.

Next is a **refresh token**. Though issued in combination with access grant/token, it’s not a part of the client-side request. Its main job is to renew the expired client app token**.**

#### OAuth 1.0 and 1.0a

oauth 1.0, 1.0a是同一个体系的。1.0早期为google工程师设计。他们随后发现这个协议存在session fixation attack。 1.0a修复了这个漏洞。但在使用过程中人们发现开发人员不能很好地实现协议所要求的数字签名。这造成大量的漏洞。于是ietf工作组重新设计了oauth2.0。新协议不再需要数字签名。这个协议和1.0不能兼容。

#### 授权模式：Grant Types

客户端必须用用户的授权，才能获取访问令牌(access token)，然后根据令牌去资源服务器获取相应资源。

oauth2.0 有四种模式分别如下：

- 授权码（authorization-code）

  This flow redirects you to log in directly with a 3rd party, meaning the client never gets access to your username/password that you type in. That very important secret is not shared in another database somewhere, it remains between you and the credential provider you trust (such as Facebook, although not sure I would trust them too much).

- 隐藏式（implicit）

  It won’t ask for any kind of code. Rather, the client app gets an access token soon after the user’s consent.

- 密码式（password）

  Useful in situations requiring to obtain a token for a scenario that falls outside the user’s context. Only the client uses it.

- 客户端凭证（client credentials）

#####  授权码:常用:lock:

不同公司(系统)公司之间的调用，比如N个小程序厂商都去调用微信的资源信息、

说明：基本流程就是拿Authorization Code换Access Token。

**每一步必须要参数：**

1. 首先你得拥有属于自己应用对应的 appkey、appsecret 分别对应 OAuth2.0 中的 client_id 与client_secret，可能一些服务端会扩展一些其他的名称。并设置好你自己的 redirect_uri 跳转回调地址，用于通知给你code码，还可以设置好 state附带返回参数，scope授权范围.
2. 当你拿到code后，就可以换取access_token了。一般都必须返回至少三个参数：refresh_token、access_token、expires_in
3. 三方应用拿到access_token后就可以请求资源服务器获取用户数据了，如果超时则用refresh_token进行重刷access_token。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/OAuth2_authcode.gif)

The authorization code workflow diagram involves the following steps:

1. The OAuth client initiates the flow when it directs the user agent of the resource owner to the authorization endpoint. The OAuth client includes its client identifier, requested scope, local state, and **a redirection URI.** The authorization server sends the user agent back to the redirection URI after access is granted or denied.
2. The authorization server authenticates the resource owner through the user agent and establishes whether the resource owner grants or denies the access request.
3. If the resource owner grants access, the OAuth client uses the redirection URI provided earlier to redirect the user agent back to the OAuth client. The redirection URI includes an authorization code and any local state previously provided by the OAuth client.
4. The OAuth client requests an access token from the authorization server through the token endpoint. The OAuth client authenticates with its client credentials and includes the authorization code received in the previous step. The OAuth client also includes the redirection URI used to obtain the authorization code for verification.
5. The authorization server validates the client credentials and the authorization code. The server also ensures that the redirection URI received matches the URI used to redirect the client in Step 3. If valid, the authorization server responds back with an access token.

######  redirect_uri:sailboat:

授权后要回调的URI，即接收Authorization Code的URI。

client需要使用Authorization Code去兑换相应的Access Token

OAuth 2.0 是一类基于回调的授权协议，以授权码模式为例，整个授权需要分为两步进行，第一步下发授权码，第二步根据授权码请求授权服务器下发访问令牌。OAuth 在第一步下发授权码时，是将授权码以参数的形式添加到回调地址后面，并以 302 跳转的形式进行下发，这样简化了客户端的操作，不需要再主动去触发一次请求，即可进入下一步流程。

回调的设计存在一定的安全隐患，坏人可以利用该机制引导用户到一个恶意站点，继而对用户发起攻击。对于授权服务器而言，也存在一定的危害，坏人可以利用该机制让授权服务器变成“肉鸡”，以授权服务器为代理请求目标地址，这样在消耗授权服务器资源的同时，也对目标地址服务器产生 DDOS 攻击。

为了避免上述安全隐患，OAuth 协议强制要求客户端在注册时填写自己的回调地址，其目的是为了让回调请求能够到达客户端自己的服务器，从而可以走获取访问令牌的流程。客户端可以同时配置多个回调地址，并在请求授权时携带一个地址，服务器会验证客户端传递上来的回调地址是否与之前注册的回调地址相同，或者前者是后者集合的一个元素，只有在满足这一条件下才允许下发授权码，同时协议还要求两步请求客户端携带的回调地址必须一致，通过这些措施来保证回调过程能够正常到达客户端自己的服务器，并继续后面拿授权码换取访问令牌的过程。

###### Authorization Code

授权码是授权流程的一个中间临时凭证，是对用户确认授权这一操作的一个短暂性表征，其生命周期一般较短，协议建议最大不要超过 10 分钟，在这一有效时间内，客户端可以通过授权码去授权服务器请求换取访问令牌，授权码应该采取防重放措施。



##### 密码模式

场景，一家公司有N个游戏或者产品，可以使用这种模式。类似天猫和淘宝 session共享？？？

密码模式中，用户向客户端提供自己的用户名和密码，这通常用在用户对客户端高度信任的情况

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/oauth-密码模式.png)

从调接口方面，简单来说：

- 第一步：直接传username，password获取token

http://localhost:8888/oauth/token?client_id=cms&client_secret=secret&username=admin&password=123456&grant_type=password&scope=all

- 第二步：拿到acceptToken之后，就可以直接访问资源

http://localhost:8084/api/userinfo?access_token=${accept_token}

##### 简化模式：不常用

简化模式（implicit grant type）不通过第三方应用程序的服务器，直接在浏览器中向认证服务器申请令牌，跳过了"授权码"这个步骤，因此称简化模式.

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/oauth-简化模式.png)

从调接口方面，简单来说：

- 第一步：访问授权，要传client_id:客户端id，redirect_uri:重定向uri，response_type为token，scope是授权范围，state是其它自定义参数

http://localhost:8888/oauth/authorize?client_id=cms&redirect_uri=http://localhost:8084/callback&response_type=token&scope=read&state=123

- 第二步：授权通过，会重定向到redirect_uri，access_token码会作为它的参数

http://localhost:8084/callback#access_token=${accept_token}&token_type=bearer&state=123&expires_in=120

- 第三步：拿到acceptToken之后，就可以直接访问资源

http://localhost:8084/api/userinfo?access_token=${accept_token}

##### 客户端模式：不常用

客户端模式（client credentials）适用于没有前端的命令行应用，即在命令行下请求令牌

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/oauth-客户端模式.png)

从调接口方面，简单来说：

- 第一步： 获取token
  http://localhost:8888/oauth/token?client_id=cms&client_secret=123&grant_type=client_credentials&scope=all
- 第二步：拿到acceptToken之后，就可以直接访问资源

http://localhost:8084/api/userinfo?access_token=${accept_token}

### JWT implements Oauth code

Basically, JWT is a token format.

OAuth is an standardised **authorization** protocol that can use JWT as a token.

#### jwt app demo：注意cookie设置地方

server端设置cookie 和client端设置cookie类似，不同的是 server端用的是`http.SetCookie`, 而client端用的是`req.AddCookie`

```go
// 设置在client端
req.AddCookie(&http.Cookie{
	Name:    "clientcookieid2",
	Value:   "id2",
	Expires: time.Now().Add(111 * time.Second),
})

// 设置在Server端
http.SetCookie(w, &http.Cookie{
	Name:    "servercookie",
	Value:   "servercookievalue",
	Expires: time.Now().Add(111 * time.Second),
})
```

下面简单的demo是将cookie设置在Server端的

当Client在header中指定cookie时，会覆盖Server端设置的cookie。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/jwt-login-1.jpg)

client故意使用过期的Cookie

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/jwt-login-2.jpg)



```go
package main

import (
	"encoding/json"
	"fmt"
	"github.com/dgrijalva/jwt-go"
	"log"
	"net/http"
	"time"
)

//设置jwt的认证签名部分的key
var jwtKey = []byte("my_secret_key")

//模拟用户数据(一般从MySQL中读取)
var users = map[string]string {
	"user1": "password1",
	"user2": "password2",
}

//请求认证时的对应结构体
type Credentials struct {
	Password string `json:"password"`
	Username string `json:"username"`
}

//加密成jwt对应结构体
type Claims struct {
	Username string `json:"username"`
	jwt.StandardClaims
}

//登录校验
func Login(w http.ResponseWriter, r *http.Request) {
	var creds Credentials

	//解析请求的凭证是否合法
	log.Println("login")
	err := json.NewDecoder(r.Body).Decode(&creds)
	if err != nil {
		w.WriteHeader(http.StatusBadRequest)
		return
	}

	//数据库存储的合法用户
	expectedPassword, ok := users[creds.Username]

	if !ok || expectedPassword != creds.Password {
		w.WriteHeader(http.StatusBadRequest)
		return
	}

	//认证通过的情况，刷新token有效期
	expirationTime := time.Now().Add(time.Minute * 5)

	//将相关信息写入jwt认证结构体中
	claims := Claims{
		Username: creds.Username,
		StandardClaims: jwt.StandardClaims{
			ExpiresAt: expirationTime.Unix(),
		},
	}

	//生成token
	token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
	tokenString, err := token.SignedString(jwtKey)
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		return
	}

	//将生成的token存入客户端(cookie或者localStore)
	http.SetCookie(w, &http.Cookie{
		Name: "token",
		Value: tokenString,
		Expires: expirationTime,
	})
}

//后续请求token校验
func Verify (w http.ResponseWriter, r *http.Request) {
	//从cookie中获取token
	c, err := r.Cookie("token")
	if err != nil {
		//cookie不存在的情况
		if err == http.ErrNoCookie {
			w.WriteHeader(http.StatusUnauthorized)
			return
		}

		//其他情况
		w.WriteHeader(http.StatusBadRequest)
		return
	}

	//获取cookie中的token值
	tknStr := c.Value
	claims := &Claims{}

	//校验从cookie中获取的token是否发生变化
	tkn, err := jwt.ParseWithClaims(tknStr, claims, func(token *jwt.Token) (interface{}, error) {
		return jwtKey, nil
	})

	if err != nil {
		if err == jwt.ErrSignatureInvalid {
			w.WriteHeader(http.StatusUnauthorized)
			return
		}

		w.WriteHeader(http.StatusBadRequest)
		return
	}

	if !tkn.Valid {
		w.WriteHeader(http.StatusUnauthorized)
		return
	}
	w.Write([]byte(fmt.Sprintf("Welcome %s!", claims.Username)))
}

//刷新token的有效期
func Refresh (w http.ResponseWriter, r *http.Request) {
	//获取cookie中的token
	c, err := r.Cookie("token")
	if err != nil {
		if err == http.ErrNoCookie {
			w.WriteHeader(http.StatusUnauthorized)
			return
		}
		w.WriteHeader(http.StatusBadRequest)
		return
	}

	//构建token
	tknStr := c.Value
	claims := &Claims{}

	tkn, err := jwt.ParseWithClaims(tknStr, claims, func(token *jwt.Token) (interface{}, error) {
		return jwtKey, nil
	})

	if err != nil {
		if err == jwt.ErrSignatureInvalid {
			w.WriteHeader(http.StatusUnauthorized)
			return
		}

		w.WriteHeader(http.StatusBadRequest)
		return
	}

	if !tkn.Valid {
		w.WriteHeader(http.StatusUnauthorized)
		return
	}

	//token认证通过, 并且token有效期少于30秒才会刷新token
	if time.Unix(claims.ExpiresAt, 0).Sub(time.Now()) > 30*time.Second {
		w.WriteHeader(http.StatusBadRequest)
		return
	}

	expirationTime := time.Now().Add(time.Minute * 5)
	claims.ExpiresAt = expirationTime.Unix()
	token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
	tokenStr, err := token.SignedString(jwtKey)
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		return
	}

	//更新cookie的有效期时间
	http.SetCookie(w, &http.Cookie{
		Name: "token",
		Value: tokenStr,
		Expires: expirationTime,
	})
}

func main() {
    // post
	http.HandleFunc("/login", Login)
    // get
	http.HandleFunc("/verify", Verify)
    // get or put?
	http.HandleFunc("/refresh", Refresh)
	log.Fatal(http.ListenAndServe(":9000", nil))
}

```



end

```go
// Copyright 2014 The Go Authors. All rights reserved.
// Use of this source code is governed by a BSD-style
// license that can be found in the LICENSE file.

// Package jwt implements the OAuth 2.0 JSON Web Token flow, commonly
// known as "two-legged OAuth 2.0".
//
// See: https://tools.ietf.org/html/draft-ietf-oauth-jwt-bearer-12
package jwt

import (
	"context"
	"encoding/json"
	"fmt"
	"io"
	"io/ioutil"
	"net/http"
	"net/url"
	"strings"
	"time"

	"golang.org/x/oauth2"
	"golang.org/x/oauth2/internal"
	"golang.org/x/oauth2/jws"
)

var (
	defaultGrantType = "urn:ietf:params:oauth:grant-type:jwt-bearer"
	defaultHeader    = &jws.Header{Algorithm: "RS256", Typ: "JWT"}
)

// Config is the configuration for using JWT to fetch tokens,
// commonly known as "two-legged OAuth 2.0".
type Config struct {
	// Email is the OAuth client identifier used when communicating with
	// the configured OAuth provider.
	Email string

	// PrivateKey contains the contents of an RSA private key or the
	// contents of a PEM file that contains a private key. The provided
	// private key is used to sign JWT payloads.
	// PEM containers with a passphrase are not supported.
	// Use the following command to convert a PKCS 12 file into a PEM.
	//
	//    $ openssl pkcs12 -in key.p12 -out key.pem -nodes
	//
	PrivateKey []byte

	// PrivateKeyID contains an optional hint indicating which key is being
	// used.
	PrivateKeyID string

	// Subject is the optional user to impersonate.
	Subject string

	// Scopes optionally specifies a list of requested permission scopes.
	Scopes []string

	// TokenURL is the endpoint required to complete the 2-legged JWT flow.
	TokenURL string

	// Expires optionally specifies how long the token is valid for.
	Expires time.Duration

	// Audience optionally specifies the intended audience of the
	// request.  If empty, the value of TokenURL is used as the
	// intended audience.
	Audience string

	// PrivateClaims optionally specifies custom private claims in the JWT.
	// See http://tools.ietf.org/html/draft-jones-json-web-token-10#section-4.3
	PrivateClaims map[string]interface{}

	// UseIDToken optionally specifies whether ID token should be used instead
	// of access token when the server returns both.
	UseIDToken bool
}

// TokenSource returns a JWT TokenSource using the configuration
// in c and the HTTP client from the provided context.
func (c *Config) TokenSource(ctx context.Context) oauth2.TokenSource {
	return oauth2.ReuseTokenSource(nil, jwtSource{ctx, c})
}

// Client returns an HTTP client wrapping the context's
// HTTP transport and adding Authorization headers with tokens
// obtained from c.
//
// The returned client and its Transport should not be modified.
func (c *Config) Client(ctx context.Context) *http.Client {
	return oauth2.NewClient(ctx, c.TokenSource(ctx))
}

// jwtSource is a source that always does a signed JWT request for a token.
// It should typically be wrapped with a reuseTokenSource.
type jwtSource struct {
	ctx  context.Context
	conf *Config
}

func (js jwtSource) Token() (*oauth2.Token, error) {
	pk, err := internal.ParseKey(js.conf.PrivateKey)
	if err != nil {
		return nil, err
	}
	hc := oauth2.NewClient(js.ctx, nil)
	claimSet := &jws.ClaimSet{
		Iss:           js.conf.Email,
		Scope:         strings.Join(js.conf.Scopes, " "),
		Aud:           js.conf.TokenURL,
		PrivateClaims: js.conf.PrivateClaims,
	}
	if subject := js.conf.Subject; subject != "" {
		claimSet.Sub = subject
		// prn is the old name of sub. Keep setting it
		// to be compatible with legacy OAuth 2.0 providers.
		claimSet.Prn = subject
	}
	if t := js.conf.Expires; t > 0 {
		claimSet.Exp = time.Now().Add(t).Unix()
	}
	if aud := js.conf.Audience; aud != "" {
		claimSet.Aud = aud
	}
	h := *defaultHeader
	h.KeyID = js.conf.PrivateKeyID
	payload, err := jws.Encode(&h, claimSet, pk)
	if err != nil {
		return nil, err
	}
	v := url.Values{}
	v.Set("grant_type", defaultGrantType)
	v.Set("assertion", payload)
	resp, err := hc.PostForm(js.conf.TokenURL, v)
	if err != nil {
		return nil, fmt.Errorf("oauth2: cannot fetch token: %v", err)
	}
	defer resp.Body.Close()
	body, err := ioutil.ReadAll(io.LimitReader(resp.Body, 1<<20))
	if err != nil {
		return nil, fmt.Errorf("oauth2: cannot fetch token: %v", err)
	}
	if c := resp.StatusCode; c < 200 || c > 299 {
		return nil, &oauth2.RetrieveError{
			Response: resp,
			Body:     body,
		}
	}
	// tokenRes is the JSON response body.
	var tokenRes struct {
		AccessToken string `json:"access_token"`
		TokenType   string `json:"token_type"`
		IDToken     string `json:"id_token"`
		ExpiresIn   int64  `json:"expires_in"` // relative seconds from now
	}
	if err := json.Unmarshal(body, &tokenRes); err != nil {
		return nil, fmt.Errorf("oauth2: cannot fetch token: %v", err)
	}
	token := &oauth2.Token{
		AccessToken: tokenRes.AccessToken,
		TokenType:   tokenRes.TokenType,
	}
	raw := make(map[string]interface{})
	json.Unmarshal(body, &raw) // no error checks for optional fields
	token = token.WithExtra(raw)

	if secs := tokenRes.ExpiresIn; secs > 0 {
		token.Expiry = time.Now().Add(time.Duration(secs) * time.Second)
	}
	if v := tokenRes.IDToken; v != "" {
		// decode returned id token to get expiry
		claimSet, err := jws.Decode(v)
		if err != nil {
			return nil, fmt.Errorf("oauth2: error decoding JWT token: %v", err)
		}
		token.Expiry = time.Unix(claimSet.Exp, 0)
	}
	if js.conf.UseIDToken {
		if tokenRes.IDToken == "" {
			return nil, fmt.Errorf("oauth2: response doesn't have JWT token")
		}
		token.AccessToken = tokenRes.IDToken
	}
	return token, nil
}
```

#### github oauth test - 前端

先申请客户端 ID（client ID）和客户端密钥（client secret），这就是应用的身份识别码。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/github-oauth.jpg)

两个前端页面

**login.html**

首页很简单，就是一个链接，让用户跳转到 GitHub。

这个 URL 指向 GitHub 的 OAuth 授权网址，带有两个参数：`client_id`告诉 GitHub 谁在请求，`redirect_uri`是稍后跳转回来的网址。

用户点击到了 GitHub，GitHub 会要求用户登录，确保是本人在操作。

登录后，GitHub 询问用户，该应用正在请求数据，你是否同意授权。

用户同意授权， GitHub 就会跳转到`redirect_uri`指定的跳转网址，并且带上授权码，跳转回来的 URL 就是下面的样子。

```html
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>index</title>
</head>
<body>
        <a href="https://github.com/login/oauth/authorize?client_id={github oauth client ID}&redirect_uri=http://IP:Port/oauth/redirect">
        Login with github
    </a>
</body>
</html>

```

访问`IP:port/login.html`自动跳转

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/oauth-github-auth.jpg)

**welcome.html**

根据access token获取github用户信息并显示

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>welcome</title>
</head>
<body>

</body>
<script>
    // 获取access_token
    const query = window.location.search.substring(1)
    const token = query.split('access_token=')[1]

    fetch('https://api.github.com/user', {
        headers: {
            // 将token放在Header中
            Authorization: 'token ' + token
        }
    })
    .then(res => res.json())
    .then(res => {
        const nameNode = document.createTextNode(`Welcome, ${res.name}`)
        document.body.appendChild(nameNode)
    })
</script>
</html>

```

#### github oauth test -后端

```go
package main

import (
	"encoding/json"
	"fmt"
	"log"
	"net/http"
)

//认证中心的注册信息
//github注册应用的地址：https://github.com/settings/applications/new

const (
	clientID = "xxx"
	clientSecret = "yyy"
)

var httpClient = http.Client{}

type OAuthAccessResponse struct {
	AccessToken string `json:"access_token"`
}

func HandleOauthRedirect(w http.ResponseWriter, r *http.Request) {
	err := r.ParseForm()
	if err != nil {
		log.Printf("could not parse query: %v", err)
		w.WriteHeader(http.StatusBadRequest)
	}

	code := r.FormValue("code")

	//通过clientID、clientSecret、code获取授权秘钥
	reqURL := fmt.Sprintf("https://github.com/login/oauth/access_token?client_id=%s&client_secret=%s&code=%s", clientID, clientSecret, code)

	req, err := http.NewRequest(http.MethodPost, reqURL, nil)
	if err != nil {
		log.Printf("could not create HTTP request: %v", err)
		w.WriteHeader(http.StatusBadRequest)
	}

	//设置返回的格式为json格式
	req.Header.Set("accept", "application/json")

	//发送http请求
	res, err := httpClient.Do(req)
	if err != nil {
		log.Printf("could not send HTTP request: %v", err)
		w.WriteHeader(http.StatusInternalServerError)
	}
	defer res.Body.Close()

	//解析结果
	var t OAuthAccessResponse
	if err := json.NewDecoder(res.Body).Decode(&t); err != nil {
		log.Printf("could not parse JSON response: %v", err)
		w.WriteHeader(http.StatusBadRequest)
	}

	w.Header().Set("Location", "/welcome.html?access_token="+t.AccessToken)
	w.WriteHeader(http.StatusFound)
}

func main() {
	fs := http.FileServer(http.Dir("./jwt"))
	http.Handle("/", fs)
	http.HandleFunc("/oauth/redirect", HandleOauthRedirect)
	http.ListenAndServe(":8080", nil)
}

```

end

##### 问题：gayhub申请错误

我调用的接口是

```bash
https://github.com/login/oauth/authorize?client_id=a1dc3941baf834777212&redirect_uri=http://116.62.122.90:31004/oauth/redirect
```

为什么后端转发的时候，端口不见了变成

```bash
http://116.62.122.90/oauth/redirect?error=redirect_uri_mismatch&error_description=The+redirect_uri+MUST+match+the+registered+callback+URL+for+this+application.&error_uri=https://docs.github.com/apps/managing-oauth-apps/troubleshooting-authorization-request-errors/#redirect-uri-mismatch
```

查了一下原因原来是在github申请oauth 时 callback URL填写错误，没加Port

https://github.com/settings/developers 可在这里查看申请的OAuth Apps

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/github-oauth.jpg)

WTF??? callback URL带了端口还是不行？？？

换个浏览器好了... 跳转还有缓存呢？



#### 参考实现代码？

https://juejin.cn/post/6979546196339589134

### gitlab配置oauth 第三方登陆

https://developer.aliyun.com/article/871743



### 引用

1. https://www.zhihu.com/question/19851243/answer/28383770
2. https://anil-pace.medium.com/json-web-tokens-vs-oauth-2-0-85dd0b32057d
3. https://stackoverflow.com/questions/39909419/what-are-the-main-differences-between-jwt-and-oauth-authentication
4. https://www.wallarm.com/what/oauth-vs-jwt-detailed-comparison
5. https://www.bilibili.com/video/BV1Nq4y1u7DK?p=1&vd_source=2795986600b37194ea1056cddb9856fa
6. https://www.cnblogs.com/mzq123/p/13044001.html
7. https://www.ibm.com/docs/en/tfim/6.2.2.6?topic=overview-oauth-20-workflow
8. https://segmentfault.com/a/1190000013659546