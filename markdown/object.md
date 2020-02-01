#### 创建对象的多种方式及优缺点

* 工厂模式

```js
function creatPerson(name){
    var o = new Object();
    o.name = name;
    o.getName = function(){
        console.log(this.name);
    }
    return o;
}
```

缺点：对象无法识别，因为所有的实例都指向一个原型

* 构造函数模式

```js
function Preson(name){
    this.name = name;
    this.getName = function(){
        console.log(this.name);
    }
}
var person1 = new Preson('cxk');
```

优点：实例可以识别为一个特定的类型

缺点：每次创建实例时，每个方法都要被创建一次

2.1 构造函数模式优化

```js
function Person(name){
    this.name = name;
    this.getName = getName;
}
function getName(){
    console.log(this.name);
}
var person1 = new Person();
```

优点：解决了每个方法都要被重新创建的问题

缺点：没有封装

* 原型模式

```js
function Person(){}
Person.prototype.name = 'cxk';
Person.prototype.getName = function(){
    console.log(this.name)
};
var person1 = new Person();
```

优点：方法不会重新创建

缺点：1. 所有的属性和方法都共享 2. 不能初始化参数

3.1 原型模式优化

```js
function Person(name){}
Person.prototype = {
    name : 'cxk',
    getName : function(){
        console.log(this.name)
    }
}
var person1 = new Person();
```

优点：封装性好了一点

缺点：重写了原型，丢失了constructor属性

3.2 原型模式优化

```js
function Person(name){}
Person.prototype = {
    constructor : Person,
    name : 'cxk',
    getName : function(){
        console.log(this.name)
    }
}
var person1 = new Person();
```

优点：实例可以通过constructor属性找到所属构造函数

缺点：原型模式该有的缺点还是有

* 组合模式

```js
function Person(name){
    this.name = name;
}
Person.prototype = {
    constructor : Person,
    getName = function(){
        console.log(rthuis.name)
    }
}
var person1 = new Person();
```

优点：该共享的共享，该私有的私有，使用最广泛的方式

缺点：有的人就是希望全部都写在一起，即更好的封装性(就是不能完美啊)

4.1 动态原型模式

```js
function Person(name){
    this.name = name;
    if(typeof this.getName != "function"){
        Person.prototype.getName = function(){
            console.log(this.name);
        }
    }
}
var person1 = new Person();
```

使用动态原型模式时，不能用对象字面量重写原型,因为：

```js
function Person(name){
    this.name = name;
    if(typeof this.getName != "function"){
        Person.prototype.getName = function(){
            console.log(this.name);
        }
    }
}
var person1 = new Person('cxk');
var person2 = new Person('wyf');
person1.getName(); // 报错，person1 没有这个方法
person1.getName(); // 注释掉上面的代码，可以正常执行
```

当执行 var person1 = new Person('cxk');时 new 关键字会把 person1 对象的 \_\_proto\_\_ 属性指向构造函数 Person 的 prototype 属性, 执行函数的时候又用对象字面量的方式重写了 Person 构造函数的 prototype 属性, 而person1 对象的 \_\_proto\_\_ 属性任然指向原来的 prototype, 而原来的 prototype 上并没有 getName 方法, 所以报错。

可以在用对象字面量的方式重写了 Person 构造函数的 prototype 属性后返回一个新实例解决这个问题：

```js
function Person(name) {
    this.name = name;
    if (typeof this.getName != "function") {
        Person.prototype = {
            constructor: Person,
            getName: function () {
                console.log(this.name);
            }
        }
        return new Person(name);
    }
}
```

* 稳妥构造函数模式

```js
function person(name){
    var o = new Object();
    o.sayName = function(){
        console.log(name);
    };
    return o;
}
var person1 = person('cxk');
person1.sayName(); // cxk
person1.name = "wyf";
person1.sayName(); // cxk
console.log(person1.name); // wyf
```

所谓稳妥对象，指的是没有公共属性，而且其方法也不引用 this 的对象。

与寄生构造函数模式有两点不同：

1. 新创建的实例方法不引用 this
2. 不使用 new 操作符调用构造函数
稳妥对象最适合在一些安全的环境中。

稳妥构造函数模式也跟工厂模式一样，无法识别对象所属类型。

#### 继承的多种方式及优缺点

***

1. 原型链继承

```js
function Parent(){
    this.name = 'cxk';
}
Parent.prototype.getName = function(){
    console.log(this.name);
}
function Child(){}
Child.prototype = new Parent();
var child1 = new Child();
child.getName(); // cxk
```

缺点：

* 引用类型的属性会被所有实例共享：

