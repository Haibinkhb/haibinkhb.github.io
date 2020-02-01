#### this

***

this有点复杂...很多问题没搞懂，简单总结下：

##### 默认绑定

还是这段代码，稍微修改下：

```js
var scope = "global scope";
function checkscope(){
    var scope2 = 'local scope';
    console.log(this.scope)
}
checkscope();
```

严格模式下会报错，非严格模式下会输出 global scope 。这是因为这里使用的是默认绑定规则：严格模式下 this 返回 undefined ，非严格模式下，this 的值为 undefined 的时候，其值会被隐式转换为全局对象。

>怎么知道这里应用了默认绑定呢？可以通过分析调用位置来看看 foo() 是如何调用的。在代码中，foo()是直接使用不带任何修饰的函数引用进行调用的，因此只能使用默认绑定，无法应用其他规则。
《你不知道的JavaScript上卷》

##### 隐式绑定

```js
function foo() {
    console.log( this.a );
}
var obj = {
    a: 2,
    foo: foo
};
obj.foo(); // 2
```

> 当 foo() 被调用时，它的落脚点确实指向 obj 对象。当函数引用有上下文对象时，隐式绑定规则会把函数调用中的 this 绑定到这个上下文对象。
《你不知道的JavaScript上卷》

严格来说这个对象obj只是有一个foo属性引用了foo() 这函数，这个函数并不属于这个对象。比如：

```js
function foo() {
    console.log( this.a );
}
var obj = {
    a: 2,
    foo: foo
};
var bar = obj.foo; // 函数别名！
var a = "oops, global"; // a 是全局对象的属性
bar(); // "oops, global"
```

虽然 bar 是 obj.foo 的一个引用，但是实际上，它引用的是 foo 函数本身，因此此时的bar() 其实是一个不带任何修饰的函数调用,所以应用了默认绑定。

##### 显式绑定

JavaScript 提供的绝大多数函数以及你自己创建的所有函数都可以使用 call(..) 和 apply(..) 方法。

它们的第一个参数是一个对象，它们会把这个对象绑定到this，接着在调用函数时指定这个 this。因为你可以直接指定 this 的绑定对象，因此我们称之为显式绑定。

```js
function foo() {
    console.log(this.a);
}
var obj = {
    a:2
}
foo.call(obj); // 2
foo.apply(obj); // 2
```

##### new绑定

```js
function foo(a) {
    this.a = a;
}
var bar = new foo(2);
console.log( bar.a ); // 2
```

使用 new 来调用 foo(..) 时，我们会构造一个新对象并把它绑定到 foo(..) 调用中的 this 上。new 是最后一种可以影响函数调用时 this 绑定行为的方法，我们称之为new 绑定。

优先级：

1. 函数是否在 new 中调用（new 绑定）？如果是的话 this 绑定的是新创建的对象。
var bar = new foo()
2. 函数是否通过 call、apply（显式绑定）或者硬绑定调用？如果是的话，this 绑定的是
指定的对象。
var bar = foo.call(obj2)
3. 函数是否在某个上下文对象中调用（隐式绑定）？如果是的话，this 绑定的是那个上
下文对象。
var bar = obj1.foo()
4. 如果都不是的话，使用默认绑定。如果在严格模式下，就绑定到 undefined，否则绑定到
全局对象。
var bar = foo()

#### 闭包

***

MDN 对闭包的定义为：

> 闭包是指那些能够访问自由变量的函数。

自由变量是什么

> 自由变量是指在函数中使用的，但既不是函数参数也不是函数的局部变量的变量。

也就是说闭包是能够访问除函数参数和函数的局部变量之外的变量的函数。

```js
var a = 1;
function f(){
    console.log(a);
}
f()
```

那岂不是函数 f 也是闭包了? 还真是！

《JavaScript权威指南》中就讲到：从技术的角度讲，所有的JavaScript函数都是闭包。

有点颠覆之前的认知了...

别着急，这是理论上的闭包，其实还有一个实践角度上的闭包，让我们看看汤姆大叔翻译的关于闭包的文章中的定义：

ECMAScript中，闭包指的是：

* 从理论角度：
  * 所有的函数。因为它们都在创建的时候就将上层上下文的数据保存起来了。哪怕是简单的全局变量也是如此，因为函数中访问全局变量就相当于是在访问自由变量，这个时候使用最外层的作用域。
* 从实践角度,以下函数才算是闭包：
  * 即使创建它的上下文已经销毁，它仍然存在（比如，内部函数从父函数中返回）
  * 在代码中引用了自由变量

```js
var scope = "global scope";
function checkscope(){
    var scope = "local scope";
    function f(){
        return scope;
    }
    return f;
}
var foo = checkscope();
foo();
```

再来分析下这段代码的执行过程：

* 最先遇到全局代码，创建全局执行上下文并压入执行上下文栈：

```js
ECStack = [
    globalContext
];
```

* 然后初始化全局执行上下文，同时checkscope 函数会被创建，并且保存作用域链到其内部属性\[\[Scope\]\]。

```js
checkscope.[[scope]] = [
    globalContext.VO
];
```

