---
title: SSL自签名证书生成与配置(基于根证书)
date: 2023-12-09 14:05:22
tags: cert
categories: shell
---

[如何让openssl生成的SSL证书被浏览器认可](https://www.cnblogs.com/mjy2wxy/p/15705680.html)

> 准备一台有openssl的服务器
- 生成根证书
  -  
  - 生成命令如下，其中：/C=CN（国家缩写）/ST=（省份）/L=（城市）/O=（组织名称）：
```
openssl req -x509 -nodes -days 3650 -newkey rsa:2048 -subj 
"/C=CN/ST=ZheJiang/L=HangZhou/O=MJY" -keyout CA-private.key 
-out CA-certificate.crt -reqexts v3_req -extensions v3_ca
```
这时会生成根证书文件CA-private.key、CA-certificate.crt

- 生成二级证书密钥
	-
```
openssl genrsa -out private.key 2048
```
- 生成签名证书请求
	-
	`openssl req -new -key private.key -subj "/C=CN/ST=ZheJiang/L=HangZhou/O=MJY/CN=127.0.0.1" -sha256 -out private.csr`
	其中ip部分根据自己实际情况修改

- 创建ext文件，这个是为了将根证书签发机构加入到SAN扩展中，这样chrome就会认为是可信任的签发机构
	- 
```
#vim private.ext
#复制如下内容到private.ext文件中

[ req ]
default_bits = 1024 distinguished_name = req_distinguished_name
req_extensions = san
extensions = san
[ req_distinguished_name ]
countryName = CN
stateOrProvinceName = Definesys
localityName = Definesys
organizationName = Definesys
[SAN]
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = IP:127.0.0.1 #其中ip后内容，改成自己需要的ip地址（服务器ip或者域名）  
#最后使用 :wq 保存退出
```
- 生成CA证书
	-
`openssl x509 -req -days 3650 -in private.csr -CA CA-certificate.crt -CAkey CA-private.key -CAcreateserial -sha256 -out private.crt -extfile private.ext -extensions SAN`

生成结果如下：
![输入图片描述](images/cert/chrome-cert-1.png)

将证书配置到nginx
![输入图片描述](images/cert/chrome-cert-2.png)

客户端将CA-certificate.crt安装到受信任的颁发机构，重启浏览器访问，就发现不安全的标识已经没了
![输入图片描述](images/cert/chrome-cert-3.png)

