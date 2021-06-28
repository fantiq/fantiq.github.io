---
title: go-grpc 中的通信证书认证过程
tags: []
categories: []
date: 2020-04-20 10:03:29
---

内容比较乱，需要整理。大致需要解释 证书生成过程/证书如何实现对数据的加密/如何通过跟证书验证加密证书的合法性

grpc 通信使用的非对称加密

<!-- more -->

## 无证书验证 只是数据加密解密

秘钥生成过程:
```
openssl genrsa -out server.key 2048
openssl ecparam -genkey -name secp384r1 -out server.key
```

证书生成过程:
```
openssl req -new -x509 -sha256 -key server.key -out server.pem -days 3650
```

```
Country Name (2 letter code) []:CN
State or Province Name (full name) []:ZheJiang
Locality Name (eg, city) []:HangZhou
Organization Name (eg, company) []:HangZhou Bianfeng Network Technology Co.,Ltd
Organizational Unit Name (eg, section) []:comb
Common Name (eg, fully qualified host name) []:bianfeng
Email Address []:admin@bianfeng.com
```
对应信息
```
Country Name (2 letter code) []: 国家
State or Province Name (full name) []:省
Locality Name (eg, city) []:市
Organization Name (eg, company) []:公司
Organizational Unit Name (eg, section) []:部门
Common Name (eg, fully qualified host name) []:域名
Email Address []:联系邮箱
```

客户端证书在Load过程中需要填写的 Host信息 需要与 证书中填写的 `Common Name` 对应


## 双向证书来源验证

客户端 服务端的证书通过一个 ca 证书进行签名生成
通信过程发送的证书 通过 ca 验证证书来源是否合法

参考链接

[grpc-go 双向认证](https://cloud.tencent.com/developer/article/1423345)

[自签名证书生成](https://ningyu1.github.io/site/post/51-ssl-cert/)


证书生成时候需要填写的信息 其中 `Common Name` 必填 很重要，客户端需要填写这个 一般是域名
若不是基于http 可以不必填写域名
可以在生成的时候使用 参数 `-subj "/C=CN/O=${公司名称}/OU=${公司部门}/CN=${域名}/ST=${省}/L=${市}/emailAddress=${邮箱}` 快速指定
查看证书信息的命令 `openssl x509 -in [pem证书] -noout -text`


#### 生成根证书
```
openssl genrsa -out ca.key 2048
openssl req -new -x509 -key ca.key -out ca.pem -subj "/CN=bianfeng"
```

#### 生成服务端证书
```
openssl genrsa -out server.key 2048
# 生成请求证书
openssl req -new -key server.key -out server.csr -subj "/CN=bianfeng"
# 通过 ca 证书对 csr签名 生成pem证书
openssl x509 -req -sha256 -CA ca.pem -CAkey ca.key -CAcreateserial -days 3650 -in server.csr -out server.pem
```

#### 生成客户端证书
```
openssl genrsa -out client.key 2048
# 生成请求证书
openssl req -new -key client.key -out client.csr -subj "/CN=bianfeng"
# 通过 ca 证书对 csr签名 生成pem证书
openssl x509 -req -sha256 -CA ca.pem -CAkey ca.key -CAcreateserial -days 3650 -in client.csr -out client.pem
```


##### grpc-go 中的配置

server side:
```
tls.Config{
    ClientAuth:   tls.RequireAndVerifyClientCert, // 对客户端证书进行验证
    Certificates: []tls.Certificate{服务端证书 用以服务端发送数据},
    ClientCAs:    certPool, // 签发客户端证书的 根证书
}
```

client side:
```
tls.Config{
    ServerName:   "bianfeng", // 对应证书的 Common Name
    Certificates: []tls.Certificate{客户端证书 用以客户端发送数据},
    RootCAs:      certPool, // 签发服务端证书的 跟证书
}
```