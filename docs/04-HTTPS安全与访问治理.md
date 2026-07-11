---
title: 04-HTTPS安全与访问治理
date: 2026-07-11 00:00:00
description: Nginx 第四阶段学习文档：HTTPS、TLS、证书、HTTP/2、HSTS、OCSP Stapling、访问控制、基础认证、限流、限连接、防盗链、安全 Header 与入口层安全治理
tags:
  - Nginx
categories:
  - Nginx
realm: wujing
cover: /image/post_cover/nginx-7stage-roadmap-min.svg
rank: 91
top_img: false
---
# 04-HTTPS安全与访问治理

## 1. 阶段定位

第四阶段是 Nginx 从“稳定代理入口”进入“安全可靠入口层”的关键阶段。前三阶段已经解决了部署、配置体系、请求匹配、反向代理、负载均衡、Header 传递、超时和 upstream 排障。本阶段重点解决下面这些问题：

```text
一个公网请求进入 Nginx 前后：
  -> HTTP 是否应该强制跳转 HTTPS
  -> HTTPS 证书如何部署、更新和保护
  -> TLS 协议版本和加密套件如何选择
  -> 证书链不完整、私钥错误、SNI 异常如何排查
  -> HTTP/2、HSTS、OCSP Stapling 如何启用
  -> 哪些接口需要 IP 白名单、基础认证或访问限制
  -> 如何限制请求速率、连接数和上传大小
  -> 如何防止图片、文件、视频等资源被盗链
  -> 如何配置基础安全 Header
  -> 400、403、413、429 等入口层问题如何定位
```

Nginx 放在入口层时，通常是用户流量进入系统的第一道可控边界。这个位置决定了它不仅要把请求转发到后端，还要承担 TLS 终止、协议治理、访问控制、流量削峰、恶意请求阻断、安全 Header 注入、日志审计和基础防护能力。

完成本阶段后，应该具备下面的能力：

- 能独立为站点配置 HTTPS，并正确处理证书链、私钥权限和自动续期。
- 能解释 TLS 握手、证书、公钥、私钥、CA、中间证书链的作用。
- 能根据生产安全要求选择 TLS 协议版本、加密套件和会话复用策略。
- 能配置 HTTP 自动跳转 HTTPS，并理解跳转对业务、SEO、接口调用和回滚的影响。
- 能开启 HTTP/2，并知道它适合什么场景、不适合解决什么问题。
- 能配置 HSTS、OCSP Stapling，并理解其收益和风险边界。
- 能配置 IP 白名单、黑名单、基础认证、限流、限连接、防盗链和上传大小限制。
- 能通过 access log、error log、curl、openssl、浏览器开发者工具定位入口层安全问题。

## 2. 学习边界

### 2.1 本阶段重点

本阶段重点关注 Nginx 入口层安全和访问治理，包括：

- HTTPS 基础概念。
- TLS 握手基础流程。
- 证书、私钥、公钥、CA、中间证书链。
- 自签证书实验。
- Let's Encrypt 真实证书部署。
- 证书文件权限和私钥保护。
- `ssl_certificate`、`ssl_certificate_key`、`ssl_protocols`、`ssl_ciphers`。
- HTTP 强制跳转 HTTPS。
- SNI 与多域名证书配置。
- HTTP/2 开启方式和验证方法。
- HSTS 作用、配置和回滚风险。
- OCSP Stapling 基础配置。
- 安全 Header 基础配置。
- `server_tokens off` 隐藏版本号。
- `allow`、`deny` 访问控制。
- `auth_basic` 基础认证。
- `limit_req_zone`、`limit_req` 请求限流。
- `limit_conn_zone`、`limit_conn` 连接数限制。
- `client_max_body_size` 上传大小限制。
- `valid_referers` 防盗链。
- 400、403、413、429 等问题排查。

### 2.2 本阶段暂不深入

下面内容与安全有关，但不是本阶段重点：

- WAF 深度规则体系。
- ModSecurity / NAXSI 规则编写。
- OpenResty Lua 动态鉴权。
- JWT、OAuth2、OIDC 网关级认证。
- mTLS 双向认证的复杂企业落地。
- Kubernetes Ingress Controller 安全注解。
- 零信任网关完整架构。
- DDoS 清洗、BGP 高防和云厂商边缘防护。
- Nginx 源码中的 SSL 模块实现。

本阶段目标不是把 Nginx 变成完整安全网关，而是掌握大多数生产入口层必须具备的基础安全能力。

## 3. HTTPS 是什么

HTTPS 可以理解为：

```text
HTTPS = HTTP + TLS
```

HTTP 本身是明文协议。浏览器、客户端、代理、运营商网络、公司网关、公共 Wi-Fi 等链路上的节点理论上都可能看到明文请求内容。如果请求里有登录凭证、Cookie、Token、订单信息、个人信息，就存在泄露和篡改风险。

HTTPS 通过 TLS 解决三个核心问题：

| 能力 | 说明 | 解决的问题 |
| --- | --- | --- |
| 加密 | 客户端和服务端协商会话密钥，后续通信加密 | 防止被窃听 |
| 认证 | 客户端验证服务端证书是否可信 | 防止访问假站点 |
| 完整性 | TLS 记录层校验数据是否被篡改 | 防止中间人篡改 |

HTTPS 不只是“证书能用”。在生产环境中，还要关注：

- 证书是否由可信 CA 签发。
- 证书域名是否匹配。
- 证书链是否完整。
- 私钥是否泄露。
- TLS 协议版本是否安全。
- 加密套件是否过旧。
- 证书是否会自动续期。
- HTTP 是否会跳转到 HTTPS。
- 安全 Header 是否正确。

## 4. TLS 基础概念

### 4.1 SSL 与 TLS

日常很多人仍然说“SSL 证书”，但严格来说，现在主流使用的是 TLS。

| 名称 | 说明 | 当前状态 |
| --- | --- | --- |
| SSL 2.0 | 早期安全协议 | 已废弃 |
| SSL 3.0 | 早期安全协议 | 已废弃 |
| TLS 1.0 | SSL 后续版本 | 已不推荐 |
| TLS 1.1 | 过渡版本 | 已不推荐 |
| TLS 1.2 | 长期主流版本 | 仍广泛使用 |
| TLS 1.3 | 新一代版本 | 推荐启用 |

生产环境一般建议：

```nginx
ssl_protocols TLSv1.2 TLSv1.3;
```

除非需要兼容非常老旧的客户端，否则不要启用 `TLSv1`、`TLSv1.1`，更不要启用 SSLv2、SSLv3。

### 4.2 对称加密与非对称加密

TLS 中同时使用了对称加密和非对称加密。

非对称加密特点：

- 有一对密钥：公钥和私钥。
- 公钥可以公开。
- 私钥必须保密。
- 用于身份认证和密钥交换等过程。
- 计算成本相对高。

对称加密特点：

- 通信双方使用同一个会话密钥。
- 加解密速度快。
- 适合大量数据传输。

TLS 的基本思路是：

```text
先用非对称机制解决身份认证和密钥协商
再用协商出的对称密钥加密后续 HTTP 数据
```

### 4.3 证书

证书本质上是一个由 CA 签名的身份文件。它证明：某个公钥属于某个域名或组织。

证书通常包含：

- 域名。
- 公钥。
- 签发者。
- 有效期。
- 签名算法。
- 使用者备用名称 SAN。
- 证书用途。
- CA 对证书内容的数字签名。

Nginx 中配置的证书文件通常是 PEM 格式：

```nginx
ssl_certificate /etc/nginx/ssl/example.com/fullchain.pem;
ssl_certificate_key /etc/nginx/ssl/example.com/privkey.pem;
```

其中：

- `ssl_certificate` 指向证书文件，生产环境通常应使用完整证书链文件。
- `ssl_certificate_key` 指向私钥文件。

### 4.4 公钥与私钥

公钥可以包含在证书中并公开给客户端。

私钥必须严格保护。如果私钥泄露，攻击者可能伪装成你的站点，或者在特定条件下解密历史或当前通信流量。

私钥保护建议：

- 不要提交到 Git。
- 不要通过聊天工具、邮件明文传输。
- 文件权限尽量收紧。
- 只允许 Nginx 启动所需用户或 root 读取。
- 证书更换时不要误删私钥。
- 怀疑泄露时立即吊销并重新签发证书。

示例权限：

```bash
sudo chown root:root /etc/nginx/ssl/example.com/privkey.pem
sudo chmod 600 /etc/nginx/ssl/example.com/privkey.pem
```

### 4.5 CA

CA 是 Certificate Authority，证书颁发机构。浏览器和操作系统内置了一批受信任的根 CA。只要服务端证书能沿着证书链追溯到受信任根 CA，浏览器就会认为证书可信。

常见 CA 或证书来源：

- Let's Encrypt。
- DigiCert。
- GlobalSign。
- Sectigo。
- 云厂商证书服务。
- 企业内部 CA。

### 4.6 根证书、中间证书和服务器证书

证书链通常不是只有一个证书，而是一条链：

```text
Root CA
  -> Intermediate CA
    -> Server Certificate
```

浏览器通常信任根证书。服务器需要发送服务器证书和中间证书，让客户端能构造完整信任链。

生产中常见错误是只配置了站点证书，没有配置中间证书，导致部分客户端报错。因此 Nginx 的 `ssl_certificate` 通常应该配置为 fullchain 文件，而不是单独 server certificate：

```nginx
ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
```

## 5. TLS 握手基础流程

### 5.1 TLS 1.2 简化流程

TLS 1.2 握手可以简化理解为：

