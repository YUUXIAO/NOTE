## TCP

> TCP全名传输控制协议，在OSI模型中属于传输层协议

TCP使用三次握手协议建立连接，它是通过序列号、确认应答、重发控制、连接管理以及窗口控制等机制实现可靠性传输的；

许多应用层协议基于TCP协议构建，典型的是HTTP、SMTP（发邮件）、IMAP（收邮件）等协议；

### 粘包

应用层传到 TCP 协议的数据，不是以 消息报为单位 向目的主机发送的，而是以字节流（二进制数据）的方式发送到下游，这些01 数据串其实是没有边界的概念的，所以数据可能被切割和组装成各种数据包，接收端收到这些数据包不知道哪到哪是一条完整的数据，还原不了原来的消息，产生粘包现象

所以纯裸TCP是不可以直接拿来用的，需要在这个基础上加上一些自定义规则，用来区分消息边界，所以要把每条要发送的数据进行包装，比如加入消息头，里面写清楚一个完整的包长度是多少，根据这个长度可以继续接收数据，截取出的就是我们要传输的消息体

#### 处理粘包

可以在 tcp的消息头里面加上：

- **加入特殊标志：** 一般是通过特殊的标志作为头尾，比如当收到了 xxx 就认为受到了新消息的起始头，直到收到了下一个头标志 xxx 才认为是一个完整的消息，类似 http 协议里的 chunked 编码
- **加入消息长度信息：**这个一般配合上面的特殊标志一起使用，在收到头标志时，里面还可以带上消息长度，以此表明在这之后多少 byte 都是属于这个消息的。如果在这之后正好有符合长度的 byte，则取走，作为一个完整消息给应用层使用。类似 HTTP 中的 Content-Length ，当接收端收到的消息长度小于 Content-Length 时，说明还有些消息没收到。那接收端会一直等，直到拿够了消息或超时

### 重传机制

> TCP 实现可靠传输的方式之一：是通过序列号与确认应答；

在 TCP 中，当发送端的数据到达接收主机时，接收端主机会返回一个确认应答消息，表示已收到消息， TCP 针对数据包丢失的情况，会用重传机制；

#### 超时重传

> 超时重传机制是以时间为驱动；

1. 每当遇到一次超时重传的时候，都会将下一次超时时间间隔设为先前值的两倍，两次超时，就说明网络环境差，不宜频繁反复发送；
2. 超时触发重传存在的问题是，超时周期可能相对较长；

#### 快速重传

> 快速重传机制，它不以时间为驱动，而是以数据驱动重传；

快速重传的工作方式是：当收到N个相同的 ACK 报文时，会在定时器过期之前，重传丢失的报文段；它只解决了一个问题，就是超时时间；

依然面临着另外一个问题，就是重传的时候，是重传之前的一个，还是重传所有的问题；

在下图，发送方发出了 1，2，3，4，5 份数据：

1. 第一份 Seq1 先送到了，于是就 Ack 回 2；
2. 结果 Seq2 因为某些原因没收到，Seq3 到达了，于是还是 Ack 回 2；
3. 后面的 Seq4 和 Seq5 都到了，但还是 Ack 回 2，因为 Seq2 还是没有收到；
4. 发送端收到了三个 Ack = 2 的确认，知道了 Seq2 还没有收到，就会在定时器过期之前，重传丢失的 Seq2；
5. 最后，接收到收到了 Seq2，此时因为 Seq3，Seq4，Seq5 都收到了，于是 Ack 回 6

### SACK选项

SACK（selective acknowledgment，选择性确认）是一种 TCP 拓展选项，**用来改进 TCP 在丢包时的传输效率**，传统的 TCP 在发生数据丢包的时，会触发拥塞控制，降低发送速率，然后等待确认

SACK选项允许接收端向发送端报告已经成功接收的数据，以及缺失的数据段，从而发送端可以更精确地重传丢失的数据段，而不必等待整个数据流程重新传输。

### D-SACK

Duplicate SACK 又称 D-SACK，使用 SACK 可以用来判断更精细的网络状况：比如数据被复制、错误重传等

#### ACK丢包

