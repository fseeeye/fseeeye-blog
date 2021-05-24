---
title: 漫谈工控协议安全 - S7Comm
tags:
  - ICS
abstract: A brief introduction to ICS protocol security.
header_image: /intro/Floor-Kids-end.png
copyright: true
date: 2020-11-24 11:00:45
top:
---

本系列文章意在分享工控协议安全问题的思想，并不着重于协议的详细解析。有一下几点原因：
1. 分析协议各个字段的内容十分繁复，各个功能码、子功能码下对应的协议结构又不尽相同，最终文章只会变成长达几十页的手册。
2. 工控协议基本都是私有协议，或在公开的协议上实现的私有化，并不能保证对其字段含义解析得完全正确，且很多协议字段对安全研究毫无意义。
3. 目前已经有很多工控协议结构的分析文章，Wireshark也已经支持大部分协议。虽然知识零碎，但足以认识协议整体结构。

# The foregoing.
本篇漫谈，我们来讲一下S7Comm协议。它是西门子在1993年推出的私有协议，应用于S7-300/400系列及200系列PLC(略有不同)，监听于`102`端口。

它在OSI七层模型中是实现了应用层、表示层和会话层的协议。实际上更为准确的说，S7Comm建立在传输层的COTP协议(RFC905)之上，作为其Payload进行传输。S7Comm推出伊始，其采用的是ISO传输协议。为了能让协议适配TCP/IP（支持地址路由），并保留原先ISO的特性（面向流量包的数据格式），西门子选择在TCP之上模拟COTP服务，并用TPKT(RFC1006)来作为过渡，封装COTP。我们将采用这种独特结构的传输层协议称为ISO-on-TCP。当然，脱离TCP/IP的架构，S7Comm也是完全可以通信的，理论上它是属于COTP的载荷，只不过要使用特殊接口/设备。(比如S7-400系列的特殊通信处理器（CP 443）)

总结来说，S7Comm以太网传输时模型如下：

| OSI layer  | Protocol |
|  ----  | ----  |
| Application Layer  | S7 communiation |
| Presentation Layer  | S7 communiation |
| Session Layer | S7 communiation |
| Transport Layer | ISO-on-TCP + TPKT + COTP |
| Network Layer | IP |
| Data Link Layer | Ethernet |
| Physical Layer | Ethernet |

目前关于S7Comm的文章都将TPKT划归会话层，将COTP划归表示层，这是不对的。仔细想想，你会发现这样划分，它们各自都没有达到OSI对该层协议规定的作用。


# Protocol Breif Intro
下面简单介绍与S7Comm相关协议。同文首所述，我们并不会把精力放在协议字段解析上。

## TPKT
作为COTP和TCP之间的过渡，在TCP之上模拟COTP协议。
![TPKT.png](./TPKT-Wireshark.png)
大端序，长度为4bytes，`Length`字段表示包括TPKT自身的载荷长度，即TCP Payload长度。一般来说，TPKT监听在102端口（而不像同样采用了TPKT+COTP的RDP协议监听在3389），这就是S7Comm最终监听在102的原因。本协议没任何难点，且和S7Comm安全研究无关，一笔带过。

## COTP
一种可靠传输协议，和TCP相似，已经随着时代演进逐渐淡出人们视野。小端序，不定长，协议第二个字段为功能码，总共四个：
1. CR: `0x0e`，Connect Request，表示连接请求。
2. CC: `0x0d`，Connect Confirm，表示连接确认。
3. DT: `0x0f`，Data，表示数据传输，其后承载S7Comm。
4. DR: `0x08`，Disconnect Request，表示断开请求。一般用不上，而是发送TCP FIN。

你可能会发现数据包中存在dst/src reference字段，在发送`CR`/`CC`为功能码的数据包时，还会会出现src-tsap、dst-tsap、tpdu-size这些参数字段，tsap那些用来指定CPU机架和槽位，tpdu-size表示最大(?)tpdu长度。类似于TCP的端口，COTP采用reference+tsap来达到多路复用效果。
> TPDU: Transport Protocol Data Unit，传输协议数据单元，简单理解为COTP的载荷，在该上下文中即是S7Comm。
> TASP: Transport Service Access Point，类比为TCP中的端口即可。

![COTP.png](./COTP-Wireshark.png)

并不用太关心这类字段，只要把握好功能码，从而理解某COTP数据包的功能即可。

## S7Comm
S7Comm是极其繁复的协议，但我们可以抛开无用的东西，关注于关键字段。它大体上分为三部分：
1. Header: 包含功能码等基本信息。
2. Parameter: 功能码的参数，所以根据功能码不同，也随之不同。
3. [Data]: 若需要传输数据时存在。

![S7Comm-intro.png](./S7Comm-Wireshark.png)

Header中有几个关键参数：
* ROSCTR / PDU Type: 即功能码，常用功能码如下
  * `0x01`: Job，即由主设备(上位机)发送的作业请求，用于完成建立通讯(Set Communication)、读/写寄存器(Read/Write Var)、读/写块（即读/写工程，Read/Write Block）、启/停PLC等常见任务。
  * `0x02`: ACK，acknowledgement without additional field，类似TCP的ACK。
  * `0x03`: Ack_Data，acknowledgement with additional field，一般用于响应Job请求。
  * `0x07`: Userdata，协议扩展，参数字段包含请求/响应ID（用于编程/调试，读取SZL，安全功能，时间设置，循环数据...）。
* PDUR: Protocol Data Unit Reference，用于标识会话。请求方仅接收相同PDUR的响应包，来达到会话控制的目的。

暂时了解上述知识即可。


