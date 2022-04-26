> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/vSh6w3eL_oQvU1mxnxsArA)

0、题记
----

我在做 Elasticsearch 相关咨询和培训过程中，发现大家普遍更关注实战中涉及的问题，下面我选取几个常见且典型的问题，和大家一起分析一下。
-------------------------------------------------------------------------

*   订单表、账单表父子文档可以实现类似 SQL 的左连接吗？通过 canal 同步到 ES 中，能否实现类似左连接的效果？具体应该如何建模？
    
*   一个人管理 1000  家连锁门店，如何更高效地查询自己管辖的商品类目？企微 一个人维护了 1000 个员工，如何快速查询自己管辖的员工信息？
    
*   随着业务的增长，一个索引的字段数据不断膨胀（商品场景变化，业务一直加字段），有什么解决方法？
    
*   一个索引字段个数设置为 1500 个，超出这个限制，会不会消耗 CPU 资源和造成写入堆积？
    
*   日志诊断用于机器学习基线，需要将 message 分离出来，怎么在写入前搞定？
    

如果我们对上述实战问题进行归类，就都可以归结为 Elasticsearch `数据建模`问题。

本文将以实战问题为基准，手把手带你实践 Elasticsearch 数据建模全流程，重点解析基于业务角度、数据量角度、Setting 、Mapping ，以及复杂索引关联，这五个层面中涉及的数据建模实战问题，让你学完即可应用到工作中。

1、为什么要做数据建模？
------------

我们选型传统的数据库，这里以 MySQL 为例，做数据存储前需要考虑的问题如下：

*   数据库要不要做读写分离？
    
*   分几张表存储？
    
*   每个表的名是什么？
    
*   每个表是按照业务划分吗？
    
*   单表数据大了怎么搞？分库分表还是其他？
    
*   每个表要有哪些字段？每个字段设置什么类型？如何设计合理的字段类型，才能保证节省存储？
    
*   哪些字段需要建索引？
    
*   哪些字段需要设置外键？
    
*   表之间要不要建立关联？如何实现关联联动查询？
    
*   关联查询可能会很慢？如何设计阶段优化建模才能提高响应速度？
    

