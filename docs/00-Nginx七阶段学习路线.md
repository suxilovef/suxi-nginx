---
title: Nginx七阶段学习路线
date: 2026-07-11 00:00:00
description: Nginx七阶段学习路线
tags:
  - Nginx
categories:
  - Nginx
realm: wujing
cover: /image/post_cover/wujing-nginx-roadmap.svg
rank: 95
top_img: false
---
# Nginx 七阶段学习路线

## 1. 路线定位

这份文档用于记录 Nginx 从入门部署、配置理解、反向代理、负载均衡、高可用、性能优化、生产排障到源码机制的完整学习路线。后续学习建议按阶段推进，不要一开始就陷入零散配置项，也不要过早进入源码。

学习目标不是“会复制一段 `nginx.conf`”，而是达到下面几个能力：

- 能独立部署、编译、升级和管理 Nginx。
- 能理解 Nginx 配置文件结构、指令继承、请求匹配和处理流程。
- 能正确配置静态资源、反向代理、负载均衡、HTTPS、缓存、限流和日志。
- 能设计 Nginx 在网关、入口层、负载均衡层、静态资源层中的使用方案。
- 能根据日志、状态、抓包和系统指标定位 4xx、5xx、超时、性能抖动等问题。
- 能做容量评估、性能压测、内核参数调优和生产运维治理。
- 能读懂关键源码路径，理解事件驱动、worker 模型、upstream、filter、phase handler 等核心机制。

Nginx 的学习不要停留在“配置项大全”。真正重要的是理解：

```text
连接如何进入 Nginx
  -> 请求如何匹配 server/location
  -> 请求如何进入不同处理阶段
  -> 请求如何转发到 upstream
  -> 响应如何经过 filter 返回客户端
  -> 故障、超时、限流、缓存和日志如何在链路中发生
```

## 2. 总体阶段

| 阶段 | 名称 | 核心目标 | 完成后的水平 |
| --- | --- | --- | --- |
| 第 1 阶段 | 单机部署与基础认知 | 理解 Nginx 安装、目录、进程模型、基础配置和静态资源服务 | 会部署、会启动、会看配置和日志 |
| 第 2 阶段 | 配置体系与请求匹配 | 理解配置上下文、指令继承、server/location 匹配、rewrite 和变量 | 能写正确、可维护的 Nginx 配置 |
| 第 3 阶段 | 反向代理与负载均衡 | 掌握 proxy、upstream、负载均衡算法、超时、重试、Header 传递 | 能把 Nginx 用作稳定入口代理 |
| 第 4 阶段 | HTTPS、安全与访问治理 | 掌握 TLS、证书、HTTP/2、限流、访问控制、防盗链和基础安全加固 | 能建设安全可靠的入口层 |
| 第 5 阶段 | 缓存、静态资源与性能优化 | 掌握静态资源优化、gzip/brotli、proxy cache、sendfile、连接和内核调优 | 能做性能优化和资源治理 |
| 第 6 阶段 | 生产排障、高可用与运维体系 | 掌握日志分析、状态观测、4xx/5xx、超时、灰度、reload、升级和高可用方案 | 能守住生产 Nginx |
| 第 7 阶段 | 源码、模块与底层机制 | 理解事件模型、master/worker、phase、upstream、filter、内存池和模块开发 | 接近专家级理解 |

完成前三个阶段后，可以认为具备 Nginx 中级偏上的配置和代理能力。真正走向资深，需要继续补第 4 到第 7 阶段。

## 3. 第 1 阶段：单机部署与基础认知

对应目录：

```text
docs/01-单机部署与基础认知/
```

核心目标：

- 理解 Nginx 是高性能事件驱动 Web Server、反向代理和负载均衡器。
- 掌握 yum/apt 安装、源码编译安装、systemd 管理和基础升级方式。
- 理解 Nginx 主配置文件、日志目录、静态资源目录、模块目录和运行用户。
- 理解 master/worker 进程模型和 reload 的基本行为。
- 能使用 `nginx -t`、`nginx -s reload`、`systemctl`、`curl`、`tail` 等命令完成基础排查。

