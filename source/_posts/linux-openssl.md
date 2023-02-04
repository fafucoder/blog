---
title: linux openssl 命令详解
date: 2021-03-09 20:28:02
tags:
- linux
categories:
- linux
---

### openssl 命令模块

openssl 命令主要包括以下几个模块：

- version模块:   用于查看openssl版本信息
- s_client/s_server模块:  通用SSL/TLS测试工具
- genrsa: 用于生成私钥
- x509： x509证书管理
- verify:   x509证书验证
- rsa:   RSA秘钥管理(例如对证书进行签名)
- rsautl:  用于完成RSA签名，验证，加密和解密功能
- req:      生成证书签名请求(CSR)
- enc:      用于加解密
- dgst:   生成信息摘要
- passwd:  生成散列密码
- rand:    生成伪随机数
- pkcs7:   PSCS#7协议数据管理
- ca:      CA管理(例如对证书进行签名)
- ........

![openssl 命令模块](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008eGmZEly1goe0dqpfj9j31790u0n53.jpg)

#### openssl s_client 模块

s_client为一个SSL/TLS客户端程序，与s_server对应，它不仅能与s_server进行通信，也能与任何使用ssl协议的其他服务程序进行通信。

![openssl s_client](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008eGmZEly1gorqrlbdctj30u010a7hq.jpg)

使用s_client 获取百度的公钥:

```shell
[root@node1 ~]# openssl s_client -connect www.baidu.com:443 -msg
CONNECTED(00000003)
depth=2 C = BE, O = GlobalSign nv-sa, OU = Root CA, CN = GlobalSign Root CA
verify return:1
depth=1 C = BE, O = GlobalSign nv-sa, CN = GlobalSign Organization Validation CA - SHA256 - G2
verify return:1
depth=0 C = CN, ST = beijing, L = beijing, OU = service operation department, O = "Beijing Baidu Netcom Science Technology Co., Ltd", CN = baidu.com
verify return:1
---
Certificate chain
 0 s:/C=CN/ST=beijing/L=beijing/OU=service operation department/O=Beijing Baidu Netcom Science Technology Co., Ltd/CN=baidu.com
   i:/C=BE/O=GlobalSign nv-sa/CN=GlobalSign Organization Validation CA - SHA256 - G2
 1 s:/C=BE/O=GlobalSign nv-sa/CN=GlobalSign Organization Validation CA - SHA256 - G2
   i:/C=BE/O=GlobalSign nv-sa/OU=Root CA/CN=GlobalSign Root CA
---
Server certificate
-----BEGIN CERTIFICATE-----
MIIKLjCCCRagAwIBAgIMclh4Nm6fVugdQYhIMA0GCSqGSIb3DQEBCwUAMGYxCzAJ
BgNVBAYTAkJFMRkwFwYDVQQKExBHbG9iYWxTaWduIG52LXNhMTwwOgYDVQQDEzNH
bG9iYWxTaWduIE9yZ2FuaXphdGlvbiBWYWxpZGF0aW9uIENBIC0gU0hBMjU2IC0g
RzIwHhcNMjAwNDAyMDcwNDU4WhcNMjEwNzI2MDUzMTAyWjCBpzELMAkGA1UEBhMC
Q04xEDAOBgNVBAgTB2JlaWppbmcxEDAOBgNVBAcTB2JlaWppbmcxJTAjBgNVBAsT
HHNlcnZpY2Ugb3BlcmF0aW9uIGRlcGFydG1lbnQxOTA3BgNVBAoTMEJlaWppbmcg
QmFpZHUgTmV0Y29tIFNjaWVuY2UgVGVjaG5vbG9neSBDby4sIEx0ZDESMBAGA1UE
AxMJYmFpZHUuY29tMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAwamw
rkca0lfrHRUfblyy5PgLINvqAN8p/6RriSZLnyMv7FewirhGQCp+vNxaRZdPrUEO
vCCGSwxdVSFH4jE8V6fsmUfrRw1y18gWVHXv00URD0vOYHpGXCh0ro4bvthwZnuo
k0ko0qN2lFXefCfyD/eYDK2G2sau/Z/w2YEympfjIe4EkpbkeBHlxBAOEDF6Speg
68ebxNqJN6nDN9dWsX9Sx9kmCtavOBaxbftzebFoeQOQ64h7jEiRmFGlB5SGpXhG
eY9Ym+k1Wafxe1cxCpDPJM4NJOeSsmrp5pY3Crh8hy900lzoSwpfZhinQYbPJqYI
jqVJF5JTs5Glz1OwMQIDAQABo4IGmDCCBpQwDgYDVR0PAQH/BAQDAgWgMIGgBggr
BgEFBQcBAQSBkzCBkDBNBggrBgEFBQcwAoZBaHR0cDovL3NlY3VyZS5nbG9iYWxz
aWduLmNvbS9jYWNlcnQvZ3Nvcmdhbml6YXRpb252YWxzaGEyZzJyMS5jcnQwPwYI
KwYBBQUHMAGGM2h0dHA6Ly9vY3NwMi5nbG9iYWxzaWduLmNvbS9nc29yZ2FuaXph
dGlvbnZhbHNoYTJnMjBWBgNVHSAETzBNMEEGCSsGAQQBoDIBFDA0MDIGCCsGAQUF
BwIBFiZodHRwczovL3d3dy5nbG9iYWxzaWduLmNvbS9yZXBvc2l0b3J5LzAIBgZn
gQwBAgIwCQYDVR0TBAIwADBJBgNVHR8EQjBAMD6gPKA6hjhodHRwOi8vY3JsLmds
b2JhbHNpZ24uY29tL2dzL2dzb3JnYW5pemF0aW9udmFsc2hhMmcyLmNybDCCA04G
A1UdEQSCA0UwggNBggliYWlkdS5jb22CDGJhaWZ1YmFvLmNvbYIMd3d3LmJhaWR1
LmNughB3d3cuYmFpZHUuY29tLmNugg9tY3QueS5udW9taS5jb22CC2Fwb2xsby5h
dXRvggZkd3ouY26CCyouYmFpZHUuY29tgg4qLmJhaWZ1YmFvLmNvbYIRKi5iYWlk
dXN0YXRpYy5jb22CDiouYmRzdGF0aWMuY29tggsqLmJkaW1nLmNvbYIMKi5oYW8x
MjMuY29tggsqLm51b21pLmNvbYINKi5jaHVhbmtlLmNvbYINKi50cnVzdGdvLmNv
bYIPKi5iY2UuYmFpZHUuY29tghAqLmV5dW4uYmFpZHUuY29tgg8qLm1hcC5iYWlk
dS5jb22CDyoubWJkLmJhaWR1LmNvbYIRKi5mYW55aS5iYWlkdS5jb22CDiouYmFp
ZHViY2UuY29tggwqLm1pcGNkbi5jb22CECoubmV3cy5iYWlkdS5jb22CDiouYmFp
ZHVwY3MuY29tggwqLmFpcGFnZS5jb22CCyouYWlwYWdlLmNugg0qLmJjZWhvc3Qu
Y29tghAqLnNhZmUuYmFpZHUuY29tgg4qLmltLmJhaWR1LmNvbYISKi5iYWlkdWNv
bnRlbnQuY29tggsqLmRsbmVsLmNvbYILKi5kbG5lbC5vcmeCEiouZHVlcm9zLmJh
aWR1LmNvbYIOKi5zdS5iYWlkdS5jb22CCCouOTEuY29tghIqLmhhbzEyMy5iYWlk
dS5jb22CDSouYXBvbGxvLmF1dG+CEioueHVlc2h1LmJhaWR1LmNvbYIRKi5iai5i
YWlkdWJjZS5jb22CESouZ3ouYmFpZHViY2UuY29tgg4qLnNtYXJ0YXBwcy5jboIN
Ki5iZHRqcmN2LmNvbYIMKi5oYW8yMjIuY29tggwqLmhhb2thbi5jb22CDyoucGFl
LmJhaWR1LmNvbYIRKi52ZC5iZHN0YXRpYy5jb22CEmNsaWNrLmhtLmJhaWR1LmNv
bYIQbG9nLmhtLmJhaWR1LmNvbYIQY20ucG9zLmJhaWR1LmNvbYIQd24ucG9zLmJh
aWR1LmNvbYIUdXBkYXRlLnBhbi5iYWlkdS5jb20wHQYDVR0lBBYwFAYIKwYBBQUH
AwEGCCsGAQUFBwMCMB8GA1UdIwQYMBaAFJbeYfG9HBYpUxzAzH07gwBA5hp8MB0G
A1UdDgQWBBSeyXnX6VurihbMMo7GmeafIEI1hzCCAX4GCisGAQQB1nkCBAIEggFu
BIIBagFoAHYAXNxDkv7mq0VEsV6a1FbmEDf71fpH3KFzlLJe5vbHDsoAAAFxObU8
ugAABAMARzBFAiBphmgxIbNZXaPWiUqXRWYLaRST38KecoekKIof5fXmsgIhAMkZ
tF8XyKCu/nZll1e9vIlKbW8RrUr/74HpmScVRRsBAHYAb1N2rDHwMRnYmQCkURX/
dxUcEdkCwQApBo2yCJo32RMAAAFxObU85AAABAMARzBFAiBURWwwTgXZ+9IV3mhm
E0EOzbg901DLRszbLIpafDY/XgIhALsvEGqbBVrpGxhKoTVlz7+GWom8SrfUeHcn
4+9Dn7xGAHYA9lyUL9F3MCIUVBgIMJRWjuNNExkzv98MLyALzE7xZOMAAAFxObU8
qwAABAMARzBFAiBFBYPxKEdhlf6bqbwxQY7tskgdoFulPxPmdrzS5tNpPwIhAKnK
qwzch98lINQYzLAV52+C8GXZPXFZNfhfpM4tQ6xbMA0GCSqGSIb3DQEBCwUAA4IB
AQC83ALQ2d6MxeLZ/k3vutEiizRCWYSSMYLVCrxANdsGshNuyM8B8V/A57c0Nzqo
CPKfMtX5IICfv9P/bUecdtHL8cfx24MzN+U/GKcA4r3a/k8pRVeHeF9ThQ2zo1xj
k/7gJl75koztdqNfOeYiBTbFMnPQzVGqyMMfqKxbJrfZlGAIgYHT9bd6T985IVgz
tRVjAoy4IurZenTsWkG7PafJ4kAh6jQaSu1zYEbHljuZ5PXlkhPO9DwW1WIPug6Z
rlylLTTYmlW3WETOATi70HYsZN6NACuZ4t1hEO3AsF7lqjdA2HwTN10FX2HuaUvf
5OzP+PKupV9VKw8x8mQKU6vr
-----END CERTIFICATE-----
subject=/C=CN/ST=beijing/L=beijing/OU=service operation department/O=Beijing Baidu Netcom Science Technology Co., Ltd/CN=baidu.com
issuer=/C=BE/O=GlobalSign nv-sa/CN=GlobalSign Organization Validation CA - SHA256 - G2
---
No client certificate CA names sent
Peer signing digest: SHA256
Server Temp Key: ECDH, P-256, 256 bits
---
SSL handshake has read 4392 bytes and written 415 bytes
---
New, TLSv1/SSLv3, Cipher is ECDHE-RSA-AES128-GCM-SHA256
Server public key is 2048 bit
Secure Renegotiation IS supported
Compression: NONE
Expansion: NONE
No ALPN negotiated
SSL-Session:
    Protocol  : TLSv1.2
    Cipher    : ECDHE-RSA-AES128-GCM-SHA256
    Session-ID: A9961C486ACB6F5924E86E822E48559DBD572EE990D790C845CBE512FE62B069
    Session-ID-ctx: 
    Master-Key: 8D0DC84049E24BDBB3B45B1A73375F04E4D65E51D59F17EBAD513C68D14B65E5B9218317050827352392808C9DFBC3DA
    Key-Arg   : None
    Krb5 Principal: None
    PSK identity: None
    PSK identity hint: None
    TLS session ticket:
    0000 - 45 aa 4e e0 47 04 bf ef-18 a2 10 48 8e d6 ec 94   E.N.G......H....
    0010 - 89 f4 c8 dc cc 86 5b 2b-fe f9 95 e6 97 14 28 45   ......[+......(E
    0020 - 1c 3f 6e ad 7c 2d f5 d8-55 a5 5a 43 83 e0 af 80   .?n.|-..U.ZC....
    0030 - 7f a4 49 5a cd d0 ea f3-0c 18 f7 e1 36 7c 17 28   ..IZ........6|.(
    0040 - 56 9a 9d 0c 6a 9f e7 c5-18 cf 28 c1 ba e5 b0 8b   V...j.....(.....
    0050 - 95 5a 83 27 95 9d ba 3e-2b 96 89 f2 3c 54 52 b0   .Z.'...>+...<TR.
    0060 - 76 b0 75 6f 03 67 32 77-0b 62 f8 bc 7d 75 07 68   v.uo.g2w.b..}u.h
    0070 - 65 fd bb 88 80 83 d2 5b-80 2e 50 97 56 a0 94 5e   e......[..P.V..^
    0080 - e6 4e 37 f6 e8 96 1c d2-cf fd 8b 74 0d 6b eb 98   .N7........t.k..
    0090 - 64 0c 4e 88 30 26 17 03-11 58 30 60 75 b5 c6 5f   d.N.0&...X0`u.._

    Start Time: 1616261167
    Timeout   : 300 (sec)
    Verify return code: 0 (ok)