以上这些疑问也均是[数据建模](http://mp.weixin.qq.com/s?__biz=MzI2NDY1MTA3OQ==&mid=2247484159&idx=1&sn=731562a8bb89c9c81b4fd6a8e92e1a99&chksm=eaa82ad7dddfa3c11e5b63a41b0e8bc10d12f1b8439398e490086ddc6b4107b7864dbb9f891a&scene=21#wechat_redirect)问题。在 MySQL 中我们往往认为建模非常有必要，但反观 Elasticsearch ，“上手快” 这类先入为主的观念已根植在很多同学心中，使得大家忽略了 Elasticsearch 数据建模的重要性。

接下来，我们基于 MySQL 做数据存储需要考虑的问题，重新审视数据建模的定义，内容如下。

*   数据模型是对描述数据、数据联系、数据语义和一致性约束进行标准化的抽象模型。
    
*   数据建模是为存储在数据库中的资源创建和分析数据模型的过程。
    
*   数据建模主要目的是表示系统内的数据类型、对象之间的关系及其属性。
    
*   数据模型有助于了解需要哪些数据以及应如何组织数据。
    

到这里，相信你已经初步明晰了数据建模的重要性。但我还想提醒你的是，“[一把梭](http://mp.weixin.qq.com/s?__biz=MzI2NDY1MTA3OQ==&mid=2247484640&idx=1&sn=2e734c41667751f17261731692c1555b&chksm=eaa82cc8dddfa5dee4e33400306ee00bc3d3d697dd41f643444f2ffcd3c95309e07ff22d9286&scene=21#wechat_redirect)用法，上来就是干” 并不是捷径，尤其到了项目中后期，极易暴露出问题。经历的项目越多，你会发现建模的时间不能省。

下面我们具体分析一下为什么要数据建模？

相比于 MySQL，Elasticsearch 有非常快捷的`优势`：

Elasticsearch 支持动态类型检查和匹配。也就是说，当我们写入索引数据的时候，可以不提前指定数据类型，直接插入数据。

以类似天眼查、企查查的工商实战数据为例（已做脱敏处理），如果利用以下语句直接创建索引和写入一条数据，岂不是很快？

```
PUT company_index/_doc/1{  "regist_id": 1XX1600000000012,  "company_name": "北京XX长江创业投资有限公司",  "regist_id_new": "191XX160066933968XC",  "legal_representative": "徐X武",  "scope_bussiness": "创业投资业务；代理其他创业投资企业等机构或个人的创业投资业务；创业投资咨询业务；为创业企业提供管理服务业务；参与设立创业投资企业与企业投资管理顾问机(依法须经批准的项目，经相关部门批准后方可开展经营活)",  "registration_status": "在营（开业）企业",  "approval_date": "201X年04月13日",  "registration_number": "191XX160066933968XC",  "establishment_time": "200X年12月03日",  "address": "北京市黄河XX路西首育青小区",  "register_capital": 3000,  "business_starttime": "20XX年12月03日",  "registration_authority": "北XX工商行政管理局",  "company_type": "其他有限责任公司",  "enttype": 1190,  "enttypename": "法定代表人:",  "pripid": "1XXX102201305305801X",  "uniscid": "1XXX160066933968XC"}
```

相比于 MySQL 中一个字段一个字段地敲定，这样操作确实节省了很多时间。但随着后续数据量激增，副作用便会很快显现出来。该处理方式的弊端：

*   首先是极大地浪费了存储空间，所有字符串类型数据都存储为 text + keyword 组合类型，这种很多业务字段都是非必须的；
    
*   其次字符串类型默认分词 standard，无法满足中文精细化分词检索的需求。
    

接下来，结合我自己工作中早期系统的一个案例，我们做进一步分析。

5 个数据节点集群（5 个分片，1 个副本），微博数据每日增量 5000W+（增量存储 150GB），核心数据磁盘 10TB 左右，很明显该系统面临存储上限问题。

我们当时就上述业务数据规划了一个大索引，比如微博数据一个索引，微信数据一个索引。但微博索引最多只能存储 20 天左右的数据，然后就得走删除索引数据的操作。由于 1 个索引只能通过 delete_by_query 删除部分数据，而 delete_by_query 的特点是版本号更新的逻辑删除，实际效果是越删数据量越大，磁盘占用率激增。加上是线上环境，压力之大，处理难度之大，经历过你就知道有多苦。

这也是很多大厂在面试候选人的时候，尤其偏爱数据建模能力强的工程师的主要原因之一。

比如下图是美团对大数据开发高级工程师的岗位要求，第一条就是 “深入理解业务，对业务服务流程进行合理的抽象和建模。”

![](https://mmbiz.qpic.cn/mmbiz_png/mjl8GCpsL9a2p3HdfNVKjklm5YTDab708rRibcuXcnvXey0hicbwxXTiajiae0478yPhTPGpuctib4H3batMMRlwBwg/640?wx_fmt=png)

从以上两个反例，以及这条招聘信息中便可以窥探出数据建模的重要性。下面我们具体说说如何做数据建模。

2、Elasticsearch 如何数据建模？
-----------------------

在做数据建模之前，会先进行架构设计，架构环节涉及选型、集群规划、节点角色划分。

本文涉及的建模倾向于索引层面、数据层面的建模。为了让你学完即可应用到工作中，我会结合项目实战进行讲解。

### 2.1 基于业务角度建模

Elasticsearch 适用范围非常广，包括电商、快递、日志等各行各业。涉及索引层面的设计，和业务贴合紧密。

其一：业务一定要细分。

分成哪几类数据，每类数据归结为一个索引还是多个索引，这是产品经理、架构师、项目经理要讨论敲定的问题。比如大数据类的数据，可以按照业务数据分为微博索引、微信索引、Twiiter 索引、Facebook 索引等。

其二：多个业务类型需不需要跨索引检索？

跨索引检索的痛点是字段不统一、不一致，需要写非常复杂的 bool 组合查询语句来实现。为了避免这种情况，最好的方式就是提前建模。每一类业务数据的相同或者相似字段，采取统一建模的方式。

下面我们举一个实际的例子加以分析。微博、微信、Twitter、Facebook 都有的字段，可以设计如下：

<table data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)"><thead data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)"><tr data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(25, 25, 25)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)" data-style="border-width: 1px 0px 0px; border-right-style: initial; border-bottom-style: initial; border-left-style: initial; border-right-color: initial; border-bottom-color: initial; border-left-color: initial; border-top-style: solid; border-top-color: rgb(204, 204, 204); background-color: white;"><th data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(40, 40, 40)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)|rgb(240, 240, 240)" data-style="border-top-width: 1px; border-color: rgb(204, 204, 204); text-align: justify; background-color: rgb(240, 240, 240); font-size: 14px; min-width: 85px;"><span data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(40, 40, 40)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)|rgb(240, 240, 240)">字段名称</span></th><th data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(40, 40, 40)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)|rgb(240, 240, 240)" data-style="border-top-width: 1px; border-color: rgb(204, 204, 204); text-align: justify; background-color: rgb(240, 240, 240); font-size: 14px; min-width: 85px;"><span data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(40, 40, 40)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)|rgb(240, 240, 240)">字段中文含义</span></th><th data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(40, 40, 40)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)|rgb(240, 240, 240)" data-style="border-top-width: 1px; border-color: rgb(204, 204, 204); text-align: justify; background-color: rgb(240, 240, 240); font-size: 14px; min-width: 85px;"><span data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(40, 40, 40)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)|rgb(240, 240, 240)">字段类型</span></th></tr></thead><tbody data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)"><tr data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(25, 25, 25)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)" data-style="border-width: 1px 0px 0px; border-right-style: initial; border-bottom-style: initial; border-left-style: initial; border-right-color: initial; border-bottom-color: initial; border-left-color: initial; border-top-style: solid; border-top-color: rgb(204, 204, 204); background-color: white;"><td data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(25, 25, 25)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)" data-style="border-color: rgb(204, 204, 204); font-size: 14px; min-width: 85px; text-align: justify;"><span data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(25, 25, 25)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)">publish_time</span></td><td data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(25, 25, 25)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)" data-style="border-color: rgb(204, 204, 204); font-size: 14px; min-width: 85px; text-align: justify;"><span data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(25, 25, 25)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)">发布时间</span></td><td data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(25, 25, 25)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)" data-style="border-color: rgb(204, 204, 204); font-size: 14px; min-width: 85px; text-align: justify;"><span data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(25, 25, 25)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)">date</span></td></tr><tr data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(32, 32, 32)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(248, 248, 248)" data-style="border-width: 1px 0px 0px; border-right-style: initial; border-bottom-style: initial; border-left-style: initial; border-right-color: initial; border-bottom-color: initial; border-left-color: initial; border-top-style: solid; border-top-color: rgb(204, 204, 204); background-color: rgb(248, 248, 248);"><td data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(32, 32, 32)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(248, 248, 248)" data-style="border-color: rgb(204, 204, 204); font-size: 14px; min-width: 85px; text-align: justify;"><span data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(32, 32, 32)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(248, 248, 248)">author</span></td><td data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(32, 32, 32)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(248, 248, 248)" data-style="border-color: rgb(204, 204, 204); font-size: 14px; min-width: 85px; text-align: justify;"><span data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(32, 32, 32)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(248, 248, 248)">作者</span></td><td data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(32, 32, 32)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(248, 248, 248)" data-style="border-color: rgb(204, 204, 204); font-size: 14px; min-width: 85px; text-align: justify;"><span data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(32, 32, 32)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(248, 248, 248)">keyword</span></td></tr><tr data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(25, 25, 25)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)" data-style="border-width: 1px 0px 0px; border-right-style: initial; border-bottom-style: initial; border-left-style: initial; border-right-color: initial; border-bottom-color: initial; border-left-color: initial; border-top-style: solid; border-top-color: rgb(204, 204, 204); background-color: white;"><td data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(25, 25, 25)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)" data-style="border-color: rgb(204, 204, 204); font-size: 14px; min-width: 85px; text-align: justify;"><span data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(25, 25, 25)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)">cont</span></td><td data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(25, 25, 25)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)" data-style="border-color: rgb(204, 204, 204); font-size: 14px; min-width: 85px; text-align: justify;"><span data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(25, 25, 25)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)">正文内容</span></td><td data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(25, 25, 25)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)" data-style="border-color: rgb(204, 204, 204); font-size: 14px; min-width: 85px; text-align: justify;"><span data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(25, 25, 25)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)">text</span></td></tr></tbody></table>