1. 「接收方」发给「发送方」的两个 ACK 确认应答都丢失了，发送方超时后重传第一个数据包（3000 ~ 3499）；
2. 「接收方」发现数据是重复收到的，于是回了一个 SACK = 3000~~3~~500，告诉「发送方」 30003500 的数据早已被接收了，因为 ACK 都到了 4000 了，已经意味着 4000 之前的所有数据都已收到，所以这个 SACK 就代表着 D-SACK；
3. 「发送方」就知道数据没有丢，是「接收方」的 ACK 确认报文丢了

#### 网络延时

1. 数据包（1000~1499） 被网络延迟了，导致「发送方」没有收到 Ack 1500 的确认报文；
2. 后面报文「接收方」收到了三个相同的 ACK 确认报文，就触发了快速重传机制，但是在重传后，被延迟的数据包（1000~1499）又到了「接收方」；
3. 「接收方」回了一个 SACK=1000~1500，因为 ACK 已经到了 3000，所以这个 SACK 是 D-SACK，表示收到了重复的包；
4. 发送方就知道快速重传触发的原因是因为网络延迟了；

#### 优势

1. 可以让「发送方」知道，是发出去的包丢了，还是接收方回应的 ACK 包丢了；
2. 可以知道是不是「发送方」的数据包被网络延迟了；
3. 可以知道网络中是不是把「发送方」的数据包给复制了；

## 滑动窗口协议

> 滑动窗口就是一种流量控制技术；

滑动窗口协议主要为了解决在网络传输数据的过程中，发送方和接收方传输数据速率不一致的问题，从而保证数据传输的可靠性，达到流量控制的效果；

其本质上是描述接收方的 TCP 数据报缓冲区大小的数据，发送方根据这个数据来计算自己最多能发送多长的数据；

- 如果发送方收到接收方的窗口大小为 0 的TCP数据报，那么将停止发送数据；
- 等到接收方发送窗口大小不为 0 的数据报的到来；

### 窗口

> 窗口大小就是指无需等待确认应答，而可以继续发送数据的最大值；

窗口的实现实际上是操作系统开辟的一个缓存空间，发送方主机在等到确认应答返回之前，必须在缓冲区中保留已发送的数据，如果按期收到确认应答，此时数据就可以从缓存区清除；

假设窗口大小为 3 个 TCP 段，那么发送方就可以「连续发送」 3 个 TCP 段，并且中途若有 ACK 丢失，可以通过「下一个确认应答进行确认」；

### 窗口大小

1. 通常窗口的大小是由接收方的决定的；
2. TCP 头里有一个字段叫 Window，也就是窗口大小；
3. Window 字段是接收端告诉发送端自己还有多少缓冲区可以接收数据，于是发送端就可以根据这个接收端的处理能力来发送数据，而不会导致接收端处理不过来；
4. 发送方发送的数据大小不能超过接收方的窗口大小，否则接收方就无法正常接收到数据；

### 发送方的滑动窗口

1. #1 是已发送并收到 ACK 确认的数据：1~31 字节；
2. #2 是已发送但未收到 ACK 确认的数据：32~45 字节；
3. #3 是未发送但总大小在接收方处理范围内（接收方还有空间）：46~51 字；
4. #4 是未发送但总大小超过接收方处理范围（接收方没有空间）：52 字节以后



