---
title: "TCP_IP详解"
date: 2014-08-22
draft: false
layout: posts
tags: ["TCP","IP","计算机网络"]
---

### OSI七层/四层模型对应关系
![image.png](https://raw.githubusercontent.com/Leowuqunqun/img/master/image202305271338611.png)



![image.png](https://raw.githubusercontent.com/Leowuqunqun/img/master/image202305271339332.png)数据链路层数



* 数据包（以太网数据包)格式，除了应用层没有头部，其他都有

![image.png](https://raw.githubusercontent.com/Leowuqunqun/img/master/image202305271339259.png)
### 数据包在传送时的封装和解封装如下所示
![image.png](https://raw.githubusercontent.com/Leowuqunqun/img/master/image202305271339427.png)
### 细说TCP
TCP是传输控制协议，是一种面向连接的、可靠的、基于字节流的传输层通信协议，
#### TCP功能
**TCP主要是将数据进行分段打包传输，对每个数据包编号控制顺序，运输中丢失、重发和丢弃处理。**

#### TCP拆包
mtu=最大传输单元（Maximum Transmission Unit）是指一种通信协议的某一层上面所能通过的**最大数据包大小**（以**字节**为单位）。
在我们常用的以太网v2中，MTU是1500字节。超过此大小的数据包就会将多余的部分拆分再单独传输 。

> 1.本地MTU值大于网络MTU值时，本地传输的数据包过大导致网络会拆包后传输，不但产生额外的数据包，而且消耗了“拆包、组包”的时间 。
2.本地MTU值小于网络MTU值时，本地传输的数据包可以直接传输，但是未能完全利用网络给予的数据包传输尺寸的上限值，传输能力未完全发挥 。
这样我们就知道：
所谓合理的设置MTU值，就是让本地的MTU值与网络的MTU值一致，既能完整发挥传输性能，又不让数据包拆分。

你可以使用ifconfig命令来查看mtu
![image.png](https://raw.githubusercontent.com/Leowuqunqun/img/master/image202305271339222.png)
![image.png](https://raw.githubusercontent.com/Leowuqunqun/img/master/image202305271340935.png)

#### TCP头
![image.png](https://raw.githubusercontent.com/Leowuqunqun/img/master/image202305271340725.png)

- Source Port & Destination Port - 源端口号和目标端口号;计算机通过端口号识别访问哪个服务,比如http服务或ftp服务;发送方端口号是进行随机端口;目标端口号决定了接收方哪个程序来接收。
- Sequence number - 32位序列号，TCP用序列号对数据包进行标记，以便在到达目的地后重新重装。在建立连接时通常由计算机生成一个随机数作为序列号的初始值。
- Acknowledgment number - 32位确认号，确认应答号。发送端接收到这个确认应答后，可以认为这个位置以前所有的数据都已被正常接收。
- Header Length - 首部长度。单位是 '4'个'字节'，如果没有可选字段，那么这里的值就是 5。表示 TCP 首部的长度为 20 字节。
- checksum - 16位校验和。用来做差错控制，TCP校验和的计算包括TCP首部、数据和其它填充字节。
- flags - 控制位。TCP的连接、传输和断开都受这六个控制位的指挥
- window size - 本地可接收数据的数目，这个值的大小是可变的。当网络通畅时将这个窗口值变大加快传输速度，当网络不稳定时减少这个值可以保证网络数据的可靠传输。它是来在TCP传输中进行流量控制的
### TCP 三握四挥

![image.png](https://raw.githubusercontent.com/Leowuqunqun/img/master/image202305271341241.png)

介绍一下这三个部分。

![image.png](https://raw.githubusercontent.com/Leowuqunqun/img/master/image202305271341957.png)

我们把这个过程分为三部分，第一部分为三次握手建立连接，第二部分为数据传输，第三次为四次挥手断开连接。
#### 三次握手
为了方便描述我们将主动发起请求的172.16.17.94:8080 主机称为客户端，将返回数据的主机172.16.17.94:8080称为服务器，以下也是。

- 第一次握手: 建立连接。客户端发送连接请求，发送SYN报文，将seq设置为0。然后，客户端进入SYN_SEND状态，等待服务器的确认。
- 第二次握手: 服务器收到客户端的SYN报文段。需要对这个SYN报文段进行确认，发送ACK报文，将ack设置为1。同时，自己还要发送SYN请求信息，将seq为0。服务器端将上述所有信息一并发送给客户端，此时服务器进入SYN_RECV状态。
- 第三次握手: 客户端收到服务器的ACK和SYN报文后，进行确认，然后将ack设置为1，seq设置为1，向服务器发送ACK报文段，这个报文段发送完毕以后，客户端和服务器端都进入ESTABLISHED状态，完成TCP三次握手。

**注意：客户端发送完第三次握手包后，不再需要服务端的确认，立即可以发送数据。**
**如果第三次握手包服务器没有收到，就直接发送数据，服务器将这个携带应用数据的包当做第三次握手（前提是这一个包中携带有ACK标记）。而后面再发送的那个名义上的第三次握手包，也就是图中黑色的那一行，被当作了重复发送的无效包，被忽略掉了，对通信没有造成影响。**
![image.png](https://raw.githubusercontent.com/Leowuqunqun/img/master/image202305271341949.png)

#### 数据传输

- 客户端先向服务器发送数据，该数据报是lenth为159的数据。
- 服务器收到报文后, 也向客户端发送了一个数据进行确认（ACK），并且返回客户端要请求的数据，数据的长度为111，将seq设置为1，ack设置为160（1 + 159）。
- 客户端收到服务器返回的数据后进行确认（ACK），将seq设置为160， ack设置为112（1 + 111）。![image.png](https://raw.githubusercontent.com/Leowuqunqun/img/master/image202305271341166.png)

#### 四次挥手
当客户端和服务器通过三次握手建立了TCP连接以后，当数据传送完毕，就要断开TCP连接了，就有了神秘的“四次挥手”。

- 第一次挥手：客户端向服务器发送一个FIN报文段，将设置seq为160和ack为112，;此时，客户端进入 **FIN_WAIT_1**状态,这表示客户端没有数据要发送服务器了，请求关闭连接;
- 第二次挥手：服务器收到了客户端发送的FIN报文段，向客户端回一个ACK报文段，ack设置为1，seq设置为112;服务器进入了**CLOSE_WAIT**状态，客户端收到服务器返回的ACK报文后，进入**FIN_WAIT_2**状态;
- 第三次挥手：服务器会观察自己是否还有数据没有发送给客户端，如果有，先把数据发送给客户端，再发送FIN报文；如果没有，那么服务器直接发送FIN报文给客户端。请求关闭连接，同时服务器进入**LAST_ACK**状态;
- 第四次挥手：客户端收到服务器发送的FIN报文段，向服务器发送ACK报文段，将seq设置为161，将ack设置为113，然后客户端进入**TIME_WAIT**状态;服务器收到客户端的ACK报文段以后，就关闭连接;此时，客户端等待2MSL后依然没有收到回复，则证明Server端已正常关闭，客户端也可以关闭连接了。

注意：在握手和挥手时确认号应该是对方序列号加1,传输数据时则是对方序列号加上对方携带应用层数据的长度。
![image.png](https://raw.githubusercontent.com/Leowuqunqun/img/master/image202305271341127.png)

