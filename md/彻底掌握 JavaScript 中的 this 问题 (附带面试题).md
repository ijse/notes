> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7350866242007580687?utm_source=gold_browser_extension)

掘金的小伙伴们大家好，我是 fox。这一篇文章是想通过自己一些对于 JavaScript 中 this 学习，做一个简单的汇总笔记，便于自己深度掌握理解。无论是入坑多年一时大意被 this 问题背刺的技术泰斗，或是刚入坑对 this 模糊不清的编程新星，都希望这篇文章能够给予各位一定的帮助！

### 一、在编程界，this 是什么

1.  在常见面向对象的编程语言中，都有 this 这个关键字，比如 Java、C++ 等等。
2.  this 通常只会出现在类的方法中，也就是你需要有一个类，在类的方法中 this 代表的是当前调用的对象。
3.  **但是在 JavaScript 中的 this 更加灵活，无论是它出现的位置还是它所代表的含义。**

### 二、JavaScript 中 this 的作用，为什么需要 this？

> 我们通过编写一个 user 对象来观察有 this 和没有 this 的区别

```
const user = {
  id: 1,
  name: 'luckyCoder'，
  address: '猿星'，
  // 使用this
  eating: function() {
    console.log(`${this.name}在${this.address}吃东西~`)
  },
  // 不使用this
  running: function() {
   console.log(`${user.name}在${user.address}跑步~`)
  }
}
user.eating() // luckyCoder在猿星吃东西~
user.running() // luckyCoder在猿星跑步~
```

> 我们通过调用 user 对象中的方法可以发现，如果不使用 this，很多问题也是有解决的方案的，我们可以直接通过点语法的方式获取到 name 和 address 的属性值。**但是这种解决方案是存在弊端的，假设对象名称 user 发生了改变。**

```
// 假设对象名称user改为info
const info = {
  id: 1,
  name: 'luckyCoder'，
  address: '猿星'，
  // 使用this
  eating: function() {
    console.log(`${this.name}在${this.address}吃东西~`)
  },
  // 不使用this
  running: function() {
   // 此处也需要同步进行修改为info
   console.log(`${info.name}在${info.address}跑步~`)
  }
}
info.eating() // luckyCoder在猿星吃东西~
info.running() // luckyCoder在猿星跑步~
```

> 通过这个案例我们发现，如果在不使用 this 的情况下修改了对象的名称，那么我们必须将其内部的方法所使用的名称同步修改，而使用 this 则不需要考虑这个问题。因此我们可以得出一个结论：**从某些角度来说，开发中如果没有 this，很多问题是有解决方案的，但是会让我们编写代码变得非常不方便。**

### 三、this 指向什么呢？

> 我们先说一个最简单的，将以下代码片段在浏览器中执行，观察 this 在全局作用域下指向什么？

```
console.log(this) // window
var name = "luckyCoder" // 通过var所创建的变量会存在于window中
console.log(this.name) // luckyCoder
console.log(window.name) // luckyCoder
```

> 但是在实际开发中很少直接在全局作用域下去使用 this，通常都是在函数中使用，而在函数中，this 的值是**动态绑定**的，**只有在函数被调用时，this 的值才会绑定上去**。因此函数不同的调用方式 this 也会有不同的值。我们通过编写一个 demo 函数，使用不同的方式调用，通过浏览器来执行，观察函数中 this 的指向。

```
function demo() {
  console.log(this)
}

// 1.直接调用这个函数
demo() // window

// 2.创建一个对象，对象中的函数指向demo
const obj = {
  name: 'luckyCoder',
  demo: demo
}
obj.demo() // obj对象

// 3.通过apply调用
demo.apply("abc") // String{"abc"}对象
```

> 我们发现同样一个函数，以不同的方式调用了三次，this 对应了三个不同的值，因此我们可以得出一个结论：**this 指向什么，跟函数所处的位置是没有关系的，跟函数被调用的方式是有关系的。**

#### 通过上述案例，我们可以总结为以下几点

1.  函数在调用时，JavaScript 会默认给 this 绑定一个值
2.  this 的绑定和定义的位置（函数编写的位置）没有关系
3.  this 的绑定和调用方式以及调用的位置有关系
4.  this 是在运行时被绑定的

#### 那么 this 到底是怎么样的绑定规则呢？我们来逐个击破！