1. 执行checkscope函数，准备阶段：创建 checkscope 函数执行上下文，checkscope 执行上下文被压入执行上下文栈：

```js
ECStack = [
    checkscopeContext,
    globalContext
];
```

* checkscope 执行上下文初始化，创建变量对象、作用域链、this；复制函数\[\[scope\]\]属性创建作用域链；初始化活动对后将活动对象压入作用域链最前端，同时 f 函数会被创建，并且保存作用域链到其内部属性\[\[Scope\]\]。

```js
checkscopeContext = {
    AO: {
        arguments: {
            length: 0
        },
        scope: undefined
    }，
    Scope: [AO, [[Scope]]],
}
```

* checkscope 函数执行阶段：随着函数的执行，修改活动对象的值，最后返回函数 f，checkscope 函数执行完毕，checkscope 执行上下文从执行上下文栈中弹出。

```js
ECStack = [
    checkscopeContext,
    globalContext
];
```

* 执行函数 f ，准备阶段：创建checkscope 函数执行上下文，checkscope 执行上下文被压入执行上下文栈：

```js
ECStack = [
    fContext,
    globalContext
];
```

* f 执行上下文初始化，创建变量对象、作用域链、this；复制函数\[\[scope\]\]属性创建作用域链；初始化活动对后将活动对象压入作用域链最前端，同时 f 函数会被创建，并且保存作用域链到其内部属性\[\[Scope\]\]。

```js
fContext = {
    AO: {
        arguments: {
            length: 0
        },
    }，
    Scope: [AO, [[Scope]]],
}
```

* f 函数执行阶段：从作用域链中查找 变量scope 并返回, f 函数执行完毕，f 函数上下文从执行上下文栈中弹出。

当 f 函数执行的时候，checkscope 函数上下文已经被销毁了啊(即从执行上下文栈中被弹出)，怎么还会读取到 checkscope 作用域下的 scope 值呢？

当我们了解了具体的执行过程后，我们知道 f 执行上下文维护了一个作用域链：

```js
fContext = {
    Scope: [AO, checkscopeContext.AO, globalContext.VO],
}
```

就是因为这个作用域链，f 函数依然可以读取到 checkscopeContext.AO 的值，说明当 f 函数引用了 checkscopeContext.AO 中的值的时候，即使 checkscopeContext 被销毁了，但是 JavaScript 依然会让 checkscopeContext.AO 活在内存中，f 函数依然可以通过 f 函数的作用域链找到它，正是因为 JavaScript 做到了这一点，从而实现了闭包这个概念。

再看之前对闭包的定义就好理解了：

> 1. 即使创建它的上下文已经销毁，它仍然存在（比如，内部函数从父函数中返回）
> 2. 在代码中引用了自由变量

#### 循环和闭包

***

```js
var data = [];
for (var i = 0; i < 3; i++) {
  data[i] = function () {
    console.log(i);
  };
}
data[0]();
data[1]();
data[2]();
```

正常情况下，我们对这段代码行为的预期是分别输出数字 0,1,2 ，然而这段代码会输出3个3。

当执行data\[0\]函数时,data\[0\] 函数的执行上下文为：

```js
data[0]Context = {
    AO: {
        arguments: {
            length: 0
        },
    },
    Scope: [AO, globalContext.VO]
}
```

活动对象里并没有 i 的值,所以会根据作用域链去全局执行上下文的变量对象(全局对象)中查找，而此时全局执行上下文为：

```js
globalContext = {
    VO: {
        data: [...],
        i: 3
    }
}
```

执行data[0]函数时，循环已经结束了，所以此时i的值为3，所以三个函数最后都输出3。

再改成闭包看看：

```js
var data = [];
for (var i = 0; i < 3; i++) {
  data[i] = (function (i) {
        return function(){
            console.log(i);
        }
  })(i);
}
data[0]();
data[1]();
data[2]();
```

当执行到 data[0] 函数之前，data[0] 函数的作用域链发生了改变：

```js
data[0]Context = {
    Scope: [AO, 匿名函数Context.AO, globalContext.AO]
}
```

data[0]Context 的 AO 还是没有 i 值，所以会沿着作用域链从匿名函数 Context.AO 中查找，而此时匿名函数 Context为：

```js
匿名函数Context = {
    AO: {
        arguments: {
            0: 0,
            length: 1
        },
        i: 0
    }
}
```

因为匿名函数接收参数，并且在循环的时候传入了实参 i ，所以活量对象里有变量 i ，因为在匿名函数Context.AO中找到了 i 的值，所以不会再去globalContext.VO 中查找了，即使 globalContext.VO 也有 i 的值(值为3)，所以这三个函数最后会输出我们预期的结果。

***

参考：

* 你不知道的JavaScript（上卷）
* [JavaScript深入之闭包](https://github.com/mqyqingfeng/Blog/issues/9)

说明：本文大量参考[冴羽大佬博客](https://github.com/mqyqingfeng/Blog)，本人出于复习和总结知识点的目的加入些许个人理解，如有冒犯，敬请谅解。
