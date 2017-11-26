---
title: 使用Openssl为Nginx生成证书
date: 2017年11月4日21:19:59
thumbnail: http://www.xiazaiba.com/uploadfiles/ico/2015/0423/2015042309095276482.png
tags: 
 - Nginx
 - Https
 - 证书
 - Openssl
categories: 
 - Nginx
---

我们在企业中，在使用到证书时，一般需要先生成“证书请求”（后缀大多为.csr），它包含你的名字和公钥，然后把这份请求交给诸如verisign等有CA服务公司（当然，连同几百美金），你的证书请求经验证后，CA用它的私钥签名，形成正式的证书发还给你。管理员再在web server上导入这个证书就行了。浏览器内置了很多商业的权威CA，所以你的网站是经过认证的。

而如果在企业内部网站有很多，很有可能公司会自己做CA。公司会先配置CA的私钥和公钥，私钥保存，自签CA，公钥则可以公开。而每个服务器需要生成证书请求提交给公司CA进行签发。这种情况下，访问内部网站时需要自行在客户端需要导入CA证书到受信任的颁发机构。

下面我们将介绍如何生成CA，如果使用CA签名证书，

<!--more-->
## 生成根证书

### a).生成根证书私钥(pem文件)

OpenSSL通常使用PEM（Privacy Enbanced Mail）格式来保存私钥，构建私钥的命令如下：
```
openssl genrsa -aes128 -passout pass:CAPassword -out ca.key 1024

该命含义如下：
genrsa——使用RSA算法产生私钥
-aes128 ——使用128位密钥的AES算法对私钥进行加密
-out——输出文件的路径
1024——指定私钥长度
```
由于Linux系统中可以使用history命令查看历史指令记录，所以出于安全方面的考量，一般不建议在命令中直接使用-passout。这与mysql登录的时候不在 -p选项里直接指定登录口令的原因是一致的。本文只是为了演示使用。

### b).生成根证书签发申请文件(csr文件)
使用上一步生成的私钥(pem文件)，生成证书请求文件(csr文件)：
```
openssl req -sha256 -new -key ca.key -out ca.csr \
-subj "/C=CN/ST=GuangDong/L=ShenZhen/O=Company/OU=Dept/CN=Name" -passin pass:CAPassword

该命令含义如下：
req——执行证书签发命令
-new——新证书签发请求
-key——指定私钥路径
-out——输出的csr文件的路径
-subj——证书相关的用户信息(subject的缩写),其中CN一般填写机器IP
-passin ——加入后可以代入CA密码，不加需要手动输入
```


### c).自签发根证书(cer文件)
csr文件生成以后，我们需要将其发送给CA认证机构进行签发，这样证书才有权威性。而认证机构是需要收费的，所以，这里我们仅做测试，我们对自己的证书生成文件进行自签发：
```
openssl x509 -req -days 3650 -sha256 -extensions v3_ca \
-signkey ca.key -in ca.csr -out ca.cer -passin pass:CAPassword

该命令的含义如下：
x509——生成x509格式证书
-req——输入csr文件
-days——证书的有效期（天）
-sha256——证书摘要采用sha256算法
-extensions——按照openssl.cnf文件中配置的v3_ca项添加扩展
-signkey——签发证书的私钥
-in——要输入的csr文件
-out——输出的cer证书文件
```

## 生成服务端证书
在实际使用时，如果是对外（客户）使用，我们对外提供的每一个网站，都需要使用权威机构签发的证书进行签发。一个证书可以包含企业的很多域名，例如百度的证书如下。
```shell
Subject: C=CN, ST=beijing, L=beijing, OU=service operation department, O=Beijing Baidu Netcom Science Technology Co., Ltd, CN=baidu.com
            X509v3 Subject Alternative Name: 
                DNS:baidu.com, DNS:baifubao.com, DNS:www.baidu.cn, DNS:www.baidu.com.cn, DNS:click.hm.baidu.com, DNS:log.hm.baidu.com, DNS:cm.pos.baidu.com, DNS:wn.pos.baidu.com, DNS:update.pan.baidu.com, DNS:mct.y.nuomi.com, DNS:*.baidu.com, DNS:*.baifubao.com, DNS:*.baidustatic.com, DNS:*.bdstatic.com, DNS:*.bdimg.com, DNS:*.hao123.com, DNS:*.nuomi.com, DNS:*.chuanke.com, DNS:*.trustgo.com, DNS:*.bce.baidu.com, DNS:*.eyun.baidu.com, DNS:*.map.baidu.com, DNS:*.mbd.baidu.com, DNS:*.fanyi.baidu.com, DNS:*.baidubce.com, DNS:*.mipcdn.com, DNS:*.news.baidu.com, DNS:*.baidupcs.com, DNS:*.aipage.com, DNS:*.aipage.cn, DNS:*.bcehost.com, DNS:baifae.com, DNS:*.safe.baidu.com, DNS:*.im.baidu.com, DNS:*.ssl2.duapps.com, DNS:*.baifae.com, DNS:*.baiducontent.com, DNS:*.dlnel.com
```
**但是，在企业内部，如果我们内部的网站够多，我们往往自己生成一个根证书（或者用外部权威机构签发一个二级CA），用这个根证书再去签发内部的各种服务端。一般这种场景才会有下面的步骤，而如果只是对外使用，使用权威机构签发的证书，直接进行nginx或tomcat等容器的配置就可以了。**