---
```



#### openssl verify 模块

openssl verify 用于验证证书是否合法，还可以用于验证子证书是否是父证书签的.

![openssl verify](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008eGmZEly1gorr0rglykj31uw0bkdip.jpg)

验证子证书是否由父证书签发的:

```shell
[root@node1 openssl2]# openssl verify rootCA.crt
rootCA.crt: C = CN, ST = Fujian, L = Fuzhou, O = Root CA, CN = Root CA
error 18 at 0 depth lookup:self signed certificate
OK
[root@node1 openssl2]# openssl verify -CAfile rootCA.crt harbor.crt
harbor.crt: OK
```

#### openssl genrsa 模块

openssl genrsa用于生成私钥(只生成私钥)

![openssl genrsa](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008eGmZEly1gorr4i3cuaj317e0h2tcz.jpg)

-out filename: 将生成的私钥保存至filename文件,若未指定输出文件,则为标准输出。
-des:生成的密钥使用des方式进行加密。
-des3:生成的密钥使用des3方式进行加密。
-passout args:加密私钥文件时,传递密码的格式,如果要加密私钥文件时单未指定该项,则提示输入密码。传递密码的args的格式,可从密码、环境变量、文件、终端等输入。
      a. pass:password:password表示传递的明文密码
      b. env:var:从环境变量var获取密码值
      c .file:filename:filename文件中的第一行为要传递的密码。若filename同时传递给"-passin"和"-passout"选项，则filename的第一行为"-passin"的值，第二行为"-passout"的值
      d. stdin:从标准输入中获取要传递的密码

numbits: 指定要生成的私钥的长度,默认为1024。该项必须为命令行的最后一项参数。

```shell
# 输入密码的方式
openssl genrsa -des3 -out rootCA.key 2048

