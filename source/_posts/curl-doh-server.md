---
layout: title
title: 用 cURL 进行 DoH 模式下的 EDNS 查询测试
date: 2024-05-19 17:49:31
tags:
- DNS
- DoH
- EDNS
- dig
categories: DNS
---

由于个人使用 [https_dns_proxy](https://github.com/aarond10/https_dns_proxy) 配合代理服务器，从而来解决 DNS 污染的问题。但是最近发现针对部分国内的 CDN 返回的地址都是针对代理服务器的IP进行的优化，而这部分 IP 莫名其妙又不在国内 IP 的段内从而导致直连访问，最终导致访问异常缓慢甚至无法正确返回。 在解决这个问题的这个过程中就需要对 DNS Server 进行一系列的查询测试，因此做了一些研究有了这个文章。这个文章包含了 DNS 优化的主流方案原理。此外由于主流的 DNS 工具都不支持 DoH，为了方便测试也提供了一个 Tricky 的方式来测试 DoH 查询。


<!-- more -->

发生问题的是 `g.alicdn.com` 这个地址，由于这个很明显是一个经典的 DNS 优化问题，所以很自然的想用 `dig` 去测试下。测试结果显示这个域名当时如果直接通过 ISP 提供的 DNS 进行查询返回的地址是没问题的，在我圣何塞的机器上对 `1.1.1.1` 进行查询返回的另外一个地址在圣何塞机器上也是没问题的，但是在如果大陆则无法正确访问，这个问题直接导致了我无法打开任何阿里系的网站。

# DNS 优化方案
由于通常 DNS 提供商都会返回的 IP 地址都会尽可能靠近浏览器从而起到加速的作用。要实现这个功能则必须先拿到发起查询的源 IP，而关于如何获取源 IP 方面就比较重要了，技术上有两种手段：`EDNS` 或者对 DNS 进行 anycast。

## EDNS
在 EDNS 模式下，对 DNS 的查询请求做了扩展，在其中加入了 `subnet` 部分的信息。DNS Server 可以直接从 DNS 查询的请求中获取客户端的源 IP 地址，并针对性的做出响应。但是这样做的好处很明显：简单可靠。即使请求被经过了层层 NAT 或者代理，四层的源 IP 可能早就丢失了的情况下也能精确返回正确的优化。但是甘蔗没有两头甜，凡是都有代价：
1. 请求很难缓存
    由于 DNS 是一个递归查询模型，即在当前 DNS 服务器对查询结果未知的情况下会递归去上游的服务器进行递归查询，并将结果缓存起来。通常情况下，缓存只需要考虑以域名为 key，以 A/AAAA 等记录为 Value 进行构建即可，即相同的域名返回相同的结果。但是在加入 `subnet` 参数后就整整增加了一个维度后，需要缓存的数据量就会成倍放大，因此 DNS Server 可能会选择不缓存这类请求从而导致整体查询变慢。

2. 存在用户隐私泄漏风险
    通常情况下，由于缓存服务器的存在相当一大部分请求将会由下游的 DNS 缓存并返回，只能看到来自下游 DNS 服务器的请求，而权威 DNS 服务器并不能准确知道哪些客户端发起了多少请求。但是当引入 EDNS 后，每次带 `subnet` 的请求权威 DNS 服务器都能知道请求来源的大致范围。

## Anycast DNS
相比 EDNS 方案，Anycast 方案走了另外一个极端，这个方案里并没有扩展 DNS 服务器的请求内容，而是利用 Anycast 技术将一个 IP 地址关联到 N 个服务器上，而此时任何一个 DNS 服务器都可以对这个查询进行响应。此时只需要铺开足够多的服务器，从而让离客户端最近的服务器来提供服务，就能够返回优化过后的 IP 地址。此时上面的两个缺点就不存在了：由于 DNS 服务器其实并没有理解`subnet`语义，因此他还是只需要简单缓存就行了。此外，由于没有携带 `subnet` 信息来对权威 DNS 服务器进行递归查询，因此权威DNS 服务器看到的只有这个下游 DNS 服务器而已，充分避免了用户隐私泄漏的问题。至于代价的话，就是这样的 DNS 无法穿透 NAT 或者代理，从而可能导致查询结果的不准确。此外这种模式依赖广泛的服务器部署，可能成本会高不少吧。

# 验证优化效果
通过 `dig` 命令对 `1.1.1.1` 直接发起域名查询, 发现 EDNS 并不工作，返回的永远是 `47.246.24.171` 和 `47.246.24.170`。

```bash
#dig g.alicdn.com @1.1.1.1

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.3.alios7.7 <<>> g.alicdn.com @1.1.1.1
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 19443
;; flags: qr rd ra; QUERY: 1, ANSWER: 3, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
;; QUESTION SECTION:
;g.alicdn.com.			IN	A

;; ANSWER SECTION:
g.alicdn.com.		597	IN	CNAME	g.alicdn.com.danuoyi.alicdn.com.
g.alicdn.com.danuoyi.alicdn.com. 57 IN	A	47.246.24.171
g.alicdn.com.danuoyi.alicdn.com. 57 IN	A	47.246.24.170

;; Query time: 199 msec
;; SERVER: 1.1.1.1#53(1.1.1.1)
;; WHEN: Sun May 19 19:31:25 CST 2024
;; MSG SIZE  rcvd: 108

#
#dig g.alicdn.com +subnet=216.58.194.0/24  @1.1.1.1

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.3.alios7.7 <<>> g.alicdn.com +subnet=216.58.194.0/24 @1.1.1.1
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 51571
;; flags: qr rd ra; QUERY: 1, ANSWER: 3, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
;; QUESTION SECTION:
;g.alicdn.com.			IN	A

;; ANSWER SECTION:
g.alicdn.com.		576	IN	CNAME	g.alicdn.com.danuoyi.alicdn.com.
g.alicdn.com.danuoyi.alicdn.com. 36 IN	A	47.246.24.170
g.alicdn.com.danuoyi.alicdn.com. 36 IN	A	47.246.24.171

;; Query time: 196 msec
;; SERVER: 1.1.1.1#53(1.1.1.1)
;; WHEN: Sun May 19 19:31:42 CST 2024
;; MSG SIZE  rcvd: 108

#
#dig g.alicdn.com +subnet=220.205.248.0/24 @1.1.1.1

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.3.alios7.7 <<>> g.alicdn.com +subnet=220.205.248.0/24 @1.1.1.1
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 40428
;; flags: qr rd ra; QUERY: 1, ANSWER: 3, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
;; QUESTION SECTION:
;g.alicdn.com.			IN	A

;; ANSWER SECTION:
g.alicdn.com.		600	IN	CNAME	g.alicdn.com.danuoyi.alicdn.com.
g.alicdn.com.danuoyi.alicdn.com. 60 IN	A	47.246.24.170
g.alicdn.com.danuoyi.alicdn.com. 60 IN	A	47.246.24.171

;; Query time: 192 msec
;; SERVER: 1.1.1.1#53(1.1.1.1)
;; WHEN: Sun May 19 19:32:00 CST 2024
;; MSG SIZE  rcvd: 108
```

重新对 `8.8.8.8` 发起查询，则正确返回了不同的地址：
```
#dig g.alicdn.com +subnet=220.205.248.0/24 @8.8.8.8

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.3.alios7.7 <<>> g.alicdn.com +subnet=220.205.248.0/24 A @8.8.8.8
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 56887
;; flags: qr rd ra; QUERY: 1, ANSWER: 3, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 512
; CLIENT-SUBNET: 220.205.248.0/24/24
;; QUESTION SECTION:
;g.alicdn.com.			IN	A

;; ANSWER SECTION:
g.alicdn.com.		600	IN	CNAME	g.alicdn.com.danuoyi.alicdn.com.
g.alicdn.com.danuoyi.alicdn.com. 60 IN	A	112.132.33.145
g.alicdn.com.danuoyi.alicdn.com. 60 IN	A	112.132.33.146

;; Query time: 80 msec
;; SERVER: 8.8.8.8#53(8.8.8.8)
;; WHEN: Sun May 19 19:36:20 CST 2024
;; MSG SIZE  rcvd: 119

#
#dig g.alicdn.com +subnet=216.58.194.0/24  @8.8.8.8

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.3.alios7.7 <<>> g.alicdn.com +subnet=216.58.194.0/24 @8.8.8.8
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 59231
;; flags: qr rd ra; QUERY: 1, ANSWER: 3, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 512
; CLIENT-SUBNET: 216.58.194.0/24/24
;; QUESTION SECTION:
;g.alicdn.com.			IN	A

;; ANSWER SECTION:
g.alicdn.com.		600	IN	CNAME	g.alicdn.com.danuoyi.alicdn.com.
g.alicdn.com.danuoyi.alicdn.com. 60 IN	A	47.246.24.170
g.alicdn.com.danuoyi.alicdn.com. 60 IN	A	47.246.24.171

;; Query time: 192 msec
;; SERVER: 8.8.8.8#53(8.8.8.8)
;; WHEN: Sun May 19 19:36:40 CST 2024
;; MSG SIZE  rcvd: 119
```

考虑到 Cloudflare 在 CDN 上的深耕，不可能会不支持 DNS 优化吧？但是经过一系列调查后发现，Cloudflare 还真就不支持 EDNS。参考 [Reddit 网友的信息](https://www.reddit.com/r/dns/comments/1585dkt/edns_client_subnet_enabled_google_dns_vs/)：
> I don't speak for Cloudflare but I've worked with them a little bit. Cloudflare is sort of philosophically opposed to ECS. They prefer to have a resolver instance running near every customer, which means that the resolver instance will always appear to be relative close to the upstream servers, and so the response will be as good as if ECS. In theory, anyways. I'm not sure I 100% agree with this but conceptually it is very tidy. The problem is that if they ever have to ship resolver traffic away, say due to an outage, or perhaps they just can't get instances to certain locations, then ECS would do a better job.

原来由于上面提到的 EDNS 可能会造成用户隐私泄漏 Cloudflare 选择更激进的直接不支持 EDNS 查询，看来是赛博菩萨 Cloudflare 比 Google 更好的诠释了什么叫 `Don't Be Evil`。


但是由于众所周知的原因，无论是 `Google DNS` 还是 `Cloudflare DNS` 的 UDP 模式在大陆都不能直接正确工作。即使借助 [https_dns_proxy](https://github.com/aarond10/https_dns_proxy) 来进行 DoH 查询还存在被阻断的问题不能稳定工作。因此，我这边会使用 DoH over proxy 的方式来使用。但是由于 `AnyCast DNS`不能穿透代理，所以只能把 DNS 改成 `Google DNS` 了。

# cURL 验证 DoH 查询结果
如果已经部署了 [https_dns_proxy](https://github.com/aarond10/https_dns_proxy) 的话，可以直接使用 `dig` 或者 `nslookup` 命令对本地的 UDP 端口进行查询测试，这个不做过多介绍了，这边我要介绍的是使用 cURL 命令来对 DoH 服务器进行测试。

DoH 实际上存在两种标准：`DNS Wireformat` 和 `JSON API`，而后者是可以直接简单通过 cURL 进行查询的。

## JSON API
可以参考的文档，实际使用比较简单：
```bash
curl -H "accept: application/dns-json" "https://cloudflare-dns.com/dns-query?name=example.com&type=AAAA"
```

> References:
  1. Google：https://developers.google.com/speed/public-dns/docs/doh?hl=zh-cn
  2. Cloudflare：https://developers.cloudflare.com/1.1.1.1/encryption/dns-over-https/make-api-requests/dns-json/

但是这种方式经过测试无法让 DNS 服务器感知源IP：可以参考 API 手册都不存在提交源 IP 的参数，而如果将 `X-Real-IP` 还是 `X-Forward` 头加入请求的话 Cloudflare 会直接返回 403。 想来者也可以理解，毕竟没可能直接信任来自互联网的 HTTP 请求中的 `X-Forward` 头。

## DNS Wireformat
相比 `JSON API` 中人类可读的 JSON 格式，这个其实是一个二进制格式，只不过是通过 HTTP body 进行发送而已。
```bash
echo -n 'q80BAAABAAAAAAAAA3d3dwdleGFtcGxlA2NvbQAAAQAB' | base64 --decode | curl -H 'content-type: application/dns-message' --data-binary @- https://cloudflare-dns.com/dns-query -o - | hexdump
```

这种方式的秘密在于 Body 中二进制的编码格式，而翻阅了 [https_dns_proxy](https://github.com/aarond10/https_dns_proxy) 的代码之后发现实际上非常简单，它只是简单的把 UDP 查询请求的 payload 通过 HTTP body 进行发送而已。这甚至可以用 Shell 进行简单的模拟，执行下面的命令将在本地 `1234` 端口启动一个一次性的 DNS 转发服务器：

```bash
#cat <<'EOF' > /tmp/helper.sh
#!/usr/bin/env bash

URL=$1
payload=$(timeout 1s cat | base64)

command=$(echo "echo -n $payload | base64 -d | curl -H 'content-type: application/dns-message' --data-binary @- https://$URL -o -")

>&2 echo $command && eval $command
EOF

chmod +x /tmp/helper.sh
ncat  -u -l 1234 --sh-exec '/tmp/helper.sh 8.8.8.8/dns-query'
```

后面就可以通过正常的 `dig` 命令进行查询了：
```bash
dig g.alicdn.com +subnet=220.205.248.0/24 @127.0.0.1 -p 1234
```

效果如图：
![demo](./images/curl-doh-server/demo.jpg)