必须掌握：

- Nginx 的典型目录结构：`/etc/nginx`、`/usr/sbin/nginx`、`/var/log/nginx`、`/usr/share/nginx/html`。
- `nginx.conf` 的基础结构：`main`、`events`、`http`、`server`、`location`。
- `user`、`worker_processes`、`worker_connections`、`pid`、`error_log`、`access_log`。
- `include` 的作用和配置拆分方式。
- `nginx -t` 配置检查机制。
- reload、restart、stop、quit 的区别。
- Nginx 与 Apache、Tomcat、Spring Boot 内置 Web 容器的定位差异。

必做实验：

- 使用包管理器安装 Nginx。
- 使用源码编译安装 Nginx，并查看 `nginx -V` 编译参数。
- 使用 systemd 管理 Nginx 启停和开机自启。
- 修改默认首页并通过浏览器或 `curl` 访问。
- 配置一个最小 `server` 块监听自定义端口。
- 故意写错配置，用 `nginx -t` 观察报错。
- 修改配置后执行 reload，观察 worker 进程变化。
- 从宿主机访问虚拟机或服务器上的 Nginx。

过关标准：

- 能独立安装并启动一台 Nginx。
- 能说明 `nginx.conf` 中每个基础块的作用。
- 能解释 master 和 worker 的职责。
- 能排查 Nginx 启动失败、端口占用、权限不足和远程访问失败。
- 能用 `nginx -V` 判断当前 Nginx 支持哪些模块。

## 4. 第 2 阶段：配置体系与请求匹配

对应目录：

```text
docs/02-配置体系与请求匹配/
```

核心目标：

- 理解 Nginx 配置不是简单文本，而是有上下文、作用域、继承和执行阶段的配置体系。
- 掌握 `server_name`、`listen`、`location` 的匹配规则。
- 掌握 URI、request URI、root、alias、index、try_files 的真实差异。
- 理解 rewrite、return、变量、正则匹配和内部跳转。
- 能写出结构清晰、可维护、可排查的 Nginx 配置。

必须掌握：

- 配置上下文：`main`、`events`、`http`、`server`、`location`、`upstream`。
- 指令继承和覆盖规则。
- `listen` 默认 server 规则。
- `server_name` 精确匹配、通配符匹配、正则匹配优先级。
- `location =`、`location ^~`、`location ~`、`location ~*`、普通前缀匹配优先级。
- `root` 和 `alias` 的区别。
- `try_files` 的执行逻辑。
- `rewrite`、`return`、`break`、`last` 的区别。
- Nginx 常用变量：`$uri`、`$request_uri`、`$host`、`$http_host`、`$remote_addr`、`$proxy_add_x_forwarded_for`。
- 正则 location 的性能和可维护性风险。

必做实验：

- 配置多个 `server_name`，验证 Host 匹配规则。
- 配置多个 `location`，验证完整匹配优先级。
- 对比 `root` 和 `alias` 在不同 URI 下的文件路径拼接结果。
- 使用 `try_files` 支持前端 history 路由。
- 使用 `return 301` 做域名跳转。
- 使用 `rewrite` 做 URI 改写，并观察 `$uri` 和 `$request_uri` 差异。
- 配置自定义 access log 格式，打印关键变量。
- 故意制造 location 误匹配并修复。

过关标准：

- 能手写一个多域名、多 location 的 Nginx 配置。
- 能准确解释一次请求最终进入哪个 `server` 和哪个 `location`。
- 能说明 `root`、`alias`、`try_files` 的差异和坑点。
- 能避免 rewrite 滥用。
- 能根据 access log 判断请求匹配和 URI 改写是否符合预期。

## 5. 第 3 阶段：反向代理与负载均衡

对应目录：

```text
docs/03-反向代理与负载均衡/
```

