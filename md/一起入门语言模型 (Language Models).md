> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [zhuanlan.zhihu.com](https://zhuanlan.zhihu.com/p/32292060)

> 语言模型（Language Model, LM）任务毫无疑问是自然语言处理领域的核心问题，正所谓历史是最好的老师。本文回顾了语言模型发展史上的几个里程碑式工作: N-gram LM、FeedForward Neural Network LM、RNN LM 和 GPT 系列。希望和大家一起掌握历史发展规律，以便更早的洞悉未来发展方向。

语言模型
----

语言模型起源于语音识别 (speech recognition)，输入一段音频数据，语音识别系统通常会生成多个句子作为候选，究竟哪个句子更合理？就需要用到语言模型对候选句子进行排序。如今语言模型的应用范围早已扩展到机器翻译、信息检索、问答、文摘等众多 NLP 领域。

那么，什么是语言模型呢？一句话，语言模型是这样一个模型：**对于任意的词序列，它能够计算出这个序列是一句话的概率**。举俩例子就明白了，比如词序列 A：“知乎 | 的 | 文章 | 真 | 水 | 啊”，这个明显是一句话，一个好的语言模型也会给出很高的概率，再看词序列 B：“知乎 | 的 | 睡觉 | 苹果 | 好快”，这明显不是一句话，如果语言模型训练的好，那么序列 B 的概率就很小很小。

大概知道了语言模型是怎么一回事，下面给出较为正式的定义。假设我们要为中文创建一个语言模型， $V$V 表示词典， $V=$V= {猫, 狗, 机器, 学习, 语言, 模型,...}， $w_{i} \in V$w_{i} \in V 。语言模型就是这样一个模型：**给定词典** $V$V **，能够计算出任意单词序列** $w_{1},w_{2},...,w_{n}$w_{1},w_{2},...,w_{n} **是一句话的概率** $p(w_{1},w_{2},...,w_{n})$p(w_{1},w_{2},...,w_{n}) **，其中，**$p\geq 0$p\geq 0 。

现在问题来了，语言模型如何计算 $p(w_{1},w_{2},...,w_{n})$p(w_{1},w_{2},...,w_{n}) ？ 最简单的方法是数数 [[1]](#ref_1)，假设训练集中共有 $N$N 个句子，我们数一下在训练集中 $(w_{1},w_{2},...,w_{n})$(w_{1},w_{2},...,w_{n}) 出现的次数，不妨假定为 $n$n ，则 $p(w_{1},w_{2},...,w_{n}) = \frac{n}{N}$p(w_{1},w_{2},...,w_{n}) = \frac{n}{N} 。

可以想象出这个模型的预测能力几乎为 0，一旦单词序列没有在训练集中出现过，模型的输出概率就是 0，显然相当不合理。

我们可以根据概率论中的链式法则 (chain rule) 把 $p$p 展开：

$$p(w_{1},w_{2},...,w_{n}) = p(w_{1})\prod_{i=2}^{n}p(w_{i}|w_{1},...,w_{i-1})\\$$

p(w_{1},w_{2},...,w_{n}) = p(w_{1})\prod_{i=2}^{n}p(w_{i}|w_{1},...,w_{i-1})\\

如果能计算 $p(w_{i}|w_{1},...,w_{i-1})$p(w_{i}|w_{1},...,w_{i-1}) ，那么就能轻松得到 $p(w_{1},w_{2},...,w_{n})$ p(w_{1},w_{2},...,w_{n}) ，所以在文献中我们经常看到语言模型的另一种等价定义：**能够计算** $p(w_{i}|w_{1},...,w_{i-1})$p(w_{i}|w_{1},...,w_{i-1}) **的模型就是语言模型**。

> 从文本生成角度来看，我们也可以给出如下的语言模型定义：给定一个短语 (一个词组或一句话)，语言模型可以生成(预测) 接下来的一个词。

N-gram 语言模型
-----------

在统计学模型横行 NLP 的时代，语言模型任务的扛把子是 N-gram 语言模型。为了简化 $p(w_{i}|w_{1},...,w_{i-1})$p(w_{i}|w_{1},...,w_{i-1}) 的计算，我们引入一阶马尔可夫假设 (first-order Markov assumption)：**每个词只依赖前一个词，**$p(w_{i}|w_{1},...,w_{i-1})\approx p(w_{i}|w_{i-1})$p(w_{i}|w_{1},...,w_{i-1})\approx p(w_{i}|w_{i-1}) 。

此时，

$p(w_{1},w_{2},...,w_{n}) \\ = p(w_{1})\prod_{i=2}^{n}p(w_{i}|w_{1},...,w_{i-1}) \\ \approx p(w_{1})\prod_{i=2}^{n}p(w_{i}|w_{i-1})$p(w_{1},w_{2},...,w_{n}) \\ = p(w_{1})\prod_{i=2}^{n}p(w_{i}|w_{1},...,w_{i-1}) \\ \approx p(w_{1})\prod_{i=2}^{n}p(w_{i}|w_{i-1})

我们也可以引入二阶马尔可夫假设：**每个词依赖前两个词。** $p(w_{i}|w_{1},...,w_{i-1}) \approx p(w_{i}|w_{i-2},w_{i-1})$p(w_{i}|w_{1},...,w_{i-1}) \approx p(w_{i}|w_{i-2},w_{i-1})

此时，

$p(w_{1},w_{2},...,w_{n}) \\ = p(w_{1})\prod_{i=2}^{n}p(w_{i}|w_{1},...,w_{i-1}) \\ \approx p(w_{1})p(w_{2}|w_{1})\prod_{i=3}^{n}p(w_{i}|w_{i-2},w_{i-1})$p(w_{1},w_{2},...,w_{n}) \\ = p(w_{1})\prod_{i=2}^{n}p(w_{i}|w_{1},...,w_{i-1}) \\ \approx p(w_{1})p(w_{2}|w_{1})\prod_{i=3}^{n}p(w_{i}|w_{i-2},w_{i-1})

有了马尔可夫假设，就可以方便的计算条件概率。接下来我们看一下什么是 N-gram 语言模型？

以 N=3 的 tri-gram 语言模型为例，它使用二阶马尔可夫假设， $p(w_{i}|w_{1},...,w_{i-1}) \approx p(w_{i}|w_{i-2},w_{i-1})$p(w_{i}|w_{1},...,w_{i-1}) \approx p(w_{i}|w_{i-2},w_{i-1}) ，对于 $p(w_{i}|w_{i-2},w_{i-1})$p(w_{i}|w_{i-2},w_{i-1}) ，如何得到它的概率值呢？答案还是数数！

$$p(w_{i}|w_{i-2},w_{i-1})=\frac{count(w_{i-2},w_{i-1},w_{i})}{count(w_{i-2},w_{i-1})}\\$$

p(w_{i}|w_{i-2},w_{i-1})=\frac{count(w_{i-2},w_{i-1},w_{i})}{count(w_{i-2},w_{i-1})}\\

其中 count(*) 表示 * 在训练集中出现的次数。总结一下，N-gram 语言模型有两个要点：

*   使用 N-1 阶马尔可夫假设简化后验概率 $p$p ，提高模型的泛化能力
*   使用数数法计算后验概率 $p$p

在 N-gram 模型中，每一个 $p(w_{i}|w_{i-n+1},...,w_{i-2},w_{i-1})$p(w_{i}|w_{i-n+1},...,w_{i-2},w_{i-1}) 就类比机器学习模型中的一个参数，所以，N-gram 模型的一大弊端就是要学习的参数太多: $|V|^{N-1}$|V|^{N-1}，实打实的指数爆炸，大量的参数根本没有在训练集中出现过，也就是 $count(w_{i-N+1},...,w_{i-1},w_{i})=0$count(w_{i-N+1},...,w_{i-1},w_{i})=0 ，导致在进行预测时，很多句子的概率都是 0，为了避免这种情况发生，大量的研究工作致力于对 count(*)=0 的情况做平滑处理 [[2]](#ref_2)，最简单的方法是所有词组出现次数加 1。

> 可不要小瞧了数数法，里面蕴含着最大似然估计 (Maximum Likelihood Estimation) 的思想。假设 N-gram 的参数 $p(w_{i}|w_{i-N+1},...,w_{i-1})$p(w_{i}|w_{i-N+1},...,w_{i-1}) 服从多类别 (categorical) 分布，数数法的结果就是最大似然估计给出的结果。对运用最大似然估计求解多类别分布参数推导过程感兴趣的可以戳此链接 [最大似然估计之 - 伯努利分布、二项分布、多类别分布、多项分布](https://zhuanlan.zhihu.com/p/32341102)

前馈神经网络语言模型 (FeedForward Neural Network Language Models)
-------------------------------------------------------

> Many hard problems remain open, like how to allow AI systems to automatically learn high-level semantics—in other words, how to represent more abstract concepts and the meaning of not just words but more complex thoughts, such as those we express in a phrase, a sentence or a paragraph. Yoshua Bengio

N-gram 语言模型将词看作离散变量，使用 one-hot 进行表示，由此带来两大弊端：

*   词与词之间不存在语义关联，简单来讲，从相似性的角度，{"猫", "狗"}要比 {"猫", "电脑"} 更相似，但是使用 one-hot 表示后，任意两个词之间的距离都相等，则距离度量失去意义，相似性关系荡然无存，这就导致 N-gram 语言模型的**泛化能力** (generalization) 受到极大限制，凡是训练集中没有出现过的就不是句子。
*   N-gram 模型的参数数量是指数级： $|V|^{N-1}$|V|^{N-1} ，为了减少参数量，只能减小 $V$V 和 $N$N 的取值， $V$V 减小要去除很多低频词导致模型效果有所下降，最致命的是 $N$N 减小让 N-gram 模型彻底在**长距离依赖** (long dependency) 问题上败下阵来，一般 $N$N 的取值也就是 3~5，面对 “Yann LeCun | 出生 | 在 | 法国 | 他 | 提出了 | LeNet | 他 | 会说 | 英语 | 和 |？” 这样的问题，束手无策。

前馈神经网络语言模型 [[3]](#ref_3) 通过结合词向量 (word embedding) 和前馈神经网络来解决上面两个问题：

*   每个词用低维稠密向量表示，这就使得语义相似的词对应的向量在空间中相邻成为可能（前提是词向量训练的效果达到预期），给模型带来了泛化能力上的提升，举个例子，测试集中有一句话 “猫 | 在 | 厨房 | 吃 | 鱼”，训练集中没有出现过，但是有一句“狗 | 在 | 卧室 | 吃 | 肉”，由于{"猫", "狗"}, {"厨房", "卧室"}, {"鱼", "肉"} 语义相似，模型可以推断 “猫 | 在 | 厨房 | 吃 | 鱼” 是一句话的概率很高。
*   语言模型归根到底是要计算 $p(w_{i}|w_{1},...,w_{i-2},w_{i-1})$p(w_{i}|w_{1},...,w_{i-2},w_{i-1}) ，(前馈) 神经网络强大的学习能力很适合拟合概率分布，并且 $w_{i}$w_{i} 所依赖的上下文信息 (context) 也会比 N-gram 中的更长。

![](https://pic3.zhimg.com/v2-85c30744367f54c8e31e5c034fd40106_r.jpg)

从上图中可以看出网络结构很简单，将 $w_{i}$w_{i} 前 n-1 个词的向量进行拼接作为网络输入，经过一次非线性变换，最后输出字典中每个词的概率作为预测结果。

$$y=b+W\cdot x+U\cdot tanh(d+H\cdot x)\\ p(w_{i}|w_{t-1},...,w_{t-n+1})=\frac{e^{y_{w_{t}}}}{\sum_{i}e^{y_{i}}}\\$$

y=b+W\cdot x+U\cdot tanh(d+H\cdot x)\\ p(w_{i}|w_{t-1},...,w_{t-n+1})=\frac{e^{y_{w_{t}}}}{\sum_{i}e^{y_{i}}}\\

图中的绿色虚线对应公式中的 $W\cdot x$W\cdot x ，在实验阶段，虚线是可选的，通过将 $W$W 设置为 0 来消除虚线。

在上世纪 90 年代就已经有神经网络语言模型相关的工作 [[4]](#ref_4)，词向量的概念 (分布式语义表示, distributed representation) 更是在上世纪 80 年代就已成型 [[5]](#ref_5)，但是毫无疑问 Yoshua Bengio 在 2003 年的工作 <A Neural Probabilistic Language Model> 是一篇集大成者，不论是创新、写作还是实验，在 20 年前，计算资源还很受限的情况，为了在 14million 的数据上训练神经网络语言模型，还是相当需要功夫的，在这之后，越来越多人开始意识到神经网络语言模型的威力，逐渐进入了学界主流。

循环神经网络语言模型 (RNN Language Models)
--------------------------------

> You shall know a word by the company it keeps (Firth,J. R. 1957:11).

上下文信息 (context) 决定了词的语义，而前馈神经网络语言模型（FFNNLM）同 N-gram 类似，每个 $w_{i}$w_{i} 只依赖前 n 个词，无疑限制了 FFNNLM 的性能，有没有更适合语言模型任务的网络结构？显然，RNN 是个相当不错的选择，序列的问题就该让序列模型去解决。[得嘞您内：读 PyTorch 源码学习 RNN（1）](https://zhuanlan.zhihu.com/p/32103001)

说起 RNNLM，就不得不提到一个人 Tomas Mikolov，博士期间专注于 RNNLM 的研究：包括 RNNLM 的训练、在标准数据集上的对比评估、训练加速技巧等各方面，也就不难理解为什么是 Mikolov 开发了 word2vec，因为词向量在语言模型中太重要了。建议对语言模型领域感兴趣的同学都去读一下 Mikolov 的博士论文 [[6]](#ref_6)。

从方法创新角度来看，从 FFNNLM 到 RNNLM，似乎只是模型上的普通过渡：用 RNN 模型替换了 FFNN，或者是再用 LSTM 替换 RNN。然而，在这看似平平无奇的模型迭代更新之下，一场新的方法论革命正在悄悄地酝酿、发酵。

2015 年 Andrew M. Dai 和 Quoc V. Le 在论文 <Semi-supervised Sequence Learning>[[7]](#ref_7) 提出了对 LSTM 使用语言模型任务进行预训练，在下游任务微调的思路。具体来讲，Dai 设计了两个预训练任务：语言模型和序列自编码。语言模型不必多讲，序列自编码任务的含义是使用 LSTM 网络对序列 $XYZ$XYZ 进行编码，再解码得到 $XYZ$XYZ 的过程。

![](https://pic2.zhimg.com/v2-8c624780f115cd587473c9260b120739_r.jpg)

预训练得到的 LSTM 模型，再用带标签的数据集微调，在多个情感分类 / 文本分类任务上得到了 SOTA 的结果，受限于历史局限性，本文研究的对象是 LSTM 模型，并且大量的实验场景预训练使用的数据集和下游微调任务的数据集一致，作者也是更多的从模型参数初始化的角度来解释，但是作者仍然前瞻性的分析了预训练数据集规模带来的影响：大规模预训练数据集能够带来效果的提升。

我个人认为 Dai 和 Le 的工作，具有相当大的历史意义，这种使用语言模型任务预训练模型，再微调的方法论简直可以称之为 RNN 时代的 BERT/GPT。

两年后，显然是受到 Dai 和 Le 的影响，GPT1 和 GPT2 的一作 Alec Radford 也对 mLSTMLM 进行了探索 [[8]](#ref_8)，差不多的套路，差不多的结论，预训练语言模型有效果，但还没有好到让人眼前一亮，总感觉差那么临门一脚。

GPT 系列
------

> GPT GPT，听说你要开辟新的天地？  
> GPT GPT，听说你要把微调都丢弃？  
> GPT GPT，是谁给你的勇气？  
> 因为我的参数有 1700 亿！

2017 年是 NLP 领域的大年，这一年 Transformer 横空出世 [[9]](#ref_9)，强大的模型拟合能力迅速席卷了 sequence-to-sequence 的各项任务，Transformer Encoder 主攻特征表示，Transformer Decoder 擅长文本生成，二者合则天下无敌，分则各自为王。

![](https://pic2.zhimg.com/v2-2094fd910fc97402da14b4aab9fa7d55_r.jpg)

OpenAI 的 GPT 系列就对 Transformer Decoder 作为语言模型的能力进行了探索，Alec Radford 等人高举无监督预训练的大旗，首先登场的是 GPT-1 模型 [[10]](#ref_10)，在规模约 5GB 的数据集上，使用 Transformer Decoder 替换 RNN 进行语言模型任务，然后在下游任务对预训练好的模型进行微调，并且对下游任务也不挑剔，甭管你是分类还是问答，没有语言模型解决不了的问题，如果有，那就把训练数据序列化，梭哈干就完事了！和 RNNLM 预训练 + 微调的套路相似，但是这一次，预训练数据集更大了，模型更强了，效果也更显著了，15 个下游任务，有 12 个都超过了当时的 SOTA，并且多个任务效果提升明显。

![](https://pic4.zhimg.com/v2-2de321d05e763e6872b7b330bbfc9a3f_r.jpg)

作为无监督学习痴汉，在 GPT-1 中作者就有想拿掉微调阶段的想法了，注意看下面右图 "Sentiment Analysis" 对应的紫色线，在预训练之后，即使不进行微调，也能达到 baseline 80% 的效果！

![](https://pic2.zhimg.com/v2-205ef22d0ec950382fa1000085071339_r.jpg)

大约半年后，GPT-2 登场了 [[11]](#ref_11)，预训练数据集由 5GB 扩大到了 40GB(并且足够 diverse)，模型参数由 1.1 亿扩大到 15 亿，这一次，作者彻底按捺不住了，直接去掉了微调阶段，只分析预训练得到的 GPT-2 在下游任务的效果 (zero-shot)。从实验结果来看，毫无疑问 GPT-2 作为通用语言模型是相当合格的，在 8 个不同的语言模型相关数据集上测试，竟然在 7 个数据集上的效果超过了当时的 SOTA！

![](https://pic4.zhimg.com/v2-095a59c984f5fda60396995dab7fbcbb_r.jpg)

显然，作者不想止步于语言模型一个任务，又继续在阅读理解、文本摘要、机器翻译和问答等任务上测试 GPT-2 的效果，重点提一点阅读理解数据集 CoQA 的效果，F1 竟然达到了 55，已经超过了多个监督模型:) 当然其他任务上也就比随机好一点点。

就在 OpenAI 不断打磨 GPT 系列的同时，其他科技巨头也没闲着，更大规模的模型不断涌现，一场围绕预训练模型的军备竞赛正如火如荼进行，时间来到 2020 年 5 月份，OpenAI 携 GPT 系列强势回归：1700 亿参数量的 GPT-3[[12]](#ref_12)！

![](https://pic2.zhimg.com/v2-eb21d1dc878b21ec589b42494f758df9_r.jpg)

这一次，作者们岂不是要将 zero-shot 进行到底？！再一看 GPT-3 论文题目，"Language Models are Few-Shot Learners"，怎么这么 “低调”？仅仅是 few-shot？再一看，哦，原来一作换人了...

GPT-3 参数量比 GPT-2 足足大了 100 多倍，未完待续

我第一次读 GPT-2 论文时，感觉太简单粗暴，没啥新意，后期 GPT-3 工作的出现，才逐渐理解了 GPT 系列的志向，作为坚定的无监督学习拥护者，作者不想止步于预训练后还要微调的现状，一个通用模型是否能够直接回答所有 NLP 问题？而语言模型作为文本理解的基础，确实也存在成为终极模型的可能性，GPT 系列则让更多人接受了这种可能性的存在，或许在未来，GPT-X 就是答案:)

参考
--

1.  [^](#ref_1_0)Language Modeling, Course notes for NLP by Michael Collins, Columbia University [http://www.cs.columbia.edu/~mcollins/lm-spring2013.pdf](http://www.cs.columbia.edu/~mcollins/lm-spring2013.pdf)
2.  [^](#ref_2_0)An Empirical Study of Smoothing Techniques for Language Modeling  [https://www.aclweb.org/anthology/P96-1041.pdf](https://www.aclweb.org/anthology/P96-1041.pdf)
3.  [^](#ref_3_0)[https://www.jmlr.org/papers/volume3/bengio03a/bengio03a.pdf](https://www.jmlr.org/papers/volume3/bengio03a/bengio03a.pdf)
4.  [^](#ref_4_0)Natural Language Processing with Modular Neural Networks and Distributed Lexicon [http://cogprints.org/1896/](http://cogprints.org/1896/)
5.  [^](#ref_5_0)Learning distributed representations of concepts [http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.408.7684&rep=rep1&type=pdf](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.408.7684&rep=rep1&type=pdf)
6.  [^](#ref_6_0)Statistical Language Models Based on Neural Networks [https://www.fit.vutbr.cz/~imikolov/rnnlm/thesis.pdf](https://www.fit.vutbr.cz/~imikolov/rnnlm/thesis.pdf)
7.  [^](#ref_7_0)Semi-supervised Sequence Learning [https://arxiv.org/pdf/1511.01432.pdf](https://arxiv.org/pdf/1511.01432.pdf)
8.  [^](#ref_8_0)Learning to Generate Reviews and Discovering Sentiment [https://arxiv.org/pdf/1704.01444.pdf](https://arxiv.org/pdf/1704.01444.pdf)
9.  [^](#ref_9_0)Attention Is All You Need [https://arxiv.org/pdf/1706.03762.pdf](https://arxiv.org/pdf/1706.03762.pdf)
10.  [^](#ref_10_0)Improving Language Understanding with Unsupervised Learning [https://openai.com/blog/language-unsupervised/](https://openai.com/blog/language-unsupervised/)
11.  [^](#ref_11_0)Better Language Models and Their Implications [https://openai.com/blog/better-language-models/](https://openai.com/blog/better-language-models/)
12.  [^](#ref_12_0)Language Models are Few-Shot Learners [https://arxiv.org/pdf/2005.14165.pdf](https://arxiv.org/pdf/2005.14165.pdf)