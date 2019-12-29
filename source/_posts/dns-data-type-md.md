---
title: DNS数据与类型
date: 2019-12-29 23:08:30
categories: 
    - 网络
    - DNS
---
DNS是WWW万维网中重要的一环，内部涉及到多种数据类型，dns的数据称为记录（record），平时我们涉及到最多的可能只有IP解析服务的A记录，但深入了解下去，发现DNS有多种用于不同用途的数据类型，常见的主要有：
- A (Host address)
- AAAA (IPv6 host address)
- CNAME (Canonical name for an alias)
- MX (Mail eXchange)
- NS (Name Server)
- PTR (Pointer)
- SOA (Start Of Authority)
- SRV (location of service)
- TXT (Descriptive text)

更多的记录类型可以参考：https://simpledns.plus/help/dns-record-types

# Zone文件
要想清楚明白记录类型，就不得不去深入了解Zone文件。DNS服务器是采用Zone文件来进行数据管理的，每个Zone相当于一个独立的管理单元，一个DNS可以管理多个zone文件，一个zone文件也可以被多个单独的dns服务器管理（如主、从、缓存服务器）。

## zone文件结构
一个域名对应着一个zong文件，以abc.com为例，zone文件如下：
```
$TTL 6h //第1行
$ORIGIN abc.com. //第2行
@ 3600 IN SOA ns1.ddd.com. root.ddd.com.( //第3行
    929142851 ; Serial //第4行
    1800 ; Refresh //第5行
    600 ; Retry //第6行
    2w ; Expire //第7行
    300 ; Minimum //第8行
    ) 
@ 2d IN NS ns1.ddd.com. //第9行
@ 2d IN NS ns2.ddd.com. //第10行
@ 2d IN NS ns3.ddd.com. //第11行
ns1 3600 IN A 120.172.234.27 //第12行
ns2 3600 IN A 120.172.234.28 //第13行
ns3 3600 IN A 120.172.234.29 //第14行
a 3600 IN A 120.172.234.27 //第15行
b 3600 IN CNAME a.abc.com. //第16行
@ 3600 IN MX a.abc.com. //第17行
@ 3600 IN TXT "TXT" //第18行
```

## 文件解释
- 第1行，这行内容给出了该域名(`abc.com.`)各种记录的默认TTL值，这里为6小时。即如果该域名的记录没有特别定义TTL，则默认TTL为有效值。
- 第2行，这行内容标识出该ZONE文件是隶属那个域名的，这里为`abc.com.`。
- 第3行，从这行开始到第8行为该域名的SOA记录部分，这里的@代表域名本身。ns1.ddd.com表示该域名的主权威DNS。root.ddd.com表示该主权威DNS管理员邮箱，等价于root@ddd.com。
- 第4行，Serial部分，这部分用来标记ZONE文件更新，如果发生更新则Serial要单增，否则MASTER不会通知SLAVE进行更新。
- 第5行，Refresh部分，这个标记SLAVE服务器多长时间主动(忽略MASTER的更新通知)向MASTER复核Serial是否有变，如有变则更新之。
- 第6行，Retry部分，如Refresh过程不能完成，重试的时间间隔。
- 第7行，Expire部分，如SLAVE无法与MASTER取得联系，SLAVE继续提供DNS服务的时间，这里为2W(两周时间)。Expire时间到期后SLAVE仍然无法联系MASTER则停止工作，拒绝继续提供服务。Expire的实际意义在于它决定了MASTER服务器的最长下线时间(如MASTER迁移，DOWN机等)。
- 第8行，Minimum部分，这个部分定义了DNS对否定回答(NXDOMAIN即访问的记录在权威DNS上不存在)的缓存时间。
- 第9-11行，定义了该域名的3个权威DNS服务器。NS记录表明要想知道该域名的ip解析，就要向该地址的服务器请求访问，这里的域名是`abc.com.`，@表示本域名。那NS服务的具体地址是什么呢，由对应的ns域名的A记录来指定，如第12-14行。通常NS记录的TTL大些为宜，这里为2天。设置过小只会增加服务器无谓的负担，同时解析稳定性会受影响。
- 第15-18行是常用的几个记录类型，详细请参考下一节。

# 数据类型
## SOA记录
一个zone文件的第一个记录（Start Of Authority），记录了整个zone文件（权威域）的全局配置，如主域名、管理员邮箱、重试时间、刷新时间等等。
## A、AAAA记录
DNS中最常用的记录类型，用于IP解析功能，A记录用于IPv4的解析，AAAA用于IPv6的解析。
## NS记录
NS（NameServer）域名服务记录，是除A记录外必需的记录，用于记录指定域名的服务器解析地址。

## CNAME记录
CNAME相关于别名功能，对于多个不同的域名，采用同一个CNAME来方便配置。
```
www IN CNAME name.abc.com.
web IN CNAME name.abc.com.
home IN CNAME name.abc.com.
name IN A 110.10.1.2
```
例如三个不同的域名www.abc.com、web.abc.com、home.abc.com可以用同一个CNAME name.abc.com来表示，方便配置与修改。

## MX记录
MX（Mail Exchange）邮箱服务器地址，用于记录邮件地址对应的服务器地址，如mailname@abc.com的邮箱地址，邮件系统会根据abc.com域名的MX记录来找到指定的邮箱服务器地址。

## PTR
PTR记录可以理解为是A记录的反解，A记录是根据域名来获取ip地址，而PTR记录则是根据ip可以反向查出ip地址对应的域名，例如给定的ip属于www.abc.com。
```
; 50 是 10.0.1.50  4个数字中的最后一个
; www.abc.com 必须是 FQDN

50 IN PTR www.abc.com.
```

## SRV
DNS SRV是DNS记录中一种，用来指定服务地址。与常见的A记录、cname不同的是，SRV中除了记录服务器的地址，还记录了服务的端口，并且可以设置每个服务地址的优先级和权重。访问服务的时候，本地的DNS resolver从DNS服务器查询到一个地址列表，根据优先级和权重，从中选取一个地址作为本次请求的目标地址。例如：
```
_ldap._tcp.example.com TTL Class SRV Priority Weight Port Target
Service: 服务名称，前缀“_”是为防止与DNS Label（普通域名）冲突。
Proto:   服务使用的通信协议，_TCP、_UDP、其它标准协议或者自定义的协议。
Name:    提供服务的域名。
TTL:     缓存有效时间。
CLASS:   类别
Priority: 该记录的优先级，数值越小表示优先级越高，范围0-65535。
Weight:   该记录的权重，数值越高权重越高，范围0-65535。     
Port:     服务端口号，0-65535。
Target:   host地址。
```
一个能够支持SRV的LDAP client可以通过查询域名，得知LDAP服务的IP地址和服务端口。

## TXT
TXT记录用于DNS一些扩展功能，最多可记录65536个字节。通常主要用于域名拥有权验证（如google-site-verification)，SPF反垃圾邮箱验证等等。后续会对TXT记录做详细的介绍。

# 参考文献
- [Red Hat -- DNS [Zone 文件]](https://www.jianshu.com/p/073c4f407395)
- [DNS扫盲系列之五：域名配置ZONE文件](https://www.cnblogs.com/niuchunjian/p/3485724.html)
- [SRV记录](https://www.lijiaocn.com/%E6%8A%80%E5%B7%A7/2017/03/06/dns-srv.html)
- https://simpledns.plus/help/dns-record-types