*   绑定规则一：**默认绑定**
*   绑定规则二：**隐式绑定**
*   绑定规则三：**显示绑定**
*   绑定规则四：**new 绑定**

### 规则一：**默认绑定**

> **通常独立的函数调用情况下会使用默认绑定规则**，独立的函数调用我们可以理解成函数没有被绑定到某个对象上进行调用，我们通过几个案例来看一下，常见的默认绑定。

#### 案例一：

```
function foo() {
  console.log(this)
}
foo() // 独立的函数调用 this => window
```

#### 案例二：

```
function foo1() {
  console.log(this)
}

function foo2() {
  console.log(this)
  foo1() // 独立的函数调用
}

function foo3() {
  console.log(this)
  foo2() // 独立的函数调用
}

foo3() // 三个函数的 this => window
```

#### 案例三：

```
const obj = {
  name: "luckyCoder",
  // foo函数在obj中定义
  foo: function() {
    console.log(this)
  }
}

// 创建一个bar并且将obj的foo属性值赋值给它
const bar = obj.foo

bar() // 独立的函数调用 this => window
```

#### 案例四：

```
function foo() {
  console.log(this)
}

// 创建一个obj对象 定义一个foo属性 并将全局下的foo函数作为属性值
const obj = {
  name: 'luckyCoder',
  foo: foo
}

// 创建一个bar并且将obj的foo属性值赋值给它
const bar = obj.foo

bar() // 独立的函数调用 this => window
```

#### 案例五：

```
function foo() {
  function bar() {
    console.log(this)
  }
  return bar
}

const fn = foo()

fn() // 独立的函数调用 this => window
```

> 我们通过案例发现，无论函数在**定义时**通过那些方式，或进行某些复杂的赋值，**只要函数的调用方式是独立的，没有任何主题的，那么就符合 this 默认绑定的规则，在浏览器中该函数的 this 指向的就是 window**。

### 规则二：**隐式绑定**

> 另外一种比较常见的调用方式是通过某个对象进行调用的，**将函数作为某个对象的方法，也就是它的调用位置中，是通过某个对象发起的函数调用**。这样的调用过程就会使用隐式绑定规则，我们通过几个案例来看一下，常见的隐式绑定。

#### 案例一：

```
function foo() {
  console.log(this)
}

foo() // 独立的函数调用 this => window

// 创建一个对象 对象中的函数指向foo
const obj = {
  name: "luckyCoder"
  foo: foo
}

obj.foo() // 通过对象点语法的方式调用函数 this => obj对象
```

#### 案例二：

```
const user = {
  id: 1,
  name: 'luckyCoder'，
  address: '猿星'，
  eating: function() {
    console.log(`${this.name}在${this.address}吃东西~`)
  }，
  running: function() {
    console.log(`${this.name}在${this.address}跑步~`)
  }
}

user.eating() // 通过对象点语法的方式调用函数 this => user对象 
user.running() // 通过对象点语法的方式调用函数 this => user对象

// 即我们在函数体中的this.name和this.address
// this为user对象，则所获取到的属性也是来自于user
```

#### 案例三：

```
const obj1 = {
  name: 'obj1',
  foo: function() {
    console.log(this)
  }
}

const obj2 = {
  name: 'obj2',
  bar: obj1.foo
}

obj2.bar() // 通过对象点语法的方式调用函数 this => obj2对象
```

> 我们通过案例可以发现，当我们通过 object.fn() 的方式调用某个函数时，**object 对象会被 JavaScript 引擎绑定到 fn 函数中的 this，这个绑定的过程是内部自动绑定的，我们无法看到其内部 this 绑定的过程，所以称之为隐式绑定**。

### 规则三：**显示绑定**

> 我们通过对隐式绑定规则的学习可以发现，它的特征是必须在调用的对象内部有一个对函数的引用，正是通过这个引用，间接的将 this 绑定到了这个对象上。**如果我们不希望在对象内部包含这个函数的引用，同时又希望在这个对象上进行强制调用，该怎么做呢？ 在 JavaScript 中所有的函数都可以使用 call 和 aplly 以及 bind 方法，通过这些方法可以帮助我们实现显示绑定规则。** 我们通过几个案例来看一下，常见的显示绑定规则。

#### 案例一：

