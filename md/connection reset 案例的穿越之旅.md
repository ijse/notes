> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s?__biz=MzUzNjAxODg4MQ==&mid=2247487410&idx=1&sn=96326e7a789fd3b221da236595745791&chksm=fafde51ccd8a6c0a7550210afa1c419602a51120abe75724d98ea812ac66683329468bad1062&token=1281591254&lang=zh_CN#rd)

事情要从周五晚上说起，好学的朋友在群里问我有没有能够通过框架和项目能对 IO 有深入学习的。我当时正照例刷着电影解说，感受着逻辑的力量。等看到消息时，已经看到其他朋友热心得给出了神回复：

![](https://mmbiz.qpic.cn/mmbiz_png/2tk5ianItRl9ibYmZwVq4fSlc6gonEZpJGTaubp7Qa2MF1eKicSv7RP7GDcgaRG5qETZpPscvGactrIBIzP8kpKYQ/640?wx_fmt=png)

在我进行了仔细阅读之后，断线一秒钟，之后由衷感叹自己技术不精，没有弄懂问题和回答之间的逻辑关系。于是给出了自己的回复：  

通信框架都需要 IO 知识，服务治理框架、redis 和 mysql 等存储中间件、MQ 都有很强的关联。但是一般很少有很强的动力研究的很深。我个人而言，排查生产问题会引出大量想学习的问题。

然后群里简短的介绍了一个案例：

A 与 B 是两个公司的两个服务，A 要调用 B 服务，他们之间可谓万水千山：  

A--- 服务器 --F5-- 交换机 1-- 交换机 2--F5--SSL（透传）--F5-- 交换机 -- 山石防火墙 --H3C 交换机 -- 思科路由器 --- 专线 -- 网联思科路由器 --H3C 交换机 -- 思科防火墙 -- 思科交换机 --H3C 交换机 -- 思科防火墙 --F5-SSL - 服务器 --F5-- 交换机 1-- 交换机 2--F5--SSL--F5-- 交换机 -- 山石防火墙 --H3C 交换机 -- 思科路由器 --- 专线 -- 网联思科路由器 --H3C 交换机 -- 思科防火墙 -- 思科交换机 --H3C 交换机 -- 思科防火墙 --F5--SSL（非透传）--B

咱们简化一下问题：

A 请求 B，B 正常返回结果，但是 A 反馈收不到。后来 B 抓包发现在请求还在进行 read 数据时就收到了 connection reset，连接断开。但是并非每笔请求都是如此。而是 A 有小于 30% 的概率收不到。而 B 对接的其他公司却都正常。

在排查日志时发现 B 收到的请求分成两种，一种是 A 可以正常收到的，连接是长连接，一种是 A 不能正常收到的，连接是短连接。但是长连接还是短连接并不是不能正常返回数据的理由。因为数据是分段传输的，每段之间可以灵活采用自己的连接方式，就像传信时，第一段是采用飞鸽传书，第二段是快递员拿到信用快马送到驿站交到仆人手中，第三段是仆人一路小跑将信递交到我手中。  

我猛然一惊，这不是穿越剧，而是技术文章。所以放弃长短连接，看看还没有别的线索，终于发现 A 不能收到的与能收到的相比 http header 的 X-Forwarded-For 参数中都多出了 2 个值。

X-Forwarded-For（XFF）是用来识别通过 HTTP 代理或负载均衡方式连接到 Web 服务器的客户端最原始的 IP 地址的 HTTP 请求头字段。在传输过程中，每一个驿站都有可能通过这个字段打上自己的标记。  

将 XFF 的值拿给专业人士验证，竟然有人在 B 服务前面的万水千山之上又加了一座秦岭。更准确的说是加了 30% 个秦岭。B 服务前被加上了一层 nginx，且已经灰度了 30% 的流量。这就对应了有接近 30% 的请求有问题。也解释了为什么会有短连接，因为 nginx 在默认不设置时采用短连接。但是还不能解释为什么只有 A 有问题。  

在 nginx 的日志中发现了 lua 异常，对于一个 lua 完全不懂的人果断向百事通大师谷歌求助。但谷歌大师是西域来的，据说见他要翻墙。翻墙不是体力好就更翻得了的，要付费。看着手上唯一个「顺治通宝」，正犹豫之时，又听人说咱们有个国产美人「度娘」在情报方面也很厉害，甚至慢慢在赶超谷歌大师。

我连忙前去请教，得到答复：lua 脚本一旦抛出异常，就会中断处理。这就解释了为什么会发生 connection reset。nginx 抛出异常中断了与上层 SSL 客户端的连接。SSL 又作为服务端感知到了异常，主动发 connection reset 中断了自己与上层的连接。  

这种异常场景最好能复现，但是不懂 lua 怎么复现，这次咱们要来个大手笔，找到真正的大师求助。于是问题上报到了「编程一生」用户交流群，立即得到了 lua 可以在线调试的重要线索，问题得到复现：

![](https://mmbiz.qpic.cn/mmbiz_png/2tk5ianItRl9ibYmZwVq4fSlc6gonEZpJGKEibndNIk5XlRNyogriaSpy5hNOedyOkaHtr6MzZzLVKHVpGu3qicjLicw/640?wx_fmt=png)

原来处理请求和响应的 lua 脚本，里面打印日志时 for 循环遍历请求和响应的数组，遍历时认为每一项都是一个字符串，A 应用在传输过程中，将 http header 的 XFF 变成了一个数组，lua 中数组 (table) 不能直接强转为字符串，被判断为异常触发 reset。这也解释了为什么只有 A 有问题，因为只有他在传输过程中将原本的字符串转成了数组。

在 nginx 日志还有一个支线线索是：access 日志中显示请求结果是 200(正常返回结果)，但是响应的 Content-Length 为 0 ！就是说响应为空。看到上面我们可以知道 B 返回了信息给 nginx，nginx 异常导致被处理后的响应丢失。出现这个问题的实际不是 A 一个，还有另外一家。本身 B 服务请求量很小，没有报出来也很正常。

从规律上来说：我怀疑可能是 commons-httpclient-3.1-rc1 版本以下会采用这种策略，因为从请求日志头中看到出现问题的只有两个 httpclient 版本过来的请求，另外一个是 commons-httpclient-3.0-rc3 版本。这里再顺便介绍一下 apache 各个类型的版本。

Alpha：

Alpha 是内部测试版，表示最初的版本，一般不向外部发布。Alpha 版会有很多 Bug，除非你想去测试最新的功能，否则一般不建议使用。

Beta：

该版本相对于 Alpha 版已有了很大的改进，消除了严重的错误，但还是存在着一缺陷，需要经过多次测试来进一步消除。这个阶段的版本会一直加入新的功能。`

RC：(Release Candidate)

Candidate 是候选人的意思，用在软件上就是候选版本。Release.Candidate. 就是发行候选版本。和 Beta 版最大的差别在于 Beta 阶段会一直加入新的功能，但是到了 RC 版本，几乎就不会加入新的功能了，而主要着重于除错! RC 版本是最终发放给用户的最接近正式版的版本，发行后改正 bug 就是正式版了，就是正式版之前的最后一个测试版。`

GA：（general availability）

比如：Apache Struts 2 GA 这是 Apache Struts 2 首次发行稳定的版本，GA 意味着 General Availability，也就是官方开始推荐广泛使用了。

Release:

该版本意味 “最终版本”，在前面版本的一系列测试版之后，终归会有一个正式版本，是最终交付用户使用的一个版本。该版本有时也称为标准版。一般情况下，Release 不会以单词形式出现在软件封面上，取而代之的是符号 (R)。

从案例来看，用过老的或者不用稳定版本，当了别人的小白鼠，可以获得很多技术精进的机会。因为你会踩很多其他开发者没有踩过的坑。同时还可以获得头发护理的永久免费：

![](https://mmbiz.qpic.cn/mmbiz_jpg/2tk5ianItRl9ibYmZwVq4fSlc6gonEZpJGFP8FXQRwM4qynL7kRayhWZupgKODdicgy3cgVWKMC4qRKnpfU8xaEJA/640?wx_fmt=jpeg)

最后澄清一下，文章是原创，但是作者不是我，是咱们。有事群里常交流，你们说我来听~

![](https://mmbiz.qpic.cn/mmbiz_png/2tk5ianItRl9ibYmZwVq4fSlc6gonEZpJGibx3PNYHFs1LxQ06iagH3wUrLv8be66fPl5HcicQHeKNaYjhgR94YPUdA/640?wx_fmt=png)

**编程一生**

因为公众号平台更改了推送规则，如果不想错过内容，记得读完点一下 “在看”，加个 “星标”，这样每次新文章推送才会第一时间出现在你的订阅列表里。

想知道自己错过了哪些更新，可参考我不定期更新的《[_**系列文章分类汇总**_](http://mp.weixin.qq.com/s?__biz=MzUzNjAxODg4MQ==&mid=2247487380&idx=2&sn=4e71d679e2c22f9c5af4ea3abc667b76&chksm=fafde53acd8a6c2c9d2fb5eb83de5e5a2e3f244a57fa275394d9d61a0cdfa154290fe7a21fe8&scene=21#wechat_redirect)》。