### a).生成服务端证书私钥(pem文件)
```
openssl genrsa -aes128 -passout pass:ServerPassword -out server.key 2048 
```
### b).生成成服务端证书签发申请文件(csr文件)
```
openssl req -new -sha256 -key server.key -out server.csr \
-subj "/C=CN/ST=GuangDong/L=ShenZhen/O=Company/OU=Dept/CN=localhost" -passin pass:ServerPassword
```
-subj——证书相关的用户信息(subject的缩写),**其中CN一般填写服务端的机器IP或域名，我们这里在本地测试，所以填写localhost**

### c).使用根证书签发服务端证书

我们使用已经找CA（权威结构）认证过的根证书（其实找CA认证是花钱的，我们使用的是自己签发的证书）对该证书进行签发：
```
openssl ca -in server.csr -out server.crt -passin pass:CAPassword \
-cert cacert.pem -keyfile cakey.pem \
-days 3650 -extensions v3_req

# 或者使用以下指令
openssl x509 -req -days 3650 -sha256 -extensions v3_req \
-CA cacert.pem -CAkey cakey.pem -CAserial ca.srl -CAcreateserial \
-passin pass:CAPassword -in server.csr -out server.crt
```
```
这里有必要解释一下这几个参数：
-CA——指定CA证书的路径
-CAkey——指定CA证书的私钥路径
-CAserial——指定证书序列号文件的路径
-CAcreateserial——表示创建证书序列号文件(即上方提到的serial文件)，创建的序列号文件默认名称为-CA，指定的证书名称后加上.srl后缀
```
## 为nginx开启https，并配置证书

```nginx
    server {
        listen       443 ssl;
        ssl_certificate      ssl3/server.crt;
        ssl_certificate_key  ssl3/server.key;

    }
```
核心配置只有三行，也可以采用`ssl on`来配置。

配置后我们就能够使用https来访问自己的页面了。但是访问的时候，如果我们的服务端使用自己生成的CA签发的证书，那么浏览器会提示我们，网站存在风险。下面我们需要导出证书到windows客户端进行安装。
## 导出证书到windows客户端进行安装
我们可以使用如下命令把证书导出为PKCS12格式：
```shell
openssl pkcs12 -export -out ca.p12 -inkey ca.key -in ca.cer

Enter pass phrase for ca.key:
Enter Export Password:
Verifying - Enter Export Password:
```
导出需要输入CA的私钥，以及输入两次导出密码（下面再windows上导入证书的时候需要使用）。

导出后，将ca.p12复制到windows环境，双击安装，输入密码，最后选择导入证书到 受信任的证书颁发机构即可。

最后再次访问可以发现网站风险已经消除。


## 其他：证书校验方法和证书格式转换命令

### 1，证书校验方法
```
# 使用CA证书校验身份证书是否合法
openssl verify -CAfile ca.cer server.cer

# 校验身份证书和私钥文件是否匹配（通过校验身份证书和私钥文件的modulus是否一致）
openssl x509 -in 身份证书 -modulus -noout 
openssl rsa -in 私钥文件 -passin pass:私钥口令  

openssl x509 -in server.cer -modulus -noout 
openssl rsa -in server.key -modulus -noout -passin pass:ServerPassword  

# 校验p12证书是否为当前CA签发，可以通过将p12证书导出PEM格式身份证书，再通过1中的方法校验

openssl pkcs12 -in 身份证书 -nokeys -passin pass:导入口令
```

### 2，证书转换 

```shell
# 根据server.cer(证书)和server.key(私钥)生成p12证书(server.p12)： 
openssl pkcs12 -export -aes128 {-descert} -in server.cer -inkey server.key -out server.p12 -passin pass:ServerPassword {-passout pass:P12Password} {-certfile ca.cer}

该命令的含义如下：
-export——导出pkcs12格式
-descert——使用3DES加密证书
-in——证书文件
-inkey——私钥文件
-passin——私钥密码
-out——输出的pkcs12文件
-passout——pkcs12文件导出密码，如果不指定该命令，则要求输入，不输入两次回车表示不加密码
-certfile——导出的证书带有CA证书

# 根据p12证书(server.p12)分解成server.cer和server.key： 

把p12格式转换为可读格式
openssl pkcs12 -in server.p12 -out server.p12.pem
---如果不想导出私钥，可加-nokeys
---如果不想加密私钥，可加-nodes
---如果不想导出证书，可加-nocerts

所以从p12文件中提取证书(.cer)
openssl pkcs12 -in server.p12 -out server.out.cer -nokeys

PFX文件中提取私钥(.key)
openssl pkcs12 -in server.p12 -nocerts -nodes -out server.out.key

PEM BASE64转为X.509文本格式
openssl x509 -in Key.pem -text -out cert.pem
```

## 参考

 1. [使用openssl进行证书格式转换](http://blog.csdn.net/linda1000/article/details/8676330)
 2. [如何使用OpenSSL工具生成根证书与应用证书](http://blog.csdn.net/mr_raptor/article/details/51854805)
 3. [OpenSSL生成根证书CA及签发子证书](https://my.oschina.net/itblog/blog/651434)
 4. [OpenSSL使用2（SSL,X.509,PEM,DER,CRT,CER,KEY,CSR,P12概念说明）（转）](http://www.cnblogs.com/EasonJim/p/6291539.html)
 5. [使用 OpenSSL 实现私钥和证书的转换](https://segmentfault.com/a/1190000006808275)