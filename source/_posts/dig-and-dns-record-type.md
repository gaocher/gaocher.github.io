---
title: dig与记录类型举例
date: 2019-12-29 23:09:02
tags:
categories: 
    - 网络
    - DNS
---
本文介绍dig的常规使用，以及以`iqiyi.com`域名为例的dns记录类型举例。之所以用`iqiyi.com`为例，是因为本人在此公司任职:)。DNS记录类型介绍可以查看{% post_link dns-data-type-md DNS记录类型%}。
# dig命令
```
dig [type] [@resolver-server] [+short] [+trace] target
```
- type指定记录类型如A、NS、MX等，具体可参考我之前的文章。默认查询为A记录。
- @resolver-server指定域名解析的本地服务器，默认是网关，但也可以手动指定，如8.8.8.8google的通用域名解析器。
- +short 简化应答形式，只显示ip结果
- +trace 显示整个域名解析的迭代过程，从根域名到顶级域名到次级域名直到主机域名。

# dig举例

## dig查询
执行命令：`dig iqiyi.com`，得到结果如下：
```
//第一段
; <<>> DiG 9.10.6 <<>> iqiyi.com 
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 6831
;; flags: qr rd ra; QUERY: 1, ANSWER: 5, AUTHORITY: 0, ADDITIONAL: 0

//第二段
;; QUESTION SECTION:  
;iqiyi.com.			IN	A

//第三段
;; ANSWER SECTION:    
iqiyi.com.		350	IN	A	101.227.188.172
iqiyi.com.		350	IN	A	101.227.188.170
iqiyi.com.		350	IN	A	101.227.188.174
iqiyi.com.		350	IN	A	101.227.188.176
iqiyi.com.		350	IN	A	101.227.188.178

//第四段
;; Query time: 7 msec  
;; SERVER: 192.168.3.1#53(192.168.3.1)
;; WHEN: Sun Dec 29 17:45:11 CST 2019
;; MSG SIZE  rcvd: 107
```
* 结果说明
    - 第一段 是查询命令说明以及结果统计信息展示
    - 第二段 查询内容
    - 第三段 查询结果内容，iqiyi.com的A记录有4个，350TTL（表明缓存时间是350s）。
    - 第四段 DNS服务器信息，192.168.3.1#53是本地DNS服务器192.168.3.1（家里的网关ip），53是端口号，DNS服务的默认端口。107是收到结果的大小107个字节。

## dig +short 简单查询
```
101.227.188.170
101.227.188.174
101.227.188.172
101.227.188.176
101.227.188.178
```
只返回查询结果的ip地址。

## dig分级查询
执行命令:`dig iqiyi.com +trace`,得到结果如下：
```
; <<>> DiG 9.10.6 <<>> iqiyi.com +trace
;; global options: +cmd
;; Received 28 bytes from 192.168.3.1#53(192.168.3.1) in 3 ms
```
发现什么都没有返回，应该是本地服务器192.168.3.1网关不支持trace的查询。

