#### bind

***

> bind() 方法会创建一个新函数。当这个新函数被调用时，bind() 的第一个参数将作为它运行时的 this，之后的一序列参数将会在传递的实参前传入作为它的参数。(来自于 MDN )

即 bind 方法会：

1. 返回一个函数
2. 可以传入参数，并把this指向第一个参数,后面的参数会作为 bind 返回的函数的参数

第一版：

```js
Function.prototype.myBind = function(obj){
    var self = this;
    return function(){
        return self.apply(obj);
    }
}
// 测试一下
var obj = {
    value : 1
}
function foo(){
    console.log(this.value)
}
var bar = foo.myBind(obj)
bar() // 1
```

第一版确定了 this 的指向，实现了返回一个函数，考虑到绑定函数可能是有返回值，所以 return self.apply(obj)。

再来实现传参：

```js
Function.prototype.myBind2 = function(obj){
    var self = this;
    // 获取 myBind2 的第二个到最后一个参数
    var args = Array.prototype.slice.call(arguments, 1);
    return function(){
        // 这里的 arguments 是 bind 返回的函数的参数
        var bindArgs = Array.prototype.slice.call(arguments)
        return self.apply(obj, args.concat(bindArgs));
    }
}
// 测试一下
var obj = {
    value : 1
}
function foo(name, age){
    console.log(this.value, name, age)
}
var bar = foo.myBind2(obj,'cxk')
bar(18) // 1 cxk 18
```

好像很轻松实现了，然而还没完，bind 还有一个特点就是：

> 一个绑定函数也能使用 new 操作符创建对象：这种行为就像把原函数当成构造器。提供的 this 值被忽略，同时调用时的参数被提供给模拟函数。

也就是说当 bind 返回的函数作为构造函数的时候，bind 时指定的 this 值会失效，但传入的参数依然生效。举个例子：

```js
var obj = {
    value : 1
}
function foo(name, age){
    this.habit = 'rap'
    console.log(this.value);
    console.log(name);
    console.log(age);
}
foo.prototype.friend = 'wyf';
var Bar = foo.bind(obj, 'cxk');
var baz = new Bar(18);
// undefinde cxk 18
console.log(baz.habit) // rap
console.log(baz.friend) // wyf
```

要实现这点就要修改返回的函数的原型：

```js
Function.prototype.myBind3 = function(obj){
    var self = this;
    var args = Array.prototype.slice.call(arguments, 1);
    var fBound = function(){
        var bindArgs = Array.prototype.slice.call(arguments);
        // 以上面的 demo 为例，当 fBound 函数(Bar)作为构造函数时 this 指向实例(baz)，此时为条件为 true，实例(baz)可以获取来自绑定函数的值；
        //作为普通函数时，this指向 window 条件为 false，将绑定函数的 this 指向 obj
        return self.apply(this instanceof fBound ? this : obj, args.concat(bindArgs));
    }
    // 修改返回函数的 prototype 为绑定函数的 prototype，实例就可以继承绑定函数的原型中的值
    fBound.prototype = this.prototype;
    return fBound;
}
```

第三版中直接将 fBound.prototype = this.prototype, 修改fBound.prototype 的时候也会直接修改绑定函数的 prototype。所以需要一个空函数中转：

```js
Function.prototype.myBind4 = function (obj) {
    var self = this;
    var args = Array.prototype.slice.call(arguments, 1);
    var fNOP = function () {};
    var fBound = function () {
        var bindArgs = Array.prototype.slice.call(arguments);
        return self.apply(this instanceof fNOP ? this : obj, args.concat(bindArgs));
    }
    fNOP.prototype = this.prototype;
    fBound.prototype = new fNOP();
    return fBound;
}
```

调用 bind 函数的必须是一个函数：

```js
if(typeof this !== "function"){
    throw Error("Function.prototype.bind - what is trying to be bound is not callable")
}
```

最终代码：

```js
Function.prototype.bind = Function.prototype.bind || function(obj){
    if(typeof this !== "function"){
        throw new Error("Function.prototype.bind - what is trying to be bound is not callable")
    };
    var self = this;
    var args = Array.prototype.slice.call(arguments, 1);
    var fNOP = function(){}
    var fBound = function(){
        var bindArgs = Array.prototype.slice.call(arguments);
        return self.apply(this instanceof fNOP ? this : obj, args.concat(bindArgs));
    }
    fNOP.prototype = this.prototype;
    fBound.prototype = new fNOP();
    return fBound;
}
```