```text
ClientHello
  -> 客户端告诉服务端支持的 TLS 版本、加密套件、随机数、SNI 等
ServerHello
  -> 服务端选择 TLS 版本和加密套件
Certificate
  -> 服务端发送证书链
Key Exchange
  -> 双方协商会话密钥
Finished
  -> 握手完成，后续使用对称密钥加密通信
```

浏览器访问 HTTPS 站点时，真正的 HTTP 请求是在 TLS 握手完成后才发送的。

### 5.2 TLS 1.3 简化流程

TLS 1.3 对握手流程进行了简化，减少往返次数，并移除了很多不安全算法。可以简单理解为：

```text
TLS 1.3 更快、更安全、更少历史包袱
```

如果 Nginx 和 OpenSSL 版本支持，建议启用 TLS 1.3。

### 5.3 SNI

SNI 是 Server Name Indication。客户端在 TLS 握手的 ClientHello 中告诉服务端自己要访问哪个域名。

SNI 解决的问题是：同一个 IP 和端口上部署多个 HTTPS 域名。没有 SNI 时，Nginx 在 TLS 握手阶段不知道客户端要访问哪个域名，只能返回默认证书。

多域名 HTTPS 示例：

```nginx
server {
    listen 443 ssl;
    server_name app.example.com;

    ssl_certificate /etc/nginx/ssl/app.example.com/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/app.example.com/privkey.pem;
}

server {
    listen 443 ssl;
    server_name admin.example.com;

    ssl_certificate /etc/nginx/ssl/admin.example.com/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/admin.example.com/privkey.pem;
}
```

客户端必须支持 SNI，才能在同一个 IP:443 上正确拿到不同域名的证书。现代浏览器和主流客户端基本都支持。

## 6. Nginx HTTPS 模块与编译检查

### 6.1 检查是否支持 SSL

Nginx 要支持 HTTPS，需要编译时带有 `http_ssl_module`。

```bash
nginx -V 2>&1 | tr ' ' '\n' | grep http_ssl_module
```

如果能看到：

```text
--with-http_ssl_module
```

说明当前 Nginx 支持 HTTPS。

### 6.2 检查是否支持 HTTP/2

HTTP/2 需要 `http_v2_module`。

```bash
nginx -V 2>&1 | tr ' ' '\n' | grep http_v2_module
```

如果能看到：

```text
--with-http_v2_module
```

说明支持 HTTP/2。

### 6.3 检查 OpenSSL 版本

TLS 能力还取决于 OpenSSL 版本。

```bash
nginx -V 2>&1 | grep -o 'OpenSSL[^)]*'
openssl version -a
```

需要注意：

- Nginx 运行时使用的 OpenSSL 可能与系统命令 `openssl` 显示的不完全一致。
- 源码编译 Nginx 时可能静态链接指定 OpenSSL 源码。
- TLS 1.3 需要较新的 OpenSSL 支持。

## 7. 自签证书实验

### 7.1 自签证书适用场景

自签证书适合学习、内网测试、临时环境。不适合公网生产站点，因为浏览器默认不信任自签 CA，会显示安全警告。

适用场景：

- 本地 HTTPS 实验。
- 内网测试环境。
- 临时验证 Nginx HTTPS 配置。
- 企业内部自建 CA 场景。

不适用场景：

- 面向公众用户的网站。
- 小程序、App、浏览器正式环境。
- 第三方回调地址。
- 支付、登录、OAuth 回调等严肃生产链路。

### 7.2 生成自签证书

创建目录：

```bash
sudo mkdir -p /etc/nginx/ssl/selfsigned
cd /etc/nginx/ssl/selfsigned
```

生成私钥和自签证书：

```bash
sudo openssl req -x509 -nodes -days 365 \
  -newkey rsa:2048 \
  -keyout selfsigned.key \
  -out selfsigned.crt \
  -subj "/C=CN/ST=Beijing/L=Beijing/O=Study/OU=Nginx/CN=nginx.local"
```

说明：

- `-x509` 表示生成自签证书。
- `-nodes` 表示私钥不加密码，便于 Nginx 启动读取。
- `-days 365` 表示有效期 365 天。
- `rsa:2048` 表示生成 2048 位 RSA 私钥。
- `CN=nginx.local` 表示证书主体域名。

### 7.3 配置 HTTPS server

```nginx
server {
    listen 443 ssl;
    server_name nginx.local;

    ssl_certificate /etc/nginx/ssl/selfsigned/selfsigned.crt;
    ssl_certificate_key /etc/nginx/ssl/selfsigned/selfsigned.key;

    ssl_protocols TLSv1.2 TLSv1.3;

    root /usr/share/nginx/html;
    index index.html;
}
```

检查配置并重载：

```bash
sudo nginx -t
sudo systemctl reload nginx
```

本地验证：

```bash
curl -k https://127.0.0.1/
curl -k --resolve nginx.local:443:127.0.0.1 https://nginx.local/
```

`-k` 表示忽略证书校验。只适合测试，不适合生产脚本。

### 7.4 浏览器验证

如果使用浏览器访问自签 HTTPS，通常会提示证书不受信任。这是正常现象，因为证书没有由浏览器信任的 CA 签发。

验证重点不是让浏览器完全信任，而是确认：

- Nginx 监听了 443。
- TLS 握手成功。
- 证书能被服务端发送。
- HTTPS 站点能返回内容。

## 8. Let's Encrypt 证书部署

### 8.1 Let's Encrypt 是什么

Let's Encrypt 是免费、自动化、开放的证书颁发机构。它适合大多数公网网站和测试域名。

特点：

- 免费。
- 自动签发。
- 支持自动续期。
- 证书有效期通常较短。
- 需要证明你控制该域名。

### 8.2 申请证书前提

申请公网证书前，需要准备：

- 一个真实域名。
- 域名 DNS 已解析到当前服务器公网 IP。
- 服务器 80 或 443 端口可从公网访问。
- 防火墙和安全组已放行。
- Nginx 可以响应该域名的 HTTP 请求。

验证域名解析和 80 端口：

```bash
dig app.example.com
nslookup app.example.com
curl -I http://app.example.com/
```

### 8.3 安装 certbot

Ubuntu / Debian：

```bash
sudo apt update
sudo apt install -y certbot python3-certbot-nginx
```

Rocky Linux / AlmaLinux / CentOS Stream：

```bash
sudo dnf install -y epel-release
sudo dnf install -y certbot python3-certbot-nginx
```

如果发行版仓库版本较旧，也可以使用 snap 安装，但生产环境需要统一运维规范，不建议混用多种安装方式。

### 8.4 使用 Nginx 插件签发

单域名：

```bash
sudo certbot --nginx -d app.example.com
```

多个域名：

```bash
sudo certbot --nginx -d example.com -d www.example.com
```

签发成功后，常见文件路径为：

```text
/etc/letsencrypt/live/app.example.com/fullchain.pem
/etc/letsencrypt/live/app.example.com/privkey.pem
```

Nginx 配置通常类似：

```nginx
ssl_certificate /etc/letsencrypt/live/app.example.com/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/app.example.com/privkey.pem;
```

### 8.5 使用 webroot 模式签发

如果不希望 certbot 自动修改 Nginx 配置，可以使用 webroot 模式。

先配置验证目录：

```nginx
server {
    listen 80;
    server_name app.example.com;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 200 "ok\n";
    }
}
```

创建目录并加载：

```bash
sudo mkdir -p /var/www/certbot
sudo nginx -t
sudo systemctl reload nginx
```

申请证书：

```bash
sudo certbot certonly --webroot \
  -w /var/www/certbot \
  -d app.example.com
```

webroot 模式的优点：

- 不自动改 Nginx 配置。
- 更适合自动化和配置审查。
- 多站点场景更可控。

### 8.6 自动续期

测试续期：

```bash
sudo certbot renew --dry-run
```

查看定时任务：

```bash
systemctl list-timers | grep certbot
sudo crontab -l
ls -l /etc/cron.d/
```

续期后需要让 Nginx 加载新证书。很多 certbot 安装方式会自动处理，也可以显式配置 deploy hook：

```bash
sudo certbot renew --deploy-hook "systemctl reload nginx"
```

生产建议：

- 对证书到期时间做监控。
- 不要等到过期后才处理。
- 续期链路应纳入变更和告警。
- reload 前先执行 `nginx -t`。

## 9. Nginx HTTPS 基础配置

### 9.1 最小 HTTPS 配置