# Communication Process
S7Comm通讯前需要建立连接，流程如下图：
![s7comm-handshake-process.png](./s7comm-handshake-process.png)

简单总结，S7Comm通讯需要先握手三次，先后分别是TCP、COTP、S7Comm。随后才开始发送S7Comm的功能包。
通常来说，S7Comm的控制指令只需要进行单次通讯，并没有什么需要注意的地方，但多次通讯就涉及到上文所述的**PDUR修正**，比如Write Block通讯流程下：
![s7comm-write-block.png](./s7comm-write-block.png)

当我们作为上位机时，处理PLC发送来的请求需要注意修正PDUR，否则PLC将拒收并断开TCP连接。例如上图处理Downlaod Block时，就需要将响应包的PDUR改为和请求包一致。


# security Analysis
经过上文简单的讲述，相信你已经对S7Comm协议整体上有了概念。下面，我们可以从多个角度来看待这个协议的安全问题。

## Authentication
作为一个应用层协议，那么它最后服务的对象就是一个应用程序(Application)，在S7Comm的场景下，这个应用程序就是S7-300/400系列PLC。身份验证准确上来讲，理应由应用程序(PLC)实现，它本身不能称为协议问题，但既然S7Comm是和S7 PLC绑定的私有协议，我们也可以宽泛的把它作为漏洞的一部分。应该通过上文学习你可以认识到，上位机和PLC建立通讯以后，接收上位机请求并处理，并不存在验证请求来自可信对象，这就是S7-300/400系列PLC(甚至可以说目前大部分PLC)最严重的问题所在。
我们打个比方，PLC比作Web服务器，上位机比作Web浏览器，那么现在的情况就是，通过浏览器访问Web服务器后，管理员操作(上传下载/开关机)按钮直接显示在了网页上，直接点击执行就可以，那么这种应用有安全性可言吗？恶意攻击者通过自己的上位机软件即可控制PLC，甚至不需要上位机软件，直接发送/重放流量即可，这就是工控安全的由来和亟待解决的重大问题。
2011年Blackhat上的一篇[WP](https://media.blackhat.com/bh-us-11/Beresford/BH_US11_Beresford_S7_PLCs_WP.pdf)成为了工控安全的开端，时至今日，解析协议后组包攻击的手段层出不穷，可以说，由于缺乏身份认证，工控领域黑客已经完全接管了S7-300。

## Integrity
互联网早期时代，中间人攻击屡见不鲜，正是因为当年协议缺乏完整性保护，随后出现了数字签名等相关技术。回看S7Comm，同样却饭完整性保护，那么流量被劫持后修改传输数据，若改动长度再修正一下length，即可被PLC或上位机认可。有没有真实的案例呢？当年震网病毒(Stuxnet)想必部分读者还有印象，那么核电站设施超负荷运作/宕机没有被工程师站检测到的原因，便是中间人攻击修改Read SZL/Var/Block传输至上位机的数据为正常状态。可见，完整性保护也是一大痛点。

## Encryption
S7Comm作为不公开的私有协议，当年的工程师可能认为这已经对协议进行了加密，难以分析。实践证明，对S7Comm协议的研究已经让它和公开协议没有区别，形同明文传输。工程师们犯了密码学的一个认知错误：私有实现的加密算法(比喻可能不太恰当)并不能像理论验证安全的公开加密算法一样保证安全。公开的数据交换大大降低了黑客的研究成本，所以S7CommPlus的快速更迭也赶不上被攻破的速度。并且随着工控互联网的推进，流量传输在互联网上将会带来数据泄漏等等安全隐患。为什么Web应用推荐采用HTTPS作加密和完整性保护？这都是有历史经验的，而工控协议远远落后时代。

# Afterword.
S7Comm协议诞生于上世纪90年代，过去这么多年，西门子设备已经占领了工控行业半壁江山，加之工控设备更替很慢，那么遗留下来的隐患设备数不胜数，它们正运行在各国各行各业的生产线上。变革是工控行业时代发展的需求，而工控安全出现正是为了解决这些生根已久的隐患。

## Reference
* S7comm·Wiki·Wireshark: https://gitlab.com/wireshark/wireshark/-/wikis/S7comm
* COTP·Wiki·Wireshark: https://gitlab.com/wireshark/wireshark/-/wikis/COTP
* TPKT·Wiki·Wireshark: https://gitlab.com/wireshark/wireshark/-/wikis/TPKT
* IsoProtocolFamily·Wiki·Wireshark: https://gitlab.com/wireshark/wireshark/-/wikis/IsoProtocolFamily
* [MS-RDPBCGR] Client X.224 Connection Request PDU: https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-rdpbcgr/e78db616-689f-4b8a-8a99-525f7a433ee2?redirectedfrom=MSDN

## Recommended reading
* Exploiting Siemens Simatic S7 PLCs WP: https://media.blackhat.com/bh-us-11/Beresford/BH_US11_Beresford_S7_PLCs_WP.pdf
* Exploiting Siemens Simatic S7 PLCs PPT: https://media.blackhat.com/bh-us-11/Beresford/BH_US11_Beresford_S7_PLCs_Slides.pdf
* Internet-facing PLCs - A New Back Orifice WP: https://www.blackhat.com/docs/us-15/materials/us-15-Klick-Internet-Facing-PLCs-A-New-Back-Orifice-wp.pdf
* Internet-facing PLCs - A New Back Orifice PPT: https://www.blackhat.com/docs/us-15/materials/us-15-Klick-Internet-Facing-PLCs-A-New-Back-Orifice.pdf
* 西门子S7COMM协议READ SZL解析: http://blog.nsfocus.net/s7comm-readszl-0427/
