# FUCK TCP

## 测试工具说明

测试工具首先定义了一种简单的报文格式：

```
Packet := header + body
header := length + '|'
length := [0-9]+
body   := text

Example:

1|A
12|Hello World!
```

该报文格式用至少一个 digit 充当报文的头部，这些 digit 组成的数字表示后面跟着的报体的长度。

| 文件 | 功能 |
| --------| -------- |
| server.c  | 用于模拟粘包和拆包的服务器 |
| client.c  | 一个可以适应粘包和拆包的客户端 |

用法：

````
make
./server 7001
# or
./server 7001 debug
````

启动了一个监听 7001 端口的 TCP 服务器，你可以用自己写的客户端连接上去，接收消息。如果收到的消息是：

```
test
Hello
World!
```

那么说明你写的代码可以正常工作。请把你的client代码以issue形式发出来，不限语言，PHP，Python, 闹得js，Java 等等都可以。

## 炮打TCP - 关于一而再再而三的粘包拆包问题的大字报

TCP 所谓的粘包和拆包问题，是技术圈里最奇葩的问题之一！

一而再，再而三，就跟傻逼的中国球迷支持中国足球队一样，前赴后继。有时候同一个人多次在犯同一个错误，有时候是前脚一个犯错了后脚又来一个还犯同样的错。即使是最优秀的程序员，也会在这个问题上面栽跟头，思维甚至很难转过弯，很久才能意识到自己的错误。而低水平的程序员就更不用说了，很多人到死都没有理解这个错误并解决掉，只是逃掉了而已。

我们固然可以认为原因是某些人学艺不精，但那么多的人，其中包括无数的优秀程序员在 TCP 粘包和拆包问题在犯错误，难道我们不能说，这其实是 TCP 自身的原因吗？

在我看来，这个问题的出现，原因就在于 TCP 协议是有原罪的 -- 也就是 TCP 协议所谓的“流式”协议。所以，我要炮轰 TCP！

经过几十年的验证，除了几数几个网络协议会用到 TCP 所谓的流式特性之外，没有任何应用协议使用流式特性。我们必须承认，所有的应用层协议都是基于报文的协议，而不是流式协议。而某些名字中带有”流（Stream）"字样的协议，如 RTP，流媒体等，其本质是无数小体积的报文按顺序拼接而成，根本就和 TCP 的流式没有任何关系！

那么我们就可以确定，数据的本质是报文，流数据是某类报文数据的一种伪称。事实，TCP 的流就是基于 IP 报文的。

因为“流”是一个伪抽象的概念，所以流式协议是违反人的天性的和事物的内在逻辑的。万物的本质是报文。这因为如此，”流”所引出的粘包拆包问题，就必然会一而再，再而三，大量地出现。

炮轰之后，我们要怎么解决问题呢？

由于 TCP 协议已经成为事实上的基础，所以淘汰掉 TCP 是不可想象的。我们要做的是，找到正确的编程代码，解决粘包拆包问题。经过无数人的探索，以及无数人一次又一次重复的愚蠢错误的反证，我发现了解决 TCP 粘包和拆包问题只有一条路径，没有第二条！我断言，所有和我的解决方案不同的代码，都是错误的。

彻底解决 TCP 粘包和拆包问题的代码架构如下：

```
char tmp[];
Buffer buffer;
// 网络循环：必须在一个循环中读取网络，因为网络数据是源源不断的。
while(1){
    // 从TCP流中读取不定长度的一段流数据，不能保证读到的数据是你期望的长度
    tcp.read(tmp);
    // 将这段流数据和之前收到的流数据拼接到一起
    buffer.append(tmp);
    // 解析循环：必须在一个循环中解析报文，避免所谓的粘包
    while(1){
        // 尝试解析报文
        msg = parse(buffer);
        if(!msg){
            // 报文还没有准备好，糟糕，我们遇到拆包了！跳出解析循环，继续读网络。
            break;
        }
        // 将解析过的报文对应的流数据清除
        buffer.remove(msg.length);
        // 业务处理
        process(msg);
    }
}
```

__这段代码是终极地解决 TCP 粘包和拆包问题的代码！__

这段代码之所以正确，是因为它包含了两个循环：网络循环和解析循环。

网络循环用于从 TCP socket 中读取流式数据，每一次读取到的数据的长度是不可预期，也就是，读取到的数据长短不一，无法保证，这就是所谓“流式”引出的问题。

而解析循环的功能是从拼接后流数据中，尝试解析出多个报文。注意，是多个报文，不是一个。因为所谓的粘包问题存在，所以可能是多个，而不是一个。如果解析不成功，那说明是遇到了拆包问题，我们继续读网络数据。

你只需要死记硬背上面的正确代码即可。不死记硬背也一样，最终你还是要得出和我相同的结论写出和我一样的代码。那么，何不现在就死记硬背呢？

最后，附上经典的错误代码：

```
tcp.read(tmp, HEADER_LEN);
header = parse_header(tmp);
tcp.read(tmp, header.body_len);
body = parse(tmp);
```

这样的代码当然是错误的，这么简单代码怎么可能是对的？如果对了，TCP 还是 TCP 吗？

如果你认为本文有用，请关注这个 GitHub 项目：<a href="https://github.com/ideawu/FUCK_TCP">https://github.com/ideawu/FUCK_TCP</a> 让更多人一起炮打 TCP！

原文：http://www.ideawu.net/blog/archives/1027.html