```js
function Parent(){
    this.colors = ['red','yellow'];
}
Parent.prototype.getColors = function(){
    console.log(this.colors);
}
function Child(){}
Child.prototype = new Parent();
var child1 = new Child();
child1.getColors(); // ['red','yellow']
child1.colors.push('blue');
child1.getColors(); // ['red','yellow','blue']
var child2 = new Child();
child2.getColors(); // ['red','yellow','blue']
```

* 在创建Child实例时不能像Parent传递参数

* 借用构造函数

```js
function Parent(name){
    this.name = name;
    this.colors = ['red','yellow'];
}
function Child(name){
    Parent.call(this, name)
}
var child1 = new Child('cxk');
child1.colors.push('blue');
console.logh(child1.colors, child.name); // ['red','yellow','blue'] , cxk
var child2 = new Child('wyf');
console.logh(child2.colors, child.name); //['red','yellow'] , wyf
```

优点：

1.避免了引用类型的属性被所有实例共享

2.可以在 Child 中向 Parent 传参

缺点：

方法都在构造函数中定义，每次创建实例都会创建一遍方法。

* 组合继承

```js
function Parent(name){
    this.name = name;
    this.colors =  ['red','yellow'];
}
Parent.prototype.getName = function(){
    console.log(this.name);
}
function Child(name, age){
    Parent.call(this, name);
    this.age = age;
}
Child.prototype = new Parent();
Child.prototype.constructor = Child;

var child1 = new Child('cxk', 18);
child1.colors.push('blue');
console.log(child1.name); // cxk
console.log(child1.age); // 18
console.log(child1.colors); // ['red','yellow','blue']
var child2 = new Child('wyf', '20');
console.log(child2.name); // wyf
console.log(child2.age); // 20
console.log(child2.colors); // ['red','yellow']
```

优点：融合原型链继承和构造函数的优点，是 JavaScript 中最常用的继承模式。

* 原型式继承

```js
function createObj(o){
    function F(){};
    F.prototype = o;
    return new F();
}
```

就是 ES5 Object.create 的模拟实现，将传入的对象作为创建的对象的原型。

缺点：

包含引用类型的属性值始终都会共享相应的值，这点跟原型链继承一样。

```js
var person = {
    name: 'cxk',
    colors: ['red', 'yellow']
}

var person1 = createObj(person);
var person2 = createObj(person);

person1.name = 'person1';
console.log(person2.name); // cxk

person1.colors.push('blue');
console.log(person2.colors); // ['red', 'yellow','blue']
```

* 寄生式继承

```js
function createObj (o) {
    var clone = Object.create(o);
    clone.sayName = function () {
        console.log('hi');
    }
    return clone;
}
```

缺点：跟借用构造函数模式一样，每次创建对象都会创建一遍方法。

1. 寄生组合式继承

组合继承最大的缺点是会调用两次父构造函数。

一次是设置子类型实例的原型的时候：

```js
Child.prototype = new Parent();
```

一次在创建子类型实例的时候：

```js
function Child(){
    Parent.call(this)
}
var child1 = new Child('cxk', '18');
```

所以 Child.prototype 和 child1 都有一个属性为colors，属性值为 \['red','yellow'\]。

间接的让 Child.prototype 访问到 Parent.prototype 可以避免这一次重复调用

```js
function Parent (name) {
    this.name = name;
    this.colors = ['red', 'yellow'];
}
Parent.prototype.getName = function () {
    console.log(this.name)
}
function Child (name, age) {
    Parent.call(this, name);
    this.age = age;
}
// 关键的三步
var F = function () {};
F.prototype = Parent.prototype;
Child.prototype = new F();
var child1 = new Child('cxk', '18');
console.log(child1);
```

封装一下:

```js
funtion createObj(o){
    function F(){};
    F.prototype = o;
    return new F();
}
function prototype(child, parent){
    var prototype = createObj(parent.prototype);
    prototype.constructor = child;
    child.prototype = prototype;
}
prototype(Child, Parent);
```

> 这种方式的高效率体现它只调用了一次 Parent 构造函数，并且因此避免了在 Parent.prototype 上面创建不必要的、多余的属性。与此同时，原型链还能保持不变；因此，还能够正常使用 instanceof 和 isPrototypeOf。开发人员普遍认为寄生组合式继承是引用类型最理想的继承范式。
《JavaScript高级程序设计》

***

参考：

* [JavaScript深入之创建对象的多种方式和优缺点](https://github.com/mqyqingfeng/Blog/issues/15)
* [JavaScript深入之继承的多种方式和优缺点](https://github.com/mqyqingfeng/Blog/issues/16)

说明：本文全部参考[冴羽大佬博客](https://github.com/mqyqingfeng/Blog)，本人出于复习和总结知识点的目的加入些许个人理解，如有冒犯，敬请谅解。
