---
layout:     post
title:      认证插件
subtitle:   服务网关kong
date:       2020-05-15
author:     bww126
header-img: img/bg/gray-bg.PNG
catalog: true
tags:
    - kong
---

## JWT

#### Payload 参数

```
iss (issuer): jwt签发者
sub (subject): jwt所面向的用户
aud (audience): 接收jwt的一方
exp (expiration time): jwt的过期时间，这个过期时间必须要大于签发时间
nbf (Not Before): 定义在什么时间之前，该jwt都是不可用的.
iat (Issued At): jwt的签发时间
jti (JWT ID): jwt的唯一身份标识，主要用来作为一次性token,从而回避重放攻击。
```


