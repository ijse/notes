> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7249933985563459640)

大家好，我卡颂。

下面这个`React`组件代码，用到 3 个`use`关键词，你理解他们的作用吗？

```
'use client'

function App() {
  using data = use(ctx);
  
  // ...
}
```

真是几天不写`React`，语法都看不懂了。本文就来聊聊这几个`use`关键词各自的意义。

欢迎加入[人类高质量前端交流群](https://juejin.cn/user/1943592291009511/pins "https://juejin.cn/user/1943592291009511/pins")，带飞

use client
----------

首先是位于代码顶部的`'use client'`声明，使用方式类似于严格模式的声明：

```
'use strict';
// 此处是严格模式下的JavaScript代码
```

`'use client'`声明是`RSC`（`React Server Component`，服务端组件）协议中的定义。

启用了`RSC`的`React`应用，所有组件默认在服务端渲染（可以通过`Next v13`体验），只有声明`'use client'`的组件文件，会在前端渲染。

假设我们的`React`应用组件结构如下，其中红色代表**服务端组件**，蓝色代表**客户端组件**（声明`'use client'`）：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/943000adea3d40d7be32b4165d053713~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

那么当应用打包后，D、E 组件会打包成独立文件。在前端，`React`可以直接渲染 A、B、C 组件。但是对于 D、E，需要以`JSONP`的形式请求回组件代码再渲染。

完整执行逻辑如下：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/db4be5af013643ffaea155156aa7b693~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

using 关键字
---------

接下来是`data`变量前的`using`关键字：

```
using data = use(ctx);
```

`using`关键字是`tc39`提案 [ECMAScript Explicit Resource Management](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Ftc39%2Fproposal-explicit-resource-management "https://github.com/tc39/proposal-explicit-resource-management") 提出的，用于为各种资源（内存、`I/O`等）提供统一的生命周期管理（何时分配、何时释放等）。

同时，`TS v5.2`率先引入了这个关键字。所以，接下来的讲解我们以`TS`中的`using`关键词为准。

`using`的作用有点类似`useEffect`的`destroy`函数。当我们在`useEffect`的`create`函数绑定了事件后，可以在`destroy`函数解绑：

```
function App() {
  useEffect(() => {
    console.log('这里是create函数')
    return () => {
      console.log('这里是destroy函数')
    }
  }, [])
}
```

类似的，当我们通过`using`关键词声明一个包含`[Symbol.dispose]`方法的对象后，当离开当前作用域时，声明的`[Symbol.dispose]`方法会执行：

```
{
  const getResource = () => {
    return {
      [Symbol.dispose]: () => {
        console.log('离开啦!')
      }
    }
  }
  using resource = getResource();
}
// 代码执行到这里会打印 离开啦!
```

在`[Symbol.dispose]`方法内主要执行一些释放资源的操作。

比如，当我们操作数据库时，如果要考虑**操作完断开数据库连接**，可能会写出如下代码：

```
const db = await connectDB();
try {
  // 执行数据库操作
} finally {
  // 断开数据库连接
  await db.close();
}
```

如果使用`using`关键词，代码如下：

```
const connect = async () => {
  const db = await connectDB();
  return {
    db,
    [Symbol.asyncDispose]: () => db.close()
  };
};

// 使用
{
  using { db } = await connect();
  // 执行数据库操作
} 
// 离开作用域自动断开连接
```

配合`async await`使用，可以降低**由于忘记释放资源造成内存泄漏**的可能性。

use 方法
------

最后是`React v18.3`之后发布的新原生`hook` —— `use`：

```
using data = use(ctx);
```

这个`hook`可以接收两种类型数据：

*   `React Context`

此时`use`的作用与`useContext`一样。

*   `promise`

此时如果这个`promise`处于`pending`状态，则最近一个祖先`<Suspense/>`组件可以渲染`fallback`。

比如，在如下代码中，如果`<Cpn />`组件或其子孙组件使用了`use`，且`promise`处于`pending`状态（比如请求后端资源）：

```
function App() {
  return (
    <div>
      <Suspense fallback={<div>loading...</div>}>
        <Cpn />
      </Suspense>
    </div>
  );
}
```

那么，页面会渲染如下结果：

```
<div>
  <div>loading...</div>
</div>
```

当请求成功后，会渲染`<Cpn />`。

总结
--

对于开篇提到的代码：

```
'use client'

function App() {
  using data = use(ctx);
  
  // ...
}
```

表示：

*   这是个客户端组件
    
*   如果传递给`use`的变量`ctx`是`React Context`，则`use`的作用等同于`useContext`
    
*   如果传递给`use`的变量`ctx`是`promise`，则配合最近的`<Suspense/>`使用
    
*   如果`use`的返回值包含`[Symbol.dispose]`，则`App`组件`render`完成后会执行`[Symbol.dispose]`方法
    

一个文件，三款`use`相关语法，你是不是已经懵逼了呢？