当发送方把数据「全部」都一下发送出去后，可用窗口的大小就为 0 了，表明可用窗口耗尽，在没收到 ACK 确认之前是无法继续发送数据了；![img](https://user-gold-cdn.xitu.io/2020/7/25/17384f119ca2a493?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

当收到之前发送的数据 32~~36 字节的 ACK 确认应答后，如果发送窗口的大小没有变化，则滑动窗口往右边移动 5 个字节，因为有 5 个字节的数据被应答确认，接下来 52~~56 字节又变成了可用窗口，那么后续也就可以发送 52~56这 5 个字节的数据了；

![img](https://user-gold-cdn.xitu.io/2020/7/25/17384f11a9324c17?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

### 接收方的滑动窗口

1. #1 + #2 是已成功接收并确认的数据（等待应用进程读取）；
2. #3 是未收到数据但可以接收的数据；
3. #4 未收到数据并不可以接收的数据



其中三个接收部分，使用两个指针进行划分:

1. RCV.WND：表示接收窗口的大小，它会通告给发送方；
2. RCV.NXT：是一个指针，它指向期望从发送方发送来的下一个数据字节的序列号；
3. 指向 #4 的第一个字节是个相对指针，它需要 RCV.NXT 指针加上RCV.WND 大小的偏移量，就可以指向 #4 的第一个字节了；

## 流量控制

TCP 提供一种机制可以让「发送方」根据「接收方」的实际接收能力控制发送的数据量；

### 死锁

1. 当接收方向发送方发送零窗口报文段后不久，接收端的接收缓存又有了一些存储空间；
2. 于是接收端向发送端发送了 Windows size = 2 的报文段；
3. 然而这个报文段在传输过程中丢失了；
4. 发送端一直等待收到接收端发送的非零窗口的通知，而接收端一直等待发送端发送数据，这样就死锁了；

解决方法：

TCP 为每个连接设有一个持续计时器，只要 TCP 连接的一方收到对方的零窗口通知，就启动持续计时器，若持续计时器设置的时间到期，就发送一个零窗口探测报文段，而对方就在确认这个探测报文段时给出了现在的窗口值；

### Nagle算法

> Nagle 算法主要为了解决TCP的传输效率问题，目的就是为了避免发送小的数据包

不过这个也算是古早年代的事了，现在的网络比之前好太多，Nagle 的优化帮助没有那么大，而且它的延迟发送，有时候也会导致调用延时变大，所以现在一般也会把它关掉

在 Nagle 算法开启的情况下，数据包在以下两个情况会被发送：

1. **长度达到：**如果包长度达到 MSS （或者含有 Fin 包），立刻发送，否则等待下一个包到来，如果下一包到来之后两个包的总长度超过 MSS 的话，就会进行拆分发送
2. **时间达到：**在等待超时（一般 200ms）的情况下，第一个包没到 mss 长度，但是又迟迟等不到第二个包，则立即发送

   ![图片](https://mmbiz.qpic.cn/mmbiz_png/FmVWPHrDdnlnuLFPj8NI57ZdUQiaibNt61dyXtL6zcgK4YAnuum27gKlobMGiawm6MtdYQKzyAPVCrYX4Cv3fmXPA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

3. 若要把发送的数据逐个字节缓存起来，则发送方需要把第一个字节发送出去，然后缓存后面的字节；
4. 在收到接收方第一个字节的确认，再将现有缓存中所有字节组成一个报文段发送出去，继续缓存后续数据；
5. 只有在收到前一个报文的确认之后发送后面的数据，这是为了减少所用带宽；
6. 当发送数据到达TCP发送窗口的一半或已达到报文段的最大长度也会立即发送报文段，而不是等待接收方确认，这是为了提高网络吞吐量；

### 糊涂窗口综合征

TCP接收方的缓存已满，若上层一次从缓存中读取一个字节，这样接收方就可以继续接纳一个字节的窗口，然后向发送方发送确认，把窗口设为1个字节，这样持续下去，网络效率会非常低；

有效的解决方法，就是让接收方等待一定时间，让缓存空间能够接纳一个最长的报文段，或者等待接收缓存已有一半的空闲空间，再发出确认报文和通知当前窗口大小；

## UDP

> UDP全名用户数据协议，在OSI模型中属于传输层协议；

一句话就是UDP没TCP可靠，就是只负责发数据，不负责对方有没有收到，一般用来处理广播数据的场景

### 与TCP的区别

1. UDP不是面向连接的，在UDP中，一个套接字可以与多个UDP服务通信；TCP中连接一旦建立，所有的会话都基于连接完成，客户端如果要与另一个TCP服务连接，需要另建一个socket来完成；
2. UDP虽然提供面向事务的不可靠信息传输服务，在网络差的情况下存在丢包严重的问题，但是它无须连接，消耗资源低，处理快速且灵活，所以常应用在那种偶尔丢一两个数据包也不会有重大影响的场景，比如音视频、直播等；
3. [DNS服务](DNS.md)是基于UDP实现的；