所以改用8.8.8.8服务器来查询，命令：`dig iqiyi.com +trace @8.8.8.8`,结果如下:
```
; <<>> DiG 9.10.6 <<>> iqiyi.com +trace @8.8.8.8
;; global options: +cmd
//首次查询 根域名服务器查询 （查询本地服务器）
.			51069	IN	NS	f.root-servers.net.
.			51069	IN	NS	h.root-servers.net.
.			51069	IN	NS	e.root-servers.net.
.			51069	IN	NS	l.root-servers.net.
.			51069	IN	NS	c.root-servers.net.
.			51069	IN	NS	b.root-servers.net.
.			51069	IN	NS	m.root-servers.net.
.			51069	IN	NS	a.root-servers.net.
.			51069	IN	NS	d.root-servers.net.
.			51069	IN	NS	i.root-servers.net.
.			51069	IN	NS	k.root-servers.net.
.			51069	IN	NS	j.root-servers.net.
.			51069	IN	NS	g.root-servers.net.
.			51069	IN	RRSIG	NS 8 0 518400 20200110170000 20191228160000 22545 . O+lE7aOii9IvLdYuWpMrTY0RkPpbc0yJLAhg/pwdxh8qZiACwS4TxuYo vxZrvoB0sJ7RyDgycViUEt++avEWx1JjzluiOXj1R0jqZQ7EXO+L+acP o88jV9F3Hqeuudj4u3ZZvM55eLnWfJaJzap/H3xi87rt5obw3zMd5QZE M2zQXSnrCiI2rSelaTeeHx6Mu+8yVigaAwRAH/8QdiAs2y2VuLbdl+C7 mHrbc3blraXSC6dzlGAEmryReOS5WOaMkSJZBctFbnX8KMmSdd83zBOv CVtdkPKGCwtUTWbbNZqrndYGqfCHf8NVLugz5R+jY3uzMvaQvdXBKwFT dcImog==
;; Received 525 bytes from 8.8.8.8#53(8.8.8.8) in 41 ms

//第一次迭代 顶级域名（TLD）查询 (查询根域名服务器)
com.			172800	IN	NS	l.gtld-servers.net.
com.			172800	IN	NS	b.gtld-servers.net.
com.			172800	IN	NS	c.gtld-servers.net.
com.			172800	IN	NS	d.gtld-servers.net.
com.			172800	IN	NS	e.gtld-servers.net.
com.			172800	IN	NS	f.gtld-servers.net.
com.			172800	IN	NS	g.gtld-servers.net.
com.			172800	IN	NS	a.gtld-servers.net.
com.			172800	IN	NS	h.gtld-servers.net.
com.			172800	IN	NS	i.gtld-servers.net.
com.			172800	IN	NS	j.gtld-servers.net.
com.			172800	IN	NS	k.gtld-servers.net.
com.			172800	IN	NS	m.gtld-servers.net.
com.			86400	IN	DS	30909 8 2 E2D3C916F6DEEAC73294E8268FB5885044A833FC5459588F4A9184CF C41A5766
com.			86400	IN	RRSIG	DS 8 1 86400 20200111050000 20191229040000 22545 . YT51a7sayHoEdZByf40buEQfUYzapxyvAwfPV12AwfWRh4crg9jIVcY6 V79GO4Yb+ezclS4ZTvT+WZ9yLdwuWnzAGVTD0fd9RLvK03nk45ZK42LP MNSHwwUOjv338vqcubwqNOyjxpEukQF3TPXgKAV/ltpGzQYmnDofCd+S uLAssjpag59wPWruFItrIvE6qD7xaDXv+oVsO/bTp7pVb7NOi+KOCpMI D8aP4xm+624JWxLZ59YXOLOy3q1YVfLiVCe4ghtJS4/6BIuRhQ3CAOmj w4QfJVrTDnyn/RY3z41BnRT8K6CkUyuDc5Nc4NlU5KX3HxdiphW1w6JM oWNrPQ==
;; Received 1169 bytes from 192.203.230.10#53(e.root-servers.net) in 170 ms

//第二次迭代 次级域名查询（查询TLD）
iqiyi.com.		172800	IN	NS	ns1.iqiyi.com.
iqiyi.com.		172800	IN	NS	ns2.iqiyi.com.
iqiyi.com.		172800	IN	NS	ns3.iqiyi.com.
iqiyi.com.		172800	IN	NS	ns4.iqiyi.com.
CK0POJMG874LJREF7EFN8430QVIT8BSM.com. 86400 IN NSEC3 1 1 0 - CK0Q1GIN43N1ARRC9OSM6QPQR81H5M9A  NS SOA RRSIG DNSKEY NSEC3PARAM
CK0POJMG874LJREF7EFN8430QVIT8BSM.com. 86400 IN RRSIG NSEC3 8 2 86400 20200102054825 20191226043825 12163 com. J8V3FpilA7JdIt7GBym3CCORYjgGlHAazZlLNBiJ0bFa92n4PrX0hPYo oUHtAA4lEaw9eSJjOIVXhnKq9AR7EgQFfMxcT8OvbBVJ4eErF1vBjd1B x4EkZM2IHIVPPv8XlziufAhiSVMnYHcZnuO8BpDaXrasvlW3U9vv/VQU dCs79XwjQR/XkFvJKvldj2EZd3FXLlRDdnwESxhlpLZmIg==
CDJHMJ049AHN95A56GE5FPTIT6CK3TVA.com. 86400 IN NSEC3 1 1 0 - CDJIFERNDE19197KLS7DLE5N5008MQB2  NS DS RRSIG
CDJHMJ049AHN95A56GE5FPTIT6CK3TVA.com. 86400 IN RRSIG NSEC3 8 2 86400 20200103053214 20191227042214 12163 com. LQOPbhYOUqtzkl49A0Sg7IVpJ8HVep5FwE5ILJ0cK/5uGsvKk1bbrM4A s0M5iiaVnQ0BwTt9FRNdYRGUVUL6YSJATaouDomYj3o/h+0kGxvrJxnF jRuDsQS56c6LDALzd+2Hv4xwiOURhv0Nl4v3rycokglC6IjN1VpGrgWN bldR9ixluPAQsBo+m3TdieQyb4zc10Ks3BAJg4UKmNQSLg==
;; Received 723 bytes from 192.26.92.30#53(c.gtld-servers.net) in 171 ms

//第三次迭代  主机域名查询 （查询权威域名服务器）
iqiyi.com.		600	IN	A	101.227.188.172
iqiyi.com.		600	IN	A	101.227.188.174
iqiyi.com.		600	IN	A	101.227.188.176
iqiyi.com.		600	IN	A	101.227.188.178
iqiyi.com.		600	IN	A	101.227.188.170
;; Received 118 bytes from 43.225.84.1#53(ns3.iqiyi.com) in 28 ms
```
根据每次迭代返回结果的服务器信息，我们可以发现DNS查询整体流程是这样的：