这样设计的好处是：字段统一，写查询 DSL 无需特殊处理，非常快捷方便。所以，在设计阶段，多个业务索引数据要尽可能地 “求同存异”。具体来说：

*   求同指的是相同或者相近含义字段，一定要统一字段名、统一字段类型；
    
*   存异指的则是特定业务数据特有字段类型，可以独立设计字段名称和类型。
    

比如微博信息来源字段有手机 App 或者网页等，别的业务索引如果没有，独立建模就可以。

类似这些建模信息可以统一 Excel 存储，统一 git 多人协作管理。

多索引管理一般优先推荐使用[模板](http://mp.weixin.qq.com/s?__biz=MzI2NDY1MTA3OQ==&mid=2247484584&idx=1&sn=accfb65830255f00c28ac1571725e493&chksm=eaa82c80dddfa596bef19161d713fe935142eda894f11d065a38495cd3e7ec59157403c5393f&scene=21#wechat_redirect)（template）和 [别名](http://mp.weixin.qq.com/s?__biz=MzI2NDY1MTA3OQ==&mid=2247484454&idx=1&sn=43e95a27bd8635b73d78a23db79407fe&chksm=eaa82c0edddfa518e979b4b4fe5f6b6cf933b30f6177c663bb006148e5a1ef0d47684a8aac92&scene=21#wechat_redirect)（alias）结合的方式。

*   [模板](http://mp.weixin.qq.com/s?__biz=MzI2NDY1MTA3OQ==&mid=2247484584&idx=1&sn=accfb65830255f00c28ac1571725e493&chksm=eaa82c80dddfa596bef19161d713fe935142eda894f11d065a38495cd3e7ec59157403c5393f&scene=21#wechat_redirect)的特点：相同前缀名称的索引可以归结为一大类，一次创建，N 多索引共享，非常方便。
    
*   [别名](http://mp.weixin.qq.com/s?__biz=MzI2NDY1MTA3OQ==&mid=2247484454&idx=1&sn=43e95a27bd8635b73d78a23db79407fe&chksm=eaa82c0edddfa518e979b4b4fe5f6b6cf933b30f6177c663bb006148e5a1ef0d47684a8aac92&scene=21#wechat_redirect)的特点：多个索引可以映射到一个别名，方便多索引以相同的名称统一对外提供服务。
    

![](https://mmbiz.qpic.cn/mmbiz_png/mjl8GCpsL9a2p3HdfNVKjklm5YTDab70vBbK88Touq35WMic6ISOibCzzU9jpvr3B53y1zcWYv0AIvRS2M8WU9QQ/640?wx_fmt=png)

### 2.2 基于数据量角度建模

如本文前面所述，我是吃过单索引激增的亏，所以对于时序性数据（日志数据、大数据类数据）等，我强烈建议你[基于时间切分索引](http://mp.weixin.qq.com/s?__biz=MzI2NDY1MTA3OQ==&mid=2247486630&idx=1&sn=caaee097196cd6047c212593ee5d340d&chksm=eaa8248edddfad98facc2b771a9993cd0372c9cf12a63fda8ad60dc9e6812fda722e56b3fb77&scene=21#wechat_redirect)，具体如下图所示。

![](https://mmbiz.qpic.cn/mmbiz_png/mjl8GCpsL9a2p3HdfNVKjklm5YTDab70jyr81FLDncHkTGsic0CgnfKmVMYH52bkYRaHRIUdaeDuK1nJS3IlFlg/640?wx_fmt=png)

当然，其他可用的方案非常多，这里我列举如下，供你选型参考。

![](https://mmbiz.qpic.cn/mmbiz_png/mjl8GCpsL9a2p3HdfNVKjklm5YTDab70ZicC4jtFDNQV2oKR0TWV1I7e0y0DRvBxz3WTXSvKc1zFT0rwm2E0hZQ/640?wx_fmt=png)

由此可见，时序管理数据的优点非常明显。

*   其一是灵活。基于时间切分索引非常方便，删除数据属于物理删除。
    
*   其二则是快速。特定业务数据配合冷热集群架构，确保高配机器对应热数据，提升检索效率和用户体验。
    

### 2.3 基于 Setting 层面建模

Setting 层面又分为静态 Setting 和动态 Setting 两种。

一种是静态 Settings，一旦设置后，后续不可修改。如 `number_of_shards`。

另一种是动态 Setting，索引创建后，后面随时可以更新。如 `number_of_replicas`, `max_result_window`, `refresh_interval` 。

仅就建模阶段最核心的问题，拆解如下。

*   问题一：索引设置多少个分片？多少个副本？
    

这里有个认知前提，就是主分片数一旦设置后就不可以修改，副本分片数可以灵活动态调整。

主分片设计一般会考量总体数据量、集群节点规模，这点在集群规划层面会着重强调。一般主分片数要考虑集群未来动态扩展，通常设置为数据节点的 1 倍或者 1~3 倍之间的值。

副本分片是保证集群的高可用性，普通业务场景建议至少设置一个副本。

*   问题二：refresh_interval 一般设置多大？
    

默认值 1s，这意味着在写入阶段，每秒都会生成一个分段。

`refresh_interval` 的目的是：数据由 `index buffer` 的堆内存缓存区刷新到堆外内存区域，形成 `segment`，以使得搜索可见。

在实际业务场景里，如果写入的数据不需要近实时搜索可见，可以适当地在模板、索引层面调大这个值，当然也可以动态调整，比如调整为 30s 或者  60s。

*   问题三：max_result_window 要不要修改默认值？
    

这里同样有个认知前提，就是对于[深度翻页](http://mp.weixin.qq.com/s?__biz=MzI2NDY1MTA3OQ==&mid=2247486505&idx=1&sn=857774dce3511cf8c28c3b35f61a607a&chksm=eaa82401dddfad174463cf172755abfd6fd0c6fccfca72bcbe00c41d8208046485c62a4b901c&scene=21#wechat_redirect)的 from + size 实现，越往后翻页越慢。其实你对比看主流搜索引擎，比如 Google、百度、360、Bing 均不支持一下跳转到最后一页，这就是最大翻页上限限制。

其实在基本业务层面也很好理解，按照相关度返回结果，前面几页是最相关的，越往后相关度越低。比如默认值 10000，也就是说如果每页显示 10 条数据，可以翻 1000 页。基本业务场景已经足够了。因此不建议调大该值。

如果需要向后翻页查询，推荐 search_after 查询方式。如果需要全量遍历或者全量导出数据，推荐 scroll 查询方式。

*   问题四：管道预处理怎么用？
    

[管道预处理](http://mp.weixin.qq.com/s?__biz=MzI2NDY1MTA3OQ==&mid=2247485003&idx=1&sn=1f0d1c933d458bb9e3c71794d354aff1&chksm=eaa82e63dddfa77584bde5083adcdb4b33ef0c35fce8749dd6b6980ff92071b567a6a7ff93f7&scene=21#wechat_redirect)的好处很多，虽然 5.X 版本就有了这个功能，但实战环境用起来还不多。

管道 `ingest pipeline` 就相当于大数据的 ETL 抽取、转换、加载的环节，或者类似 `logstash filter` 处理环节。一些数据打标签、字段类型切分、加默认字段、加默认值等的预处理操作都可以借助 `ingest pipelie` 实现。

这里给出索引层面 `Setting` 设置的简单模板，供你进一步学习参考，如下定义了 indexed_at 缺省的管道，同时在索引 my_index_0001 指定了该缺省管道，这样做的好处，是每个新增的数据都会加了插入时刻的时间戳：indexed_at 字段，无需我们在业务层面手动处理，非常灵活和方便。

更多设置，推荐阅读官方文档，地址如下：

https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules.html#index-modules-settings

```
PUT _ingest/pipeline/indexed_at{  "description": "Adds indexed_at timestamp to documents",  "processors": [    {      "set": {        "field": "_source.indexed_at",        "value": "{{_ingest.timestamp}}"      }    }  ]}PUT my_index_0001{  "settings": {    "number_of_replicas": 1,    "number_of_shards": 3,    "refresh_interval": "30s",    "index": {      "default_pipeline": "indexed_at"    }  },   "mappings": {    "properties": {      "cont": {        "type": "text",        "analyzer": "ik_max_word",        "fields": {          "keyword": {            "type": "keyword"          }        }      }    }  }}
```

### 2.4 基于 Mapping 层面建模

Mapping 层面核心是字段名称、字段类型、分词器选型、多字段 multi_fields 选型，以及字段细节（是否索引、是否存储等）的敲定。

#### （1）字段命名要规范

索引名称不允许用大写，字段名称官方没有限制，但是可以参考 Java 编码规范。我还真见过学员用中文或者拼音命名的，非常不专业，大家一定要避免。

#### （2）字段类型要合理

要结合业务类型选择合适的字段类型。比如 integer 能搞定的，就不要用 long、float 或 double。

注意，字符串类型在 5.X 版本之后分为两种类型:

*   一种是 keyword，适合精准匹配、排序和聚合操作；
    
*   另一种是 text，适合全文检索。默认值 text & keyword 组合不见得是最优的，选型时候要结合业务选择。比如优先选择 keyword 类型，keyword 走倒排索引更快。
    

再举个例子，实战中情感值介于 0~100 之间，50 代表中性，0~50 代表负面，50~100 代表正面。如果使用 integer 查询的时候要 range query，而实际存储可以增加字段：0~50 设置为 -1，50 设置为 0，50~100 设置为 1，三种都是 keyword 类型，检索时直接走 term 检索会非常快。

#### （3）分词器要灵活

实战中中文分词器用得比较多，中文分词又分为 ansj，结巴，IK 等。以 IK 举例，可以细分为 ik_smart 粗粒度分词、ik_max_word 细粒度分词。

在工作中，要结合业务选择合适的分词器，分词器一旦设定是不可以修改的，除非 reindex。

分词器选型后，都会有动态词典的更新问题。更新的前提是不要仅使用开源插件原生词典，而是要在平时业务中自己多积累特定业务数据词典、词库。

如果要动态更新：一般推荐第三方更新插件借助数据库更新实现。如果普通分词都不能满足业务需要，可以考虑 [`ngram` 自定义分词](http://mp.weixin.qq.com/s?__biz=MzI2NDY1MTA3OQ==&mid=2247484728&idx=1&sn=eeb76ad84c98af16fc16d6dc5d5d11af&chksm=eaa82d10dddfa406e5847d61a0ef2cd1230cb10185dd32bf0e1d1a64c7980658f0268264af96&scene=21#wechat_redirect)方式实现更细粒度分词。

#### （4）multi_fields 适机使用

同一个字段根据需要可以设置多种类型。实战业务中，对用特定中文词明明存在，却无法召回的情况，采用字词混合索引的方式得以满足。

所谓字词混合，实际就是 standard 分词器实现单字拆解，以及 ik_max_word 实现中文切词结合的方式。检索的时候 bool 对两种分词器结合，就可以实现相对精准的召回效果。

```
PUT mix_index{  "mappings": {      "properties": {        "content": {          "type": "text",          "analyzer": "ik_max_word",          "fields": {            "standard": {              "type": "text",              "analyzer": "standard"            },            "keyword": {              "type": "keyword",              "ignore_above": 256            }          }        }      }    }}POST mix_index/_search{  "query": {    "bool": {      "should": [        {          "match_phrase": {            "content": "佟大"          }        },        {          "match_phrase": {            "content.standard": "佟大"          }        }      ]    }  }}
```

为了方便你记忆和使用，这里我把字段细节总结在如下这张表格中。

<table data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)"><thead data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)"><tr data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(25, 25, 25)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)" data-style="border-width: 1px 0px 0px; border-right-style: initial; border-bottom-style: initial; border-left-style: initial; border-right-color: initial; border-bottom-color: initial; border-left-color: initial; border-top-style: solid; border-top-color: rgb(204, 204, 204); background-color: white;"><th data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(40, 40, 40)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)|rgb(240, 240, 240)" data-style="border-top-width: 1px; border-color: rgb(204, 204, 204); text-align: left; background-color: rgb(240, 240, 240); font-size: 14px; min-width: 85px;"><br data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(40, 40, 40)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)|rgb(240, 240, 240)"></th><th data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(40, 40, 40)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)|rgb(240, 240, 240)" data-style="border-top-width: 1px; border-color: rgb(204, 204, 204); text-align: left; background-color: rgb(240, 240, 240); font-size: 14px; min-width: 85px;"><br data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(40, 40, 40)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)|rgb(240, 240, 240)"></th><th data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(40, 40, 40)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)|rgb(240, 240, 240)" data-style="border-top-width: 1px; border-color: rgb(204, 204, 204); text-align: left; background-color: rgb(240, 240, 240); font-size: 14px; min-width: 85px;"><br data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(40, 40, 40)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)|rgb(240, 240, 240)"></th></tr></thead><tbody data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)"><tr data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(25, 25, 25)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)" data-style="border-width: 1px 0px 0px; border-right-style: initial; border-bottom-style: initial; border-left-style: initial; border-right-color: initial; border-bottom-color: initial; border-left-color: initial; border-top-style: solid; border-top-color: rgb(204, 204, 204); background-color: white;"><td data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(25, 25, 25)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)" data-style="border-color: rgb(204, 204, 204); font-size: 14px; min-width: 85px;"><span data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(25, 25, 25)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)">核心参数</span></td><td data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(25, 25, 25)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)" data-style="border-color: rgb(204, 204, 204); font-size: 14px; min-width: 85px;"><span data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(25, 25, 25)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)">默认值</span></td><td data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(25, 25, 25)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)" data-style="border-color: rgb(204, 204, 204); font-size: 14px; min-width: 85px;"><span data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(25, 25, 25)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)">释义</span></td></tr><tr data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(32, 32, 32)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(248, 248, 248)" data-style="border-width: 1px 0px 0px; border-right-style: initial; border-bottom-style: initial; border-left-style: initial; border-right-color: initial; border-bottom-color: initial; border-left-color: initial; border-top-style: solid; border-top-color: rgb(204, 204, 204); background-color: rgb(248, 248, 248);"><td data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(32, 32, 32)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(248, 248, 248)" data-style="border-color: rgb(204, 204, 204); font-size: 14px; min-width: 85px;"><span data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(32, 32, 32)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(248, 248, 248)">enabled</span></td><td data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(32, 32, 32)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(248, 248, 248)" data-style="border-color: rgb(204, 204, 204); font-size: 14px; min-width: 85px;"><span data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(32, 32, 32)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(248, 248, 248)">true</span></td><td data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(32, 32, 32)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(248, 248, 248)" data-style="border-color: rgb(204, 204, 204); font-size: 14px; min-width: 85px;"><span data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(32, 32, 32)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(248, 248, 248)">仅适用于 Mapping 顶层以及 Object 对象，设置为 false 后该字段将不再被解析。</span></td></tr><tr data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(25, 25, 25)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)" data-style="border-width: 1px 0px 0px; border-right-style: initial; border-bottom-style: initial; border-left-style: initial; border-right-color: initial; border-bottom-color: initial; border-left-color: initial; border-top-style: solid; border-top-color: rgb(204, 204, 204); background-color: white;"><td data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(25, 25, 25)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)" data-style="border-color: rgb(204, 204, 204); font-size: 14px; min-width: 85px;"><span data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(25, 25, 25)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)">index</span></td><td data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(25, 25, 25)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)" data-style="border-color: rgb(204, 204, 204); font-size: 14px; min-width: 85px;"><span data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(25, 25, 25)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)">true</span></td><td data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(25, 25, 25)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)" data-style="border-color: rgb(204, 204, 204); font-size: 14px; min-width: 85px;"><span data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(25, 25, 25)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)">控制是否对字段值进行索引，设置为 false 的字段不能被查询。</span></td></tr><tr data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(32, 32, 32)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(248, 248, 248)" data-style="border-width: 1px 0px 0px; border-right-style: initial; border-bottom-style: initial; border-left-style: initial; border-right-color: initial; border-bottom-color: initial; border-left-color: initial; border-top-style: solid; border-top-color: rgb(204, 204, 204); background-color: rgb(248, 248, 248);"><td data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(32, 32, 32)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(248, 248, 248)" data-style="border-color: rgb(204, 204, 204); font-size: 14px; min-width: 85px;"><span data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(32, 32, 32)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(248, 248, 248)">doc_values</span></td><td data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(32, 32, 32)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(248, 248, 248)" data-style="border-color: rgb(204, 204, 204); font-size: 14px; min-width: 85px;"><span data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(32, 32, 32)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(248, 248, 248)">true</span></td><td data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(32, 32, 32)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(248, 248, 248)" data-style="border-color: rgb(204, 204, 204); font-size: 14px; min-width: 85px;"><span data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(32, 32, 32)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(248, 248, 248)">正排索引，除了 text 类型外的其他类型默认开启，用于聚合和排序分析。</span></td></tr><tr data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(25, 25, 25)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)" data-style="border-width: 1px 0px 0px; border-right-style: initial; border-bottom-style: initial; border-left-style: initial; border-right-color: initial; border-bottom-color: initial; border-left-color: initial; border-top-style: solid; border-top-color: rgb(204, 204, 204); background-color: white;"><td data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(25, 25, 25)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)" data-style="border-color: rgb(204, 204, 204); font-size: 14px; min-width: 85px;"><span data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(25, 25, 25)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)">fielddata</span></td><td data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(25, 25, 25)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)" data-style="border-color: rgb(204, 204, 204); font-size: 14px; min-width: 85px;"><span data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(25, 25, 25)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)">false</span></td><td data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(25, 25, 25)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)" data-style="border-color: rgb(204, 204, 204); font-size: 14px; min-width: 85px;"><span data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(25, 25, 25)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)">是否为 text 类型启动 fielddata，实现 text 字段排序和聚合分析。</span></td></tr><tr data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(32, 32, 32)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(248, 248, 248)" data-style="border-width: 1px 0px 0px; border-right-style: initial; border-bottom-style: initial; border-left-style: initial; border-right-color: initial; border-bottom-color: initial; border-left-color: initial; border-top-style: solid; border-top-color: rgb(204, 204, 204); background-color: rgb(248, 248, 248);"><td data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(32, 32, 32)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(248, 248, 248)" data-style="border-color: rgb(204, 204, 204); font-size: 14px; min-width: 85px;"><span data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(32, 32, 32)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(248, 248, 248)">store</span></td><td data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(32, 32, 32)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(248, 248, 248)" data-style="border-color: rgb(204, 204, 204); font-size: 14px; min-width: 85px;"><span data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(32, 32, 32)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(248, 248, 248)">false</span></td><td data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(32, 32, 32)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(248, 248, 248)" data-style="border-color: rgb(204, 204, 204); font-size: 14px; min-width: 85px;"><span data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(32, 32, 32)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(248, 248, 248)">是否存储该字段值。</span></td></tr><tr data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(25, 25, 25)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)" data-style="border-width: 1px 0px 0px; border-right-style: initial; border-bottom-style: initial; border-left-style: initial; border-right-color: initial; border-bottom-color: initial; border-left-color: initial; border-top-style: solid; border-top-color: rgb(204, 204, 204); background-color: white;"><td data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(25, 25, 25)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)" data-style="border-color: rgb(204, 204, 204); font-size: 14px; min-width: 85px;"><span data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(25, 25, 25)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)">coerce</span></td><td data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(25, 25, 25)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)" data-style="border-color: rgb(204, 204, 204); font-size: 14px; min-width: 85px;"><span data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(25, 25, 25)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)">true</span></td><td data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(25, 25, 25)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)" data-style="border-color: rgb(204, 204, 204); font-size: 14px; min-width: 85px;"><span data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(25, 25, 25)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)">是否开启自动数据类型转换功能，比如 字符串转数字、浮点转整型。true 代表可以转换，false 代表不可以转换。</span></td></tr><tr data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(32, 32, 32)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(248, 248, 248)" data-style="border-width: 1px 0px 0px; border-right-style: initial; border-bottom-style: initial; border-left-style: initial; border-right-color: initial; border-bottom-color: initial; border-left-color: initial; border-top-style: solid; border-top-color: rgb(204, 204, 204); background-color: rgb(248, 248, 248);"><td data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(32, 32, 32)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(248, 248, 248)" data-style="border-color: rgb(204, 204, 204); font-size: 14px; min-width: 85px;"><span data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(32, 32, 32)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(248, 248, 248)">fields</span></td><td data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(32, 32, 32)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(248, 248, 248)" data-style="border-color: rgb(204, 204, 204); font-size: 14px; min-width: 85px;"><span data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(32, 32, 32)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(248, 248, 248)">根据业务需要而定</span></td><td data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(32, 32, 32)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(248, 248, 248)" data-style="border-color: rgb(204, 204, 204); font-size: 14px; min-width: 85px;"><span data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(32, 32, 32)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(248, 248, 248)">灵活使用多字段解决多样的业务需求。</span></td></tr><tr data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(25, 25, 25)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)" data-style="border-width: 1px 0px 0px; border-right-style: initial; border-bottom-style: initial; border-left-style: initial; border-right-color: initial; border-bottom-color: initial; border-left-color: initial; border-top-style: solid; border-top-color: rgb(204, 204, 204); background-color: white;"><td data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(25, 25, 25)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)" data-style="border-color: rgb(204, 204, 204); font-size: 14px; min-width: 85px;"><span data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(25, 25, 25)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)">dynamic</span></td><td data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(25, 25, 25)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)" data-style="border-color: rgb(204, 204, 204); font-size: 14px; min-width: 85px;"><span data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(25, 25, 25)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)">true</span></td><td data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(25, 25, 25)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)" data-style="border-color: rgb(204, 204, 204); font-size: 14px; min-width: 85px;"><span data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(25, 25, 25)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(255,255,255)">控制 mapping 的动态自动更新。</span></td></tr><tr data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(32, 32, 32)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(248, 248, 248)" data-style="border-width: 1px 0px 0px; border-right-style: initial; border-bottom-style: initial; border-left-style: initial; border-right-color: initial; border-bottom-color: initial; border-left-color: initial; border-top-style: solid; border-top-color: rgb(204, 204, 204); background-color: rgb(248, 248, 248);"><td data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(32, 32, 32)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(248, 248, 248)" data-style="border-color: rgb(204, 204, 204); font-size: 14px; min-width: 85px;"><span data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(32, 32, 32)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(248, 248, 248)">date_detection</span></td><td data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(32, 32, 32)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(248, 248, 248)" data-style="border-color: rgb(204, 204, 204); font-size: 14px; min-width: 85px;"><span data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(32, 32, 32)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(248, 248, 248)">true</span></td><td data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(32, 32, 32)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(248, 248, 248)" data-style="border-color: rgb(204, 204, 204); font-size: 14px; min-width: 85px;"><span data-darkmode-color-16509382451261="rgb(163, 163, 163)" data-darkmode-original-color-16509382451261="#fff|rgb(63, 63, 63)" data-darkmode-bgcolor-16509382451261="rgb(32, 32, 32)" data-darkmode-original-bgcolor-16509382451261="#fff|rgb(248, 248, 248)">是否自动识别类型。</span></td></tr></tbody></table>

