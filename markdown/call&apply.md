> ECMAScript 规范给所有函数都定义了 call 和 apply 两个方法,他们的作用都是显示的绑定 this 。它们的第一个参数都是一个对象，它们会把 this 指向这个对象。不同的是 apply 的第二个参数是一个作为函数参数的数组。

```js
var obj = {
    name:'cxk'
}
function foo(firstname, lastname){
    return firstName + ' ' + this.name + ' ' + lastName
}
foo.apply(obj, ['wo', 'shiye']) // wo cxk shiye
```

而call方法后面传入的是一个参数列表，而不是单个数组。

```js
var obj = {
    name:'cxk'
}
function foo(firstname, lastname){
    return firstName + ' ' + this.name + ' ' + lastName
}
foo.call(obj, 'wo', 'shiye') // wo cxk shiye
```

他们的作用是一样的，对于什么时候该用什么方法，其实不用纠结。如果你的参数本来就存在一个数组中，那自然就用 apply，如果参数比较散乱相互之间没什么关联，就用 call。

#### call

***

先来模拟实现 call ：

既然 call 方法会把 this 指向传入的对象，那直接把函数添加到对象上，再用对象调用不就是把 this 指向对象了

```js
var obj ={
    value : 1,
    foo: function(){
        console.log(this.value)
    }
}
obj.foo() // 1
```

确实可以，但是给对象本身添加了一个属性，不妥，得用 delete 删除掉。

所以模拟的步骤为：

1. 将函数设为对象的属性
2. 执行函数
3. 删除函数

```js
var obj = {};
function foo(){};
obj.foo = foo;
obj.foo();
delete obj.foo;
```

根据这个思路尝试第一版：

```js
Function.prototype.myCall = function(obj){
    obj.fn = this; // 这里的 this 就是调用 myCall 方法的函数
    obj.fn();
    delete obj.fn;
}

//测试一下
var obj = {
    value : 1
}
function foo(){
    console.log(this.value)
}
foo.myCall(obj) // 1
```

实现了！再来解决参数的问题，前面说过call接收的是一个参数列表，那怎么获取参数呢？当然是 Arguments 对象，取第二到最后一个就是我们想要的参数了。比如：

```js
var obj = {
    name:'cxk'
}
function f(firstname, lastname){
    return firstName + ' ' + this.name + ' ' + lastName
}
f.myCall(obj,'wo','shiye');
// 此时 arguments 为：
var arguments = {
    0:obj,
    1: 'wo',
    2: 'shiye',
    length: 3
}
// 我们可以遍历类数组对象 arguments，取出想要的参数存入 args 数组中
var args = [];
for(var i = 1; i < arguments.length; i++){
    args.push(arguments[i])
}
```

接着把这个参数数组放到要执行的函数的参数里面去

```js
// ...
obj.fn(args.join(','))
```

这样当然是不行的(只传入了一个参数),用ES6的 ... 运算符可以轻松实现：

```js
// ...
obj.fn(...args)
```

不过 call 是 ES3 的方法,为了模拟实现一个 ES3 的方法,得用 eval 方法拼成一个函数,类似这样：

```js
// ...
eval('obj.fn(' + args + ')')
```

eval() 函数接收一个字符串，计算并执行其中的的 JavaScript 代码。 这里 args 会自动调用 Array.toString() 这个方法。所以 args 不能直接 push(arguments\[ i \]) ,而是要 push('arguments[' + i + ']'),这样的话 args 为 \["arguments\[ 1 \]", "arguments\[ 2 \]", "arguments\[ 3 \]"],这样 eval 相当于接受了这样一个字符串 'obj.fn(' + arguments\[ 1 \],arguments\[ 2 \] + ')'。所以第二版完整代码：

```js
Function.prototype.myCall2 = function(obj){
    obj.fn = this; // 这里的 this 就是调用 myCall2 方法的函数
    var args = [];
    for(var i = 1; i < arguments.length; i++){
        args.push('arguments[' + i + ']');
    }
    eval('obj.fn(' + args + ')');
    delete obj.fn;
}

//测试一下
var obj = {
    name : 'cxk'
}
function foo(firstname, lastname){
    console.log(firstname + ' ' + this.name + ' ' + lastname)
}
foo.myCall2(obj,'wo','shiye') // wo cxk shiye
```

还有要注意的地方：

1. 第一个参数可以是 null, 当为 null 时视为指向 window。
2. 函数是可以有返回值的！

```js
var obj = {
    value : 1
}
function foo(name, age){
    return {
        vaule: this.value
        name: name
        age: age
    }
}
console.log(foo.call(obj,'cxk', 18))
// {
//    value: 1,
//    name: 'cxk',
//    age: 18
// }
```

所以第三版：

```js
Function.prototype.myCall3 = function(obj){
    var obj = obj || window;
    obj.fn = this; // 这里的 this 就是调用 myCall3 方法的函数
    var args = [];
    for(var i = 1; i < arguments.length; i++){
        args.push('arguments[' + i + ']');
    }
    var result = eval('obj.fn(' + args + ')');
    delete obj.fn;
    return result;
}

//测试一下
var value = 1;
var obj = {
    value : 2
}
function foo(name, age){
    return {
        vaule: this.value,
        name: name,
        age: age
    }
}
foo.myCall3(null) // 1
foo.myCall3(obj,'cxk', 18)
// {
//    value: 2,
//    name: 'cxk',
//    age: 18
// }
```

#### apply

***

apply 和 call 类似,直接给代码：

```js
Function.prototype.myApply = function(obj, arr){
    var obj = obj || window;
    obj.fn = this;
    var result;
    if(!arr){ // 没有第二个参数
        result = obj.fn()
    }else{
        var args = [];
        for(var i = 0; i < arr.length; i++){
            args.push('arr[' + i + ']');
        }
        result = eval('obj.fn(' + args + ')')
    }
    delete obj.fn;
    return result
}
```

***

参考：

* [JavaScript深入之闭包](https://github.com/mqyqingfeng/Blog/issues/11)

* [JavaScript中apply、call的详解](https://github.com/lin-xin/blog/issues/7)

说明：本文大量参考[冴羽大佬博客](https://github.com/mqyqingfeng/Blog)，本人出于复习和总结知识点的目的加入些许个人理解，如有冒犯，敬请谅解。
