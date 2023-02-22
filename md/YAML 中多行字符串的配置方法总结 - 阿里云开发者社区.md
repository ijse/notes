> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [developer.aliyun.com](https://developer.aliyun.com/article/826760)

> YAML 中多行字符串的配置方法总结

YAML 中多行字符串的配置方法总结
==================

2021-12-08 441 举报

**简介：** YAML 中多行字符串的配置方法总结

+ 关注继续查看

有时候我们会在配置文件中设置一段文字说明，这时通常会出现两种需求：

1.  文字中可能出现段落，希望在配置中按段落方式编写，显示打印的时候也能出现段落换行。
2.  文字很长，为方便编辑，可能在配置文件中分段写，但是显示的时候不喜欢出现配置中的段落换行。

简单的说，就是：

1.  配置与显示，都严格按段落展示
2.  配置按段落，显示不需要按段落

假设，我们需要配置这样一段文字：

```
I am a coder.My blog is didispace.com.
```

下面，就针对上面的两种情况来看看可以怎么来实现：

[](https://blog.didispace.com/yaml-string-multi-line/#%E9%85%8D%E7%BD%AE%E4%B8%8E%E6%98%BE%E7%A4%BA%EF%BC%8C%E9%83%BD%E4%B8%A5%E6%A0%BC%E6%8C%89%E6%AE%B5%E8%90%BD%E5%B1%95%E7%A4%BA)配置与显示，都严格按段落展示
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

这个需求下，我们希望配置和显示都按句子换行，就是这样：

```
I am a coder.

My blog is didispace.com.
```

### 方法一：直接使用`\n`来换行

这样写：

```
string: "I am a coder.\n\

         My blog is didispace.com."
```

最终输出：

```
I am a coder.

My blog is didispace.com.
```

通过`\n`在显示的时候换行，通过配置行末的`\`让这个字符串换行继续写（这个必须有，如果没有第二行行首会多一个空格）。

**注意**：这里必须使用双引号来定义字符串，不能用单引号。因为单引号是不支持`\n`换行的。

### [](https://blog.didispace.com/yaml-string-multi-line/#%E6%96%B9%E6%B3%95%E4%BA%8C%EF%BC%9A%E4%BD%BF%E7%94%A8%EF%BD%9C%E3%80%81%EF%BD%9C-%E3%80%81%EF%BD%9C)方法二：使用`｜`、`｜+`、`｜-`

在方法一种，其实我们在文字中加入了几个转义符号，其实对于阅读并不方便。在方法二中，将介绍更适合阅读的几种形式：

```
string1: |

  I am a coder.

  My blog is didispace.com.



string2: |+

  I am a coder.

  My blog is didispace.com.



string3: |-

  I am a coder.

  My blog is didispace.com.
```

如上面一共有三种配置都会自动按配置中所写的换行来换行，但是在文末会有一些区别，有的会增加一个空行，有的不会，有的会新增两个空行，具体说明如下：

*   `|`：文中自动换行 + 文末新增一空行
*   `|+`：文中自动换行 + 文末新增两空行
*   `|-`：文中自动换行 + 文末不新增行

[](https://blog.didispace.com/yaml-string-multi-line/#%E9%85%8D%E7%BD%AE%E6%8C%89%E6%AE%B5%E8%90%BD%EF%BC%8C%E6%98%BE%E7%A4%BA%E4%B8%8D%E9%9C%80%E8%A6%81%E6%8C%89%E6%AE%B5%E8%90%BD)配置按段落，显示不需要按段落
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

这个需求下，我们希望配置里是按行写的，但是显示是如下面这样在一行的：

```
I am a coder.My blog is didispace.com.
```

### 方法一：直接在字符串中换行写

最粗暴的写法，反正不用换行，那就直接写了：

```
string: 'I am a coder.

         My blog is didispace.com.'
```

这里不论用双引号还是单引号都是可以的。因为不存在需要转移的内容，所以总体还算清晰。

### [](https://blog.didispace.com/yaml-string-multi-line/#%E6%96%B9%E6%B3%95%E4%BA%8C%EF%BC%9A%E4%BD%BF%E7%94%A8-gt-%E3%80%81-gt-%E3%80%81-gt)方法二：使用`>`、`>+`、`>-`

比较好的表述方式就是使用`>`、`>+`、`>-`来定义，比如下面这几种：

```
string1: >

  I am a coder.

  My blog is didispace.com.



string2: >+

  I am a coder.

  My blog is didispace.com.



string3: >-

  I am a coder.

  My blog is didispace.com.
```

这三种都不会对配置中的换行进行实际换行，但是依然在文末的处理会有一些小区别，具体如下：

*   `>`：文中不自动换行 + 文末新增一空行
*   `>+`：文中不自动换行 + 文末新增两空行
*   `>-`：文中不自动换行 + 文末不新增行

**版权声明：**本文内容由阿里云实名注册用户自发贡献，版权归原作者所有，阿里云开发者社区不拥有其著作权，亦不承担相应法律责任。具体规则请查看《[阿里云开发者社区用户服务协议](https://developer.aliyun.com/article/768092)》和《[阿里云开发者社区知识产权保护指引](https://developer.aliyun.com/article/768093)》。如果您发现本社区中有涉嫌抄袭的内容，填写[侵权投诉表单](https://yida.alibaba-inc.com/o/right)进行举报，一经查实，本社区将立刻删除涉嫌侵权内容。

[YAML 配置](https://www.aliyun.com/sswb/418244.html)

[开发者社区](/) > [云计算](/group/cloud/) > [文章](/group/cloud/article/)