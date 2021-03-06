---
title: jwt原理
date: 2020-8-8
tags: 
- web
- jwt
---

## 传统的基于token的用户验证

- 1. 用户登录
    * 服务返回token，并在服务端保存token
- 2. 用户验证
    * 用户再次访问时需要携带token，比较数据库中的token


## Json Web Token

- 1. 用户登录
    * 服务返回token，但不保存token
- 2. 用户验证
    * 用户再次访问时需要携带token，再做token校验


### jwt实现过程

- 1. 用户提交用户名和密码，如果登录成功，使用jwt创建一个token并给用户返回
    * jwt生成的token由三段字符串生成，并由`.`连接
        + 第一段字符串：HEADER。内部包含算法和token类型，json转换成字符串然后做base64url加密
        ``` json
        {
            "alg": "HS256",
            "typ": "JWT"
        }
        ```
        + 第二段字符串：PAYLOAD。内容自定义值，但一般都有`exp`超时时间，且不包含敏感信息
        ``` json
        {
            "id": "123",
            "name": "ring",
            "exp"  0000000000,
        }
        ```
        + 第三段字符串。第一和第二部分的密文拼接，然后加密。加密方式由第一段字符串给出， **并且`加盐`** 。在这次加密后(HS256，第一次加密不能反解)再对密文进行base64url加密
- 2. 校验token
    * 1. 对token进行切割，`.`
    * 2. 对第二段进行base64url解密，获取PAYLOAD信息。然后检测超时时间
    * 3. 把第一第二段拼接进行HS256解密(第一段指定的加密方式)加密。让密文和第三段base64url解密后的密文比较
- 能防止篡改吗？
    * 由于第三段密文是`加盐`后加密得到的，而盐只有自己知道，所以不能篡改


### 应用

官网给出了对很多语言、算法的支持



        