# 环境变量的方式
export passwd=admin
openssl genrsa -des3 -out rootCA.key -passout env:passwd 2048
```

#### openssl rsa 模块

rsa模块用于生成的密码的管理，包括查看秘钥的信息，更改秘钥的密码等

```shell
# 去除genrsa生成的密码(每次都要输入密码怪麻烦的~)
openssl rsa -in rootCA.key -out rootCA.key

# 查看私钥的信息
openssl rsa -in rootCA.key -noout -text
```

#### openssl req 模块

req大致有3个功能：生成证书请求文件、验证证书请求文件和创建根CA。

![openssl req](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008eGmZEly1gorsowumboj31460u0k2v.jpg)

```shell
# 查看csr的信息
openssl req -in harbor.csr -noout -text

# 查看csr 的subject 信息
openssl req -new -in harbor.csr -subject -noout

# 验证csr 是否合法
openssl req -in harbor.csr -verify

# 创建csr
openssl req -new -key harbor.key -out harbor.csr

# 创建csr和私钥(
# -newkey选项和-new选项类似，只不过-newkey选项可以直接指定私钥的算法和长度，所以它主要用在openssl req自动创建私钥。
openssl req -newkey rsa:2038 -keyout harbor.key -out harbor.csr 

```

#### openssl x509 模块

x509 模块主要的功能有 签署证书请求文件、生成自签名证书、转换证书格式等。

![openssl x509](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008eGmZEly1gort5xzo7tj30yu0u019a.jpg)

```shell
# 查看公钥信息
openssl x509 -in harbor.crt -noout -text

