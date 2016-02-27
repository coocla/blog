title: 利用let's encrypt为网站免费启用https
categories: Linux
tags: [linux, https, let's encrypt, 免费证书]
date: 2016-02-27 23:15:12
---
## 概览
Let's Encrypt旨在为每个站点提供免费的基于SSL证书，以加速HTTP向HTTPS的过渡，恰逢上周HTTP2的发布，对于HTTPS的优化，其速度显著优于HTTP1.x(某些应用场景)
## 特点
* 免费：任何站点都可以免费申请一个针对其域名的有效证书
* 自动：证书的申请以及配置在web服务器上整个过程完全自动化, 并且支持后台更新
* 安全：提供业内最新的安全技术和最好的实践
* 透明：所有证书的签发与撤销记录均对需要调查的人员开放
* 开放：自动化执行的发放与更新将遵循开放标准，并且允许自定义插件
* 合作：对整个互联网都有益的社区行动, 不由任何一个组织控制<!--more-->

## 实践
### 准备
* 操作系统：CentOS6.6

### 安装
```
git clone https://github.com/letsencrypt/letsencrypt
cd letsencrypt
./letsencrypt-auto run --debug
```
安装过程中可能会安装较多依赖的第三方库,这对于线上环境无疑是不够友好和安全的，我们可以选择使用docker。
```
docker run -it --rm -p 443:443 -p 80:80 --name letsencrypt \
           -v "/etc/letsencrypt:/etc/letsencrypt" \
           -v "/var/lib/letsencrypt:/var/lib/letsencrypt" \ 
           quay.io/letsencrypt/letsencrypt:latest auth
```
我们利用官方已经制作好的letencrypt镜像
### 快速上手
为域名**coocla.org**生成一个证书
```
letsencrypt -d coocla.org certonly
```
如果一切顺利, 查看/etc/letsencrypt将会看到如下结构:

```
/etc/letsencrypt/
├── accounts
│   └── acme-staging.api.letsencrypt.org
│       └── directory
│           └── a68aa061212sd65a51234eaeadda9081
│               ├── meta.json
│               ├── private_key.json
│               └── regr.json
├── archive
│   └── coocla.org
│       ├── cert1.pem
│       ├── chain1.pem
│       ├── fullchain1.pem
│       └── privkey1.pem
├── csr
│   └── 0000_csr-letsencrypt.pem
├── keys
│   └── 0000_key-letsencrypt.pem
├── live
│   └── coocla.org
│       ├── cert.pem -> ../../archive/coocla.org/cert1.pem
│       ├── chain.pem -> ../../archive/coocla.org/chain1.pem
│       ├── fullchain.pem -> ../../archive/coocla.org/fullchain1.pem
│       └── privkey.pem -> ../../archive/coocla.org/privkey1.pem
└── renewal
    └── coocla.org.conf
```
* live 目录下存放的将会链接到最新的证书和私钥
* csr keys 用来存放当前代理的授权密钥对
* account  用来存放证书的管理信息, 这里涉及ACME
* renewal  存放当前代理所管理的域的信息

使用申请的证书及私钥就可以配置web服务器了
> 注意: 由于letsencrypt CA的证书从2015-11-17日开始才被信任, 如果你使用在此时间之前申请的证书, 可能会遇到浏览器不信任的错误提示.

### 原理
letsencrypt 通过 ACME() 协议, 使人们可以轻松建立HTTPS服务, 并且在无人干预的情况下可以自动获取浏览器所信任的证书. 实现这些, 需要两步:
1. 首先, letsencrypt客户端会向letsencrypt CA(下文简称为:CA)证明web服务器控制一个域
2. 其次, letsencrypt客户端(下文简称为:代理)可以请求, 更新, 吊销该域的证书

### 域的验证
CA 通过公钥来验证服务器管理员, 当代理第一次与CA通信时, 它将生成一个新的密钥对, 并且告诉CA该服务器控制一个或多个域名.
当代理启动时, 将会询问CA需要什么才可以证明该代理控制(管理)域**coocla.org**, CA将回复一个或多个问题, 需要代理进行证明, 例如CA可能会给代理一个选择:
* 提供域**coocla.org**下的记录
* 在域**https://coocla.org**下配置一个资源

![域的验证](http://7xk38j.com1.z0.glb.clouddn.com/letsencrypt/howitworks_challenge.png)
期间, CA也会提供一个临时的秘钥给代理, 要求代理使用该秘钥进行签署, 以证明它所管理的key
当代理完成CA提供的证明问题后, 将会通知CA它已经处于就绪状态, 此时如果这些证明问题都通过后, 并且签名也通过后, 代理的公钥将会被CA授权管理域**coocla.org**, 这对秘钥就被称作为授权密钥对, 摘自官网图片:
![授权秘钥对](http://7xk38j.com1.z0.glb.clouddn.com/letsencrypt/howitworks_authorization.png)
### 证书的颁发、更新与吊销
一旦代理对证书进行颁发、更新或吊销时, 只需要简单的将证书的管理信息和授权密钥对发送给CA.
当代理获取证书时, 需要构建一个PKCS#10的证书签名请求, 要求CA发行关于域及指定公钥的证书.
通常一个证书签名请求中包含由公钥对应的私钥的签名, 并且后续所有的证书签名请求中代理均会使用授权秘钥进行签名, 当CA收到证书签名请求后, 它只需要验证这两个签名是否均正常. 如果一切正常, 它将使用代理发送的公钥颁发一个证书返回给代理.
![颁发证书](http://7xk38j.com1.z0.glb.clouddn.com/letsencrypt/howitworks_certificate.png)
吊销证书的过程也类似, 代理使用授权秘钥发起吊销的请求到CA, CA来验证请求, 如果是这样, 它将发表吊销消息到吊销通道中(像CRLs, OCSP), 这样浏览器就会知道不应该信任已经吊销的证书.
![吊销证书](http://7xk38j.com1.z0.glb.clouddn.com/letsencrypt/howitworks_revocation.png)
### 官方支持的插件
| 插件 | 支持验证 | 默认安装 |
|--------|---|---|
|standalone|Y|N|
|apache|Y|Y|
|webroot|Y|N|
|manual|Y|N|
|nginx|Y|Y|

### 证书的位置
所有生成的秘钥和证书均可在/etc/letsencrypt/live/$domain 目录下找到. 这里可以使用链接的方式链接到所需要配置的web服务配置目录下, 因为/etc/letsencrypt下的文件将会是最新更新的文件,拥有最新的内容.
> /etc/letsencrypt/archive 和 /etc/letsencrypt/keys 包含了所有的keys、certificates. 只有/etc/letsencrypt/live 链接的才是最新的更新内容.

以下文件可以用:
* privkey.pem   证书的私钥文件, 在 apache 服务器配置中,它被用作*SSLCertificateKeyFile*, 在 nginx 配置中, 被用作 *ssl_certificate_key*
* cert.pem    服务器的证书文件
* chain.pem    浏览器所需要的所有的证书, 除去服务器本身的证书. 例如: 根或中级证书
* fullchain.pem    所有的证书, 包含服务器本身的证书，等于 cert.pem 和 chain.pem 的总和

在使用过程中, 必须要使用 chain.pem 或 fullchain.pem, 不能仅仅使用cert.pem, 否则将会引发错误
### 配置文件
```
# 使用4096长度的RSA秘钥
rsa-key-size = 4096

# 总是使用临时测试服务器
server = https://acme-staging.api.letsencrypt.org/directory

# 使用指定的地址更新或取消注册的证书
# email = foo@example.com

# 使用文本接口代替ncurses
# text = True

# 使用独立的认证端口443
# authenticator = standalone
# standalone-supported-challenges = tls-sni-01

# 取消webroot的验证. 使用 webroot-path 的值代替它, 这个路径是指web服务器的根
# authenticator = webroot
# webroot-path = /usr/share/nginx/html
```
默认的配置文件从以下几个位置加载:
* /etc/letsencrypt/cli.ini
* $XDG_CONFI_HOST/letsencrypt/cli.ini
* ~/.config/letsencrypt/cli.ini

### 注意
> 1. Let's Encrypt的证书的有效期限是90天, 因此使用者必须至少每三个月更新一次你的证书
> 2. 因为letsencrypt的工作原理, 代理在向CA发起证书的发行或吊销时, 需要能够证明自己可以控制该域名, 所以代理需要运行在域名解析的那台服务器上

更新证书只需要使用相同的值再次执行 **letsencrypt** (或 **letsencrypt-auto** ), 配合相关的CLI命令参数, 或者使用配置文件, 或者配合crontab进行使用







</br>