核心目标：

- 掌握 Nginx 作为反向代理和应用入口的核心能力。
- 理解 upstream、负载均衡算法、连接复用、超时、重试和故障摘除。
- 掌握请求头、真实 IP、协议、Host、WebSocket、长连接等代理细节。
- 能将 Nginx 正确接入 Spring Boot、Tomcat、Node.js、Go、静态前端和微服务网关。

必须掌握：

- `proxy_pass` 带 URI 和不带 URI 的差异。
- `upstream` 的定义方式。
- 负载均衡算法：round-robin、weight、ip_hash、least_conn、hash。
- `max_fails`、`fail_timeout` 的真实含义和局限。
- `proxy_connect_timeout`、`proxy_send_timeout`、`proxy_read_timeout`。
- `proxy_next_upstream` 的重试条件和幂等风险。
- `proxy_set_header Host`、`X-Real-IP`、`X-Forwarded-For`、`X-Forwarded-Proto`。
- `keepalive` upstream 连接池。
- WebSocket 代理配置。
- gRPC、HTTP/2 代理的基本概念。
- 后端服务返回 502、503、504 的常见原因。

必做实验：

- 准备两个后端 HTTP 服务，通过 Nginx upstream 做轮询转发。
- 配置权重并观察请求分布。
- 停止一个后端服务，观察 Nginx 错误日志和转发行为。
- 配置真实 IP 传递，并在后端打印 Header。
- 对比 `proxy_pass http://backend` 和 `proxy_pass http://backend/` 的 URI 转发差异。
- 配置 WebSocket 代理并验证连接升级。
- 配置 upstream keepalive，观察连接复用。
- 故意设置过短超时，复现 504。

过关标准：

- 能独立配置 Nginx 代理一个后端应用集群。
- 能解释 `proxy_pass` URI 拼接规则。
- 能说明各种超时参数分别发生在哪个阶段。
- 能根据日志区分 502、503、504。
- 能合理设置 Header，保证后端拿到真实客户端信息。

## 6. 第 4 阶段：HTTPS、安全与访问治理

对应目录：

```text
docs/04-HTTPS安全与访问治理/
```

核心目标：

- 掌握 HTTPS 证书部署、TLS 协议、安全套件和证书更新。
- 理解 HTTP/2、HSTS、OCSP Stapling 等入口层安全能力。
- 掌握访问控制、限流、限连接、防盗链、上传大小限制和基础防护。
- 能设计一套符合生产要求的 Nginx HTTPS 和访问治理配置。

必须掌握：

- TLS 握手基础流程。
- 证书、公钥、私钥、CA、中间证书链。
- `ssl_certificate`、`ssl_certificate_key`、`ssl_protocols`、`ssl_ciphers`。
- HTTP 到 HTTPS 强制跳转。
- HSTS 的作用和风险。
- HTTP/2 开启方式和适用场景。
- `client_max_body_size`。
- `allow`、`deny`、`auth_basic`。
- `limit_req_zone`、`limit_req`。
- `limit_conn_zone`、`limit_conn`。
- 防盗链：`valid_referers`。
- 隐藏版本号：`server_tokens off`。
- 基础安全 Header：`X-Frame-Options`、`X-Content-Type-Options`、`Content-Security-Policy`、`Referrer-Policy`。

必做实验：

- 使用自签证书部署 HTTPS。
- 使用真实域名和 Let's Encrypt 证书部署 HTTPS。
- 配置 HTTP 自动跳转 HTTPS。
- 配置 TLS 协议版本，禁用不安全版本。
- 开启 HTTP/2 并用浏览器开发者工具验证。
- 配置 HSTS 并理解回滚风险。
- 配置 IP 白名单和基础认证。
- 配置接口限流，使用压测工具验证 429。
- 配置上传大小限制，复现并解决 413。
- 配置防盗链并验证图片访问行为。

过关标准：