# 查看公钥颁发者信息/所有者信息
openssl x509 -in harbor.crt -text -issuer/-subjet

# 生成公钥
openssl x509 -req -in harbor.csr -signkey harbor.key -out harbor.crt

# 签署CA证书
openssl x509 -req -CA rootCA.crt -CAkey rootCA.key -in sub.csr -out sub.crt -days 365 -CAcreateserial
```

#### openssl ca 模块

ca命令能够签发证书请求文件生成证书

![openssl ca](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008eGmZEly1gortfb0gl3j311u0u013u.jpg)

```shell
openssl ca -config openssl.cnf -days 365 -create_serial -in harbor.csr -out harbor.crt  -extensions ca_ext -extensions req_ext -notext
```

### 创建一个自签名证书

```shell
# 创建根证书
openssl req -x509 -days 365 -newkey rsa:2048 -out rootCA.crt -keyout rootCA.key -nodes

# 创建子证书的私钥跟rsa
openssl genrsa -des3 -out sub.key 2048
openssl rsa -in sub.key -out sub.key #去除私钥的密码
openssl req -new -key sub.key -out sub.csr

# 使用根证书签名子证书
openssl x509 -req -CA rootCA.crt -CAkey rootCA.key -in sub.csr -out sub.crt -days 365 -set_serial 123 #-set_serial 用于设置sub.crt 的SN序列号

# 验证子证书
openssl verify -CAfile rootCA.crt sub.crt