```
function foo() {
  console.log(this)
}

// foo直接调用指向的是全局对象(浏览器 => window)
foo() // this => window

// foo函数的原型对象中有JavaScript帮助我们实现的call/apply方法 可以直接使用
foo.call() // this => window
foo.apply() // this => window
foo.call(null) // this => window
foo.apply(null) // this => window
foo.call(undefined) // this => window
foo.apply(undefined) //this => window
// 上述调用皆等同于foo()

// foo直接调用和使用call/apply调用 
// 在不传递任何参数或参数为null/undefined的情况下是一样的
// 它们的区别就在于 通过call/apply可以通过传递第一个参数 
// 使被调用的函数的this指向这个参数
```

#### 案例二：

```
function foo() {
  console.log(this)
}

const obj = {
  name: "obj"
  foo: foo // 通过foo.apply(obj)调用时 这段代码可以省略
}

// 通过对隐式绑定的学习 我们知道 如果希望foo函数被调用时this是指向obj的
// 那么我们需要给obj添加一个foo属性并且属性值指向foo函数 
// 再进行对象点语法的方式调用
// 从而达到隐式绑定规则 this则指向obj
obj.foo() // this => obj

// 那么我们也可以使用call/apply的方式来实现显示绑定规则
// 并且也无需在obj中创建foo属性
foo.apply(obj) // 显示绑定规则 this => obj

// call/apply是可以指定this的绑定对象
```

#### 案例三：call 和 apply 的区别

```
const obj = {}

function sum(num1, num2) {
  console.log(num1 + num2, this)
}

// 当我们在调用类似sum这类函数时 是需要传递一些参数的
// 如果希望通过call/apply来调用并指定this的绑定对象 同时也需要传递一些参数时
// 就会用到call/apply的第二个参数 他们的不同之处也在于传递参数的方式
sum(10, 20) // 30, window

// 使用剩余参数的方式传递
sum.call(obj, 20, 30) // 将sum函数中的this显示绑定给obj 并且传递两个参数20, 30
// 使用数组的方式传递
sum.apply(obj, [20, 30]) // 将sum函数中的this显示绑定给obj 并且传递两个参数20, 30
```

#### 案例四：bind 方法

```
const obj = {}

function foo() {
  console.log(this)
}

// 当通过foo函数的bind方法进行调用时 会返回一个新的函数
// 新的函数的this就是在调用bind方法时传入的参数obj
const newFn = foo.bind(obj)

// 看起来像是一个独立的函数调用 
// 但是这个独立函数在调用之前 通过bind方法显示的绑定了一个obj对象
newFn() // this => obj

// 相当于默认绑定和显示绑定bind冲突了 但显示绑定的优先级会更高
```

> 通过案例我们发现，call 和 aplly 方法作用类似，第一个参数可以是一个对象，在调用这个函数时，会将 this 绑定到这个传入的对象上。第二个参数用法上有所差异，call 使用剩余参数的方式传递而 apply 使用数组的方式传递。而 bind 方法则会返回一个新的函数，新的函数的 this 同样为 bind 方法的第一个参数。**虽然这三个方法在用法上虽然有所区别，但是它的 this 绑定过程我们是可以观测到的，所以称之为显示绑定。**

### 规则四：**new 绑定**

> 在学习 new 绑定规则之前，首先最好有 JavaScript 面向对象编程的基础。在 JavaScript 中，函数是可以当做一个类的构造函数来使用，也就是使用 new 关键字，当使用 new 关键字来调用函数时，会创建一个全新的对象，这个对象会被执行 prototype 连接，**并且这个新对象会绑定到函数调用的 this 上，这个绑定过程称之为 this 的 new 绑定**。还是通过几个案例，来观察 new 绑定下的 this。

#### 案例一

```
function Person() {
  // 伪代码
  const obj = {}
  this = obj
  return obj
}

// 在使用new关键字调用函数时 
// 在函数内部会生成一个对象 
// 并且会将生成的这个对象 赋值给函数的this 
// 最终会将这个对象返回(return)

const p = new Person() // 那么我们就可以在外部拿到这个对象
```

```
function Person(name, age) {
  // 往创建的对象中添加属性
  this.name = name
  this.age = age
  
  // 因构造函数被调用了两次 产生了两个新对象 
  // 第一次this为所创建出的a对象
  // 第二次this为所创建出的b对象
  console.log(this)
}

// 调用构造函数并传入参数 返回新的对象
const a = new Person('a', 18)
const b = new Person('b', 20)

console.log(a) // => {name: 'a', age: 18} 
console.log(b) // => {name: 'b', age: 20}
```

