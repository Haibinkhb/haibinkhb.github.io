#### let 命令

***

let 用法类似 var，用于声明变量，但是 let 命令声明的变量只在 let 命令的代码块内有效。

```js
{
    let a = 1;
    var b = 2;
}

console.log(b); // 2
console.log(a); // ReferenceError: a is not defined.
```

所以 for 循环很适合使用 let 命令。

```js
for(let i = 0; i < 5; i++){
    // ...
}
console.log(i) // ReferenceError: a is not defined.
```

经典的闭包试题

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

上面代码中，变量i是var命令声明的，在全局范围内都有效，所以全局只有一个变量i。每一次循环，变量i的值都会发生改变，而循环内被赋给数组a的函数内部的console.log(i)，里面的i指向的就是全局的i。也就是说，所有数组a的成员里面的i，指向的都是同一个i，导致运行时输出的是最后一轮的i的值，最后输出都是3。

如果使用let，声明的变量仅在块级作用域内有效，最后输出的分别是0，1，2。

```js
var data = [];
for (let i = 0; i < 3; i++) {
  data[i] = function () {
    console.log(i);
  };
}
data[0]();
data[1]();
data[2]();
```

上面代码中，变量i是let声明的，当前的i只在本轮循环有效，所以每一次循环的i其实都是一个新的变量。你可能会问，如果每一轮循环的变量 i 都是重新声明的，那它怎么知道上一轮循环的值，从而计算出本轮循环的值？这是因为 JavaScript 引擎内部会记住上一轮循环的值，初始化本轮的变量 i 时，就在上一轮循环的基础上进行计算。

另外，for循环还有一个特别之处，就是设置循环变量的那部分是一个父作用域，而循环体内部是一个单独的子作用域。

```js
for (let i = 0; i < 3; i++) {
  let i = 'cxk';
  console.log(i);
}
// cxk
// cxk
// cxk
```

上面代码正确运行，输出了 3 次abc。这表明函数内部的变量i与循环变量i不在同一个作用域，有各自单独的作用域。

##### 不存在变量提升

var命令会发生“变量提升”现象，即变量可以在声明之前使用，值为undefined。为了纠正这种现象，let命令改变了语法行为，它所声明的变量一定要在声明后使用，否则报错。

```js
console.log(foo); // undefined
var foo = 1;
-----------------
console.log(bar); // 报错 ReferenceError
let bar = 1;
```

##### 暂时性死区

只要块级作用域内存在 let 命令，它所声明的变量就“绑定”（binding）这个区域，不再受外部的影响。

```js
var tmp = 123;
if (true) {
  tmp = 'abc'; // ReferenceError
  let tmp;
}
```

因为块级作用域内用 let 声明了局部变量 tmp，tmp 就被绑定在这个块级作用域中，所以在 tmp 被声明前使用就会报错。

ES6 明确规定，如果区块中存在let和const命令，这个区块对这些命令声明的变量，从一开始就形成了封闭作用域。凡是在声明之前就使用这些变量，就会报错。

总之，在代码块内，使用let命令声明变量之前，该变量都是不可用的。这在语法上，称为“暂时性死区”（temporal dead zone，简称 TDZ）。

“暂时性死区”也意味着typeof不再是一个百分之百安全的操作。

```js
typeof a; // ReferenceError
let a = 2;
```

作为比较，如果一个变量根本没有被声明，使用typeof反而不会报错。

```js
typeof undeclared_variable // "undefined";
```

解构赋值时会存在比较隐蔽的‘‘死区’’

```js
function bar(x = y, y = 2){
  return [x, y];
}
bar() // 报错
```

上面代码中，调用bar函数之所以报错（某些实现可能不报错），是因为参数x默认值等于另一个参数y，而此时y还没有声明，属于“死区”。如果y的默认值是x，就不会报错，因为此时x已经声明了。

```js
function bar(x = 2, y = x){
  return [x, y];
}
bar(); // [2, 2]
```

因为暂时性死区。使用let声明变量时，只要变量在还没有声明完成前使用，就会报错。

```js
var x = x; // 不报错

let x = x; // ReferenceError: x is not defined
```

上面这行就属于这个情况，在变量x的声明语句还没有执行完成前，就去取x的值，导致报错”x 未定义“。

##### 不允许重复声明

```js
// 报错
function bar(){
  let a = 2;
  var a = 10;
}
// 报错
function foo(){
  let a = 2;
  let a = 10;
}
// 报错
function fn1(arg) {
  let arg;
}
// 不报错
function fn2(arg) {
  {
    let arg;
  }
}
```

##### 块级作用域

ES5 只有全局作用域和函数作用域，没有块级作用域，这带来很多不合理的场景。

```js
var tmp = new Date();
function foo(){
  console.log(tmp);
  if(false){
    var tmp = 123
  }
}
foo(); // undefined
```

因为内层的 var tmp 导致变量提升覆盖了外层的tmp,所以执行函数后输出结果为 undefined。

第二种场景，用来计数的循环变量泄露为全局变量。

```js
var str = 'hello';
for(var i = 0; i < str.length; i++){
  console.log(str[i]);
}
console.log(i); // 5
```

上面代码中，变量i只用来控制循环，但是循环结束后，它并没有消失，泄露成了全局变量。

let 为 JavaScript 新增了块级作用域。

```js
{
  let a = 'out'
  {let a = 'inner'}
  console.log(a); // 
}
```