{% asset_img DNS_Query_Flow.png This is an image %}

- 首次查询 从本地服务器（local dns server）获取根域名服务器地址，然后由**本地服务器**开始迭代dns查询过程。
- 第一次迭代 由本地服务器访问根域名服务器，根域名服务器返回关于顶级域名`.com`的NS记录。
- 第二次迭代 由本地服务器访问上一步获得的`.com`地址的顶级域名服务器，并从中获得关于`iqiyi.com`的NS记录。
- 第三次迭代 由本地服务器访问由上一步获得的`iqiyi.com`NS的服务器地址，即`iqiyi.com`的权威服务器，然后获得iqiyi.com的A记录。
- 最终，本地服务器返回关于`iqiyi.com`的IP地址给客户端，然后客户端再访问目标服务器。

# 记录类型举例
1. SOA记录 `dig iqiyi.com soa`
```
; <<>> DiG 9.10.6 <<>> iqiyi.com soa
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 20850
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;iqiyi.com.			IN	SOA

;; ANSWER SECTION:
iqiyi.com.		86400	IN	SOA	ns1.iqiyi.com. dnsadmin.iqiyi.com. 2019122905 1800 600 1209600 600

;; Query time: 33 msec
;; SERVER: 192.168.3.1#53(192.168.3.1)
;; WHEN: Sun Dec 29 22:46:03 CST 2019
;; MSG SIZE  rcvd: 87
```
2. NS记录 `dig iqiyi.com ns`
```
; <<>> DiG 9.10.6 <<>> iqiyi.com ns
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 21342
;; flags: qr rd ra; QUERY: 1, ANSWER: 4, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;iqiyi.com.			IN	NS

;; ANSWER SECTION:
iqiyi.com.		1132	IN	NS	ns2.iqiyi.com.
iqiyi.com.		1132	IN	NS	ns3.iqiyi.com.
iqiyi.com.		1132	IN	NS	ns4.iqiyi.com.
iqiyi.com.		1132	IN	NS	ns1.iqiyi.com.

;; Query time: 11 msec
;; SERVER: 192.168.3.1#53(192.168.3.1)
;; WHEN: Sun Dec 29 22:47:10 CST 2019
;; MSG SIZE  rcvd: 110
```
3. ns3.iqiyi.com的A记录 `dig ns3.iqiyi.com`
```
; <<>> DiG 9.10.6 <<>> ns3.iqiyi.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 11375
;; flags: qr rd ra ad; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;ns3.iqiyi.com.			IN	A

;; ANSWER SECTION:
ns3.iqiyi.com.		296	IN	A	43.225.84.1

;; Query time: 51 msec
;; SERVER: 192.168.3.1#53(192.168.3.1)
;; WHEN: Sun Dec 29 22:49:07 CST 2019
;; MSG SIZE  rcvd: 47
```
4. MX记录 `dig iqiyi.com mx`
```
; <<>> DiG 9.10.6 <<>> iqiyi.com mx
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 16303
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;iqiyi.com.			IN	MX

;; ANSWER SECTION:
iqiyi.com.		300	IN	MX	10 mx1.iqiyi.com.

;; Query time: 9 msec
;; SERVER: 192.168.3.1#53(192.168.3.1)
;; WHEN: Sun Dec 29 22:50:26 CST 2019
;; MSG SIZE  rcvd: 47
```
5. TXT记录 `dig iqiyi.com txt`
```
; <<>> DiG 9.10.6 <<>> iqiyi.com txt
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 28756
;; flags: qr rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;iqiyi.com.			IN	TXT

;; ANSWER SECTION:
iqiyi.com.		600	IN	TXT	"92qpmvb8qgzqbvndvbgtrn9nlys14bxh"
iqiyi.com.		600	IN	TXT	"v=spf1 ip4:202.108.14.100 a mx ~all"

;; Query time: 64 msec
;; SERVER: 192.168.3.1#53(192.168.3.1)
;; WHEN: Sun Dec 29 22:51:45 CST 2019
;; MSG SIZE  rcvd: 131
```
6. SRV、PTR
SRV与PRT不常用，所以iqiyi.com的域名服务器并没有配置这两项。

# 参考
[DNS 原理入门](https://www.ruanyifeng.com/blog/2016/06/dns.html)