```nginx
server {
    listen 443 ssl;
    server_name app.example.com;

    ssl_certificate /etc/letsencrypt/live/app.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/app.example.com/privkey.pem;

    location / {
        proxy_pass http://app_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

这个配置能工作，但还不够完整。生产环境还需要补充：

- TLS 协议版本。
- 加密套件。
- 会话缓存。
- 安全 Header。
- HTTP 到 HTTPS 跳转。
- 日志。
- 访问控制或限流策略。

### 9.2 listen 443 ssl

开启 HTTPS 的核心是：

```nginx
listen 443 ssl;
```

如果忘记 `ssl`，可能导致 443 端口按明文 HTTP 处理，客户端访问 HTTPS 会失败。

旧版本 Nginx 中也常见：

```nginx
ssl on;
```

现在不建议使用这种老写法，推荐直接在 `listen` 中声明 `ssl`。

### 9.3 证书与私钥必须匹配

证书和私钥不匹配时，`nginx -t` 可能报类似错误：

```text
SSL_CTX_use_PrivateKey("/path/privkey.pem") failed
key values mismatch
```

检查证书和私钥是否匹配：

```bash
openssl x509 -noout -modulus -in fullchain.pem | openssl md5
openssl rsa -noout -modulus -in privkey.pem | openssl md5
```

两个 md5 值一致，通常表示证书和私钥匹配。

如果是 ECDSA 私钥，检查命令会不同：

```bash
openssl x509 -noout -pubkey -in fullchain.pem | openssl pkey -pubin -outform pem | openssl sha256
openssl pkey -pubout -in privkey.pem -outform pem | openssl sha256
```

### 9.4 证书链完整性

检查服务端实际返回的证书链：

```bash
openssl s_client -connect app.example.com:443 -servername app.example.com -showcerts </dev/null
```

重点观察：

- 证书 Subject 是否是目标域名。
- SAN 中是否包含目标域名。
- Verify return code 是否为 0。
- 是否返回了中间证书。

如果看到类似：

```text
Verify return code: 0 (ok)
```

说明证书链验证通过。

如果部分浏览器正常、部分客户端失败，优先怀疑：证书链不完整、客户端根证书库过旧、TLS 版本或加密套件不兼容、SNI 没有正确传递。

## 10. TLS 协议版本

### 10.1 推荐配置

常见推荐：

```nginx
ssl_protocols TLSv1.2 TLSv1.3;
```

这表示只允许 TLS 1.2 和 TLS 1.3。

### 10.2 不推荐启用旧协议

不建议：

```nginx
ssl_protocols SSLv3 TLSv1 TLSv1.1 TLSv1.2;
```

原因：

- SSLv3 已严重不安全。
- TLS 1.0 和 TLS 1.1 已被主流浏览器淘汰。
- 安全扫描通常会将旧协议标为风险。
- 支付、金融、政企等场景通常要求禁用旧协议。

### 10.3 兼容性取舍

如果业务必须兼容非常老旧设备，需要先明确：

- 老客户端占比是多少。
- 访问的是公网用户还是内部设备。
- 是否能升级客户端。
- 是否能单独提供兼容域名。
- 是否接受安全扫描风险。

一种折中方式是把兼容性流量隔离到独立域名：

```text
secure.example.com   -> 只支持 TLS 1.2/1.3
legacy.example.com   -> 临时兼容旧客户端，严格限制访问范围
```

不要为了少量旧客户端降低所有用户的安全基线。

## 11. 加密套件 ssl_ciphers

### 11.1 ssl_ciphers 是什么

`ssl_ciphers` 用于控制 TLS 1.2 及以下可用的加密套件。TLS 1.3 的套件通常不完全由该指令控制，具体取决于 Nginx 和 OpenSSL 版本。

示例：

```nginx
ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305';
ssl_prefer_server_ciphers off;
```

这类配置强调：

- 使用 ECDHE 支持前向安全。
- 使用 GCM 或 CHACHA20-POLY1305 等现代算法。
- 避免 RC4、3DES、NULL、EXPORT、MD5 等过时算法。

### 11.2 ssl_prefer_server_ciphers

```nginx
ssl_prefer_server_ciphers off;
```

在现代配置中，通常可以让客户端和服务端协商更合适的套件。老旧配置常写 `on`，但需要结合实际安全基线评估。

### 11.3 不要盲目复制超长套件字符串

很多线上配置会从网络复制一大串 `ssl_ciphers`，但这有几个问题：

- 可能包含过时算法。
- 可能与当前 OpenSSL 不兼容。
- 可能降低兼容性。
- 可能让配置不可维护。
- 可能在安全扫描中暴露弱套件。

建议基于组织安全基线、OpenSSL 版本和客户端画像统一维护。

## 12. SSL 会话复用

### 12.1 为什么需要会话复用

TLS 握手比普通 HTTP 连接建立成本更高。会话复用可以减少重复握手开销，提高 HTTPS 性能。

常见配置：

```nginx
ssl_session_cache shared:SSL:10m;
ssl_session_timeout 10m;
ssl_session_tickets off;
```

### 12.2 ssl_session_cache

```nginx
ssl_session_cache shared:SSL:10m;
```

含义：

- 使用共享内存保存 SSL 会话。
- 多个 worker 可以共享会话缓存。
- `10m` 是共享内存大小，不是缓存时间。

### 12.3 ssl_session_timeout

```nginx
ssl_session_timeout 10m;
```

表示会话缓存有效时间。

时间不宜过长。过长可能增加密钥材料长期有效带来的风险；过短则复用收益下降。

### 12.4 ssl_session_tickets

```nginx
ssl_session_tickets off;
```

Session Ticket 可以实现另一种会话恢复机制，但如果 ticket key 管理不当，会削弱前向安全。

简单生产建议：

- 不清楚如何安全轮换 ticket key 时，先关闭。
- 如果需要开启，应建立 ticket key 生成、分发、轮换机制。

## 13. HTTP 自动跳转 HTTPS

### 13.1 为什么要跳转

开启 HTTPS 后，仍然可能有用户访问 HTTP：

```text
http://app.example.com
```

如果 HTTP 不跳转，可能出现：

- 用户仍在使用明文访问。
- Cookie 或 Token 暴露风险增加。
- 搜索引擎收录 HTTP 和 HTTPS 两套地址。
- 接口回调地址混乱。

因此公网站点通常应配置 HTTP 到 HTTPS 的跳转。

### 13.2 推荐跳转配置

```nginx
server {
    listen 80;
    server_name app.example.com;

    return 301 https://$host$request_uri;
}
```

说明：

- `301` 表示永久重定向。
- `$host` 使用规范化后的 Host。
- `$request_uri` 保留原始路径和查询参数。

例如：

```text
http://app.example.com/api/users?page=1
```

跳转到：

```text
https://app.example.com/api/users?page=1
```

### 13.3 301 与 302 选择

| 状态码 | 含义 | 适用场景 |
| --- | --- | --- |
| 301 | 永久重定向 | 确认长期使用 HTTPS |
| 302 | 临时重定向 | 灰度验证或临时切换 |
| 308 | 永久重定向且保持方法 | API 场景可考虑 |

对于普通网站，常用 301。

对于接口迁移、灰度验证、存在回滚不确定性的场景，可以先用 302 或 308 做验证。

### 13.4 跳转常见错误

错误一：丢失路径。

```nginx
return 301 https://$host;
```

这样会把所有请求跳到根路径，丢失 URI。

错误二：跳转到固定错误域名。

```nginx
return 301 https://example.com$request_uri;
```

如果当前 server 同时处理多个域名，可能跳错。

错误三：在代理后端也做跳转，形成循环。

```text
client -> Nginx HTTPS -> backend HTTP
backend 判断自己是 HTTP -> 返回 HTTPS 跳转
client 再访问 HTTPS -> 后端仍认为 HTTP -> 无限跳转
```

解决思路：

- Nginx 传递 `X-Forwarded-Proto $scheme`。
- 后端框架信任代理 Header。
- 后端不要基于本地连接协议盲目跳转。

## 14. HTTPS 代理到后端

### 14.1 入口 HTTPS，后端 HTTP

最常见架构：

```text
client --HTTPS--> Nginx --HTTP--> backend
```

Nginx 负责 TLS 终止，后端使用内网 HTTP。

优点：

- 后端配置简单。
- 内网性能开销较低。
- 证书集中管理。
- 入口层统一做安全治理。

典型配置：

```nginx
location /api/ {
    proxy_pass http://app_backend/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

### 14.2 入口 HTTPS，后端 HTTPS

有些场景需要 Nginx 到后端也使用 HTTPS：

```text
client --HTTPS--> Nginx --HTTPS--> backend
```

适用场景：

- 跨机房或跨公网代理。
- 零信任网络要求。
- 后端只暴露 HTTPS。
- 合规要求内部链路也加密。

基础配置：

```nginx
location /api/ {
    proxy_pass https://backend.example.internal;
    proxy_ssl_server_name on;
    proxy_set_header Host backend.example.internal;
}
```

### 14.3 proxy_ssl_server_name

当代理到 HTTPS 后端时，建议开启：

```nginx
proxy_ssl_server_name on;
```

它的作用是让 Nginx 访问后端 HTTPS 时发送 SNI。

如果后端同一 IP 承载多个 HTTPS 域名，而 Nginx 没有发送 SNI，后端可能返回默认证书，导致校验失败或业务错路由。

### 14.4 是否校验后端证书

如果要校验后端证书，可以配置：

```nginx
proxy_ssl_verify on;
proxy_ssl_trusted_certificate /etc/nginx/ssl/internal-ca.pem;
proxy_ssl_verify_depth 2;
```

如果是内部自签 CA，需要把内部 CA 证书配置给 `proxy_ssl_trusted_certificate`。

不校验证书虽然简单，但会降低 HTTPS 后端链路的安全意义。

## 15. HTTP/2

### 15.1 HTTP/2 是什么

HTTP/2 是 HTTP 协议的新版本，常见于 HTTPS 站点。

它的特点包括：

- 多路复用。
- 头部压缩。
- 二进制帧。
- 单连接并发多个请求。
- 减少 HTTP/1.1 队头阻塞影响。

HTTP/2 更适合：

- 静态资源较多的网站。
- 浏览器访问场景。
- 多小文件并发请求。
- 高延迟网络下改善加载体验。

### 15.2 HTTP/2 不是万能性能开关

HTTP/2 不能解决所有问题：

- 不能让慢后端变快。
- 不能替代缓存和压缩。
- 不能消除业务接口耗时。
- 不能提升服务器带宽上限。
- 不能解决错误的前端资源组织。

如果后端接口本身很慢，开启 HTTP/2 不会让接口计算变快。

### 15.3 开启 HTTP/2

老写法常见：

```nginx
listen 443 ssl http2;
```

较新版本 Nginx 支持单独写：

```nginx
listen 443 ssl;
http2 on;
```

为了兼容性，很多环境仍然使用：

```nginx
server {
    listen 443 ssl http2;
    server_name app.example.com;

    ssl_certificate /etc/letsencrypt/live/app.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/app.example.com/privkey.pem;
}
```

### 15.4 验证 HTTP/2

使用 curl：

```bash
curl -I --http2 https://app.example.com/
```

如果 curl 支持 HTTP/2，可能看到：

```text
HTTP/2 200
```

也可以使用浏览器开发者工具：

- 打开 Network。
- 查看 Protocol 列。
- 如果显示 `h2`，表示使用 HTTP/2。

如果看不到 Protocol 列，需要在 Network 表头右键勾选。

### 15.5 HTTP/2 常见问题

常见问题：

- Nginx 没有编译 `http_v2_module`。
- OpenSSL 版本过旧。
- 客户端不支持 HTTP/2。
- 浏览器通常只在 HTTPS 下使用 HTTP/2。
- 中间 CDN 或负载均衡没有开启 HTTP/2。
- 配置修改后没有 reload。

排查命令：

```bash
nginx -V 2>&1 | grep http_v2_module
curl -I --http2 -v https://app.example.com/
```

## 16. HSTS

### 16.1 HSTS 是什么

HSTS 是 HTTP Strict Transport Security。服务端通过响应头告诉浏览器：以后访问该域名必须使用 HTTPS。

配置示例：

```nginx
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
```

浏览器收到后，在 `max-age` 时间内会自动把 HTTP 访问升级为 HTTPS。

### 16.2 HSTS 解决什么问题

HSTS 主要解决：

- 用户手动输入 HTTP 地址。
- 书签中保存 HTTP 地址。
- 首次跳转后的后续访问安全。
- 部分 SSL stripping 攻击风险。

普通 HTTP 到 HTTPS 跳转仍有第一次明文请求。HSTS 可以让浏览器后续直接访问 HTTPS。

### 16.3 HSTS 配置项

```nginx
add_header Strict-Transport-Security "max-age=31536000" always;
```

只作用当前域名。

```nginx
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
```

作用当前域名和所有子域名。

```nginx
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
```

表示希望加入浏览器 HSTS preload 列表。

### 16.4 HSTS 风险

HSTS 最大风险是回滚困难。如果配置了：

```text
max-age=31536000; includeSubDomains
```

浏览器会在一年内强制该域名和所有子域名使用 HTTPS。如果某个子域名还没有 HTTPS，就可能无法访问。

因此首次上线建议：

```nginx
add_header Strict-Transport-Security "max-age=300" always;
```

观察无问题后逐步增加：

```text
300 -> 3600 -> 86400 -> 31536000
```

### 16.5 HSTS 不要随便 preload

`preload` 会把域名提交到浏览器内置列表。一旦进入列表，即使用户从未访问过该站点，也会默认强制 HTTPS。

加入 preload 前必须确认：

- 主域名支持 HTTPS。
- 所有子域名支持 HTTPS。
- 长期不会回退 HTTP。
- 证书自动续期可靠。
- 运维团队理解移除 preload 的复杂性。

不满足条件时，不要配置 `preload`。

## 17. OCSP Stapling

### 17.1 OCSP 是什么

OCSP 用于检查证书是否被吊销。

传统流程中，浏览器可能向 CA 的 OCSP 服务查询证书状态。但这会带来：

- 额外延迟。
- 隐私问题。
- CA OCSP 服务不可用时体验不稳定。

OCSP Stapling 让 Nginx 预先向 CA 获取 OCSP 响应，并在 TLS 握手时一并返回给客户端。

### 17.2 基础配置

```nginx
ssl_stapling on;
ssl_stapling_verify on;
resolver 223.5.5.5 119.29.29.29 valid=300s;
resolver_timeout 5s;
ssl_trusted_certificate /etc/letsencrypt/live/app.example.com/chain.pem;
```

说明：

- `ssl_stapling on` 开启 OCSP Stapling。
- `ssl_stapling_verify on` 校验 OCSP 响应。
- `resolver` 用于解析 OCSP 服务域名。
- `ssl_trusted_certificate` 用于指定可信证书链。

### 17.3 验证 OCSP Stapling

```bash
openssl s_client -connect app.example.com:443 \
  -servername app.example.com \
  -status </dev/null
```

如果成功，可能看到：

```text
OCSP Response Status: successful
```

如果没有看到，不一定代表 HTTPS 不可用，但说明 OCSP Stapling 没有正常返回。

### 17.4 常见问题

常见原因：

- 没有配置 `resolver`。
- 服务器无法访问 CA 的 OCSP 服务。
- `ssl_trusted_certificate` 配置错误。
- 使用自签证书。
- CA 不支持或响应异常。

生产建议：OCSP Stapling 是优化项，不是第一优先级。先保证证书、TLS 和续期可靠，再处理它。

## 18. 安全 Header 总览

### 18.1 为什么需要安全 Header

安全 Header 可以让浏览器执行一些安全策略，降低常见前端攻击风险。

Nginx 可以统一注入这些 Header：

- `Strict-Transport-Security`。
- `X-Frame-Options`。
- `X-Content-Type-Options`。
- `Referrer-Policy`。
- `Content-Security-Policy`。
- `Permissions-Policy`。

安全 Header 不是万能防线，但入口层统一配置可以提升基础安全水平。

### 18.2 add_header 的 always

Nginx 的 `add_header` 默认只对部分成功响应生效。为了让错误响应也带上 Header，常用：

```nginx
add_header X-Content-Type-Options "nosniff" always;
```

`always` 可以让 Header 在更多响应状态下生效。

### 18.3 add_header 继承陷阱

`add_header` 有继承行为陷阱：

```nginx
server {
    add_header X-Frame-Options "SAMEORIGIN" always;

    location /api/ {
        add_header X-Content-Type-Options "nosniff" always;
    }
}
```

在这个例子中，`location /api/` 里定义了新的 `add_header`，可能导致上层 `X-Frame-Options` 不再继承。

因此建议：

- 把通用安全 Header 放到统一 include 文件。
- 在需要自定义 Header 的 location 中完整重复必要 Header。
- 配置后用 curl 验证实际响应。

## 19. 常见安全 Header

### 19.1 X-Frame-Options

作用：控制页面是否允许被 iframe 嵌入，降低点击劫持风险。

常见值：

```nginx
add_header X-Frame-Options "SAMEORIGIN" always;
```

含义：

- `DENY`：不允许任何站点嵌入。
- `SAMEORIGIN`：只允许同源页面嵌入。

如果你的系统需要被第三方 iframe 嵌入，不能简单配置 `DENY` 或 `SAMEORIGIN`，需要结合 CSP 的 `frame-ancestors` 精细控制。

### 19.2 X-Content-Type-Options

作用：阻止浏览器进行 MIME sniffing。

配置：

```nginx
add_header X-Content-Type-Options "nosniff" always;
```

它可以减少浏览器把非脚本资源错误当脚本执行的风险。

### 19.3 Referrer-Policy

作用：控制浏览器跳转或加载资源时发送多少 Referer 信息。

推荐示例：

```nginx
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
```

含义：

- 同源请求发送完整 Referer。
- 跨源 HTTPS 请求只发送 origin。
- HTTPS 到 HTTP 降级请求不发送 Referer。

### 19.4 Content-Security-Policy

CSP 用于限制页面能加载哪些脚本、样式、图片、字体、接口等资源。

简单示例：

```nginx
add_header Content-Security-Policy "default-src 'self'; frame-ancestors 'self'; object-src 'none'" always;
```

CSP 很强，但也很容易误伤业务。

上线建议：

1. 先梳理前端资源来源。
2. 使用 `Content-Security-Policy-Report-Only` 观察。
3. 收集违规报告。
4. 调整策略。
5. 再切换到正式 `Content-Security-Policy`。

不要在复杂前端应用上直接复制严格 CSP，否则可能导致页面白屏、脚本失败、第三方组件异常。

### 19.5 Permissions-Policy

用于控制浏览器能力，例如摄像头、麦克风、定位等。

示例：

```nginx
add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;
```

如果业务需要这些能力，需要按需放开。

## 20. 隐藏版本号 server_tokens

### 20.1 server_tokens 的作用

默认情况下，Nginx 错误页面或响应头可能暴露版本号。

关闭方式：

```nginx
server_tokens off;
```

通常放在 `http` 块：

```nginx
http {
    server_tokens off;
}
```

关闭后，响应头可能从：

```text
Server: nginx/1.24.0
```

变为：

```text
Server: nginx
```

### 20.2 隐藏版本号不是漏洞修复

`server_tokens off` 只能减少信息暴露，不能修复漏洞。

真正的安全治理仍然包括：

- 跟踪安全公告。
- 定期升级 Nginx 和 OpenSSL。
- 禁用不安全协议。
- 限制管理端访问。
- 做好日志审计和告警。

## 21. 访问控制 allow 与 deny

### 21.1 基础语法

`allow` 和 `deny` 可以按 IP 控制访问。

示例：

```nginx
location /admin/ {
    allow 10.0.0.0/8;
    allow 192.168.1.0/24;
    deny all;

    proxy_pass http://admin_backend/;
}
```

含义：

- 允许 `10.0.0.0/8`。
- 允许 `192.168.1.0/24`。
- 拒绝其他所有 IP。

### 21.2 匹配顺序

Nginx 按顺序检查 `allow` / `deny`，匹配到就停止。

示例：

```nginx
allow 192.168.1.10;
deny 192.168.1.0/24;
allow all;
```

含义：

- `192.168.1.10` 允许。
- 其他 `192.168.1.0/24` 拒绝。
- 其他 IP 允许。

### 21.3 白名单模板

管理后台常见配置：

```nginx
location /admin/ {
    allow 203.0.113.10;
    allow 203.0.113.11;
    deny all;

    proxy_pass http://admin_backend/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

### 21.4 多层代理下的真实 IP 问题

如果 Nginx 前面还有 CDN、SLB、Ingress 或另一个 Nginx，`allow` / `deny` 默认看到的可能不是用户真实 IP，而是上一层代理 IP。

示例链路：

```text
client -> CDN -> SLB -> Nginx -> backend
```

Nginx 的 `$remote_addr` 可能是 SLB IP，而不是 client IP。

这时不能直接对 `$http_x_forwarded_for` 做简单信任，必须结合 `real_ip` 模块：

```nginx
set_real_ip_from 10.0.0.0/8;
real_ip_header X-Forwarded-For;
real_ip_recursive on;
```

只有可信代理传来的真实 IP 才能被信任。

### 21.5 allow deny 常见误区

常见误区：

- 忘记最后写 `deny all`，导致白名单失效。
- 在多层代理场景中拿代理 IP 做限制。
- 直接信任用户可伪造的 `X-Forwarded-For`。
- 把动态办公网出口 IP 写死，导致员工经常无法访问。
- 在多个 location 中重复维护白名单，后续难以更新。

建议把白名单拆到 include 文件：

```nginx
include conf.d/allow_admin_ips.conf;
```

## 22. 基础认证 auth_basic

### 22.1 auth_basic 是什么

`auth_basic` 是 HTTP Basic Authentication。浏览器访问时会弹出用户名密码输入框。

适合：

- 临时测试环境。
- 简单内部页面。
- 低频管理入口的额外保护。
- 暂未接入统一认证的临时站点。

不适合：

- 正式复杂权限系统。
- 多角色权限管理。
- 高安全要求后台认证。
- 替代业务登录系统。

### 22.2 安装 htpasswd 工具

Ubuntu / Debian：

```bash
sudo apt install -y apache2-utils
```

Rocky / CentOS：

```bash
sudo dnf install -y httpd-tools
```

### 22.3 创建密码文件

```bash
sudo htpasswd -c /etc/nginx/.htpasswd admin
```

追加用户：

```bash
sudo htpasswd /etc/nginx/.htpasswd ops
```

注意：

- 第一次创建使用 `-c`。
- 第二次追加不要使用 `-c`，否则会覆盖原文件。
- 密码文件不要放在站点 root 目录下。

### 22.4 配置基础认证

```nginx
location /admin/ {
    auth_basic "Restricted Area";
    auth_basic_user_file /etc/nginx/.htpasswd;

    proxy_pass http://admin_backend/;
}
```

访问 `/admin/` 时，浏览器会要求输入用户名和密码。

### 22.5 与 IP 白名单组合

可以同时使用 IP 白名单和基础认证：

```nginx
location /admin/ {
    allow 203.0.113.10;
    deny all;

    auth_basic "Admin Area";
    auth_basic_user_file /etc/nginx/.htpasswd;

    proxy_pass http://admin_backend/;
}
```

默认情况下，访问控制和基础认证都要通过。

如果想实现“IP 白名单或密码二选一”，需要使用 `satisfy any`：

```nginx
location /admin/ {
    satisfy any;

    allow 203.0.113.10;
    deny all;

    auth_basic "Admin Area";
    auth_basic_user_file /etc/nginx/.htpasswd;

    proxy_pass http://admin_backend/;
}
```

这表示满足任意一个访问条件即可通过。

## 23. 上传大小限制 client_max_body_size

### 23.1 指令作用

`client_max_body_size` 用于限制客户端请求体大小。

示例：

```nginx
client_max_body_size 10m;
```

如果上传超过限制，Nginx 会返回：

```text
413 Request Entity Too Large
```

或：

```text
413 Payload Too Large
```

### 23.2 配置位置

可以放在 `http`、`server`、`location` 中。

全局默认：

```nginx
http {
    client_max_body_size 10m;
}
```

某个站点单独放大：

```nginx
server {
    server_name upload.example.com;
    client_max_body_size 100m;
}
```

某个上传接口单独放大：

```nginx
location /api/upload/ {
    client_max_body_size 200m;
    proxy_pass http://upload_backend/;
}
```

### 23.3 不要全局无限放大

不建议：

```nginx
client_max_body_size 0;
```

`0` 表示不限制请求体大小。除非非常明确，否则不要使用。

风险：

- 大文件请求占用带宽。
- 临时文件占用磁盘。
- worker 处理压力增大。
- 恶意上传可能拖垮入口层。
- 后端未必能承受大请求体。

### 23.4 413 排查

排查步骤：

1. 确认是 Nginx 返回 413，还是后端返回 413。
2. 检查 `client_max_body_size` 所在上下文。
3. 确认请求是否命中了预期 `server` 和 `location`。
4. 检查是否还有上层 CDN、SLB、Ingress 限制。
5. 检查后端框架上传限制。

命令：

```bash
curl -v -F "file=@big.zip" https://app.example.com/api/upload/
```

日志中重点看：

```text
client intended to send too large body
```

## 24. 请求限流 limit_req

### 24.1 limit_req 解决什么问题

`limit_req` 用于限制单位时间内的请求速率。

适合处理：

- 登录接口防爆破。
- 短信验证码接口防刷。
- 搜索接口防滥用。
- 下载接口削峰。
- 爬虫访问频率控制。
- 后端接口保护。

它不适合单独应对大规模 DDoS。大流量攻击通常需要 CDN、高防、云清洗、网络层防护配合。

### 24.2 基础语法

限流分两步：

第一步，在 `http` 块定义共享内存区域：

```nginx
limit_req_zone $binary_remote_addr zone=per_ip:10m rate=10r/s;
```

第二步，在 `server` 或 `location` 中使用：

```nginx
location /api/ {
    limit_req zone=per_ip burst=20 nodelay;
    proxy_pass http://app_backend/;
}
```

### 24.3 limit_req_zone 参数

```nginx
limit_req_zone $binary_remote_addr zone=per_ip:10m rate=10r/s;
```

含义：

| 部分 | 说明 |
| --- | --- |
| `$binary_remote_addr` | 限流 key，按客户端 IP 限流 |
| `zone=per_ip:10m` | 共享内存区域名称和大小 |
| `rate=10r/s` | 平均速率 10 请求/秒 |

为什么常用 `$binary_remote_addr`：

- 比 `$remote_addr` 更省内存。
- 适合按 IP 维度限流。

### 24.4 burst

```nginx
limit_req zone=per_ip burst=20;
```

`burst` 表示允许排队的突发请求数量。

如果 rate 是 `10r/s`，burst 是 `20`，可以理解为：

```text
平均每秒处理 10 个请求
短时间突发超过平均速率时，最多允许 20 个请求排队等待
超过队列容量的请求被拒绝
```

### 24.5 nodelay

```nginx
limit_req zone=per_ip burst=20 nodelay;
```

加上 `nodelay` 后，burst 内的请求不会延迟排队，而是立即处理。

适合：

- 允许短时间突发。
- 不希望用户请求被 Nginx 排队变慢。
- 后端能承受一定突发。

不加 `nodelay` 时，Nginx 会按 rate 延迟处理 burst 内请求，这可能让用户感觉接口变慢。

### 24.6 限流返回码

默认限流拒绝可能返回 503。生产中常希望返回 429：

```nginx
limit_req_status 429;
```

可以放在 `http`、`server` 或 `location`。

示例：

```nginx
http {
    limit_req_zone $binary_remote_addr zone=per_ip:10m rate=10r/s;
    limit_req_status 429;
}
```

### 24.7 登录接口限流示例

```nginx
http {
    limit_req_zone $binary_remote_addr zone=login_ip:10m rate=5r/m;
    limit_req_status 429;

    server {
        listen 443 ssl http2;
        server_name app.example.com;

        location = /api/login {
            limit_req zone=login_ip burst=5 nodelay;
            proxy_pass http://app_backend;
        }
    }
}
```

含义：

- 每个 IP 平均每分钟 5 次登录请求。
- 允许短时间突发 5 次。
- 超过返回 429。

### 24.8 验证限流

使用 `curl` 快速请求：

```bash
for i in $(seq 1 30); do
  curl -s -o /dev/null -w "%{http_code}\n" https://app.example.com/api/login &
done
wait
```

使用 `hey`：

```bash
hey -n 100 -c 20 https://app.example.com/api/login
```

观察：

- 是否出现 429。
- access log 中状态码分布。
- error log 是否出现 limiting requests。

### 24.9 限流 key 选择

常见 key：

| key | 含义 | 适用场景 |
| --- | --- | --- |
| `$binary_remote_addr` | 按 IP 限流 | 通用入口保护 |
| `$server_name` | 按站点限流 | 全站总量控制较少单独使用 |
| `$uri` | 按 URI 限流 | 接口维度控制 |
| `$remote_addr$uri` | 按 IP + URI 限流 | 防单 IP 刷接口 |
| `$http_authorization` | 按 Token 限流 | API 调用方控制，需谨慎 |

更复杂的用户级限流一般交给应用网关、业务系统、OpenResty 或 API Gateway 实现。

### 24.10 多层代理下的限流问题

如果 Nginx 前面有 CDN 或 SLB，按 `$binary_remote_addr` 限流可能限制的是代理节点 IP。

结果：

- 大量正常用户被当成同一个 IP。
- 限流误伤严重。

解决：

- 正确配置 `real_ip`。
- 只信任可信代理传来的真实 IP。
- 再基于真实 IP 限流。

示例：

```nginx
set_real_ip_from 10.0.0.0/8;
real_ip_header X-Forwarded-For;
real_ip_recursive on;

limit_req_zone $binary_remote_addr zone=per_real_ip:10m rate=10r/s;
```

## 25. 连接数限制 limit_conn

### 25.1 limit_conn 解决什么问题

`limit_conn` 用于限制并发连接数。

适合：

- 限制单 IP 长连接数量。
- 防止下载连接占满资源。
- 防止 WebSocket/SSE 连接滥用。
- 对特定域名或接口做连接保护。

它限制的是连接数，不是请求速率。

### 25.2 基础语法

定义共享内存区域：

```nginx
limit_conn_zone $binary_remote_addr zone=conn_per_ip:10m;
```

使用限制：

```nginx
location /download/ {
    limit_conn conn_per_ip 5;
    proxy_pass http://download_backend/;
}
```

含义：每个 IP 同时最多 5 个连接访问 `/download/`。

### 25.3 limit_conn_status

默认拒绝状态码也可能不是你期望的，可以设置：

```nginx
limit_conn_status 429;
```

或者使用 503，取决于团队规范。

### 25.4 limit_req 与 limit_conn 对比

| 指令 | 控制对象 | 典型场景 |
| --- | --- | --- |
| `limit_req` | 请求速率 | 登录、短信、搜索、API 防刷 |
| `limit_conn` | 并发连接数 | 下载、长连接、慢连接保护 |

两者可以组合使用：

```nginx
location /api/ {
    limit_req zone=per_ip burst=20 nodelay;
    limit_conn conn_per_ip 20;
    proxy_pass http://app_backend/;
}
```

### 25.5 长连接场景注意

WebSocket、SSE、长轮询这类请求会长期占用连接。

如果对它们配置过低的 `limit_conn`，可能导致正常用户被拒绝。

建议：

- 长连接接口单独 location。
- 单独设置连接数限制。
- 结合业务用户身份做更精细治理。
- 不要套用普通 API 的限流参数。

## 26. 防盗链 valid_referers

### 26.1 防盗链是什么

防盗链是指限制其他网站直接引用你的图片、视频、文件等资源。

典型盗链：

```html
<img src="https://static.example.com/images/product.jpg">
```

如果大量第三方站点引用你的资源，会消耗你的带宽和流量费用。

### 26.2 Referer 原理

浏览器从 A 页面加载 B 资源时，通常会带上 Referer：

```text
Referer: https://www.other-site.com/page.html
```

Nginx 可以根据 Referer 判断请求来源。

但 Referer 不是强安全凭证：

- 可以被客户端伪造。
- 有些浏览器或隐私插件会去掉 Referer。
- HTTPS 到 HTTP 可能不发送 Referer。
- App、爬虫、脚本不一定发送 Referer。

因此防盗链适合减少普通盗链，不适合作为强鉴权。

### 26.3 valid_referers 基础配置

```nginx
location ~* \.(jpg|jpeg|png|gif|webp|mp4|zip)$ {
    valid_referers none blocked server_names *.example.com example.com;

    if ($invalid_referer) {
        return 403;
    }

    root /data/static;
}
```

含义：

- `none`：允许没有 Referer 的请求。
- `blocked`：允许 Referer 被防火墙或代理隐藏的请求。
- `server_names`：允许当前 `server_name`。
- `*.example.com example.com`：允许指定域名。

### 26.4 是否允许 none

是否允许 `none` 要看业务：

允许 `none` 的理由：

- 用户直接打开图片链接。
- 浏览器隐私策略可能不发送 Referer。
- App 或小程序可能没有 Referer。
- 搜索引擎或合法抓取可能不带 Referer。

不允许 `none` 的理由：

- 希望更严格地阻止直接访问。
- 资源只允许站内页面加载。
- 下载资源需要更强控制。

如果资源很敏感，不应依赖 Referer，应使用签名 URL、临时 Token、鉴权下载等方式。

### 26.5 返回替代图片

图片盗链时可以返回默认图：

```nginx
location ~* \.(jpg|jpeg|png|gif|webp)$ {
    valid_referers none blocked server_names *.example.com example.com;

    if ($invalid_referer) {
        rewrite ^ /images/no-hotlink.png break;
    }

    root /data/static;
}
```

需要注意：这里使用 `if` 是常见防盗链写法，但不要把复杂逻辑都堆到 `if` 中。

## 27. 基础 Bot 与扫描防护

### 27.1 能做什么

Nginx 可以基于 URI、User-Agent、方法等做一些轻量拦截。

适合：

- 阻断明显扫描路径。
- 拒绝不允许的方法。
- 隐藏敏感文件。
- 限制异常 User-Agent。

不适合：

- 完整 WAF 规则。
- 复杂攻击识别。
- 深度业务风控。
- 大规模 DDoS 防护。

### 27.2 禁止访问隐藏文件

```nginx
location ~ /\. {
    deny all;
}
```

这可以阻止访问：

- `.git`。
- `.env`。
- `.svn`。
- `.htaccess`。

也可以更明确：

```nginx
location ~* /(\.git|\.svn|\.env|\.DS_Store) {
    deny all;
}
```

### 27.3 限制 HTTP 方法

如果站点只允许常见方法：

```nginx
if ($request_method !~ ^(GET|POST|HEAD|OPTIONS|PUT|DELETE)$) {
    return 405;
}
```

更保守的静态资源站点：

```nginx
location / {
    limit_except GET HEAD {
        deny all;
    }

    root /data/www;
}
```

`limit_except` 比在复杂 location 中乱用 `if` 更清晰。

### 27.4 拦截敏感路径

```nginx
location ~* /(phpmyadmin|wp-admin|wp-login\.php|manager/html) {
    return 404;
}
```

这类配置可以减少无效扫描日志，但不要误伤真实业务路径。

## 28. CORS 与安全边界

### 28.1 CORS 是什么

CORS 是浏览器跨域资源共享机制。它控制的是浏览器是否允许前端 JavaScript 读取跨域响应。

它不是服务端接口鉴权，也不是防止别人访问接口的安全机制。

即使没有 CORS，攻击者仍然可以用 curl、Postman、脚本直接请求接口。

### 28.2 不要随便配置星号

危险配置：

```nginx
add_header Access-Control-Allow-Origin "*" always;
add_header Access-Control-Allow-Credentials "true" always;
```

`*` 和 credentials 不能安全组合。涉及 Cookie、Authorization、用户态数据时，不能简单放开所有来源。

### 28.3 简单 CORS 示例

只允许指定前端域名：

```nginx
set $cors_origin "";

if ($http_origin ~* ^https://(www\.)?example\.com$) {
    set $cors_origin $http_origin;
}

add_header Access-Control-Allow-Origin $cors_origin always;
add_header Access-Control-Allow-Credentials "true" always;
add_header Access-Control-Allow-Methods "GET, POST, PUT, DELETE, OPTIONS" always;
add_header Access-Control-Allow-Headers "Authorization, Content-Type, X-Requested-With" always;

if ($request_method = OPTIONS) {
    return 204;
}
```

注意：这是教学示例。复杂跨域策略更建议在应用层或 API Gateway 统一处理，避免 Nginx 配置过度复杂。

## 29. 入口层日志审计

### 29.1 为什么安全治理必须配日志

限流、访问控制、防盗链、安全 Header 配完后，如果没有日志，就很难判断是否生效，也无法发现误伤。

入口层日志至少应能回答：

- 谁访问了。
- 访问哪个域名。
- 访问哪个 URI。
- 返回了什么状态码。
- 花了多久。
- 是否被限流。
- 来自哪个 Referer。
- 使用什么 User-Agent。
- 上游返回什么状态。
- 请求 ID 是什么。

### 29.2 推荐安全日志格式

```nginx
log_format security_main '$remote_addr - $remote_user [$time_local] '
                         '"$request" $status $body_bytes_sent '
                         'host="$host" scheme="$scheme" '
                         'request_time=$request_time '
                         'referer="$http_referer" '
                         'ua="$http_user_agent" '
                         'xff="$http_x_forwarded_for" '
                         'request_id="$request_id" '
                         'upstream_addr="$upstream_addr" '
                         'upstream_status="$upstream_status" '
                         'upstream_response_time="$upstream_response_time"';
```

使用：

```nginx
access_log /var/log/nginx/app_access.log security_main;
error_log /var/log/nginx/app_error.log warn;
```

### 29.3 限流日志

限流触发时，error log 可能出现类似：

```text
limiting requests, excess: 1.234 by zone "per_ip"
```

可以通过 error log 判断：

- 哪个 zone 触发。
- 哪个客户端触发。
- 是否频繁误伤。
- rate 和 burst 是否过低。

### 29.4 访问控制日志

403 可能来自：

- `deny all`。
- `auth_basic` 失败。
- 防盗链。
- 静态目录无权限。
- 后端应用返回。

排查时必须结合：

- access log 状态码。
- error log 提示。
- 命中的 server/location。
- 响应 Header。
- 后端是否收到请求。

## 30. 常见状态码排查

### 30.1 400 Bad Request

入口层常见 400 原因：

- 客户端发送了非法 HTTP 请求。
- HTTPS 请求发到了 HTTP 端口。
- HTTP 请求发到了 HTTPS 端口。
- Header 太大。
- Host 异常。
- TLS 握手失败。

排查命令：

```bash
curl -v http://app.example.com:443/
curl -vk https://app.example.com/
openssl s_client -connect app.example.com:443 -servername app.example.com
```

error log 关键字：

```text
client sent plain HTTP request to HTTPS port
client sent invalid request
SSL_do_handshake() failed
```

### 30.2 403 Forbidden

常见原因：

- `allow` / `deny` 拒绝。
- `auth_basic` 未通过。
- 防盗链拒绝。
- 目录无 index 且禁止 autoindex。
- 文件权限不足。
- SELinux 限制。
- 后端返回 403。

排查步骤：

1. 看 access log 中 status 是否 403。
2. 看 error log 是否有 access forbidden、permission denied。
3. 确认后端是否收到请求。
4. 检查 location 中是否有 deny、防盗链、auth_basic。
5. 检查 root/alias 对应文件权限。

### 30.3 413 Payload Too Large

常见原因：

- `client_max_body_size` 太小。
- 上层 CDN 或 SLB 上传限制。
- 应用框架上传限制。
- 请求命中了错误 location。

排查重点：

```bash
nginx -T | grep -n "client_max_body_size"
```

以及 error log：

```text
client intended to send too large body
```

### 30.4 429 Too Many Requests

常见原因：

- `limit_req` 触发。
- `limit_conn` 触发。
- 上层网关或 CDN 限流。
- 后端应用限流。

排查重点：

- 是否配置 `limit_req_status 429`。
- error log 中是否有 limiting requests。
- 限流 key 是否错误。
- 是否把 CDN IP 当成用户 IP。
- rate 和 burst 是否与业务流量匹配。

## 31. HTTPS 常见问题排查

### 31.1 证书过期

查看证书有效期：

```bash
echo | openssl s_client -servername app.example.com -connect app.example.com:443 2>/dev/null \
  | openssl x509 -noout -dates
```

输出示例：

```text
notBefore=Jul 11 00:00:00 2026 GMT
notAfter=Oct  9 23:59:59 2026 GMT
```

生产建议：

- 证书到期前 30 天告警。
- 证书到期前 15 天升级告警。
- 证书到期前 7 天严重告警。
- 自动续期失败必须能及时发现。

### 31.2 域名不匹配

浏览器提示证书域名不匹配时，检查：

```bash
openssl s_client -connect app.example.com:443 -servername app.example.com </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -ext subjectAltName
```

重点看 SAN：

```text
DNS:app.example.com
DNS:www.example.com
```

现代浏览器主要看 SAN，不再只看 CN。

### 31.3 忘记 SNI

如果用 IP 访问：

```bash
openssl s_client -connect 203.0.113.10:443
```

可能拿到默认证书。

要指定 SNI：

```bash
openssl s_client -connect 203.0.113.10:443 -servername app.example.com
```

curl 指定解析：

```bash
curl -vk --resolve app.example.com:443:203.0.113.10 https://app.example.com/
```

### 31.4 私钥权限错误

Nginx 启动或 reload 报权限错误时：

```text
BIO_new_file("/etc/nginx/ssl/privkey.pem") failed
Permission denied
```

检查：

```bash
ls -l /etc/nginx/ssl/privkey.pem
sudo nginx -t
```

如果 Nginx master 以 root 启动，通常 root 可读即可。如果使用非 root 启动，需要确保运行用户可读。

### 31.5 证书链不完整

表现：

- Chrome 正常，Java 客户端失败。
- 部分 Android 设备失败。
- curl 报 unable to get local issuer certificate。
- 第三方回调提示证书不可信。

检查：

```bash
openssl s_client -connect app.example.com:443 -servername app.example.com -showcerts </dev/null
```

解决：

- `ssl_certificate` 使用 fullchain。
- 不要只配置站点证书。
- 确认证书文件顺序正确。

## 32. 生产 HTTPS 配置模板

### 32.1 通用 HTTPS 站点模板

```nginx
server {
    listen 80;
    server_name app.example.com;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl http2;
    server_name app.example.com;

    ssl_certificate /etc/letsencrypt/live/app.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/app.example.com/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305';
    ssl_prefer_server_ciphers off;

    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    ssl_session_tickets off;

    add_header Strict-Transport-Security "max-age=86400" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    access_log /var/log/nginx/app_access.log security_main;
    error_log /var/log/nginx/app_error.log warn;

    location / {
        proxy_pass http://app_backend;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Port $server_port;
    }
}
```

### 32.2 管理后台模板

```nginx
server {
    listen 443 ssl http2;
    server_name admin.example.com;

    ssl_certificate /etc/letsencrypt/live/admin.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/admin.example.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;

    add_header Strict-Transport-Security "max-age=86400" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    location / {
        satisfy all;

        allow 203.0.113.10;
        allow 203.0.113.11;
        deny all;

        auth_basic "Admin Area";
        auth_basic_user_file /etc/nginx/.htpasswd;

        proxy_pass http://admin_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### 32.3 API 限流模板

```nginx
http {
    limit_req_zone $binary_remote_addr zone=api_per_ip:20m rate=20r/s;
    limit_req_zone $binary_remote_addr zone=login_per_ip:10m rate=5r/m;
    limit_conn_zone $binary_remote_addr zone=conn_per_ip:10m;

    limit_req_status 429;
    limit_conn_status 429;

    server {
        listen 443 ssl http2;
        server_name api.example.com;

        location = /api/login {
            limit_req zone=login_per_ip burst=5 nodelay;
            proxy_pass http://api_backend;
        }

        location /api/ {
            limit_req zone=api_per_ip burst=40 nodelay;
            limit_conn conn_per_ip 30;
            proxy_pass http://api_backend;
        }
    }
}
```

### 32.4 静态资源防盗链模板

```nginx
server {
    listen 443 ssl http2;
    server_name static.example.com;

    ssl_certificate /etc/letsencrypt/live/static.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/static.example.com/privkey.pem;

    root /data/static;

    location ~* \.(jpg|jpeg|png|gif|webp|mp4|zip)$ {
        valid_referers none blocked server_names *.example.com example.com;

        if ($invalid_referer) {
            return 403;
        }

        expires 7d;
        access_log /var/log/nginx/static_access.log security_main;
    }
}
```

## 33. 配置拆分建议

### 33.1 推荐拆分结构

可以将安全相关配置拆分：

```text
/etc/nginx/
  nginx.conf
  conf.d/
    app.conf
    admin.conf
    static.conf
    ssl_params.conf
    security_headers.conf
    limit_zones.conf
    real_ip.conf
    allow_admin_ips.conf
```

### 33.2 ssl_params.conf

```nginx
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305';
ssl_prefer_server_ciphers off;
ssl_session_cache shared:SSL:10m;
ssl_session_timeout 10m;
ssl_session_tickets off;
```

使用：

```nginx
server {
    listen 443 ssl http2;
    server_name app.example.com;

    include conf.d/ssl_params.conf;

    ssl_certificate /etc/letsencrypt/live/app.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/app.example.com/privkey.pem;
}
```

### 33.3 security_headers.conf

```nginx
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-Content-Type-Options "nosniff" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;
```

HSTS 是否放入通用文件要谨慎。如果所有 HTTPS 站点都准备好，可以统一放；如果有些域名还需要灰度，应按站点单独配置。

### 33.4 limit_zones.conf

`limit_req_zone` 和 `limit_conn_zone` 必须放在 `http` 上下文。

```nginx
limit_req_zone $binary_remote_addr zone=api_per_ip:20m rate=20r/s;
limit_req_zone $binary_remote_addr zone=login_per_ip:10m rate=5r/m;
limit_conn_zone $binary_remote_addr zone=conn_per_ip:10m;

limit_req_status 429;
limit_conn_status 429;
```

在 `nginx.conf` 的 `http` 中 include：

```nginx
http {
    include conf.d/limit_zones.conf;
    include conf.d/*.conf;
}
```

### 33.5 allow_admin_ips.conf

```nginx
allow 203.0.113.10;
allow 203.0.113.11;
deny all;
```

使用：

```nginx
location /admin/ {
    include conf.d/allow_admin_ips.conf;
    proxy_pass http://admin_backend/;
}
```

这样后续维护办公出口 IP 更方便。

## 34. 安全上线检查清单

### 34.1 HTTPS 检查

上线前检查：

- 443 端口已开放。
- 80 端口跳转到 HTTPS。
- 证书域名匹配。
- 证书链完整。
- 私钥权限正确。
- TLS 1.0/1.1 已禁用。
- TLS 1.2/1.3 可用。
- 自动续期已验证。
- reload 后证书实际生效。

命令：

```bash
sudo nginx -t
curl -I http://app.example.com/
curl -I https://app.example.com/
openssl s_client -connect app.example.com:443 -servername app.example.com </dev/null
```

### 34.2 安全 Header 检查

检查：

```bash
curl -I https://app.example.com/
```

确认是否包含：

- `Strict-Transport-Security`。
- `X-Frame-Options`。
- `X-Content-Type-Options`。
- `Referrer-Policy`。
- 必要时包含 `Content-Security-Policy`。

### 34.3 访问控制检查

检查：

- 白名单 IP 可以访问。
- 非白名单 IP 不能访问。
- Basic Auth 用户密码正确。
- 错误密码无法访问。
- 后端没有绕过 Nginx 暴露公网。
- 日志能记录拒绝行为。

### 34.4 限流检查

检查：

- 低频正常请求不受影响。
- 高频请求返回预期状态码。
- rate、burst 与业务容量匹配。
- 多层代理下真实 IP 正确。
- 429 比例能被监控。
- 误伤时有快速调整方案。

### 34.5 防盗链检查

检查：

- 自家页面可正常加载资源。
- 外站引用被拒绝或替换。
- 直接访问资源行为符合预期。
- App、小程序、下载器等客户端未被误伤。
- Referer 为空时策略符合业务要求。

## 35. 必做实验

### 35.1 基础实验

- 使用 OpenSSL 生成自签证书。
- 配置 Nginx 监听 443 并启用自签 HTTPS。
- 使用 `curl -k` 验证自签 HTTPS。
- 使用 `openssl s_client` 查看证书内容。
- 使用 Let's Encrypt 为真实域名签发证书。
- 配置 `fullchain.pem` 和 `privkey.pem`。
- 配置 HTTP 自动跳转 HTTPS。
- 配置 `ssl_protocols TLSv1.2 TLSv1.3`。
- 使用浏览器验证证书可信状态。
- 使用 curl 验证 HTTP/2。

### 35.2 安全治理实验

- 配置 `server_tokens off` 并观察响应头变化。
- 配置 HSTS，先使用较短 `max-age` 验证。
- 配置基础安全 Header 并使用 `curl -I` 验证。
- 配置 IP 白名单，验证允许和拒绝行为。
- 配置 `auth_basic`，验证用户名密码访问。
- 配置 `client_max_body_size`，复现 413。
- 配置 `limit_req`，使用压测工具复现 429。
- 配置 `limit_conn`，模拟连接数超限。
- 配置 `valid_referers`，验证图片防盗链。
- 配置禁止访问 `.git`、`.env` 等敏感路径。

### 35.3 进阶练习

- 使用 webroot 模式签发 Let's Encrypt 证书，避免自动修改 Nginx 配置。
- 为两个域名配置不同证书，验证 SNI。
- 配置 HTTPS 后端代理并开启 `proxy_ssl_server_name`。
- 开启 OCSP Stapling 并使用 `openssl s_client -status` 验证。
- 配置登录接口和普通 API 不同限流策略。
- 在 CDN / SLB / Nginx 多层代理场景下配置 `real_ip` 并验证限流 key。
- 为管理后台配置 IP 白名单 + Basic Auth 双重保护。
- 为静态资源配置防盗链，并评估是否允许空 Referer。
- 为前端站点配置 CSP Report-Only，观察是否误伤资源加载。
- 写一个脚本检查证书剩余有效期并输出告警。

### 35.4 生产思考题

- HTTPS 证书即将过期，为什么不能只依赖人工记忆？
- 为什么 `ssl_certificate` 推荐使用 fullchain，而不是单独站点证书？
- HTTP 跳 HTTPS 使用 301 后，如果发现需要回滚会有什么问题？
- HSTS 配置 `includeSubDomains` 前，需要确认哪些事情？
- 为什么限流时不能在多层代理场景下直接使用代理 IP？
- 登录接口是否应该和普通 API 使用同一套限流策略？
- 防盗链为什么不能替代真实鉴权？
- 管理后台只配置 Basic Auth 是否足够？还应叠加哪些措施？
- 如果用户反馈偶发 429，应该如何判断是攻击、误伤还是容量不足？
- 如果后端收到的协议总是 HTTP，导致生成错误回调地址，应检查哪些配置？

## 36. 过关标准

完成本阶段后，应能做到：

- 能独立给 Nginx 站点配置 HTTPS。
- 能解释 HTTPS、TLS、证书、公钥、私钥、CA、证书链之间的关系。
- 能使用自签证书完成本地 HTTPS 实验。
- 能使用 Let's Encrypt 为真实域名签发和续期证书。
- 能配置 HTTP 到 HTTPS 的自动跳转，并说明 301、302、308 的差异。
- 能配置 TLS 1.2 / TLS 1.3，并说明为什么要禁用旧协议。
- 能开启 HTTP/2，并使用 curl 或浏览器开发者工具验证。
- 能配置 HSTS，并说明 `max-age`、`includeSubDomains`、`preload` 的风险。
- 能配置基础安全 Header，并理解 CSP 可能误伤业务。
- 能使用 `allow` / `deny` 做 IP 访问控制。
- 能使用 `auth_basic` 保护临时后台或测试环境。
- 能使用 `client_max_body_size` 控制上传大小，并排查 413。
- 能使用 `limit_req` 做请求速率限制，并验证 429。
- 能使用 `limit_conn` 做连接数限制，并区分它和限流的差异。
- 能使用 `valid_referers` 配置基础防盗链。
- 能排查 400、403、413、429 等入口层问题。
- 能设计一份基础生产 HTTPS 与访问治理配置骨架。

## 37. 本阶段推荐配置骨架

下面是一份可作为学习和小型生产场景起点的配置骨架：

```nginx
http {
    server_tokens off;

    log_format security_main '$remote_addr - $remote_user [$time_local] '
                             '"$request" $status $body_bytes_sent '
                             'host="$host" scheme="$scheme" '
                             'request_time=$request_time '
                             'referer="$http_referer" '
                             'ua="$http_user_agent" '
                             'xff="$http_x_forwarded_for" '
                             'request_id="$request_id" '
                             'upstream_addr="$upstream_addr" '
                             'upstream_status="$upstream_status" '
                             'upstream_response_time="$upstream_response_time"';

    limit_req_zone $binary_remote_addr zone=api_per_ip:20m rate=20r/s;
    limit_req_zone $binary_remote_addr zone=login_per_ip:10m rate=5r/m;
    limit_conn_zone $binary_remote_addr zone=conn_per_ip:10m;
    limit_req_status 429;
    limit_conn_status 429;

    upstream app_backend {
        server 10.0.0.11:8080 max_fails=2 fail_timeout=10s;
        server 10.0.0.12:8080 max_fails=2 fail_timeout=10s;
        keepalive 64;
    }

    server {
        listen 80;
        server_name app.example.com;

        location /.well-known/acme-challenge/ {
            root /var/www/certbot;
        }

        location / {
            return 301 https://$host$request_uri;
        }
    }

    server {
        listen 443 ssl http2;
        server_name app.example.com;

        ssl_certificate /etc/letsencrypt/live/app.example.com/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/app.example.com/privkey.pem;

        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305';
        ssl_prefer_server_ciphers off;
        ssl_session_cache shared:SSL:10m;
        ssl_session_timeout 10m;
        ssl_session_tickets off;

        add_header Strict-Transport-Security "max-age=86400" always;
        add_header X-Frame-Options "SAMEORIGIN" always;
        add_header X-Content-Type-Options "nosniff" always;
        add_header Referrer-Policy "strict-origin-when-cross-origin" always;
        add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;

        access_log /var/log/nginx/app_access.log security_main;
        error_log /var/log/nginx/app_error.log warn;

        client_max_body_size 20m;

        location = /api/login {
            limit_req zone=login_per_ip burst=5 nodelay;

            proxy_pass http://app_backend;
            proxy_http_version 1.1;
            proxy_set_header Connection "";
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        location /api/ {
            limit_req zone=api_per_ip burst=40 nodelay;
            limit_conn conn_per_ip 30;

            proxy_pass http://app_backend/;
            proxy_http_version 1.1;
            proxy_set_header Connection "";
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header X-Forwarded-Host $host;
            proxy_set_header X-Forwarded-Port $server_port;
        }

        location /admin/ {
            allow 203.0.113.10;
            deny all;

            auth_basic "Admin Area";
            auth_basic_user_file /etc/nginx/.htpasswd;

            proxy_pass http://app_backend/admin/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        location ~* /\. {
            deny all;
        }
    }
}
```

使用前必须根据实际情况调整：

- `server_name`。
- 证书路径。
- upstream 后端 IP 和端口。
- 是否需要 HTTP/2。
- HSTS `max-age`。
- 是否启用 `includeSubDomains`。
- 限流 key、rate 和 burst。
- 上传大小限制。
- 管理后台白名单。
- Basic Auth 用户文件。
- 是否存在 CDN / SLB / 多层代理真实 IP 问题。

## 38. 本阶段总结

第四阶段的核心不是“把 HTTPS 配起来就结束”，而是建立入口层安全治理模型：

```text
域名解析
  -> HTTP 跳转 HTTPS
  -> TLS 握手
  -> 证书链验证
  -> 协议与加密套件协商
  -> HTTP/2 与安全 Header
  -> 访问控制
  -> 请求限流
  -> 连接限制
  -> 上传大小限制
  -> 防盗链
  -> 日志审计
  -> 入口层问题排查
```

学习本阶段时，最重要的几个判断是：

- 证书是否可信、完整、未过期、域名匹配。
- 私钥是否安全保存，是否有泄露风险。
- TLS 协议版本是否满足当前安全基线。
- HTTP 到 HTTPS 跳转是否保留路径和参数。
- HSTS 是否有充分灰度和回滚预案。
- 安全 Header 是否真正出现在响应中，是否误伤业务。
- 限流和限连接是否基于正确的客户端身份。
- 403、413、429 是 Nginx 返回，还是上层或后端返回。
- 防盗链是否符合真实客户端访问方式。
- 管理入口是否同时具备网络层和认证层保护。

掌握本阶段后，就可以把 Nginx 建设为更安全、更可靠、更可治理的入口层。下一阶段建议进入缓存、静态资源与性能优化，继续补齐资源效率和性能调优能力。
