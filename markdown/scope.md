#### 作用域

***

> 作用域是指源代码定义变量的区域。作用域规定了如何查找变量，也就是确定当前执行代码对变量的访问权限。
《*[JavaScript深入之词法作用域和动态作用域](https://github.com/mqyqingfeng/Blog/issues/3)*》

JavaScript采用词法作用域（静态作用域），函数的作用域在函数定义时就已经确定了。

```js
//例1
var scope = "global";
function checkscope(){
    var scope = "local";
    function f(){
        return scope;
    }
    return f();
}
checkscope();
```

```js
var scope = "global";
function checkscope(){
    var scope = "local";
    function f(){
        return scope;
    }
    return f;
}
checkscope()();
```

这两段代码都会输出： local

因为JavaScript采用的是词法作用域，函数的作用域基于函数创建的位置。

> JavaScript 函数的执行用到了作用域链，这个作用域链是在函数定义的时候创建的。嵌套的函数 f() 定义在这个作用域链里，其中的变量 scope 一定是局部变量，不管何时何地执行函数 f()，这种绑定在执行 f() 时依然有效。
*《JavaScript权威指南》*

#### JavaScript代码的执行顺序

***

先看一段代码：

```js
//例2
var a = 1;
function f1() {
    f2()
}
function f2(){
    console.log(a);
    var a = 2;
}
f1();
```

这段代码会输出 undefined 。

这是因为JavaScript引擎对JavaScript代码进行执行之前,需要进行预先处理,然后再执行处理后的代码。

也就是说JavaScript在浏览器中运行的过程分为两个阶段：**“预解析”（准备工作）** 和 **执行阶段** 。

#### **“预解析”**

***

##### 可执行代码块

JavaScript引擎并非一行一行地分析和执行程序，而是一段一段地分析执行。这个“一段一段”中的“段”怎么划分呢?  

JavaScript 的可执行代码(executable code)有三种类型：*全局代码、函数代码 和 eval代码*。

##### 执行上下文栈

我们上面提到的所谓javascript预解释正是创建函数的执行上下文。比如当执行到一个函数时，就会进行预解析，也就是创建函数执行上下文（execution context）。

问题来了，一个程序肯定会有很多函数，如何管理这么多执行上下文呢？所以 JavaScript 引擎创建了执行上下文栈（Execution context stack，ECS）来管理执行上下文。

为了模拟执行上下文栈的行为，让我们定义执行上下文栈是一个数组：

```js
ECStack = [];
```

有点晕... 我们来分析一下JavaScript引擎执行例2代码时的预解析流程：

当 JavaScript 开始要解释执行代码的时候，最先遇到的就是全局代码，所以初始化的时候首先就会向执行上下文栈压入一个全局执行上下文，用 globalContext 表示它，只有当整个应用程序结束的时候，ECStack 才会被清空，并且我们知道栈是一种先进后出的数据结构，所以程序结束之前， ECStack 最底部永远有个 globalContext：

```js
ECStack = [
    globalContext
];
```

接下来JavaScript 遇到下面的这段代码了：

```js
function f1() {
    f2()
}
function f2(){
    console.log(a);
    var a = 2;
}
f1();
```

当执行一个函数的时候，就会创建一个执行上下文，并且压入执行上下文栈，当函数执行完毕的时候，就会将函数的执行上下文从栈中弹出。知道了这样的工作原理，让我们来看看如何处理上面这段代码：

```js
// 伪代码
// f1() 遇到函数f1()创建函数执行上下文并压入执行上下文栈。
ECStack.push(<f1> functionContext);
// f1中调用了f2，所以此时还要创建f2的执行上下文
ECStack.push(<f2> functionContext);
// f2 执行完毕
ECStack.pop();
// f1 执行完毕
ECStack.pop();
// javascript接着执行下面的代码，但是ECStack底层永远有个globalContext，直到程序结束。
```

#### 变量对象

前面讲到当JavaScript执行一段可执行代码时会创建对应的执行上下文。对于每个执行上下文都有三个重要属性：

* 变量对象(Variable object，VO)
  
* 作用域链(Scope chain)
  
* this

先来看变量对象：

变量对象(variableObject)是与执行上下文相关的数据作用域,一个与上下文相关的特殊对象，其中存储了在上下文中定义的变量和函数声明和函数的形参（函数上下文才有）

不同的可执行代码创建的执行上下文下的变量对象会稍有不同：

* 全局上下文中的变量对象就是全局对象，在客户端 JavaScript 中，全局对象就是Window 对象。

* > 在函数上下文中，我们用活动对象(activation object, AO)来表示变量对象。活动对象和变量对象其实是一个东西，只是变量对象是规范上的或者说是引擎实现上的，不可在 JavaScript 环境中访问，只有到当进入一个执行上下文中，这个执行上下文的变量对象才会被激活，所以才叫 activation object 呐，而只有被激活的变量对象，也就是活动对象上的各种属性才能被访问。活动对象是在进入函数上下文时刻被创建的，它通过函数的 arguments 属性初始化。arguments 属性值是 Arguments 对象。
《[JavaScript深入之变量对象](https://github.com/mqyqingfeng/Blog/issues/5) )》

我们用一段伪代码表示创立的执行上下文：

```js
executionContextObj = {
    scopeChain: { /* 变量对象 + 所有父级执行上下文中的变量对象 */ },
    variableObject: { /*  函数参数 / 参数, 内部变量以及函数声明 */ },
    this: {}
}
```

当进入执行上下文时，这时候还没有执行代码,属于解析阶段，这个阶段会创建:作用域链（scope chain）、变量对象（variableObject）、设置this。

先来看创建变量对象的细节：

* 根据函数的参数，创建并初始化arguments object
  * 创建由名称和对应值组成的一个变量对象的属性
  * 没有实参时，属性值设为 undefined
* 扫描函数内部代码，查找函数声明（Function declaration）
  * 对于所有找到的函数声明，将函数名和函数引用存入变量对象中
  * 如果变量对象已经存在其它相同名称的属性（函数和变量），则完全替换这个属性
* 扫描函数内部代码，查找变量声明（Variable declaration）
  * 由名称和对应值（undefined）组成一个变量对象的属性被创建
  * 如果变量名称跟已经声明的形式参数或函数相同，则变量声明不会干扰已经存在的这类属性

举个例子：

```js
function f1(a) {
    function f2(){}
    var b = 2
    a = function(){}
}
f1(1);
```

在进入执行上下文后，这时候的 AO 是：

```js
AO = {
    arguments: {
        0: 1,
        length: 1
    },
    f2: reference to function f2(){},
    b: undefined
}
```

然后假设预解析完毕进入执行阶段，执行完毕后此时 AO ：

```js
arguments: {
        0: reference to FunctionExpression "a",
        length: 1
    },
    f2: reference to function f2(){},
    b: 2,
```

简单的总结下：

1. 全局上下文的变量对象初始化是全局对象

2. 函数上下文的变量对象初始化只包括 Arguments 对象
3. 在进入执行上下文时会给变量对象添加形参、函数声明、变量声明等初始的属性值
4. 在代码执行阶段，会再次修改变量对象的属性值

#### 作用域链

再来看执行上下文的另一个重要属性：作用域链。

>在[《JavaScript深入之变量对象》](https://github.com/mqyqingfeng/Blog/issues/5)中讲到，当查找变量的时候，会先从当前上下文的变量对象中查找，如果没有找到，就会从父级(词法层面上的父级)执行上下文的变量对象中查找，一直找到全局上下文的变量对象，也就是全局对象。这样由多个执行上下文的变量对象构成的链表就叫做作用域链。
[《JavaScript深入之作用域链》](https://github.com/mqyqingfeng/Blog/issues/6)

##### 函数创建

前面讲到函数的作用域在函数定义的时候就已经确定了。

这是因为函数内部有一个属性\[\[scope\]\],当函数创建的时候，就会保存所有父变量对象到其中。

```js
function foo() {
    function bar() {
        ...
    }
}
```

函数创建时，各自的\[\[scope\]\]为：

```js
foo.[[scope]] = [
  globalContext.VO
];

bar.[[scope]] = [
    fooContext.AO,
    globalContext.VO
];
```

当函数被执行，进入函数上下文，创建变量对象后，就会将变量对象添加到作用域链的最前端。Scope = \[AO\].concat(\[\[Scope\]\]);这时Scope才代表完整的作用域链。

再来看前面对作用域链的定义：

>当查找变量的时候，会先从当前上下文的变量对象中查找，如果没有找到，就会从父级(词法层面上的父级)执行上下文的变量对象中查找，一直找到全局上下文的变量对象，也就是全局对象。这样由多个执行上下文的变量对象构成的链表就叫做作用域链。

就很好理解了。

再来捋一捋：

```js
var scope = "global scope";
function checkscope(){
    var scope2 = 'local scope';
    return scope2;
}
checkscope();
```

用上面例子来总结一下函数执行上下文中作用域链和变量对象的创建过程：

首先会遇到全局代码，创建全局执行上下文，并压入执行上下文栈，此时 checkscope 函数会被创建，并且保存作用域链到其内部属性\[\[Scope\]\]

```js
checkscope.[[scope]] = [
    globalContext.VO
];
```

然后执行 heckscope 函数，创建 checkscope 函数执行上下文，checkscope 函数执行上下文被压入执行上下文栈：

```js
ECStack = [
    checkscopeContext,
    globalContext
];
```

checkscope 函数并不立刻执行，开始做准备工作，第一步：复制函数\[\[scope\]\]属性创建作用域链:

```js
checkscopeContext = {
    Scope: checkscope.[[scope]],
}
```

第二步：用 arguments 创建活动对象，随后初始化活动对象，加入形参、函数声明、变量声明：

```js
checkscopeContext = {
    AO: {
        arguments: {
            length: 0
        },
        scope2: undefined
    }，
    Scope: checkscope.[[scope]],
}
```

第三步：将活动对象压入 checkscope 作用域链顶端:

```js
checkscopeContext = {
    AO: {
        arguments: {
            length: 0
        },
        scope2: undefined
    },
    Scope: [AO, [[Scope]]]
}
```

准备工作做完，开始执行函数，随着函数的执行，修改 AO 的属性值:

```js
checkscopeContext = {
    AO: {
        arguments: {
            length: 0
        },
        scope2: 'local scope'
    },
    Scope: [AO, [[Scope]]]
}
```

查找到 scope2 的值，返回后函数执行完毕，函数上下文从执行上下文栈中弹出：

```js
ECStack.pop();
ECStack = [
    globalContext
];
```

***

参考：

* [JavaScript深入之词法作用域和动态作用域](https://github.com/mqyqingfeng/Blog/issues/3)
* [JavaScript深入之执行上下文栈](https://github.com/mqyqingfeng/Blog/issues/4)
* [JavaScript深入之变量对象](https://github.com/mqyqingfeng/Blog/issues/5)
* [JavaScript深入之作用域链](https://github.com/mqyqingfeng/Blog/issues/6)
* [JavaScript的『预解释』与『变量提升』](http://www.cxymsg.com/guide/hoisting.html#%E5%89%8D%E8%A8%80)

说明：本文大量参考[冴羽大佬博客](https://github.com/mqyqingfeng/Blog)，本人出于复习和总结知识点的目的加入些许个人理解，如有冒犯，敬请谅解。