我们再来分析一下数据建模的流程，如下图所示。

![](https://mmbiz.qpic.cn/mmbiz_png/mjl8GCpsL9a2p3HdfNVKjklm5YTDab70bs0QR3eQ8H1TpJUiamBcSkedTxCxEoVWYTsMrgbAnr1IadPCdgc6STQ/640?wx_fmt=png)

数据建模的流程图

首先，根据业务选择合适的数据类型。

注意字符串类型分为两种 text 和 keyword 类型；尽量选择贴近实际大小的数据类型；nested 和 join [复杂类型](http://mp.weixin.qq.com/s?__biz=MzI2NDY1MTA3OQ==&mid=2247484382&idx=1&sn=da073a257575867b8d979dac850c3f8e&chksm=eaa82bf6dddfa2e0bf920f0a3a63cb635277be2ae286a2a6d3fff905ad913ebf1f43051609e8&scene=21#wechat_redirect)需根据业务特点选型，具体会在下一部分详细阐述。

其次，判定是否需要检索，如果不需要，index 设置为 false 即可。

然后，判定是否需要排序和聚合操作，如果不需要可以设置 doc_values 为 false。

最后，考虑一下是否需要另行存储，会结合使用 store 和  _source 字段。

Mapping 层面要**强调**的是：尽量不要使用默认的 dynamic 动态字段类型，强烈建议 strict 严格控制字段，避免字段 “暴涨” 导致不可预知的风险，比如字段数超过默认 1000 个的上限、磁盘大于预期的激增等。

### 2.5 基于复杂索引关联建模

要摒弃 MySQL 的多表关联建模思想，因为 MySQL 中的范式思想都不再适用于 Elasticsearch。回顾文章开头的几个多表关联问题，Elasticsearch 能提供的核心解决方案如下。

#### （1） 宽表方案

这是空间换时间的方案，就是允许部分字段冗余存储的存储方式。实战举例如下。

用户索引：user。

博客索引：blogpost。

一个用户可以发表多篇博客。按照传统的 MySQL 建表思想：两个表建立个用户外键，即可搞定一切。而对于 Elasticsearch，我们更愿意在每篇博文后面都加上用户信息（这就是宽表存储的方案），看似存储量大了，但是一次检索就能搞定搜索结果。

```
PUT user/_doc/1{  "name":     "John Smith",  "email":    "john@smith.com",  "dob":      "1970/10/24"}PUT blogpost/_doc/2{  "title":    "Relationships",  "body":     "It's complicated...",  "user":     {    "id":       1,    "name":     "John Smith"   }}GET /blogpost/_search{  "query": {    "bool": {      "must": [        {          "match": {            "title": "relationships"          }        },        {          "match": {            "user.name": "John"          }        }      ]    }  }}
```

#### （2） nested 方案

*   适用场景：1 对少量，子文档偶尔更新、查询频繁的场景。
    

如果需要索引对象数组并保持数组中每个对象的独立性，则应使用嵌套 Nested 数据类型而不是对象 Oject 数据类型。

[nested 文档](http://mp.weixin.qq.com/s?__biz=MzI2NDY1MTA3OQ==&mid=2247487231&idx=2&sn=fbe1b054b2f6eaa0d5e98911b837a51b&chksm=eaa826d7dddfafc1b7c9d1b4df4b0064ce62558458f08f4b7b4a1e00c4f691a9f63bb5fc4c10&scene=21#wechat_redirect)的优点是可以将父子关系的两部分数据（如博客 + 评论）关联起来，我们可以基于 nested 类型做任何的查询。但缺点是查询速度相对较慢，更新子文档需要更新整篇文档。

#### （3） [join 父子文档](http://mp.weixin.qq.com/s?__biz=MzI2NDY1MTA3OQ==&mid=2247483998&idx=1&sn=6c407a0e0a30c1237451ddd1b40f5c0b&chksm=eaa82a76dddfa3604354a969ec1520121294e188ea2ba5120a3a29026c379fb45f53394d4a27&scene=21#wechat_redirect)方案

*   适用场景：子文档数据量要明显多于父文档的数据量，存在 1 对多量的关系；子文档更新频繁的场景。
    

比如 1 个产品和供应商之间就是 1 对 N 的关联关系。当使用父子文档时，使用 has_child 或者 has_parent 做父子关联查询。优点是父子文档可独立更新，但维护 Join 关系需要占据部分内存，查询较 Nested 更耗资源。

注意：5.X 之前版本叫父子文档（多 type 实现），6.X 之后高版本是 join 类型（单 type 类型）。

#### （4） 业务层面实现关联

需通过多次检索获取所需的关键字段，业务层面自己写代码实现。

这里小结一下，以上四种方式便是 Elasticsearch 能实现的全量多表关联方案。实战建模阶段，一定要结合自己的业务场景，尽量往上靠，先通过 kibana dev tool 模拟实现，找到契合自己业务的多表关联方案。

此外我还要强调的是：多表关联都会有性能问题，数据量极大且检索性能要求高的场景需要慎用。这里我摘取了官方文档对应的描述如下，供你参考。

尤其应该[避免多表关联](http://mp.weixin.qq.com/s?__biz=MzI2NDY1MTA3OQ==&mid=2247484282&idx=1&sn=4f2aee1c83f7b3742ed34897b1b2390d&chksm=eaa82b52dddfa2448d850f00e9be5f8302df06a9b1fe3e0c071811f86fb84216aca4e803ec2a&scene=21#wechat_redirect)。Nested 嵌套可以使查询慢几倍，而 Join 父子关系可以使查询慢数百倍。

3、总结
----

最后，我们再来总结一下建模其他核心考量因素。

![](https://mmbiz.qpic.cn/mmbiz_png/mjl8GCpsL9a2p3HdfNVKjklm5YTDab70aFqzMP6TMGGoqy4mo5LlJXdWrFwSpXbaa4nRTFQEzSZibOsdMVKdL0w/640?wx_fmt=png)

*   尽量[空间换时间](http://mp.weixin.qq.com/s?__biz=MzI2NDY1MTA3OQ==&mid=2247484728&idx=1&sn=eeb76ad84c98af16fc16d6dc5d5d11af&chksm=eaa82d10dddfa406e5847d61a0ef2cd1230cb10185dd32bf0e1d1a64c7980658f0268264af96&scene=21#wechat_redirect)：能多个字段解决的不要用脚本实现。
    
*   尽量[前期数据预处理](http://mp.weixin.qq.com/s?__biz=MzI2NDY1MTA3OQ==&mid=2247485003&idx=1&sn=1f0d1c933d458bb9e3c71794d354aff1&chksm=eaa82e63dddfa77584bde5083adcdb4b33ef0c35fce8749dd6b6980ff92071b567a6a7ff93f7&scene=21#wechat_redirect)，不要后期脚本。优先选择 ingest process 数据预处理实现，尽量不要留到后面 script 脚本实现。
    
*   能指定路由的提前指定路由。写入的时候指定路由，检索的时候也同样适用路由。
    
*   能前置的尽量前置，让后面检索聚合更加清爽。比如 index sorting 前置索引字段排序是非常好的方式。
    

数据建模是 Elasticsearch 开发实战中非常重要的一环，也是项目管理角度中的设计环节的重中之重，你一定要重视！千万不要着急写业务代码，以 “**代码之前，设计先行**” 作为行动准绳。

感谢你的时间！有 Elasticsearch 建模问题欢迎留言交流。

本文成文于：2021 年 8 月 11 日

推荐
--

  

1[、重磅 | 死磕 Elasticsearch 方法论认知清单（2021 年国庆更新版）](http://mp.weixin.qq.com/s?__biz=MzI2NDY1MTA3OQ==&mid=2247487494&idx=1&sn=731687e8a09d2da56fa844c4e494ab62&chksm=eaa8382edddfb138627d276b4f60f4245d457a8a5f787daf65a0c0be95231f67b55d871b907f&scene=21#wechat_redirect)

[2](http://mp.weixin.qq.com/s?__biz=MzI2NDY1MTA3OQ==&mid=2247487494&idx=1&sn=731687e8a09d2da56fa844c4e494ab62&chksm=eaa8382edddfb138627d276b4f60f4245d457a8a5f787daf65a0c0be95231f67b55d871b907f&scene=21#wechat_redirect)[、](http://mp.weixin.qq.com/s?__biz=MzI2NDY1MTA3OQ==&mid=2247487072&idx=1&sn=13694eb4b907ae5cffa7ac37bb1ed248&chksm=eaa82648dddfaf5edb55fcc5ea74021367b81f03a22d9fb5655409adcb3e400e6da19f945ed3&scene=21#wechat_redirect)[如何从 0 到 1 打磨一门 Elasticsearch 线上直播课？](http://mp.weixin.qq.com/s?__biz=MzI2NDY1MTA3OQ==&mid=2247488915&idx=1&sn=a086a4f5a598f77d1c3a047c5d843cf0&chksm=eaa83dbbdddfb4adb8ef2573cd1f7fb725d250860b8f57616be38a0b7c900028c43abb411cd6&scene=21#wechat_redirect)（口碑不错）

3、[如何系统的学习 Elasticsearch ？](http://mp.weixin.qq.com/s?__biz=MzI2NDY1MTA3OQ==&mid=2247486294&idx=1&sn=50b43356e3a236df90773b4991819a9d&chksm=eaa8237edddfaa6828367df84d3697e99f3f3023d544bfb3446d9fe45d7b2e5e8ea0c89a4cbb&scene=21#wechat_redirect)

4、[Elasticsearch 是一把梭，用起来再说？！](http://mp.weixin.qq.com/s?__biz=MzI2NDY1MTA3OQ==&mid=2247484640&idx=1&sn=2e734c41667751f17261731692c1555b&chksm=eaa82cc8dddfa5dee4e33400306ee00bc3d3d697dd41f643444f2ffcd3c95309e07ff22d9286&scene=21#wechat_redirect)

5、[干货 | 论 Elasticsearch 数据建模的重要性](http://mp.weixin.qq.com/s?__biz=MzI2NDY1MTA3OQ==&mid=2247484159&idx=1&sn=731562a8bb89c9c81b4fd6a8e92e1a99&chksm=eaa82ad7dddfa3c11e5b63a41b0e8bc10d12f1b8439398e490086ddc6b4107b7864dbb9f891a&scene=21#wechat_redirect)

![](https://mmbiz.qpic.cn/mmbiz_png/mjl8GCpsL9ZoCQGlxWp4G56gqia6ANT1Z9WB22YUEZ2Xib9YBZ90fLUQLyudxgjENibxzn9wCtBx3sQeE5CJnRE1Q/640?wx_fmt=png)

更短时间更快习得更多干货！  

和全球 1600+ Elastic 爱好者一起精进！

![](https://mmbiz.qpic.cn/mmbiz_gif/mjl8GCpsL9aP1cRicwD8ibiaWicGzrrI5hFt9BVtE6mkbaIePyxJ1ic0SicaUEzTI3mhrgjNvLvJprDmf8Sqk9EphbRw/640?wx_fmt=gif)

比同事抢先一步学习进阶干货！