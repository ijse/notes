> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [sspai.com](https://sspai.com/post/72951)

> 之前我在 能少一个是一个：我用 Raycast 替代了这些应用 这篇文章中介绍了一些我用 Raycast 替代了的应用。

之前我在 [能少一个是一个：我用 Raycast 替代了这些应用](https://sspai.com/post/72540) 这篇文章中介绍了一些我用 Raycast 替代了的应用。评论区收到了读者的一些问题，回复的同时也总结了一番，所以想再写一篇文章作为补充，主要是**针对 Quicklink 功能的问题解答和快捷小技巧**，让你使用起来更为方便。

首先是几点建议
-------

### 第一，尽可能完整地走一遍 Walkthrough

刚安装 Raycast，应用会置顶一个 Walkthrough 指令，也就是新用户指南。里面有介绍非常多的基础使用方法，甚至是用了很久可能都没有发现的小技巧。

![](https://cdn.sspai.com/2022/04/28/ceae40aa8e9a18c95c41bedfe84eb852.png)Walkthrough 里的新手任务

比如可以在 Quicklink 指令设置中添加一些链接的预设；再比如设置别名（alias）后，输入 alias + space 就可以快捷进入指令，不需要按多次回车。这些都是在 Walkthrough 可以学到的内容，一遍下来，所有任务都完成（或者了解过）后，就能大致知道 Raycast 能做些什么了。

> Q: 找不到 Walkthrough 怎么办？  
> A: 如果你已经完成所有任务，或先前按 `⌃` + `X` 忽略掉了它，也可以直接搜索 Walkthrough 来找到。

![](https://cdn.sspai.com/2022/04/28/43b015a64af34f999005dd163b1c902c.png)搜索 Walkthrough

### 第二，多用 `⌘` + `K` 看看有什么更多操作

每一个指令都有主操作和更多操作，即使是用惯了的指令，`⌘` + `K` 里面或许也藏着不少实用操作呢。又或者应用版本更新，也会带来一些新变化**（所以有能力的话，changelog 也可以好好读一下）**。

比如复制了图片，在剪贴板历史可以直接提取文字：

![](https://cdn.sspai.com/2022/04/28/09387ff28aacedc918d611cd7ab9b0ed.png)

比如商店中 Obsidian 的插件，可以直接在某个文档最后插入文字（而不需要打开 Obsidian 应用）：

![](https://cdn.sspai.com/2022/04/28/784d0225dca2a66c9cd8a0a6cbf9e832.png)图片来自插件介绍页

### 第三，也不妨看看每一个插件和指令的设置项

许多插件和指令在 `Raycast Preferences > Extensions` 中是有更多定制项的，记得翻一翻，会有不少新发现。

比如 File Search 可以设置仅查找「文件名」还是「文件名 + 内容」：

![](https://cdn.sspai.com/2022/04/28/07e4d9ed6b8a9a9ba412247b372f6b49.png)File Search 设定

比如剪贴板历史可以设置保留历史记录的时间、忽略的应用等（这个在上一篇文章有提到）：

![](https://cdn.sspai.com/2022/04/28/29ece7855df496a2e9869dc47a11b160.png)Clipboard History 设定

说完了整体的建议，下面将进入之前文章评论区最常见的一个问题 —— Quicklink 怎么好像还是不够 Quick？

Quicklink 怎么用起来更 Quick
----------------------

Quicklink 的最大用途之一，可能是用于「指定某个搜索引擎搜索」或「指定在某个网站执行站内搜索」。

比如「少数派站内搜索」就可以这样写：`https://sspai.com/search/post/{Query}` 。其中，`{Query}` 是搜索词的占位语。

![](https://cdn.sspai.com/2022/04/28/cb0a534177b0e1003c0063c6694d4476.png)Quicklink: 少数派站内搜索

这样再输入 Quicklink 的名称（如 Search SSPAI）和打开方式（用什么浏览器），保存后，我们在 Raycast 的主搜索框输入 `sspai` 等字样就可以使用这个搜索指令了：

![](https://cdn.sspai.com/2022/04/28/b43f94d2a4eea9d5b8862fc8c3cb0aca.png)使用刚刚创建的「少数派站内搜索」

用法很简单，不过默认的设置还是有一些问题，主要是**「搜索需要敲好几次回车，不流畅」**以及**「不能用一个快捷键搜索当前选中的关键字」**。以下是可行的解决办法：

### alias —— 解决「搜索需要敲好几次回车，不流畅」

按照上面的设置，「少数派站内搜索」是这样一个过程： `输入 sspai 定位到该 Quicklink > 回车进入指令 > 输入搜索词 > 回车执行搜索` 。实际上是两次回车，但如果不习惯回车键的话，敲击起来是比较麻烦的。（而且，假如你有不止一个与 `sspai` 相关联的指令，可能还需要上下移动焦点来选择需要的那个。）

我不太熟悉 Alfred，不过从读者的评论来看，「Alfred 可以敲一次回车或空格就进入搜索」，其实 Raycast 也是可以实现的，需要用到 alias（别名）。

Raycast 的别名功能，官方是这样描述的：

> Typing the alias, followed by a space, directly opens a Command, saving you an extra step.

也就是说，**为指令设置别名后，只要输入「别名 + 空格键」，就可以直接进入这个指令了，不需要按回车**（按空格就行，空格按起来相对方便一些）。**而且别名是唯一的，不会与其他相似指令混淆。**

比如设置「少数派站内搜索」的别名为 `ssp`，那么只要输入 `ssp` + `space` 就可以进入指令了。

![](https://cdn.sspai.com/2022/04/28/551af574031396876b39761e81aaf5f6.gif)别名 + 空格

  
`ssp + 空格键 + 搜索词 + 回车键` ，这一串的操作可比一开始简化不少。（语言描述可能不太直观，建议大家自己试一下。）

> 另外，别名也不仅适用于 Quicklink，在其他指令如 File Search、Search Menu Items 上也是可以简化操作步骤的，方法和效果同样。

### Quick Search —— 解决「不能用一个快捷键搜索当前选中的关键字」

什么意思呢，还是以「少数派站内搜索」为例，比如我高亮选中了一个词，**最好按一个快捷键就可以唤起少数派搜索，并且搜索词是我选中的这个词**：

![](https://cdn.sspai.com/2022/04/28/7f0d96aac5fc6b15c179ef2d100cd260.png)

这个功能也是可以实现的，需要

1.  在 Quicklink 偏好设置中开启 Quick Search: Pass selected text as argument（将选中的词作为搜索参数）。
2.  为需要开启快速搜索的 Quicklink 设置快捷键。

![](https://cdn.sspai.com/2022/04/28/d6141052743e17d0af40faf155f2420e.png)开启 Quick Search![](https://cdn.sspai.com/2022/04/28/0d6dc68b81b48dc5a6d2229d3421fa2e.png) 为需要开启 Quick Search 的 Quicklink 设置快捷键

**两者缺一不可。**Quick Search 打开、快捷键设置好后，就可以实现 `选中某个词 > 按下快捷键 > 直接跳转搜索结果` 这个 chain 了，非常方便。

那以上就是主要的两个问题的解答了。再介绍几个不那么重要、但建议你可以了解的小技巧：

### Add to Favorites or Fallback

常用的 Quicklink，比如你经常使用 Search SSPAI，就可以把它加入 Favorites 置顶：

![](https://cdn.sspai.com/2022/04/28/8f08022cd6ace568ad9c434f7f68f6bb.png)按 ⌘ + K 可以找到加入 Favorites 的操作![](https://cdn.sspai.com/2022/04/28/0452e4f1aec8732f625358aa44ab58cd.png)置顶后的效果

也可以加入 Fallback，意思是当没有匹配的指令时，Raycast 会向你展示一系列默认指令，它们通常由内建的一些搜索（如 File Search、Search Menu Items）和 Quicklink 组成。

比如输入「iOS 待办应用推荐」这样一句话，看起来就不是什么指令，就可以在 Fallback 里面选要执行什么操作。这时如果选择「Search DuckDuckGo」，就会打开鸭鸭走的「iOS 待办应用推荐」搜索结果页面了。

![](https://cdn.sspai.com/2022/04/28/4223eabda3057e3195ba4a7ae60a0cd5.png)Fallback Commands

如果要定制 Fallback Commands，可以搜索 Manage Fallback Commands。所有你添加的 Quicklink 都可以设置为 Fallback，也可以改变显示顺序。

![](https://cdn.sspai.com/2022/04/28/216cd094cccfda9283c4026114dd20ac.png)Manage Fallback Commands

（向大家请教 fallback 应该翻译成什么好？）

### 同一个 link 可以添加好几条，用不同浏览器打开

这个比较简单，只是一个思路。比如同时在用两个浏览器，都需要「Search DuckDuckGo」，就可以写两条，名称上做区分，打开方式分别为两个浏览器就可以了。

当然你也可以用 [Browserosaurus](https://browserosaurus.com/) 等方式，只使用一个默认浏览器，所有 link 都通过它来判断打开方式。（Quicklink 的打开方式选择 Browserosaurus 即可；两种方法各有千秋。）

![](https://cdn.sspai.com/2022/04/28/article/3981ab17f964bb9124cfcba084727f0b)Browserosaurus

### 暂时隐藏和完全删除 Quicklink

也比较简单。暂时隐藏就是让这个 Quicklink 不要出现在 Raycast 的查找结果中，只要在 Extensions 设置中把勾选去掉就可以了：

![](https://cdn.sspai.com/2022/04/28/253e74dd3909912a85b050a716de76e9.png)暂时隐藏 Quicklink

完全删除 Quicklink 则需要 `右键 > Delete`。（同样，完整卸载商店插件也是 `右键 > Uninstall`。）

![](https://cdn.sspai.com/2022/04/28/1349494eff8ed46e28e6f55254786c9c.png)完全删除 Quicklink

那说了这么多 Quicklink，还是总结一下它的用法好了。

Quicklink 可以做什么
---------------

### 简单使用，主要是这 3 种

*   直接打开某一个网址。比如：`https://sspai.com/`
*   以 `{Query}` 作为搜索词的占位语，来执行特定网站的搜索。比如上文提到的「少数派站内搜索」：`https://sspai.com/search/post/{Query}` 。（很像浏览器添加搜索引擎的方法，只不过浏览器用的是 `%s`。）

![](https://cdn.sspai.com/2022/04/28/3af3bc54e1722b4b3f7e8e5f8f2219fc.png)Brave 添加搜索引擎

*   打开文件或文件夹路径。比如：`~/Downloads`。（也可以直接点击右侧按钮选择。）

![](https://cdn.sspai.com/2022/04/28/6a995275ebfea94bbadaf3e1b1adcd19.png)打开文件或文件夹

### 配合 URL Schemes 使用（同样可以带参数）

既然 Quicklink 用来打开链接，那么 URL Schemes 自然也是一种链接，一样可以与 Quicklink 配合使用。

我以深受大家喜爱的文本记录工具 Drafts 为例。 [Drafts 提供了许多 URL Schemes](https://docs.getdrafts.com/docs/automation/urlschemes) ，比如创建一个 Draft 的 URL Scheme 是这样的：

![](https://cdn.sspai.com/2022/04/28/ced347c9f9a6bb6449c7cc5a8e6f1801.png)

把 `text=` 后面的部分使用 `{Query}` 占位，就成了 `drafts://x-callback-url/create?text={Query}`。

![](https://cdn.sspai.com/2022/04/28/c9ee3b547352c4f8b10d88ae924ca473.png)

这样创建一个 Create New Draft 的 Quicklink，按喜好设置别名或快捷键，就可以在 Raycast 里面直接创建新的 Draft 了。

![](https://cdn.sspai.com/2022/04/28/18fc0a90bf642f47f2a50e3a7ac46559.png)

创建完成后，会打开 Drafts 应用，展示刚刚创建的这一条内容：

![](https://cdn.sspai.com/2022/04/28/efcb8b92dd62beecace3b5030300cfd1.png)

不过这样操作还是麻烦的，它有几个问题：

1.  输入框太小，不能多行文本。
2.  参数只能输入一个，因为占位语 `{Query}` 只有一个嘛。但是其实这个 URL Scheme 是可以有多个参数的，比如你想同时输入 `text` 和 `tag` 就不行了。

![](https://cdn.sspai.com/2022/04/28/497320ce8c8413da778cf88168c72d3e.png)其实这个 URL Scheme 是可以有多个参数的

所以，使用起来可能还没有 Drafts 自带的快捷键方便：

![](https://cdn.sspai.com/2022/04/28/fb03d8d12bbf62c416c3e12f79700ff2.png)Drafts 自带的快捷键

比如 Quick capture shortcut，可以唤起一个小窗口来输入：

![](https://cdn.sspai.com/2022/04/28/b8f87185b67836333c214aa59af05464.png)Quick Capture 的小窗

然而，我觉得这个 Create New Draft 的 Quicklink 最实用的地方是**配合上文提到的 Quick Search 来使用**。

### Quick Search + URL Schemes

Quick Search 这个名称可能会有一些误导，其实它并不仅仅用来「Search 搜索」，而是可以**将当前高亮选中的内容作为参数传递（pass）到我们的 Quicklink 中**。

![](https://cdn.sspai.com/2022/04/28/a5383d83fa00f074fab729ef3000a8dc.png)

所以，设置快捷键以后（比如 `⌃` + `D` 代表 Create New Draft），遇到什么想要 capture 的词句段落，就可以执行 `高亮选中 > 按下 ⌃ + D > 直接记录到 Drafts 并打开` 这个动作了，甚至省下了「复制到剪贴板」这一步。

> Tips: 可以把 Drafts 主窗口变小一点，放在角落，这样跳出来也不会打断浏览。

效果是这样的：

![](https://cdn.sspai.com/2022/04/28/6bceddb854d88329c20e54057616c638.png)1. 设置好快捷键![](https://cdn.sspai.com/2022/04/28/dc13572f483211b6f054327224704ec1.gif) 2. 浏览网页的时候只要「高亮选择 + 快捷键」就可以 capture 到 Drafts 中了

**这个 Quicklink 在浏览网页、查资料、读文章的时候真的非常实用。**全部剪到 Drafts，后续慢慢整理。（唯一的问题是，当原文本有**分段**的时候，这个 Quicklink 会失效… 所以**只能一段一段来剪藏**。）

![](https://cdn.sspai.com/2022/04/28/98a7fee1e03fa9f4ebad8a17b5e62758.png)像这样跨段选择就会失效

> 但是呢，我举这个例子是提供一个思路，就是说 Quicklink 可以配合 URL Schemes 使用，会有不一样的玩法；而 Drafts 因为它本身已经够方便了（它本身就是一个非常强大的效率工具），所以这个例子不明显。**各种自动化的方式结合起来是很美妙的！**

结语
--

我真的很喜欢 Raycast，很期待它每次的更新、更新了以后按 `⌘` + `space` 跳出来的 changelog 小惊喜，连它**不打断的静默安装**都让我觉得非常非常贴心。也很喜欢时不时去插件商店看看又上了什么好东西可以玩。

我倒是觉得，它能提高多大的生产力不重要（但确实使用以后已经提高一大段了），而是它让探索化繁为简、减少重复、提高效率的过程变得很有趣很快乐，这一点很重要。**用起来舒服很重要。**

至于生产力不生产力，我觉得最重要的还是自己本身的做事方法、专注能力、思考能力等，工具只是辅助，或者说像我一样「总是重复、总是用不聪明的方法完成一项任务」会觉得很烦躁、想折腾得好一点的时候，起到一个支持的作用。

Raycast 的官推发布过一张图片（也是用它们的[代码转图片工具 ray.so](https://ray.so/) 制作的），说：

![](https://cdn.sspai.com/2022/04/28/6c0e2bd38c3ff8bb9a334396fbde0e9a.png)

> We don't do marketing, we just tweet about it, and people share it with their friends and companies.
> 
> 我们也没有怎么宣传，我们就是发发推，然后大家就推荐给朋友和同事们了。

确实呢😆。

后续，我可能会慢慢更新一些新功能、好用的插件、Hidden Gems 等内容，也是为我喜欢的应用做一个宣传和总结吧～感谢阅读。

#### 关联阅读

*   [能少一个是一个：我用 Raycast 替代了这些应用](https://sspai.com/post/72540)
*   [极具潜力的效率启动器 App，Raycast 脚本功能详解](https://sspai.com/post/64339)
*   [新增悬浮便签、浏览器书签查询…… 快捷启动器 Raycast 更好用了](https://sspai.com/post/64192)

> 下载 [少数派 2.0 客户端](https://sspai.com/page/client)、关注 [少数派公众号](https://sspai.com/s/J71e)，解锁全新阅读体验 📰

> 实用、好用的 [正版软件](https://sspai.com/mall)，少数派为你呈现 🚀