- 能独立给站点配置 HTTPS。
- 能解释证书链和私钥保护的重要性。
- 能设计基础访问控制、限流和限连接策略。
- 能排查 400、403、413、429 等入口层问题。
- 能说明 HSTS、HTTP/2、安全 Header 的适用边界。

## 7. 第 5 阶段：缓存、静态资源与性能优化

对应目录：

```text
docs/05-缓存静态资源与性能优化/
```

核心目标：

- 从“能代理请求”进入“能优化请求链路”。
- 掌握静态资源缓存、压缩、零拷贝、连接复用和 proxy cache。
- 理解 Nginx 性能瓶颈通常来自 CPU、内存、磁盘、网络、后端延迟或配置不当。
- 能根据业务类型设计静态资源和动态接口的不同优化策略。

必须掌握：

- `sendfile`、`tcp_nopush`、`tcp_nodelay`。
- `keepalive_timeout`、`keepalive_requests`。
- `gzip`、`gzip_types`、`gzip_comp_level`。
- brotli 模块的定位和使用边界。
- 浏览器缓存：`Cache-Control`、`Expires`、`ETag`、`Last-Modified`。
- 静态资源版本化和强缓存策略。
- `open_file_cache`。
- `proxy_buffering`、`proxy_buffers`、`proxy_busy_buffers_size`。
- `proxy_cache_path`、`proxy_cache`、`proxy_cache_key`、`proxy_cache_valid`。
- 缓存穿透、缓存雪崩、缓存刷新和缓存绕过。
- 大文件下载、 Range 请求、限速和断点续传。
- worker 数、连接数、文件描述符和系统内核参数。

必做实验：

- 配置静态资源强缓存和协商缓存。
- 对比开启 gzip 前后的响应大小和 CPU 消耗。
- 配置 `sendfile` 并观察大文件传输表现。
- 开启 `open_file_cache` 优化大量小文件访问。
- 配置 `proxy_cache` 缓存后端接口响应。
- 设计不同 URI 的缓存策略：HTML 不强缓存，JS/CSS/图片强缓存。
- 模拟后端短暂故障，观察缓存是否能兜底。
- 使用压测工具对比缓存命中和未命中的吞吐与延迟。
- 调整 `worker_processes`、`worker_connections`、系统 `ulimit` 并观察连接上限。

过关标准：

- 能独立配置静态资源缓存策略。
- 能说明 gzip、brotli、sendfile、keepalive 的性能影响。
- 能正确使用 proxy cache，而不是盲目缓存动态接口。
- 能根据日志和指标判断缓存命中率。
- 能解释 worker 连接数、文件描述符和系统连接上限之间的关系。

## 8. 第 6 阶段：生产排障、高可用与运维体系

对应目录：

```text
docs/06-生产排障高可用与运维体系/
```

核心目标：

- 从“会配置 Nginx”进入“能守生产 Nginx”。
- 掌握日志分析、错误定位、状态观测、平滑 reload、灰度发布和高可用部署。
- 掌握 Nginx 与系统层、网络层、后端服务层之间的问题边界。
- 能写出生产故障排查 SOP 和变更发布方案。

必须掌握：

- access log 字段设计：请求时间、upstream 时间、状态码、upstream 状态、请求 ID。
- error log 级别和典型错误含义。
- `$request_time`、`$upstream_response_time`、`$upstream_connect_time`、`$upstream_header_time`。
- 4xx 问题：400、401、403、404、405、408、413、414、429、499。
- 5xx 问题：500、502、503、504。
- Nginx stub_status、status 模块和第三方监控模块。
- 日志切割、归档和采集。
- 平滑 reload、二进制热升级和回滚。
- Nginx 单机不可用风险和 Keepalived/VIP 高可用方案。
- 云负载均衡、Ingress Controller、API Gateway 与 Nginx 的职责边界。
- 灰度发布、蓝绿发布、按 Header/Cookie/IP 分流。
- 配置变更评审、发布窗口、回滚预案。

必做实验：

