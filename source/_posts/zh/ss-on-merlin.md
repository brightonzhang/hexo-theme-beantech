---
title: Linux网络小实践
catalog: true
date: 2019-01-15 11:39:19
subtitle: SS on Merlin
header-img: "/img/2019/3180568124.jpg"
tags:
- Linux 
- Network 
---

# 准备

## 网络模型

说网络总是绕不开网络模型，所以先祭出此表。

| OSI七层模型                    | TCP/IP模型 | 相关通信协议与标准              |
| ------------------------------ | ---------- | ------------------------------- |
| 应用层<br />表现层<br />会话层 | 应用层     | HTTP，FTP，SMTP，POP3，NFS，SSH |
| 传送层                         | 传送层     | TCP UDP                         |
| 网络层                         | 网络层     | IP，ICMP                        |
| 数据链路层<br />物理层         | 链路层     |                                 |



## IPv4与IPv6

<p>IPv4地址一共32位，用点分十进制表示，每一个部分是8位。子网掩码有两种表示：<br />
1) ip: 192.168.1.3; mask:255.255.255.0，ip&mask得到子网，表示ip的前24位是网络位，后8位是主机位，也就是前24位相同的ip地址是同一个子网的。<br />
2) CIDR标记法: 192.168.1.3/24，直接用位数来表示子网，意义是ip 192.168.1.3的32位地址中的前24位表示网络位，后8位表示主机位，IP前24位相同，表示是同一个子网的。</p>
IPv6地址一共128位，用十六进制表示，中间用“:”隔开，每一部分是16位。如：21DA:D3:0:2F3B:2AA:FF:FE28:9C5A。IPv6可以将每4个十六进制数字中的前导零位去除做简化表示，但每个分组必须至少保留一位数字。IPv6还可以将冒号十六进制格式中相邻的连续零位进行零压缩，用双冒号“::”表示。 如多点传送地址FF02:0:0:0:0:0:0:2压缩后，可表示为FF02::2。在一个特定的地址中，零压缩只能使用一次，否则我们就无法知道每个“::”所代表的确切零位数了。 

IPv6 前缀：前缀是地址中具有固定值的位数部分或表示网络标识的位数部分。IPv6的子网标识、路由器和地址范围前缀表示法与IPv4采用的CIDR标记法相同，其前缀可书写为：地址/前缀长度。例如21DA:D3::/48是一个路由器前缀，而21DA:D3:0:2F3B::/64是一个子网前缀。 ::1/128，即0:0:0:0:0:0:0:1，等于::1，回环地址，相当于ipv4中的localhost（127.0.0.1），ping locahost可得到此地址。

## 路由与route命令

路由（routing）是通过互联的网络把信息从源地址传输到目的地址的活动。路由发生在OSI网络参考模型中的第三层即网络层。路由引导分组转送，经过一些中间的节点后，到它们最后的目的地。路由通常根据路由表——一个存储到各个目的地的最佳路径的表——来引导分组转送。
Linux上可以通过route命令来查看管理路由表。

```shell
brighton@WINTERFELL:/tmp/home/root# route 
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
192.168.1.1     *               255.255.255.255 UH    0      0        0 eth0
192.168.50.0    *               255.255.255.0   U     0      0        0 br0
192.168.1.0     *               255.255.255.0   U     0      0        0 eth0
127.0.0.0       *               255.0.0.0       U     0      0        0 lo
default         192.168.1.1     0.0.0.0         UG    0      0        0 eth0
```




## 防火墙与iptables

防火墙通过订定一些有顺序的规则，管制进入到网络的数据封包。Linux防火墙主要有封包过滤型的 Netfilter 与依据服务软件程序作为分析的 TCP Wrappers 两种。

