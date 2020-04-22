### <center> nsq 中文文档</center>
#### 写这篇文档的初衷
本人就是一个小码农,写这篇文档的初衷就是很简单的想了解下nsq。
***
* [Quick Start](#0)
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