- 自定义 JSON 格式 access log，接入请求 ID。
- 制造 404、403、413、499、502、504 并定位原因。
- 使用 `stub_status` 观察连接状态。
- 使用 `curl -w` 分析 DNS、连接、TLS、首包、总耗时。
- 使用 `ss`、`lsof`、`top`、`free`、`iostat`、`sar` 排查系统瓶颈。
- 使用 `tcpdump` 或 Wireshark 抓包分析连接问题。
- 编写 Nginx 日志切割脚本或使用 logrotate。
- 模拟 reload 期间请求是否中断。
- 搭建 Keepalived + Nginx 双机 VIP。
- 配置按 Cookie 或 Header 灰度到新 upstream。

过关标准：

- 能根据日志快速定位入口层、网络层、后端层问题边界。
- 能解释 499、502、504 的常见生产原因。
- 能设计 Nginx 监控告警项。
- 能写出 Nginx 变更发布和回滚 SOP。
- 能设计基础高可用方案，避免单点故障。

## 9. 第 7 阶段：源码、模块与底层机制

对应目录：

```text
docs/07-源码模块与底层机制/
```

核心目标：

- 从“会部署、会配置、会排障”进入“理解 Nginx 为什么这样工作”。
- 重点看核心路径，不追求一开始读完整个源码。
- 理解 Nginx 高性能来自事件驱动、非阻塞 IO、多进程 worker、内存池、模块化 pipeline，而不是某个单独配置项。
- 能在遇到复杂问题时知道应该去源码哪个模块找答案。

必须掌握：

- Nginx 源码目录结构。
- master/worker 启动流程。
- 配置解析和模块初始化流程。
- cycle、connection、request、upstream 等核心结构。
- 事件模块：epoll/kqueue/select。
- 非阻塞 IO 和事件循环。
- HTTP 请求处理阶段 phase handler。
- rewrite、access、content、log 等阶段的执行位置。
- upstream 模块转发流程。
- filter 链：header filter、body filter。
- 内存池 `ngx_pool_t`。
- buffer 和 chain。
- slab、shared memory、zone。
- timer、红黑树和事件超时管理。
- reload 和热升级内部机制。
- 动态模块和第三方模块开发基础。

建议阅读顺序：

```text
源码目录和编译参数
  -> master/worker 启动流程
  -> 配置解析与模块初始化
  -> event 模块和连接接受
  -> HTTP 请求解析
  -> phase handler 请求处理阶段
  -> upstream 反向代理流程
  -> filter 响应过滤链
  -> 内存池、buffer、chain
  -> shared memory 与限流缓存模块
  -> reload、热升级与 worker 退出
```

必做实验：

- 下载 Nginx 源码并完成 debug 编译。
- 使用 `nginx -V` 对照编译模块和源码目录。
- 跟踪一次 master/worker 启动流程。
- 跟踪一次配置解析流程。
- 跟踪一次 HTTP 请求从 accept 到 access log 的完整路径。
- 跟踪一次 `proxy_pass` 到 upstream 的转发流程。
- 跟踪一次响应经过 header/body filter 的过程。
- 理解 `limit_req` 使用 shared memory 的方式。
- 编写一个最小 HTTP content handler 模块。
- 编写一个最小 header filter 或 log 阶段模块。

过关标准：

- 能从源码角度解释 Nginx 为什么能支撑高并发。
- 能解释 master/worker、事件循环、非阻塞 IO 的关系。
- 能说明一次 HTTP 请求在 Nginx 内部经历哪些阶段。
- 能解释 upstream 和 filter 的核心实现流程。
- 能读懂常见模块的实现方式，并具备写简单模块的能力。

## 10. 推荐学习顺序

建议严格按下面顺序推进：

```text
01 单机部署与基础认知
  -> 02 配置体系与请求匹配
  -> 03 反向代理与负载均衡
  -> 04 HTTPS、安全与访问治理
  -> 05 缓存、静态资源与性能优化
  -> 06 生产排障、高可用与运维体系
  -> 07 源码、模块与底层机制
```

