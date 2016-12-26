SYN-Flood简易实现和原理解析
===========================

## Prepare Reading
### Relevant Reading
 - IP RFC: [https://tools.ietf.org/html/rfc791](https://tools.ietf.org/html/rfc791)
 - TCP RFC: [https://tools.ietf.org/html/rfc793](https://tools.ietf.org/html/rfc793)
 - TCP state machine: [http://tcpipguide.com/free/t_TCPOperationalOverviewandtheTCPFiniteStateMachineF-2.htm](http://tcpipguide.com/free/t_TCPOperationalOverviewandtheTCPFiniteStateMachineF-2.htm)

### Why am I doing this?

&emsp;&emsp;最近在重读UNP(Unix Network Programming)，发现以前不懂的太多，只能一味的接受书上所述，未能提出过疑问，甚至有些东西都看不大懂。重读之后，发现忽略掉的细节有点太多，最先发现的就是`listen(sockfd, backlog)`里的`backlog`参数是一个有限而且很小的`kernel`维护的一个队列，很容易就被耗尽了，这就给了不怀好意的人可乘之机，即所谓的`SYN FLOOD`，正文就是浅说`SYN FLOOD`的简易实现和原理解析。<br>

## SYN FLOOD原理解析
&emsp;&emsp;`SYN FLOOD`就是通过半连接去耗尽目标主机、目标端口的`kernel`资源，导致目标主机拒绝所有来自目标端口的合法请求，即所谓的`Dely Of Service`(DOS攻击)，如果通过一个集群攻击同一个目标主机，即`Distributed Dely Of Service`(DDOS攻击)。<br>
&emsp;&emsp;这里就不提老生常谈的`TCP`三次握手，四次握手，以及`TCP`状态转换图了，这块知识生疏了可以去翻翻`Prepare Reading`里的内容。
&emsp;&emsp;当`client`发送完一个`SYN`请求，`server`会回一个`SYN/ACK`，然后等待下一个`ACK`，那么此次连接就算成功建立了。那么，如果我们不给`server`回这个`ACK`呢，那么`server`的TCP状态会一直维持在`SYN_RCVD`状态，并触发超时重传，一定次数后回到`CLOSED`状态，但是每个`socket`的半连接资源是有限的，这样就会耗尽某个服务的半连接资源，导致拒绝服务攻击。<br>
&emsp;&emsp;&emsp;<img src="https://github.com/linghuazaii/blog/blob/master/image/syn-flood/backlog.png"><br>
&emsp;&emsp;&emsp;<img src="https://github.com/linghuazaii/blog/blob/master/image/syn-flood/ip_packat_sample.png"><br>
&emsp;&emsp;非常抱歉，我忘记截`TCP`包的图了，大伙儿自己用`wireshark`或者`tcpdump`抓一个看看就知道啦。<br>  

## SYN FLOOD简易实现  
&emsp;&emsp;首先我们肯定是不能走正常的`TCP`系统调用的，因为这样会触发`TCP`状态转换的正常流程，即使你设置了一个`NONBLOCKING`的`socket`去`connect``server`，虽然`connect`发送完`SYN`后立即返回，然后你可以关闭该`socket`不发最后一个`ACK`，但是当`server`的`SYN/ACK`过来时，`kernel`发现此端口并未打开，会直接发送`RST`给`server`，导致`server`关闭`TCP`连接，并不能产生一个半连接。<br>
&emsp;&emsp;那么正确的做法是什么呢？<br>
&emsp;&emsp;通过`raw socket`，`raw socket`可以直接绕过`TCP`发送各种`TCP`包，`raw socket`存在的目的本来是为测试用的，所以你必须有`root`权限才能使用。怎么才能有`root`权限呢？其一，`root`设置了某个应用的`SUID`位，类似于`passwd`；其二，你在`sudo`组里面；其三，你就是`root`。<br>
&emsp;&emsp;下面看一下`IP`和`TCP`的header，摘自`RFC`:<br>
&emsp;&emsp;&emsp;<img src="https://github.com/linghuazaii/blog/blob/master/image/syn-flood/ip_header.png"><br>
&emsp;&emsp;&emsp;<img src="https://github.com/linghuazaii/blog/blob/master/image/syn-flood/tcp_header.png"><br>  
&emsp;&emsp;具体字段的具体含义我就不一一解释了，具体可以去翻`RFC`，因为我也是这么过来的。<br>
&emsp;&emsp;下面说一下代码里面需要注意的地方：<br>
```
int syn_socket = socket(AF_INET, SOCK_RAW, IPPROTO_RAW);
rc = setsockopt(syn_socket, IPPROTO_IP, IP_HDRINCL, &on, sizeof(on));
```
&emsp;&emsp;一定要设置`IP_HDRINCL`使用我们自己的`IP header`，这样才能造成一个FLOOD。为什么呢？因为我们必须随机`源IP地址`,让目标主机将`SYN/ACK`发往各个不同的`源IP地址`，如果路由是通的，那么`源IP地址`主机将会发一个`RST`导致连接关闭，不通则会导致多次超时重传，目标主机维护一堆没用的半连接，服务资源耗尽，目的达到。<br>
```
syn_header.ip_header.check = 0;
syn_header.ip_header.check = checksum((uint16_t *)&syn_header.ip_header, sizeof(syn_head    er.ip_header));

syn_header.tcp_header.th_sum = 0;
syn_header.tcp_header.th_sum = tcp4_checksum(syn_header.ip_header, syn_header.tcp_header
```
&emsp;&emsp;一定要将`iphdr.check`和`tcphdr.th_sum`赋`0`，然后再计算`checksum`,`RFC`里有提到这一点，所以我们也这么实现。
```
syn_header.ip_header.tot_len = htons(IP_HDRLEN + TCP_HDRLEN);

syn_header.tcp_header.th_sport = htons((uint16_t)config.local_port);
syn_header.tcp_header.th_dport = htons((uint16_t)config.remote_port);
```
&emsp;&emsp;设置`header`的时候多字节成员一定要进行网络地址转换`htons()`、`htonl()`之类。  

## Talk Is Cheap, Show You My Code!
&emsp;&emsp;Github地址：[https://github.com/linghuazaii/syn-flood-implement](https://github.com/linghuazaii/syn-flood-implement)<br>
&emsp;&emsp;刚开始犯了一个很愚蠢的错误：为什么我发完`SYN`也收到`SYN/ACK`了，`kernel`却发送了`RST`？当时是本机测本机，原因是：逗逼楼主，`raw socket`又不会触发`TCP`状态机，一个`CLOSED`状态的`socket`收到`SYN/ACK`，你还指望我给你回一个`ACK`，呵呵，`RST`死你~<br>
&emsp;&emsp;&emsp;<img src="https://github.com/linghuazaii/blog/blob/master/image/syn-flood/syn-packet.png"><br>
&emsp;&emsp;但是我也并没能突破`kernel`的保护机制，`kernel`为每个`socket`维护的半连接队列增加到一定程度就不再自动增加了，到这里，学习目的就达到啦，有闲心可以翻翻`kernel`源码，应该可以很容易突破`kernel`的保护机制，道高一尺魔高一丈嘛。<br>
&emsp;&emsp;&emsp;<img src="https://github.com/linghuazaii/blog/blob/master/image/syn-flood/half-connection.png"><br>

## 闲话
 - `HTTPS`出现的一个原因是因为可以监控网络上的`TCP`包，窃取`SYN Sequence`值和其他信息来造成中间人攻击，所以`HTTPS`出现了，最主要的目的是提供认证。
 - `raw socket`还可以用来对四次握手制造`SYN FLOOD`。
 - `raw socket`可以通过向指定端口发送`SYN`包并检查是否收到`SYN/ACK`来实现隐藏的端口扫描，并且不会被目标主机记录进日志文件。
 - `raw socket`可以用来搞事啊~
 - 暂时就写这么多吧。  

## 结语
&emsp;&emsp;**Merry Christmas！！！**
