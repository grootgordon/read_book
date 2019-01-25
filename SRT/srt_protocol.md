# SRT协议
srt是基于UDT传输协议，是用户级别的协议，其保留UDT的核心思想和机制，但是做了多项改进，包括控制报文的修改，针对直播流改进了流控，改进了拥塞算法，报文加密算法。本文介绍srt协议本身。更多的相关实现在：https://github.com/Haivision/srt

## 简介
srt传输协议为不可靠网络提供安全，可靠的数据传输，如因特网。任何数据都可以在srt协议上传输，特别是对音视频流数据优化最为明显。<br/>
在任何时候，srt都能用于视频流的汇聚/分发节点，提供几乎最好的质量和最低延时的视频。<br/>
当报文作为流在源与目的之间传输，srt对网络进行检测和适应。srt帮助补偿jitter和带宽抖动，针对噪声网络中的拥塞。其错误恢复机制对因特网的丢包影响最小化。srt也支持aes加密来保证端到端的安全。<br/>
SRT是基于UDT协议的(UDP-based data transfer)。虽然UDT是设计用来在公网中高吞吐的报文传输，UDT对视频直播并没有优势。SRT是一个大的改进UDT版本，其对视频直播有大的提升。<br/>
SRT在IP网络中的低延时视频传输，是以mpeg-ts格式作为UDP的单播/多播。其方案对网络提供可靠性，通过使能FEC来减轻任何丢包。在城市间，国家间设置跨洋间的低延时通信都带有更多的挑战。虽然用卫星通信，或MPLS专网来实现，但是价格太贵。公用因特网的连接，虽然便宜很多，但是需要大量的带宽来实现丢包恢复。<br/>
即使UDT并不是为直播流而设计，但是其丢包恢复机制能提供基本的丢包恢复功能。SRT的最早版本包括新报文的重传功能，能对直播流的丢包做快速响应。<br/>
为了达到低延时，SRT不得不引入分时机制。一个流在因特网中传输，很多特效会被完全影响，包括延时/jitter/丢包。进而导致解码问题，音视频解码器不能解码对应时间戳上的未收到的报文。如果应用buffer缓存来避免，但是却会带来延时。<br>

![常规网络情况](https://github.com/runner365/read_book/blob/srt/SRT/pic/net_condition01.png)

SRT的机制在接收方新创建了重要的特性，极大的降低buffer的需要。这些机制是SRT协议自身的一部分，所以一旦报文从SRT的一端发到接收端，流自身状态已经被恢复成流本身的状态。<br/>

![srt网络情况](https://github.com/runner365/read_book/blob/srt/SRT/pic/net_condition02.png)

最初SRT协议又Haivision Systems公司开发，在2017年4月Wowza Media Systems将其开源。开源的SRT遵守MPL-2.0开源协议。选用MPL-2.0协议，因为想在对开源SRT的兼容性，和估计开源社区去改进SRT协议之间做好平衡。任何第三方开发者都能自由的使用SRT开源代码。但是如果他们修改和优化代码，就必须把这些优化代码提交到开源社区。<br/>
在2017年4月，Haivision和Wowza公司成立了SRT联盟(www.srtalliance.org)，致力于持续发展该协议。

## SRT的UDT4适配
UDT是一种ARQ(自动重传请求)协议。其应用的是ARQ的第三种演进方案(选择性重传)。 UDT4的应用在ietf中提出，在draft-gg-udt-03中。<br/>
UDT致力于容量的最大使用，也就是当发送数据的时候，应用必须保证输入buffer是足够的。当以确定的比特率发送实时视频，包的发送速度对比读取文件的速度是非常慢的。buffer耗尽会导致一些UDT算法的复位。也就是说，当拥塞发生，发送方的算法会挂住UDP API，也会把报文放入丢弃队列，这样新的报文就不能被处理。实时的视频不能被挂住的，所以报文可能被应用丢弃(因为发送API会被block住)。不像UDT，SRT对实时报文，重传报文，丢失或要丢弃的旧豹纹都共享当前的带宽。<br/>
SRT的早期开发就引入了大量针对UDT version4的改进: <br/>
* 基于字节统计
* 基于毫秒单位来做延时控制(测量buffer的时间单位，更好的适配延时，使其不受流比特率的影响)
* ACK消息的统计信息(接受速率和估计连接容量)
* 控制报文时间戳(这个在UDT的draft中有，但是不在UDT4的应用中)
* 时间戳漂移更正算法
* 周期的NAK报告(UDT4去使能这个特性，用于unACKed报文的超时重传，其对实时流应用会消耗过多的带宽)
* 基于时间戳的报文发送(可配置的延时)
* SRT的握手是基于UDT自定义的控制报文，用于交换点到点间的配置和应用信息，以确保无缝升级，当升级协议和保证向后兼容
* 基于配置和输入速率的最大输出速率
* 加密优化
早期SRT的开发，是在内网用Haivison的Makito X系列的编解码器，其能模拟包的丢弃。<br/>
当中间报文的丢失，会导致解码器的接收buffer耗尽(没有报文送去解码)。丢失的报文不能及时的重传。解决方法是更早的触发未确认收到报文的重传(ARQ的自动重传)。然而，这会导致带宽使用的迸发。基于超时重传的发生是当报文没有很快的得到ack确认。在UDT4中，重传发生只是当丢失表空的时候。<br/>
重传所有ack超时的报文，能解决这样的实验场景，当无网络拥塞，有随机丢包影响报文传输。在多次重传后，所有的报文都应该能被发送，接收者的队列不会被挂住。但这会消耗大量的带宽，因为没有加入速率控制。

## 包结构
SRT保有UDT的UDP报文结构，但是有很多改进。基于UDT IETF internet Draft(draft-gg-udt-03.txt)的细节。

### 数据和控制报文
每个承载SRT的UDP报文都带有SRT头(其紧跟在UDP头后面)。在所有的协议版本中，SRT头包含4个32bits字段：
* PH_SEQNO
* PH_MSGNO
* PH_TIMESTAMP
* PH_ID
SRT有两类报文，PH_SEQNO字段的第一个bit用来标识报文类型，0是数据报文，1是控制报文。如下的例子中，就是数据报文的例子，其'packet type' bit=0：<br/>
注意：在SRT version 1.3中的报文结构变化。为了提高早期版本的适应能力，新旧报文格式都在下面列出(大字节序)。<br/>

<pic>

* FF=(2bits) 如报文中的位置
<pre>
  1) 10b = 1st
  2) 00b = middle
  3) 01b = last
  4) 11b = single
