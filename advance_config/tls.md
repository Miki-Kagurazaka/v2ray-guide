# TLS
从 v1.19 起引入了 TLS，TLS 中文译名是传输层安全，如果你没听说过，请 Google 了解一下。以下给出些我认为介绍较好的文章链接：

 [阮一峰的网络日志：SSL/TLS协议运行机制的概述](http://www.ruanyifeng.com/blog/2014/02/ssl_tls.html)

 [维基百科：传输层安全协议](https://zh.wikipedia.org/wiki/%E5%82%B3%E8%BC%B8%E5%B1%A4%E5%AE%89%E5%85%A8%E5%8D%94%E8%AD%B0)

 [编程随想的博客：扫盲 HTTPS 和 SSL/TLS 协议[1]：背景知识、协议的需求、设计的难点](https://program-think.blogspot.com/2014/11/https-ssl-tls-1.html)

 [编程随想的博客：扫盲 HTTPS 和 SSL/TLS 协议[2]：可靠密钥交换的难点，以及身份认证的必要性](https://program-think.blogspot.com/2014/11/https-ssl-tls-2.html)

 但是，Shadowsocks 的作者 clowwindy 却认为[翻墙不该用 SSL](https://gist.github.com/clowwindy/5947691)。那么到底该不该用？对此我不作评论，各位自行思考。这里我只教大家如何开启 TLS。

 ## 注册一个域名

如果已经注册有域名了可以跳过。
域名有免费的有付费的，总体来说付费的会优于免费的，具体差别请 Google。如果你不舍得为一个域名每年花点钱，用个免费域名也可以。为了方便，这里我将以免费域名为例。

关于如何注册一个免费域名，我发现有个家伙写得很详细，就不多说了。请参考：

[教你申请.tk/.ml/.cf/.gq/.ga等免费域名](https://www.dou-bi.co/dbwz-3/)

至于注册其它付费的域名请 Google 吧，差不多都是大同小异的。

**注册好域名之后务必记得设置 DNS 解析到你的 VPS !**

**据了解，在 freenom 注册的域名在对应的 IP 上要有一个网站，否则注册之后域名会被回收。如果您只是想用免费域名在 V2Ray 用一下 TLS，又不愿意（懒得、不会）建站，建议您看看您的亲朋好友谁有手上有域名的，向他们要一个二级域名就行了**

以下假设注册的域名为 mydomain.me，请将之替换成自己的域名。


## 证书生成
使用 TLS 需要证书，证书也有免费付费的，同样的这里使用免费证书，证书认证机构为 [Let's Encrypt](https://letsencrypt.org/)。
证书的生成有许多方法，这里使用的是最简单的方法：使用 [acme.sh](https://github.com/Neilpang/acme.sh) 脚本生成，本部分说明部分内容参考于[acme.sh README](https://github.com/Neilpang/acme.sh/blob/master/README.md)

不严格来说，证书有两种，一种是 ECC 证书（内置公钥是 ECDSA 公钥），一种是 RSA 证书（内置 RSA 公钥）。简单来说，同等长度 ECC 比 RSA 更安全,也就是说在具有同样安全性的情况下，ECC 的密钥长度比 RSA 短得多（加密解密会更快）。但问题是 ECC 的兼容性会差一些，Android 4.x 以下和 Windows XP 不支持。只要您的设备不是非常老的老古董，强烈建议使用 ECC 证书。

以下将给出两种证书的生成方法。

证书生成只需在服务器上操作。

### 安装 acme.sh
执行以下命令，acme.sh 会安装到 ~/.acme.sh 目录下：
```
curl  https://get.acme.sh | sh
```
### 使用 acme.sh 生成证书

执行以下命令生成证书：
```
acme.sh  --issue -d mydomain.me --standalone -k ec-256
```
`-k` 表示密钥长度，后面的值可以是 `ec-256` 、`ec-284`、`2048`、`3072`、`4096`、`8192`，带有 `ec` 表示生成的是 ECC 证书，没有则是 RSA 证书。在安全性上 256 位的 ECC 证书等同于 3072 位的 RSA 证书。

上面的命令会临时监听 80 端口，请确保执行该命令前 80 端口没有使用。或者使用 `--httpport` 指定其它未使用的端口，如：
```
acme.sh  --issue -d mydomain.me --standalone -k ec-256 --httpport 88
```

### 安装证书和密钥

#### ECC 证书
将证书和密钥安装到 /etc/v2ray 中：
```
acme.sh --installcert -d mydomain.me --fullchainpath /etc/v2ray.crt --keypath /etc/v2ray/v2ray.key --ecc
```
#### RSA 证书
```
acme.sh --installcert -d mydomain.me --fullchainpath /etc/v2ray.crt --keypath /etc/v2ray/v2ray.key
```

**注意：无论什么情况，密钥(即上面的v2ray.key)都不能泄漏**

## 配置 V2Ray

服务器配置：
```javascript
{
  "inbound": {
    "port": 443, // 建议使用 443 端口
    "protocol": "vmess",    
    "settings": {
      "clients": [
        {
          "id": "23ad6b10-8d1a-40f7-8ad0-e3e35cd38297",  
          "alterId": 64
        }
      ]
    },
    "streamSettings": {
      "network": "tcp",
      "security": "tls", // security 要设置为 tls 才会启用 TLS
      "tlsSettings": {
        "certificates": [
          {
            "certificateFile": "/etc/v2ray/v2ray.crt", //证书文件
            "keyFile": "/root/acmessl/v2ray.key" //密钥文件
          }
        ]
      }
    }
  },
  "outbound": {
    "protocol": "freedom",
    "settings": {}
  }
}
```

客户端配置：

```javascript
{
  "inbound": {
    "port": 1080,
    "protocol": "socks",
    "settings": {
      "auth": "noauth"
    }
  },
  "outbound": {
    "protocol": "vmess",
    "settings": {
      "vnext": [
        {
          "address": "mydomain.tk",
          "port": 443,
          "users": [
            {
              "id": "23ad6b10-8d1a-40f7-8ad0-e3e35cd38297",
              "alterId": 64
            }
          ]
        }
      ]
    },
    "streamSettings": {
      "network": "tcp",
      "security": "tls" // 客户端的 security 也要设置为 tls
    }
  }
}
```