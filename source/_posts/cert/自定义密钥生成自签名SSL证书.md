---
title: 自定义密钥生成自签名SSL证书
date: 2023-12-09 14:15:17
tags: cert
categories: shell
---


## 生成SSL证书
-  生成RSA密钥：需要设置密码，后面的操作都需要使用这个密码；执行命令：
 localssl.key由自己定义
` openssl genrsa -des3 -out D:/ssl/localssl.key 2048 `

-   拷贝一个不需要密码的密钥
`openssl rsa -in D:/ssl/localssl.key -out D:/ssl/localssl_nopass.key`

- 生成一个证书请求, 填写证书信息
`openssl rsa -in D:/ssl/localssl.key -out D:/ssl/localssl_nopass.key`

- 用上面的密钥和CSR对进行证书签名,生成证书文件(days是证书有效期)
`openssl x509 -req -days 3650 -in D:/ssl/localssl.csr -signkey D:/ssl/localssl.key -out D:/ssl/localssl.crt`

## 配置nginx的ssl，支持https
```
 server {
        listen       80;
        listen    443 ssl;
        server_name  192.168.1.15;
	
        ssl_certificate  D:\ssl\localssl.crt;
        ssl_certificate_key  D:\ssl\localssl_nopass.key;
}

```