</pre>
* O = (1bit) 表示消息是否按序发送，按序1，不按序0。在文件/消息模式下(传统的UDT)，当该bit为0，那么后面的消息是可以立刻发送而不用等待前面消息的发送完成。但是这在直播业务中是没有用的，因为当TSBPD模式开启时，为数据抽取而服务器的功能完全不同。
* KK=(2bits)表示是否数据已经被加密：
<pre>
  1) 00b: 不加密
  2) 01b: 加密，偶数key
  3) 10b: 加密，奇数key
</pre>
* R=(1bit)重传报文。如果是0，表示该报文是第一次发送；如果是1，就是重传报文。
在数据报文中，第3，4字段如下的定义：
* Timestamp: 报文时间戳
* ID: 报文分发的目的socket id，如果是connect的发起报文，该字段就是0

更多数据报文的细节在本文后面介绍。 <br/>
<br/>

SRT协议控制报文头("packet type" bit=1)，其结构如下(未包含udp头)：

<pic>

对于控制报文，头两个字段分别解释如下：
* 头32bit：
<pre>
  1) bit0: 类型，1就是控制报文
  2) bits1-15: 消息类型
  3) bits16-31: 消息扩展类型
  -----------------------------------------------------------------------
  | type | Extended Type | description                                  |
  -----------------------------------------------------------------------
  |  0   |        0      | handshake                                    |
  -----------------------------------------------------------------------
  |  1   |        0      | keepalive                                    |
  -----------------------------------------------------------------------
  |  2   |        0      | ack                                          |
  -----------------------------------------------------------------------
  |  3   |        0      | nak(loss report)                             |
  -----------------------------------------------------------------------
  |  4   |        0      | congestion warning                           |
  -----------------------------------------------------------------------
  |  5   |        0      | shutdown                                     |
  -----------------------------------------------------------------------
  |  6   |        0      | ackack                                       |
  -----------------------------------------------------------------------
  |  7   |        0      | drop request                                 | 
  -----------------------------------------------------------------------
  |  8   |        0      | Peer Error                                   |
  -----------------------------------------------------------------------
  |0x7fff|        -      | Message Extension                            |
  -----------------------------------------------------------------------
  |0x7fff|        1      | SRT_HSREQ:SRT handleshake request            |
  -----------------------------------------------------------------------
  |0x7fff|        2      | SRT_HSRSP:SRT handleshake response           |
  -----------------------------------------------------------------------
  |0x7fff|        3      | SRT_KMREQ:Encryption Keying Material Request |
  -----------------------------------------------------------------------
  |0x7fff|        4      | SRT_KMRSP:Encryption Keying Material response|
  -----------------------------------------------------------------------
<pre/>
扩展消息机制是为未来的扩展，SRT可能因为某些原因今后会用到。后面的SRT扩展握手中会提及。
* 第二个32bits：
<pre>
  1) Additional info -- 其在控制消息中被用作扩展空间字段。它的解析依赖于特殊消息类型，握手消息不使用它。
</pre>

### Handshake报文
Handshake控制报文('packet type' bit=1) 是用来在两点之间建立连接的。早期的SRT用handshake来交换参数，在连接建立之后，但是1.3版本吧交换参数作为handshake的自身的一部分。后面的Handshake一节专门用来解释。<br/>

<pic...>

### KM 错误反馈报文
Key Messge Error Response控制报文('packet type' bit=1)是用来交换错误状态消息。在加密一节中详细介绍。<br/>

<pic...>

### ACK报文
ACK控制报文('packet type' bit=1) 是用来提供报文发送状态和RTT信息的。在SRT数据传输和控制一节中详细介绍。<br/>

<pic...>

### Keep-alive报文
Keep-alive报文('packet type' bit=1) 是用来每10ms交换信息，来保证SRT流在连接断开后字段重连的。

<pic...>

### NAK控制报文
NAK控制报文('packet type' bit=1) 是用来报告失败的报文传输。在SRT数据传输和控制一节中详细介绍。

<pic...>

### SHUTDOWN控制报文
shutdown控制报文('packet type' bit=1) 用来发器关闭SRT连接。

<pic...>

### ACKACK控制报文
ACKACK控制报文('packet type' bit=1) 用来回复收到ACK报文，并且可以用来计算RTT。在SRT数据传输和控制一节中介绍。

<pic...>

### 扩展控制报文
扩展控制报文('packet type' bit=1) 用来为原始UDT用户控制消息。它们被用在SRT扩展握手报文中，可以通过独立的消息，或内嵌在HANDSHAKE中。

## SRT数据交互
下表描述数据交互(包括控制数据)。注意，两点间角色的变换。举例，在会话过程中节点可以作为发起者，和监听者，然后也能成为发送和接受者在数据传输过程中。<br/>