#### new

***

> new 运算符创建一个用户定义的对象类型的实例或具有构造函数的内置对象类型之一

人话：

1. 可以访问 Trainee 函数里的属性
2. 可以访问 Trainee.prototype 中的属性

```js
function Trainee(name, age){
    this.name = name;
    this.age = age;
    this.habit = 'rap'
}
Trainee.prototype.hairstyle = 'middleScore';
Trainee.prototype.greet = function(){
    console.log('我是练习时长两年半的练习生' + this.name)
}
var p = new Trainee('cxk', 18);
console.log(p.name, p.habit, p.hairstyle) // cxk rap middleScore
p.greet() // 我是练习时长两年半的练习生cxk
```

因为 new 是关键字，没法覆盖，所以写一个函数来模拟：

```js
function Trainee(name, age){
    ...
}

function objectFactory(Trainee, ...){}; //和 new Trainee(...)一样的效果
```

因为 new 会返回一个对象，所以要创建对象并返回，而且对象会具有 Trainee 构造函数里的属性(如：this.name = name)，所以可以用 Trainee.apply(obj, arguments)来给对象添加属性。

对象还可以访问 Trainee.prototype 中的属性，所以 obj.\_\_proto\_\_ =  Trainee.prototype。

所以第一版代码：

```js
function objectFactory(){
    var obj = new Object();
    Constructor  = [].shift.call(arguments);
    obj.__proto__ = Constructor.prototype;
    Constructor.apply(obj, arguments);
    return obj;
}
```

1. 用new Object() 的方式新建了一个对象 obj
2. 取出第一个参数，就是我们要传入的构造函数。此外因为 shift 会修改原数组，所以 arguments 会被去除第一个参数
3. 将 obj 的原型指向构造函数，这样 obj 就可以访问到构造函数原型中的属性
使用 apply，改变构造函数 this 的指向到新建的对象，这样 obj 就可以访问到构造函数中的属性
4. 返回 obj

***

测试一下：

```js
functiono bjectFactory(){
    var obj = new Object();
    Constructor  = [].shift.call(arguments);
    obj.__proto__ = Constructor.prototype;
    Constructor.apply(obj, arguments);
    return obj;
}
function Trainee(name, age){
    this.name = name;
    this.age = age;
    this.habit = 'rap'
}
Trainee.prototype.hairstyle = 'middleScore';
Trainee.prototype.greet = function(){
    console.log('我是练习时长两年半的练习生' + this.name)
}
var p = objectFactory(Trainee, 'cxk', 18)
console.log(p.name, p.age, p.habit, p.hairstyle) // cxk 18 rap middleScore
p.greet() // 我是练习时长两年半的练习生cxk
```

构造函数可能会有返回值：

```js
function Trainee(name, age){
    this.name = name;
    this.age = age;
    return {
        name: name,
        habit: 'rap'
    }
}
var p = new Trainee('cxk', 18);
console.log(p.name, p.age, p.habit) // cxk undefined rap
```

构造函数返回了一个对象，在实例 p 中只能访问返回的对象中的属性。

构造函数返回还有可能返回一个基本类型值：

```js
function Trainee(name, age){
    this.name = name;
    this.habit = 'rap'
    return 'sing dance basketball'
}
var p = new Trainee('cxk', 18);
console.log(p.name, p.age, p.habit) // cxk undefined rap
```

尽管有返回值，但是相当于没有返回值进行处理。

所以还需要判断返回的值是不是一个对象，如果是一个对象，我们就返回这个对象，如果没有，我们该返回什么就返回什么。

```js
function objectFactory(){
    var obj = new Object();
    Construtor = [].shift.call(arguments);
    obj.__proto__ = Construtor.prototype;
    var result = Construtor.apply(obj, arguments);
    return typeof result === "object" ? result : obj;
}
```

参考：

* [JavaScript深入之闭包bind的模拟实现](https://github.com/mqyqingfeng/Blog/issues/12)

说明：本文大量参考[冴羽大佬博客](https://github.com/mqyqingfeng/Blog)，本人出于复习和总结知识点的目的加入些许个人理解，如有冒犯，敬请谅解。
