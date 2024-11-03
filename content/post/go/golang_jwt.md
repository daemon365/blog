---
title: golang jwt
date: '2020-05-20T00:00:00+08:00'
tags:
- go
showToc: true
categories:
- go
---



## 什么是JWT？

JWT全称JSON Web Token是一种跨域认证解决方案，属于一个开放的标准，它规定了一种Token实现方式，目前多用于前后端分离项目和OAuth2.0业务场景下。

## JWT作用？

JWT就是一种基于Token的轻量级认证模式，服务端认证通过后，会生成一个JSON对象，经过签名后得到一个Token（令牌）再发回给用户，用户后续请求只需要带上这个Token，服务端解密之后就能获取该用户的相关信息了。

## 下载jwt

```bash
go get -u github.com/golang-jwt/jwt
```

## 生成JWT和解析JWT

我们在这里直接使用jwt-go这个库来实现我们生成JWT和解析JWT的功能。

定义需求

我们需要定制自己的需求来决定JWT中保存哪些数据

```go
type MyClaims struct {
	UserID   uint64 `json:"user_id"`
	Username string `json:"username"`
	jwt.StandardClaims
}
```

定义JWT的过期时间和Secret(盐)：

```go
const TokenExpireDuration = time.Hour * 24 * 2 // 过期时间 -2天
var Secret = []byte("i am zhaohaiyu") // Secret(盐) 用来加密解密
```

## 生成JWT

```go
//生成 jwt token
func genToken(userID uint64, username string) (string, error) {
	var claims = MyClaims{
		userID,
		username,
		jwt.StandardClaims{
			ExpiresAt: time.Now().Add(TokenExpireDuration).Unix(), // 过期时间
			Issuer:    "zhaohaiyu",                                // 签发人
		},
	}
	token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
	signedToken, err := token.SignedString([]byte(Secret))
	if err != nil {
		return "", fmt.Errorf("生成token失败:%v", err)
	}
	return signedToken, nil
}
```

## 解析JWT

```go
//验证jwt token
func ParseToken(tokenStr string) (*MyClaims, error) {
	token, err := jwt.ParseWithClaims(tokenStr, &MyClaims{}, func(token *jwt.Token) (i interface{}, err error) { // 解析token
		return Secret, nil
	})
	if err != nil {
		return nil, err
	}
	if claims, ok := token.Claims.(*MyClaims); ok && token.Valid { // 校验token
		return claims, nil
	}
	return nil, errors.New("invalid token")
}
```

## 在gin框架中使用JWT

```go
r.POST("/auth/token", GetTokenHandler)
```

### 登录生成token

```go
func GetTokenHandler(c *gin.Context) {
	// 接收参数
	var user UserInfo
	err := c.ShouldBind(&user)
	if err != nil {
		c.JSON(http.StatusOK, gin.H{
			"code":     1001,
			"err_info": "参数错误",
		})
		return
	}
	// 验证密码
	if user.Username == "root" && user.Password == "123" {
		// 生成Token
		tokenString, err := GenToken(user.Username)
		if err != nil {
			c.JSON(http.StatusOK, gin.H{
				"code":     1001,
				"err_info": "生成token错误",
			})
			return
		}
		c.JSON(http.StatusOK, gin.H{
			"code": 2000,
			"msg":  "success",
			"data": gin.H{"token": tokenString},
		})
		return
	} else {
		c.JSON(http.StatusOK, gin.H{
			"code": 2002,
			"msg":  "用户名或密码错误",
		})
		return
	}
	return
}
```

### jwt中间件验证

```go
// JWThMiddleware 中间件
func JWThMiddleware() func(c *gin.Context) {
	return func(c *gin.Context) {
		// 客户端携带Token有三种方式 1.放在请求头 2.放在请求体 3.放在URI
		// 这里假设Token放在Header的token中
		token := c.Request.Header.Get("token")
		if token == "" {
			// 处理 没有token的时候
			c.Abort() // 不会继续停止
			return
		}
		// 解析
		mc, err := ParseToken(token)
		if err != nil {
			// 处理 解析失败
			c.Abort()
			return
		}
		// 将当前请求的userID信息保存到请求的上下文c上
		c.Set("userID", mc.UserID)
		c.Next()
	}
}
```