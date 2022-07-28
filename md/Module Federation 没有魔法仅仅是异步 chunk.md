> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [zhuanlan.zhihu.com](https://zhuanlan.zhihu.com/p/352936804)

相信大家已经对 Module Federation(MF) 这个 Webpack5 的特性有所了解了，当我们在惊讶于这个功能的革命性的同时，是否会觉得它是一种魔法呢？会不会觉得它是 Webpack5 的专属功能呢？不，我想说的是 Module Federation 没有魔法，仅仅是异步 chunk 罢了，同时它也不是 Webpack5 的专属功能。

本文会摒弃一些底层细节，从上层帮助大家去理解这项技术。

简略的介绍一下吧
--------

  
Module Federation（简称 MF）是一种让 “模块” 可以在多个应用之间 “联合” 起来使用的一种技术，在 MF 中，一个应用将会由多个互相独立的应用”所暴露出的模块“聚合而成，这种应用形态通常称为微前端。

业界已经存在一些优秀的微前端技术方案，如 qiankun/icestark 等，但是它们与 MF 有着本质的不同，那就是对 “微前端” 的定义：

<table data-draft-node="block" data-draft-type="table" data-size="normal" data-row-style="normal"><tbody><tr><th>方案</th><th>微的定义</th><th>微前端的定义</th></tr><tr><td>Module Federation</td><td>模块</td><td>由多个互相独立的模块聚合而成的应用</td></tr><tr><td>qiankun/icestark 等</td><td>应用</td><td>由多个互相独立的应用聚合而成的应用</td></tr></tbody></table>

定义上的不同决定了技术实现的不同：

<table data-draft-node="block" data-draft-type="table" data-size="normal" data-row-style="normal"><tbody><tr><th>方案</th><th>技术实现</th></tr><tr><td>Module Federation</td><td>模块本质上是 JS 代码片段，这种代码片段一般称为 chunk。因此，模块的聚合，实际上是 chunk 的聚合。</td></tr><tr><td>qiankun/icestark 等</td><td>应用本质上是 HTML，而在 SPA 中，HTML 又是 main.js 进行填充的。因此，应用的聚合，实际上是 main.js 的聚合。</td></tr></tbody></table>

技术实现的不同又决定了使用场景的不同：

<table data-draft-node="block" data-draft-type="table" data-size="normal" data-row-style="normal"><tbody><tr><th>方案</th><th>使用场景</th></tr><tr><td>Module Federation</td><td>是一种技术升级的创造性工作，有一定成本，目的是为了让系统具备更强大的能力。</td></tr><tr><td>qiankun/icestark 等</td><td>是一种维持现状的保守性工作，成本极小，目的是为了让系统拥有更长久的生命力。</td></tr></tbody></table>

那么谁才是微前端呢？我觉得都是，只不过微的粒度以及使用场景不同罢了。

先来说说 Webpack4
-------------

Module Federation 并不是一个从无到有的技术，而是基于 webpack4 现有功能的一项技术创新。要理解其中的原理，我们步子不能迈得太大，我们要先理解 webpack4 有哪些功能，然后再去理解 webpack5 是如何利用现有功能实现技术创新的。

我们首先要理解是：webpack 的编译时和运行时。

### 编译时

在这个阶段，我们只需要关注 webpack 的输出结果即可，主要包括：

*   index.html
*   main.js
*   runtime.js (一般会直接注入到 main.js 中)
*   同步 chunk
*   异步 chunk

这里面最关键的是 runtime.js 这个文件，它是 webpack 的运行时代码，也就是 webpack 需要在浏览器中运行的代码。

可能有同学会困惑为什么 webpack 要在浏览器中运行。需要注意的是，这里的运行并不是指 webpack 要在浏览器中进行编译，而是指 webpack 要在浏览器中运行模块解析的相关操作，因为 webpack 输出结果并不是 esm，而浏览器也没有解析 cjs 的功能，所以必须要额外提供一部分代码，从而能让 webpack 的输出结果能在浏览器中正常解析运行。这部分代码就是 webpack 运行时代码，即 runtime.js。

其他比较常见，就不一一解释了，有问题可以留言区评论。

### 运行时

所谓运行时，就是**使用 runtime.js 中定义的方法，去解析执行 main.js 中提供的模块**的过程。

这里就不贴图了，大家应该都见过，就是那一堆非常乱的代码，乱到我们很少去关注里面的内容。因为这些代码都是 webpack 在编译时通过 node 直接把字符串拼接成代码塞进去的，所以会很乱。

但是我们仔细研究下就会发现，这里面其实只有三样东西：

*   modulesMap：一个 Map，它的 key 是模块路径，value 是模块内容
*   require：一个方法，用来解析同步模块，它会从 modulesMap 中找
*   require.ensure：一个方法，用来解析异步模块，通过网络请求加载异步 chunk，并把其中的内容合并进 modulesMap

> 名字是随意起得，我们只需要知道它们的作用即可。

那么运行时的执行过程，又可以理解成，通过 require 解析并执行 modulesMap 中的入口模块的过程。

### Webpack4 下的项目

现在我们看一下基于这些东西，webpack4 下的项目是怎样的。

因为 webpack4 没有 MF 这样的功能，因此 webpack 运行时做的事情非常简单：

1.  require 模块并执行（第一个模块是入口模块），执行过程中
2.  如果遇到静态 import，回到 1
3.  如果遇到动态 import，前往 4
4.  先 require.ensure 把异步模块相关的异步模块加载到 modulesMap，回到 1
5.  开始执行用户代码

体现在实际代码大概是这样的：

```
// 静态import，转化为require，直接从modulesMap里拿
import React from 'react' 

// 动态import，转化为require.ensure，先异步获取，再从modulesMap里拿
const AsyncComponent = React.lazy(() => import('./async-component'))

// 模块解析完毕后，开始执行用户代码
const App = () => {
  return <div>
      <React.suspense>
          <AsyncComponent/>
      </React.suspense>
  </div>
}
React.render(<AsyncComponent/>, document.body)
```

webpack4 的运行时代码的功能就这么些，再就没有了，非常简单。

### Webpack5 下的项目

在配置了 MF 的 webpack5 项目下，情况就会复杂很多，因为模块的类型被扩充了。

在 webpack4 下，模块只有两种：**同步模块和异步模块**，而在 webpack5 下，基于这两种模块又衍生出两种模块：**同步共享模块和异步远端模块**。这样，为了更好地区分，我们在这里梳理一下：

<table data-draft-node="block" data-draft-type="table" data-size="normal" data-row-style="normal"><tbody><tr><th>模块类型</th><th>示例</th></tr><tr><td>同步非共享模块</td><td>import React from 'react'<br>不做任何 MF 配置的模块</td></tr><tr><td>同步共享模块（webpack5 特有）</td><td>import React from 'react'<br>同时把 React 设置为 MF 的共享模块：new ModuleFederationPlugin({shared: { react: { 一些配置} }})</td></tr><tr><td>异步本地模块</td><td>import ('./async-component')<br>就是通过动态 import 引用的本地模块</td></tr><tr><td>异步远端模块（webpack5 特有）</td><td>import('remote-app/async-component')<br>通过动态 import 引用的远端 (remote-app) 模块</td></tr></tbody></table>

那么新衍生出的这两种模块是做什么的呢？这里简单说一下。

<table data-draft-node="block" data-draft-type="table" data-size="normal" data-row-style="normal"><tbody><tr><td>同步共享模块</td><td>用于解决一个微前端系统的多个子系统之间依赖共享的问题，remote 应用可以直接复用 host 应用上已经加载的共享模块，而不需要再次下载。<br>它跟 externals 很像，但是更安全更强大。</td></tr><tr><td>异步远端模块</td><td>host 应用可以通过异步加载的方式使用的 remote 应用暴露出的模块。<br>这种模块内部也可能会使用到共享模块，比如上述'remote-app/async-component'会用到 React。这样当 host 应用加载这个远端模块时，这个模块便可直接复用 host 应用已经加载好的 React 资源了。</td></tr></tbody></table>

现在我们再来看看，在 webpack5 下的运行时会发生怎样的变化：

1.  require 模块并执行（第一个模块是入口模块），执行过程中
2.  如果遇到静态 import

1.  如果这个模块是**非共享同步模块**，回到 1
2.  如果这个模块是**共享同步模块**，前往 4

4.  如果遇到动态 import

1.  如果这个模块是**本地异步模块**，前往 4
2.  如果这个模块是**远端异步模块**，也前往 4

6.  先 require.ensure 把异步模块相关的异步模块加载到 modulesMap

1.  如果是来自 2.b，会初始化 shared-scope，用来处理依赖共享相关逻辑
2.  如果是来自 3.a，没有额外操作
3.  如果是来自 3.b，会利用 shared-scope 来复用依赖，实现依赖共享

先不用管具体细节，你只需要关注到 “这里依然还是那 4 个步骤”，只不过每个步骤多出了一些条件判断，用来处理多出来的那两种模块类型。

**也就是说，即便在 MF 中，不管是共享模块还是远端模块，其实还是使用的 require.ensure 去加载一些异步 chunk 罢了。只不过稍有不同的是，因为牵扯到依赖共享的逻辑，会有一个 shared-scope 的概念，用来实现依赖共享的相关逻辑。**

体现在实际代码大概是这样的：

```
new ModuleFederationPlugin({
  shared: {
    react: { 一些配置 }
  },
  remotes: {
    'remote-app': "remote-app2@http://localhost:3001/remoteEntry.js",
  },
})

// 1. 静态import，同时是共享模块，转化为require.ensure
// 2. 初始化shared-scope，用来处理依赖共享相关逻辑
import React from 'react' 

// 1. 动态import，同时是远端模块，转化为require.ensure
// 2. 利用shared-scope来复用依赖，实现依赖共享
const AsyncComponent = React.lazy(() => import('remote-app/async-component'))

// 模块解析完毕后，开始执行用户代码
const App = () => {
  return <div>
      <React.suspense>
          <AsyncComponent/>
      </React.suspense>
  </div>
}
React.render(<AsyncComponent/>, document.body)
```

到这里，我们可以看到，MF 的实现其实并没有魔法，仅仅是异步 chunk 罢了。这整个过程跟 webpack5 是没有绑定关系的，也就是说 MF 并非 webpack5 的专属功能，[Rollup](https://link.zhihu.com/?target=https%3A//github.com/module-federation/rollup-federation) 和 [webpack4](https://link.zhihu.com/?target=https%3A//github.com/webpack/webpack/issues/10352%23issuecomment-587162038) 都可以实现 MF。

到这里，本文的主要内容就结束了，希望能对你有所帮助。

### 剩下的细节

本文依然有很多地方没有刻意深入，主要有以下两点：

1.  为什么 “**共享同步模块**” 这种静态 import 也要是异步 chunk 而不是同步 chunk

因为如果是同步 chunk，那么这个模块就会与 main.js 绑定起来，试想一下如果你的应用会作为 remote 集成进别人的应用，你希望把这个模块强行塞给别人吗？还是说你把这个模块做成异步的，当集成进别人的应用时，别人根据依赖共享配置，动态决定是否使用这个模块？必然是后者。

2. shared-scope 是什么？

是一个对象，每个 MF 应用都会有一个自己的 shared-scope 对象，里面会维护着类似的信息：

```
// host初始化后的shared-scope
{
  react: {
    16.3.0: () => module
  },
  lodash: {
    1.4.0: () => module
  }
}
```

意思是说，如果你指定的 react 的依赖共享配置中，react 的版本 16.3.0 符合配置要求，那么你就可以直接用这个 16.3.0 对应的 module，而不需要重新下载资源。

同时，当 host 加载一个 remote 应用时，remote 应用也会有自己的 shared-scope，但是它不会重新建一个，而是直接在 host 的 shared-scope 的基础上进行初始化，类似这样：

```
// remote初始化后的shared-scope
{
  react: {
    16.3.0: () => module // 来自host
    16.3.1: () => module // 来自remote
  },
  lodash: {
    1.4.0: () => module // 来自host
    1.4.1: () => module // 来自remote
  }
}
```

此时，如果 remote 的 react 的依赖共享配置中，react 的版本 16.3.0 符合配置要求，那么 remote 就不会再去加载自己的 16.3.1 版本的 react，而是直接用 host 已有的 16.3.0 版本的 react。

> 关于依赖共享配置，可以直接看这个[例子](https://link.zhihu.com/?target=https%3A//github.com/module-federation/module-federation-examples/tree/master/basic-host-remote)

更多的细节还是需要大家自己研究，本文到此为止啦。

------------------- 2021-12-25 新增补充 ---------------------

@yzl2000

这位小哥在评论区中提出的一种特殊情况，即 host 和 remote 的 shared 的模块的版本是一样时，host 不会使用自己的模块，而是反而使用了 remote 的模块。

这个现象源自 webpack 对于共享模块的处理方式：

1.  host 先初始化自己的 shared-scope
2.  host 加载 remote，remote 基于 host 的 shared-scope 初始化自己的 shared-scope
3.  host 开始从 shared-scope 中加载 shared module

如果模块的版本是一样的，在第二步中 remote 会将 host 之前设置过的同版本的 module 给覆盖掉，进而导致 host 尝试从中读取 module 时，会读到 remote 的 module。

想要了解更为细致的共享模块处理方式，可以看看这个小哥的文章：

[https://ph3xmz5sya.feishu.cn/docs/doccnI0D08Ou6w2VX2gLkootBfh](https://link.zhihu.com/?target=https%3A//ph3xmz5sya.feishu.cn/docs/doccnI0D08Ou6w2VX2gLkootBfh)