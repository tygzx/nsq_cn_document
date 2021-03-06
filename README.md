### <center> nsq 中文文档</center>
#### 写这篇文档的初衷
本人就是一个小码农,写这篇文档的初衷就是很简单的想了解下nsq。
***
* [Quick Start](#0)
* [FEATURES & GUARANTEES](#1)
* [FAQ](#2)
* [PERFORMANCE](#3)
* [Design](#4)
<h3 id ="0">Quick Start</h3>
跟着下面这些步骤将会在你自己的机器上运行一个小的NSQ集群。消息会被发送,消费,然后存在本地的磁盘上。

1.根据[安装文档](https://nsq.io/deployment/installing.html)上的教程来安装nsq

2 打开一个shell 下,开启nsqlookupd
>
    nsqlookupd

3 打开另外一个shell,开启nsqd
>
    nsqd --lookupd-tcp-address=127.0.0.1:4160

4 在打开一个shell ,开启nsqadmin
>
    nsqadmin --lookupd-http-address=127.0.0.1:4161

5 发送你的第一个nsq 消息(需要现在先在集群里面创建topic)
>
    curl -d 'hello world 1' 'http://127.0.0.1:4151/pub?topic=test'

6 最后在另外一个shell,开启nsq_to_file
>
    nsq_to_file --topic=test --output-dir=/tmp --lookupd-http-address=127.0.0.1:4161
7 发送更多的消息到nsqd
>
    curl -d 'hello world 2' 'http://127.0.0.1:4151/pub?topic=test'
    curl -d 'hello world 3' 'http://127.0.0.1:4151/pub?topic=test'
8 为了确认nsq 是否正常运行,可以在浏览器中打开`http://127.0.0.1:4171/ `看nsqadmin的ui和看到统计。也可以通过查看写到/tmp 文件夹下的log 文件的内容

Tips: 在本章中值得注意的是nsq_to_file 命令不是直接告诉产生topic test 的。它是直接从nsqlookupd 收到消息的，因为连接的原因，所以没有消息丢失？(这段话存疑,后面可能需要修改)

***

<h3 id ="1">FEATURES & GUARANTEES</h3>
NSQ 是一个实时的发布消息的平台


FEATURES
- 支持
- 动态伸缩(没有broker,无缝添加更多的节点到集群)
- 低风险的消息分发？
- combination load-balanced and multicast style message routing
- 
- 主要是基于内存的(除了a high-water mark messages 被保存在磁盘上)
- 消费者运行发现服务来寻找生产者
- TLS
- 自定义数据格式?
- 依赖少(容易部署)配置少
- 使用tcp 协议的客户端能够被任何语言支持
- 有管理生产者的http 界面
- 
- 健壮性好的集群管理界面


Guarantees

任何一个分布式系统,你的的目标都是让交换更加智能。我们想要得到这样的一个预期:NSQ 在真实的生产环境中将会怎样表现

- 消息默认不是可持久的
    - 虽然系统支持通过--mem-queue-size 选项来让消息保存在磁盘上,但是nsq 主要还是一个基于内存的系统。
    通过将--mem-queue-size 设置为0来确保所有进来的消息保存在磁盘上。在这种情况下,如果一个节点挂掉了，你很容易再现失败的场景。
    There is no built in replication. However, there are a variety of ways this tradeoff is managed such as deployment topology and techniques which actively slave and persist topics to disk in a fault-tolerant fashion.
- 消息至少被分发一次
    - 在上面的情况下,假设被给的nsqd 节点没有失败情况下，这意味着因为很多的原因,消息能够被分发多次(client 超时,断开连接,重新加入队列等)It is the client’s responsibility to perform idempotent operations or de-dupe.
- 消息被接收是无序的
    - 你不能相信消费者接收消息的顺序。和消息分发政策相似。
    this is the result of requeues,因为基于内存和磁盘的两种存储方式结合起来。和这个nsqd 节点不分享任何事情。
    It is relatively straightforward to achieve loose ordering (i.e. for a given consumer its messages are ordered but not across the cluster as a whole) by introducing a window of latency in your consumer to accept messages and order them before processing (although, in order to preserve this invariant one must drop messages falling outside that window).
- 消费者会寻找所有的生产者
    - 发现服务(nsqlookupd)是 designed to be eventually consistent。nsqlookupd 节点不调整和保存任何查询的状态和答案的。Network partitions do not affect availability in the sense that both sides of the partition can still answer queries. Deployment topology has the most significant effect of mitigating these types of issues.

***

FAQ
- 部署
    - nsqd 推荐的部署结构是什么？
        - 我们强烈推荐每运行一个生产者服务就在增加一个nsqd服务。nsqd 是一个相对轻量级的占用大量内存的程序。
        This pattern aids in structuring message flow as a consumption problem rather than a production one.另外一个优点是他本质是形成了一个独立的,可分享的topic数据仓库在被给的主机地址上。
        这个要求不是绝对的。只是比较简单。
    - 为什么nsqlookupd 不能被生产者用来发现消息将被发送到哪里。
        - nsq 提升了消费者一端的发现模型通过减轻配置负担
    但是他没有对解决一个server 应该发布到哪儿没有任何意义。这是一个鸡和蛋的问题.这个不存在任何的优势对于发布问题
    By co-locating nsqd (see question above), you sidestep this problem entirely (your service simply publishes to the local nsqd) and allow NSQ’s runtime discovery system to work naturally.
    - 我只是想把nsqd 作为一个工作队列在一个单节点中。这种情况是否适合。
        - 是的。nsqd 能单独运行的非常好。
        nsqlookupd在大的分布式环境中是有益的。
    - 我应该运行多少个nsqlookupd
        - 这个主要是看你集群的规模和,nsqd 节点的数量和消费者的数量还有你想到达到的默认的容错率
        一般来说,对于数百个消费者和host节点。3到5个就可以工作的很好了。nsqlookup 不要求和查询的回答同等。在集群的元数据是始终保持一致的。
- 发送
    - 我需要一个客户端用来发布信息吗
        - 只需要使用http 就ok 了。他是简单的,容易的。在大部分的程序环境中都可以使用。在事实上大部分的nsq 的部署节点都是使用http 部署的
    - 为什么强制客户端处理tcp 协议的pub 和mpub 协议
        - 我们认为nsq 的默认操作模式应该是安全优先的。我们想这个协议是简单的和一致的。
    - 什么时候一个 pub 和mpub 失败
        - 1:  这topic 的名字不是正常的形式。比如字符不对或者长度不对。可以看[topic and channel name spec](https://nsq.io/clients/tcp_protocol_spec.html#notes)
        - 2:消息太长了 (消息的长度可以作为nsqd 的一个参数)
        - 3:topic 在消息传送到一半的被删除掉了
        - 4:nsqd 在消息传递到一半的时候中途退出了
        - 5:客户端连接失败了在发布的时候
    - 我如何减少上述第三个场景的发生
        - 删除topic 是一个相对来说不常见的操作。如果你需要删除一个topic。orchestrate the timing such that publishes eliciting topic creations will never be performed until a sufficient amount of time has elapsed since deletion.
- 设计和理论
    - 推荐的topic 和channels 的命名
        - pass
    - 单个的nsqd 能够支持多少个topic 和channels 的数量上的限制呢。
        - 这个没有内部的限制数据。他只被主机上的cpu 的数量和内存大小限制
    - How are new topics announced to the cluster?
        - The first PUB or SUB to a topic will create the topic on an nsqd. Topic metadata will then propagate to the configured nsqlookupd. Other readers will discover this topic by periodically querying the nsqlookupd
    - nsq 能做rpc 吗？
        - 是的。他是可能的。但是他并不是为了这种情况而设计的。
        - 我们准备发布一些文档能够有顺序的到底而不是同时到达。如果你们有兴趣的话。
    - pynsq 特定的
        - 为什么你们强制我们使用tornado
            - pynsqd 原本就是设计用来作为一个专注于消费端上的一个包。而tornado 是python 里一个很简单的异步客户端的实现。
            - tornado 的api是简单的而且性能也表现的非常好。
        - tornado 的事件循环(IOLoop)对于publish 是必需的吗？
            - 不是的。nsqd 暴露出http端(/pub和/mpub)for dead-simple programming language agnostic publishing.
            如果你担心使用http 的过量损耗,没有必要。/mpub 会自动批量发送。
        - 什么时候我将会使用writer 
            - 当一个高性能和低损耗是必要的时候
            - writer 使用tcp 协议的pub 和mpub 命令。使用tcp 协议将会比http 协议有更低的损耗
        - 当我只是想要fire 和forget(我能够容忍一些消息丢失)
            - 使用wirter ，但是不指定一个publish 方法的callback
***

<h3 id ="0">PERFORMANCE</h3>

- 分布式性能
    - 代码的主仓库里包含了一个脚本(bench/bench.py).这个脚本能够自动执行一个并发测试.他包含了n个节点和一些运行的nsqd节点和一些运行的and some running load-generating utilities (PUB and SUB),然后解析他们的输出到一个汇总里
- 设置
    - The following runs reflect the default parameters of 6 c3.2xlarge, the cheapest instance type that supports 1gbit links. 3 nodes run an nsqd instance and the rest run instances of bench_reader (SUB) and bench_writer (PUB), to generate load depending on the benchmark mode.
    ````shell
        ./bench/bench.py --access-key=... --secret-key=... --ssh-key-name=..
    ````
- 生产者吞吐量
    - 下面的这个并发测试只测量生产者容量。不需要其他额外的加载。这个消息的size 是100字节然后消息被分发到超过3个topic
    > 
        ./bench/bench.py --access-key=... --secret-key=... --ssh-key-name=... --mode=pub --msg-size=100 run
    An ingress of ~2.07mm msgs/sec, 消费者大概197MB/S的宽带
- 生产者和消费者的吞吐量
    - 下面的这个并发测试更加真实的反应了真实世界中生产者和消费者的条件。同样的消息的大小是100byte.然后消息被发送到超过3个topic 里。each with a single channel (24 clients per channel).
    >
        ./bench/bench.py --access-key=... --secret-key=... --ssh-key-name=... --msg-size=100 run
    - At an ingress of ~842k and egress of ~806k msgs/s, consuming an aggregate 156mb/s of bandwidth, we’re now maxing out the CPU capacity on the nsqd nodes. By introducing consumers, nsqd needs to maintain per-channel in-flight accounting so the load is naturally higher.消费者的数量是明显的比生产者的数量少的。因为使用者的命令是生产者的2倍Adding another 2 nodes (one nsqd and one load-generating) attains over 1mm msgs/s:
    >
        ./bench/bench.py --access-key=... --secret-key=... --ssh-key-name=... --msg-size=100 run
- 单个节点的性能
    - 免责声明:请记住nsq 的设计是为了用在分布式的潮流上的。单个节点的性能是重要的。但是这个不是我们想要实现的。同时。虽然并发测试是一件很愚蠢的事情，but here’s a few anyway to ignite the flame:

    - GOMAXPROCS=1 (single publisher, single consumer)
    >
        ./bench.sh 
    - GOMAXPROCS=4 (4 publishers, 4 consumers)
    >
        ./bench.sh 

<h3 id ="4">Design</h3>
如果你想看类似于ppt的请点击这个链接 [slide deck](https://speakerdeck.com/snakes/nsq-nyc-golang-meetup).

nsq 是在simplequeue 基础上的项目。设计这个项目的目的主要是:
- 为了支持高可用性和低消耗的架构
- 满足更加强大的消息分发的需求
- 限制单个进程的内存使用量(将一些消息存到磁盘上)
- 消费者和生产者配置简单
- provide a straightforward upgrade path
- 性能提升

- 简单的配置和管理者
    - 单个nsqd实例是被设计用来即时处理多个流。流也被成为主题。每个主题有1到多个管道。每个管道receives a copy of all the messages for a topic. 在实践中,每个管道都对应着一个消费者服务。
    - 主题和管道都不是事先配置好的。当第一次向主题发送消息时,主题将会自动创建当订阅某个主题的管道的时候管道将自动创建。
    - 主题和管道都有自己独立的数据缓冲区防止一个很慢的消费者造成的对于其他管道的积压
    - 每个管道都有可能有多个客户端连接。假设所以连接到同一个管道的客户端都在准备收到消息的状态,每个消息都会随机分发到一个客户端。比如看[图](https://f.cloud.github.com/assets/187441/1700696/f1434dc8-6029-11e3-8a66-18ca4ea10aca.gif)。
    - 总结一下就是消息是一个消息会从一个主题中传送到多个管道中
    (这就是每个管道收到主题的复制消息)。但是从管道到客户端是随机挑选的。
    - nsq 也包括一个一个帮助的应用nsqlookupd:消费者能通过nsqlooked 找到单个nsqd 的实例的地址In terms of configuration, this decouples the consumers from the producers (they both individually only need to know where to contact common instances of nsqlookupd, never each other), reducing complexity and maintenance
    - 每个nsqd有一个和nsqlookupd的tcp长连接。over which it periodically pushes its state这个信息通常用来告知是哪个nsqd江将发送给
    nsqlookupd.然后发送给消费者。对于消费者来说。
    一个http 请求/lookup 端是暴露用来拉的。
    - 对于一个新的不同的主题的消费者。只需要简单的开启一个NSQ的客户端。在这个NSQ 客户端中简单的配置一下你的nsqlookupd实例的地址。然后其他的就无须改动了。极大的减少了服务器的开销和复杂度
    - 提示: 在将来版本的实验性的nsqlookupd 中将被用来返回能够被连接的客户端地址或者其他的智能策略。The current implementation is simply all. Ultimately, the goal is to ensure that all producers are being read from such that depth stays near zero.还有很重要的一点提示:nsqd 和nsqlookupd这两个守护进程是被用来设计独立操作的。不需要通过任何一个兄弟进程来进行通信
    - 我们认为有一种方式来查看,观测集群也是一件很重要的事情。因此我们内置了nsqadmin来做这件事情。它提供了一个网页界面来浏览topics/channels/consumers的统计和 inspect depth and other key statistics for each layer.此外它还支持一些管理命令。比如移除和置空一个管道(which is a useful tool when messages in a channel can be safely thrown away in order to bring depth back to 0).
- 直接的升级之路
    - 这是我们产品最大的优点。我们的产品系统处理大容量的交易事物。这些交易事物都建立在我们目前已经存在的消息工具上。因此我们需要一个办法来缓慢的有系统的来升级我们的基础设施的部分，在几乎没有任何影响的情况下。
    - 第一 在生产者一侧，我们内置了nsqd 来匹配simplequeue.很特别的，nsqd暴露了一个http /put 接口，和simplequeue 是一样的to POST binary data (with the one caveat that the endpoint takes an additional query parameter specifying the “topic”). Services that wanted to switch to start publishing to nsqd only have to make minor code changes.
    - 第二 我们内置了一些库在python 和go 之中This eased the transition on the message consumer side by limiting the code changes to bootstrapping. All business logic remained the same
    - 最后。我们我们内置了一些工具方法用来兼容老的和新的组件。他们都是有效的在我们的examples文件夹下
    - nsq_pubsub : 在一个nsq的集群中暴露一个pubsub的http 接口
    - nsq_to_file : 将一个topic 里的消息持久化到一个文件里
    - nsq_to_http: perform HTTP requests for all messages in a topic to (multiple) endpoints
- 消除SPOFs
    - nsq 是设计被用来分布式的.nsqd 的客户端和所有的实例都是通过tcp进行连接的。在客户端和实例之间没有任何中间人,没有任何消息中间件和spof
    ![image.png](https://upload-images.jianshu.io/upload_images/1250148-dca43ba92eee15f2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
    这种架构减少了单链条的的需要。让你能够直接从所有的生产者之间消费。理论上对应生产者来说只要有足够多的client 。你的消息就能就确保处理。
    对于nsqlookupd 来说。高的可用性可以通过多运行几个实例来保证。他们是不会互相交流的。数据始终是保持一致的。消费者能够拉在他们配置中的nsqdlookup的实例。然和将所有的消息合并在一起。即使节点出现问题也不会让系统产生停止
- 确保消息传送到达
    - nsq 确保一个消息至少被分发一次。复制消息是可能出现的、
    Consumers should expect this and de-dupe or perform idempotent operations.
    - 这种确保是被强制作为协议的一部分和运行按照以下的流程(假设客户端已经成功的连接和订阅了主题)
        - 客户端他们已经准备好接收消息
        - nsq 发送一个消息然后临时将他们存储在本地
        - 客户端返回fin,req信号来告诉nsq 消息是发送成功了还是失败了。如何客户端没有回复nsq 。或者回复超时了。nsq 将会自动的将消息重新加入到队列之中
    - 上述的措施将保证如果出现消息丢失的情况，那么一定是nsqd 突然shutdown了。在这种情况下任何在内存里的消息都是丢失
    它将消息的丢失放在了最大限度的重要性上面。甚至这种情况也能减少。一个解决办法就是部署多余的nsqd对。因为you’ve written your consumers to be idempotent, doing double-time on these messages has no downstream impact and allows the system to endure any single node failure without losing messages.
    The takeaway is that NSQ provides the building blocks to support a variety of production use cases and configurable degrees of durability.
- 有限的内存的足印
    - nsqd 提供了一个可配置的选项--mem-queue-size。这个选项将决定保存在内存中的消息的总数。如果这个队列的消息的强度超过了保存的消息的极限，那么这些消息将会被保存在磁盘上。这个内存一个被给的nsqd 进程将会被处理成mem-queue-size *
    当然一个聪明的观察者已经意识到了这是一个方便的方法来获得一个高确保率的消息传递。通过将这个内存的值设置为1或者0。磁盘备份的队列是设计来恢复从未知的nsqd的停止上来恢复数据的
    - 当然相对于消息传送上的保证上，我们也可以通过向nsqd 发送一个停止信号。来安全的保存当前在内存中的信息
    - 一个提示。任何一个主题或者管道的名字以#ephemeral 为结尾的将不会保存到磁盘上。将会直接丢弃掉信息This enables consumers which do not need message guarantees to subscribe to a channel. 这些临时的管道也将会消息。在最后一次客户端断开连接的时候。对于一个临时的主题来说。他意味着至少一个管道被创建,被消费，然后被删除
- 性能
    