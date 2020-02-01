> typeof 是一元操作符，放在其单个操作数的前面，操作数可以是任意类型。返回值为表示操作数类型的一个字符串。

使用 typeof 对JavaScript 共六种数据类型 Undefined、Null、Boolean、Number、String、Object 的值进行操作的时候返回的结果却不是一一对应，分别是：

undefined、object、boolean、number、string、object

都变成小写的字符串。Null 和 Object 类型都返回了 object 字符串。虽然不能一一对应，但是 typeof 却能检测出函数类型:

```js
function f(){}
console.log(typeof f) // function
```

所以 typeof 能检测出六种类型的值，但是，除此之外 Object 下还有很多细分的类型呐，如 Array、Function、Date、RegExp、Error 等。如果用 typeof 去检测这些类型都会返回 object。

幸好 Object.prototype.toString 方法可以帮我们，调用 Object.prototype.toString 会返回一个由 "[object " 和 class 和 "]" 组成的字符串，而 class 是要判断的对象的内部属性。

```js
console.log(Object.prototype.toString.call(undefined)); // [object Undefined]
console.log(Object.prototype.toString.call(null)); // [object Null]
console.log(Object.prototype.toString.call(new Date())); // [object Date]
...
```

可以看到这个 class 值就是识别对象类型的关键！正是因为这种特性，我们可以用 Object.prototype.toString 方法识别出更多类型。

写一个 type 函数能检测各种类型的值，如果是基本类型，就使用 typeof，引用类型就使用 toString。此外鉴于 typeof 的结果是小写，我也希望所有的结果都是小写：

```js
var classType = {};
'Boolean Number String Function Array Date RegExp Object Error Null Undefined'.split(' ').map(function(item, index){
    classType['[object '+ item + ']'] = item.toLowerCase()
})
function type(obj){
    return typeof obj === 'object' || typeof obj === 'function' ?
    classType[Object.prototype.toString.call(obj)] || 'objcet' : typeof obj;
}
```

然而在 IE6 中，null 和 undefined 会被 Object.prototype.toString 识别成 [object Object]！

```js
var classType = {};

// 生成class2type映射
"Boolean Number String Function Array Date RegExp Object Error".split(" ").map(function(item, index) {
    classType["[object " + item + "]"] = item.toLowerCase();
})

function type(obj) {
    // 一箭双雕
    if (obj == null) {
        return obj + ""; // undefined null
    }
    return typeof obj === "object" || typeof obj === "function" ?
        class2type[Object.prototype.toString.call(obj)] || "object" :
        typeof obj;
}
```

有了 type 函数后，我们可以对常用的判断直接封装，比如 isFunction, isArray:

```js
function isFunction(obj){
    return type(obj) === 'function'
}
var isArray = Array.isArray || function(obj){
    return type(obj) === 'array'
}
```

##### EmptyObject

jQuery提供了 isEmptyObject 方法来判断是否是空对象，代码简单，我们直接看源码：

```js
function isEmptyObject( obj ) {
        var name;
        for ( name in obj ) {
            return false;
        }
        return true;
}
```

就是判断是否有属性，for 循环一旦执行，就说明有属性，有属性就会返回 false。

##### Window对象

Window 对象作为客户端 JavaScript 的全局对象，它有一个 window 属性指向自身

```js
function isWindow(obj){
    return obj !== null && obj = obj.window;
}
```

##### isArrayLike

isArrayLike，看名字可能会让我们觉得这是判断类数组对象的，其实不仅仅是这样，jQuery 实现的 isArrayLike，数组和类数组都会返回 true。

```js
function(obj){
    var length = !!obj && 'length' in obj && obj.length;
    var typeRes = type(obj);
    // 排除掉函数和 Window 对象
    if (typeRes === "function" || isWindow(obj)) {
        return false;
    }
    return typeRes === 'array' || length === 0 || typeof length === 'number' && length > 0 && (length -1)
}
```

重点分析 return 这一行，使用了或语句，只要一个为 true，结果就返回 true。

所以如果 isArrayLike 返回true，至少要满足三个条件之一：

1. 是数组
2. 长度为0
3. length 属性大于 0 的数字类型，并且 length - 1 必须存在

```js
function a(){
    console.log(isArrayLike(arguments))
}
a();
```

如果我们去掉length === 0 这个判断，就会打印 false，然而我们都知道 arguments 是一个类数组对象，这里是应该返回 true 的。

数组是可以这样写的：

```js
[,,3]
```

第三个条件：length 是数字，并且 length > 0 且最后一个元素存在。

当我们写一个对应的类数组对象就是：

```js
var arrayLike = {
    2: 3,
    length: 3
}
```

也就是说当我们在数组中用逗号直接跳过的时候，我们认为该元素是不存在的，类数组对象中也就不用写这个元素，但是最后一个元素是一定要写的，要不然 length 的长度就不会是最后一个元素的 key 值加 1。

##### isElement

判断是不是 DOM 元素:

```js
isElement = function(obj){
    return !!(obj && obj.nodeType === 1);
}
```

***

参考：

* [JavaScript专题之类型判断(上)](https://github.com/mqyqingfeng/Blog/issues/28)
* [JavaScript专题之类型判断(下)](https://github.com/mqyqingfeng/Blog/issues/30)

说明：本文全部参考[冴羽大佬博客](https://github.com/mqyqingfeng/Blog)，本人出于复习和总结知识点的目的加入些许个人理解，如有冒犯，敬请谅解。
