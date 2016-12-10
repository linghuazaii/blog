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