- Netfilter (封包过滤机制)，所谓的封包过滤，亦即是分析进入主机的网络封包，将封包的表头数据捉出来进行分析，以决定该联机为放行或抵挡的机制。 由于这种方式可以直接分析封包表头数据，所以包括硬件地址(MAC), 软件地址 (IP), TCP, UDP, ICMP 等封包的信息都可以进行过滤分析的功能，因此用途非常的广泛。(其实主要分析的是 OSI 七层协议的 2, 3, 4 层啦)。在 Linux 上面我们使用核心内建的 Netfilter 这个机制，而 Netfilter 提供了 iptables 这个软件来作为防火墙封包过滤的指令。
- TCP Wrappers (程序控管)，另一种抵挡封包进入的方法，为透过服务器程序的外挂 (tcpd) 来处置的。与封包过滤不同的是， 这种机制主要是分析谁对某程序进行存取，然后透过规则去分析该服务器程序谁能够联机、谁不能联机。 由于主要是透过分析服务器程序来控管，因此与启动的端口无关，只与程序的名称有关。 举例来说，我们知道 FTP 可以启动在非正规的 port 21 进行监听，当你透过 Linux 内建的 TCP wrappers 限制 FTP 时， 那么你只要知道 FTP 的软件名称 (vsftpd) ，然后对他作限制，则不管 FTP 启动在哪个埠口，都会被该规则管理的。

### iptables

iptables实现防火墙功能的原理是：在数据包经过内核的过程中有五处关键地方，分别是PREROUTING、INPUT、OUTPUT、FORWARD、POSTROUTING，iptables可以在这5处地方写规则，对经过的数据包进行处理，规则一般的定义为“如果数据包头符合这样的条件，就这样处理数据包”。 

**规则表**                                                                                 

* filter表：filter表用来对数据包进行过滤，根据具体的规则要就决定如何处理一个数据包。对应内核模块：iptable_fileter。共包含三个链。

* nat表：nat(Network Address Translation,网络地址转换)表主要用于修改数据包 ip地址，端口号等信息。对应的内核模块为iptable_nat，共包含三个链。

* mangle表：mangle表用来修改数据包的TOS(Type Of Service,服务类型)，TTL（Time To Live，生存周期）值，或者为数据包设置Mark标记，以实现流量整形，策略路由等高级应用。对应的内核模块iptable_mangle，共包含五个链。

* raw表：raw表示自1.2.9以后版本的iptables新增的表，主要来决定是否对数据包进行状态跟踪。对应的内核模块为iptable_raw,共包含两个链。

**规则链**                                                                                            

- INPUT链：当收到访问防火墙本机地址的数据包(入站)时，应用此链中的规则。
- OUTPUT链：当防火墙本机向外发送数据包(出站)时，应用此链中的规则。
- FORWARD链:当接收到需要通过防火墙中转发送给其他地址的数据包(转发)是，应用测链中的规则。
- PREROUTING链：在对数据包做路由选择之前，应用测链中的规则。
- POSTROUTING链：在对数据包做路由选择之后，应用此链中的规则。

**数据包过滤工作流程**                                                                          

```
-> PREROUTING -> Routing Decision -> FORWARD -> POSTROUTING ->
                         |                           ^
                         v                           |
                       INPUT ->------------------->OUTPUT
```

规则表应用优先级：raw→mangle→nat→filter
各条规则的应用顺序：链内部的过滤遵循“匹配即停止”的原则，如果对比完整个链也没有找到和数据包匹配的规则，则会按照链的默认策略进行处理。

-  入站数据流向：数据包到达防火墙后首先被PREROUTING链处理(是否修改数据包地址等)，然后进行路由选择（判断数据包发往何处），如果数据包的目标地址是防火墙本机（如：Internet用户访问网关的Web服务端口），那么内核将其传递给INPUT链进行处理(决定是否允许通过等)。
- 转发数据流向：来自外界的数据包到达防火墙后首先被PREROUTTING链处理，然后再进行路由选择；如果数据包的目标地址是其他的外部地址，则内核将其传递给FORWARD链进行处理(允许转发，拦截，丢弃)，最后交给POSTROUTING链(是否修改数据包的地址等)进行处理。
- 出站数据流向：防火墙本机向外部地址发送的数据包(如在防火墙主机中测试公网DNS服务时)，首先被OUTPUT链处理，然后进行路由选择，再交给POSTROUTING链(是否修改数据包的地址等)进行处理。

基本语法：