# 查看子证书的公钥信息(可以看到issuer为根证书的信息)
openssl x509 -in sub.crt -noout -text 
```



### 创建带SAN的自签名证书

#### 什么叫SAN

SAN(Subject Alternative Name) 是 SSL 标准 x509 中定义的一个扩展。使用了 SAN 字段的 SSL 证书，可以扩展此证书支持的域名，使得一个证书可以支持多个不同域名的解析。(下面是百度证书中的SAN信息)

![百度SAN](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008eGmZEly1goqrfszbjkj31f30u07qz.jpg)

#### 如何创建带SAN的自签证书

​	上个步骤中，创建的自签名证书由于没有带SAN信息，所以无法在浏览器中使用(一般作为中间证书使用)，如果要在浏览器中使用，必须带有SAN信息，

​	在真实场景下，根证书自身就是可信的(内置在操作系统中), 然后由根证书签发中间证书，中间证书再签发最终的SAN证书, 所以一般的流程为先创建根证书，然后创建中间证书，由中间证书签发最终的证书。接下来将创建一个带SAN的自签名证书

1. 创建根证书

   ````shell
   openssl req -x509 -days 365 -newkey rsa:2048 -out rootCA.crt -keyout rootCA.key -nodes
   ````

2. 生成中间证书的私钥和CSR。

   ```shell
   openssl req -new -nodes -keyout intermediate.key -out intermediate.csr
   ```

3. 添加san配置信息如下(只需要修改域名信息即可), 其中alt_names 保存域名信息(暂时保存为openssl.cnf)

   ```shell
   [ req ]
   default_bits       = 2048
   distinguished_name = req_distinguished_name
   req_extensions     = req_ext
   
   [ ca ]
   default_ca = intermediate_ca
   
   [ intermediate_ca ]
   dir = .
   private_key = $dir/rootCA.key
   certificate = $dir/rootCA.crt
   new_certs_dir = $dir/
   serial = $dir/crt.srl
   database = $dir/db/index
   default_md = sha256
   policy = policy_any
   email_in_dn = no
   
   [ req_distinguished_name ]
   countryName                 = Country Name (2 letter code)
   stateOrProvinceName         = State or Province Name (full name)
   localityName                = Locality Name (eg, city)
   organizationName            = Organization Name (eg, company)
   commonName                  = Common Name (eg, your name or your server\'s hostname)
   emailAddress                = Email Address
   
   [ policy_any ]
   domainComponent = optional
   countryName = optional
   stateOrProvinceName = optional
   localityName = optional
   organizationName = optional
   organizationalUnitName = optional
   commonName = optional
   emailAddress = optional
   
   [ ca_ext ]
   keyUsage                = critical,keyCertSign,cRLSign
   # 注意这里设置了CA:true，表明使用该配置生成的证书是CA证书，可以用于签发用户证书
   basicConstraints        = critical,CA:true
   subjectKeyIdentifier    = hash
   authorityKeyIdentifier  = keyid:always
   
   [ req_ext ]
   subjectAltName          = @alt_names
   
   [alt_names]
   # 填写域名
   DNS.1   = hello.com
   DNS.2   = san.com
   DNS.3   = harbor.com
   DNS.4   = *.baidu.com
   ```

4. 创建数据库

   ```shell
   mkdir db
   touch db/index
   ```

5. 生成证书的私钥和CSR

   ```shell
   openssl req -newkey rsa:2048 -nodes -out harbor.csr -keyout harbor.key -config openssl.cnf
   ```

![csr](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008eGmZEly1goqs6p3em4j31de0h8goq.jpg)

验证CSR中是否包含文件

```shell
openssl req -in harbor.csr -noout -text
```

![验证csr](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008eGmZEly1goqs8wu6t8j30zf0u07ik.jpg)

6. 生成公钥

```shell
openssl ca -config openssl.cnf -days 365 -create_serial -in harbor.csr -out harbor.crt  -extensions ca_ext -extensions req_ext -notext
```

验证公钥是否包含SAN

```shell
openssl x509 -in harbor.crt -noout -text
```

![验证公钥](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008eGmZEly1gorpma0xcvj30uk0u0wub.jpg)

### 参考文档

- https://blog.csdn.net/baidu_36649389/article/details/54379935  //openssl 命令
- [openssl SAN证书](https://my.oschina.net/sskxyz/blog/1554093?utm_source=debugrun&utm_medium=referral) //如何创建带SAN证书
- [openssl SAN证书](https://geekflare.com/san-ssl-certificate/)  //英文版
- [cakey.pem 找不到解决方法](https://blog.csdn.net/php_boy/article/details/6660697)  //创建SAN证书中，无法找到cakey.pem的解决方法