不要一开始就背完整配置项，也不要过早进入源码。前三阶段解决 Nginx 的部署、配置和代理基础；第四、第五阶段解决安全和性能；第六阶段解决生产治理；第七阶段再回到源码，理解会更稳。

## 11. 推荐配套实验环境

建议准备下面的实验环境：

| 角色 | 建议数量 | 用途 |
| --- | --- | --- |
| Nginx 节点 | 1-2 台 | 部署、代理、HTTPS、高可用实验 |
| 后端应用节点 | 2-3 台 | 模拟 Spring Boot、Tomcat、Node.js 或静态服务 |
| 客户端压测节点 | 1 台 | 使用 wrk、ab、hey、curl、浏览器访问 |
| DNS/域名环境 | 1 套 | 验证多域名、HTTPS、证书和 Host 匹配 |
| 日志/监控环境 | 可选 | 接入 Prometheus、Grafana、ELK、Loki 等 |

推荐工具：

- 命令行：`curl`、`wget`、`openssl`、`dig`、`nslookup`。
- 压测：`wrk`、`ab`、`hey`、`siege`。
- 系统观测：`top`、`htop`、`free`、`vmstat`、`iostat`、`sar`、`ss`、`lsof`。
- 网络分析：`tcpdump`、Wireshark。
- 日志分析：`awk`、`grep`、`jq`、GoAccess。
- 调试分析：`strace`、`gdb`、火焰图工具。

## 12. 重点知识地图

Nginx 深入学习可以围绕下面几条主线展开：

```text
配置主线：
nginx.conf -> context -> server -> location -> rewrite -> variable

代理主线：
client -> Nginx -> upstream -> backend -> response filter -> client

性能主线：
worker -> connection -> event loop -> non-blocking IO -> buffer -> sendfile

安全主线：
TLS -> access control -> rate limit -> security headers -> log audit

运维主线：
log -> status -> metrics -> alert -> reload -> rollback -> high availability

源码主线：
cycle -> module -> event -> request -> phase -> upstream -> filter -> log
```

## 13. 常见误区

学习 Nginx 时要特别避免下面这些误区：

- 只会复制配置，不理解请求匹配和执行阶段。
- 混淆 `root` 和 `alias`，导致静态资源路径错误。
- 不理解 `proxy_pass` URI 拼接规则，导致转发路径异常。
- 只看 HTTP 状态码，不看 error log 和 upstream 时间。
- 把所有 502 都归因于 Nginx，忽略后端连接拒绝、进程崩溃和网络问题。
- 随意开启缓存，缓存了不该缓存的用户态动态数据。
- 盲目调大 worker 和连接数，却忽略文件描述符、端口、带宽和后端能力。
- 使用 rewrite 解决所有问题，导致配置难以维护。
- HTTPS 只关注证书能不能用，不关注协议版本、安全套件和续期。
- 生产环境直接修改配置并 reload，没有评审、备份、测试和回滚方案。
- 过早读源码，基础配置和生产问题还没掌握就陷入细节。

## 14. 最终能力定位

如果只完成前三阶段：

```text
Nginx 中级偏上
具备部署、配置、请求匹配、反向代理和负载均衡基础能力
```

如果完成前五阶段：

```text
Nginx 高级应用能力
能设计安全、稳定、高性能的入口层和静态资源方案
```

如果完成前六阶段：

```text
Nginx 生产方案能力
能做生产排障、监控告警、高可用、灰度发布和运维治理
```

如果完成全部七阶段：

```text
Nginx 高级工程师到专家入门水平
具备部署、配置、代理、安全、性能、运维、源码和模块开发综合能力
```

真正的专家水平还需要持续积累线上故障经验、复杂网络环境经验、业务架构经验、源码级问题定位经验，以及对 OpenResty、Ingress Controller、Envoy、HAProxy、API Gateway 等相邻技术的横向理解。