```
iptables [-t 表] [操作命令] [链] [规则匹配器] [-j 目标动作]
iptables [-t filter] [-AI INPUT,OUTPUT,FORWARD] [-io interface] [-p tcp,udp.icmp,all] [-s ip/nerwork] [–sport ports] [-d ip/network] [–dport ports] [-j ACCEPT DROP REJECT REDIRECT MASQUERADE LOG DNAT SNAT MIRROR QUEUE RETURN MARK]
常用操作命令： 
-A 在指定链尾部添加规则 
-D 删除匹配的规则 
-R 替换匹配的规则 
-I 在指定位置插入规则
-L/S 列出指定链或所有链的规则 
-F 删除指定链或所有链的规则 
-N 创建用户自定义链
-X 删除指定的用户自定义链 
-P 为指定链设置默认规则策略，对自定义链不起作用
-Z 将指定链或所有链的计数器清零 
-E 更改自定义链

常见规则匹配器说明：
-p tcp|udp|icmp|all 匹配协议，all会匹配所有协议 
-s addr[/mask] 匹配源地址 
-d addr[/mask] 匹配目标地址 
–sport port1[:port2] 匹配源端口(可指定连续的端口） 
–dport port1[:port2] 匹配目的端口(可指定连续的端口） 
-o interface 匹配出口网卡，只适用FORWARD、POSTROUTING、OUTPUT。
-i interface 匹配入口网卡，只使用PREROUTING、INPUT、FORWARD。 

目标动作说明：
ACCEPT 允许数据包通过 
DROP 丢弃数据包 
REJECT 丢弃数据包，并且将拒绝信息发送给发送方
```

### ipset

ipset是iptables的扩展,它允许创建匹配整个地址集合的规则。普通的iptables链只能单IP匹配, 进行规则匹配时，是从规则列表中从头到尾一条一条进行匹配，这像是在链表中搜索指定节点费力。ipset 提供了把这个 O(n) 的操作变成 O(1) 的方法：就是把要处理的 IP 放进一个集合，对这个集合设置一条 iptables 规则。像 iptable 一样，IP sets 是 Linux 内核提供，ipset 这个命令是对它进行操作的一个工具。
另外ipset的一个优势是集合可以动态的修改，即使ipset的iptables规则目前已经启动，新加的入ipset的ip也生效。

ipset命令简介：

```
ipset create blacklist hash:ip
# 创建名为blacklist的集合，以 hash 方式存储，存储内容是 IP 地址。

iptables -I INPUT -m set --match-set blacklist src -j DROP
# 如果源地址(src)属于blacklist这个集合，就进行 DROP 操作。这条命令中，blacklist是作为黑名单的，如果要把某个集合作为白名单，添加一个 ‘!’ 符号就可以。
iptables -I INPUT -m set ! --match-set whitelist src -j DROP

ipset add blacklist 192.168.50.131
# 将192.168.50.131加入blacklist集合
```



## 代理

### 正向代理

### 透明代理



## 常用工具与命令

### ping

###traceroute

这两个指令可以透过 ICMP 封包的辅助来确认与回报网络主机的状态。

通过traceroute我们可以知道信息从你的计算机到互联网另一端的主机是走的什么路径。

Traceroute程序的设计是利用ICMP及IP header的TTL（Time To Live）栏位（field）。首先，traceroute送出一个TTL是1的IP datagram（其实，每次送出的为3个40字节的包，包括源地址，目的地址和包发出的时间标签）到目的地，当路径上的第一个路由器（router）收到这个datagram时，它将TTL减1。此时，TTL变为0了，所以该路由器会将此datagram丢掉，并送回一个「ICMP time exceeded」消息（包括发IP包的源地址，IP包的所有内容及路由器的IP地址），traceroute 收到这个消息后，便知道这个路由器存在于这个路径上，接着traceroute 再送出另一个TTL是2 的datagram，发现第2 个路由器...... traceroute 每次将送出的datagram的TTL 加1来发现另一个路由器，这个重复的动作一直持续到某个datagram 抵达目的地。当datagram到达目的地后，该主机并不会送回ICMP time exceeded消息，因为它已是目的地了，那么traceroute如何得知目的地到达了呢？

Traceroute在送出UDP datagrams到目的地时，它所选择送达的port number 是一个一般应用程序都不会用的号码（30000 以上），所以当此UDP datagram 到达目的地后该主机会送回一个「ICMP port unreachable」的消息，而当traceroute 收到这个消息时，便知道目的地已经到达了。所以traceroute 在Server端也是没有所谓的Daemon 程式。

Traceroute提取发 ICMP TTL到期消息设备的IP地址并作域名解析。每次 ，Traceroute都打印出一系列数据,包括所经过的路由设备的域名及 IP地址,三个包每次来回所花时间。