> 当我们通过一个 new 关键字调用一个函数时，这个时候 this 是在调用这个函数时创建出来的对象 这个绑定过程就是 new 绑定

### 四、规则的优先级

> 学习了四条规则之后，在实际开发中我们只需要去查找函数的调用应用了哪条规则即可判断 this 的指向，**但是如果一个函数调用位置应用了多条规则，优先级谁更高呢？** 例如以下代码：

```
const obj = {
  foo: function() {
    console.log(this)
  }
}

// 调用foo函数时 既有隐式绑定又有new绑定
new obj.foo()
```

#### 显示绑定高于隐式绑定

> 我们先记住一个结论：**默认规则的优先级是最低的**，因为存在其他规则时，就会通过其他规则的方式来绑定 this，而**显示绑定优先级高于隐式绑定**，我们通过案例来进行论证。

```
const obj = {
   name: 'obj',
   foo: function() {
     console.log(this)
   }
 }
 
 const info = {}
 
 // 通过obj对象点语法的方式拿到foo函数并使用call/apply方法来调用
 obj.foo.call(info) // this => info对象
 obj.foo.apply(info) // this => info对象
```

```
function foo() {
  console.log(this)
}

const info = {}

const obj = {
  name: 'obj',
  foo: foo.bind(info)
}

// 通过对象点语法的方式获取到foo函数 
// 而foo函数的值是已经被bind绑定之后返回的新函数
obj.foo() // this => info
```

#### new 绑定高于隐式绑定

```
const obj = {
  name: 'obj'
  foo: function() {
    console.log(this)
  }
  
  const f = new obj.foo() // this => foo函数所创建的对象
}
```

#### new 绑定高于显示绑定

```
// 首先要知道new关键字是不能和apply/call一起来使用的
// 因为new和apply/call都是用于主动调起一个函数的
// 所以使用new和bind方法来进行比较

function foo() {
  console.log(this)
}

const info = {}

const bar = foo.bind(info)

const f = new bar() // this => foo函数所创建的对象
```

> 通过案例我们可以得出最终结论：this 的绑定规则优先级为，**new 绑定 > 显示绑定 > 隐式绑定 > 默认绑定**

### 五、箭头函数的 this

> 箭头函数不可以使用 new 关键字进行调用，并且也不适用于其他三个绑定规则，如果在箭头函数使用了 this，那么 this 引用就会从上层作用域中查找到对应的 this，我们通过案例来进行观察。

```
const foo = () => {
  // 等同于我们在函数中使用某个变量 
  // 在函数中找不到的话则沿着作用域链向上查找
  console.log(this) 
}

const obj = {
  foo: foo
}

const info = {}

foo() // this => window
obj.foo() // this => window
foo.call(info) // this => window
const f = new foo() // 箭头函数不可以使用new关键字进行调用 报错
```

### 六、this 相关面试题

> 相信看到这里，大家对 this 的问题一定有了一定的掌握，那么尝试做几道面试题检验一下自己的学习成果吧。在这里我就不附加对应的答案了，大家可自行运行代码对照。

#### 题一

```
const name = 'window'

const person = {
  name: 'person',
  sayName: function() {
    console.log(this.name)
  }
}

functin sayName() {
  const a = person.sayName
  a()
  person.sayName()
  (person.sayName)()
  (b = person.sayName)()
}

sayName()
```

#### 题二

```
const person1 = {
  name: 'person1',
  foo1: function() {
    console.log(this.name)
  },
  foo2: () => console.log(this.name),
  foo3: function() {
    return function() {
      console.log(this.name)
    }
  },
  foo4: function() {
    return () => {
      console.log(this.name)
    }
  }
}

const person2 = { name: 'person2' }

person1.foo1()
person1.foo1.call(person2)

person1.foo2()
person1.foo2.call(person2)

person1.foo3()()
person1.foo3.call(person2)()
person1.foo3().call(person2)

person1.foo4()()
person1.foo4.call(person2)()
person1.foo4().call(person2)
```

至此本篇已经完结啦，此篇大量内容采用了 coderwhy 老师的 JavaScript 高级课程以及公众号 this 相关的文章，大家如果感兴趣可以自行查阅了解更多的知识。如果你成功的做出了这两道面试题，相信在以后面试笔试实际场景中在碰到 this 绑定相关的也是手拿把掐了，希望你能够继续保持学习的热情，不惧怕前端的知识复杂、琐碎，达成新的高度。