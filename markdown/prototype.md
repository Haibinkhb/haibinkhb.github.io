#### 原型

***

``` js
function Person(){};
Person.prototype.name = 'cxk'; // 只有函数才会有prototype属性
let p1 = new Person();
let p2 = new Person();
console.log(p1.name, p2.name)   // cxk, cxk
```

上面代码中创建了一个构造函数 Person ,然后使用 new (或者说是：函数 Person 使用 new 关键字来构造调用？)创建了两个实例对象(p1, p2)。

绝大部分函数都有一个 prototype 属性,它会指向一个对象，它指向的这个对象就是实例的原型。也就是说上例中的 Person 构造函数的 prototype 属性的值就是 p1, p2 的原型。

#### \_\_proto\_\_

***

每一个 JavaScript 对象(除了 null )都具有的一个属性，叫\_\_proto\_\_，这个属性会指向该对象的原型。

``` js
function Person() {}
let p = new Person();
console.log(p.__proto__ === Person.prototype); // true
```

当读取实例的属性时，如果找不到，就会去查找和对象关联的原型中的属性，如果还查不到，就去找原型的原型，一直找到最顶层为止。就像第一个例子 中 p1, p2 都没有定义 name 属性，查找的时候找不到就会去找原型( Person.prototype )中找，所以返回的都是 “cxk”。简单证明一下：

``` js
function Person() {}
let p1 = new Person();
let p2 = new Person();
Person.prototype.name = 'cxk';
p1.name = 'wyf';
console.log(p1.name, p2.name) // wyf, cxk
```

#### 原型的原型

***

前面说到读取实例的属性时如果找不到会去原型中查找，如果还找不到，就去找原型的原型... 原型的原型又是什么？

前面说过函数的 prototype 属性指向的是一个对象，也就是说原型是一个对象，所以它也有一个 \_\_proto\_\_ 属性,这个属性会指向该对象的原型...那还有完没完？

``` js
let obj = new Object();
obj.name = 'cxk'
console.log(obj.name) // cxk
```

我们知道通过 JavaScript 内置的构造函数 Object 可以创建对象，其实原型对象就是通过 Object 构造函数生成的，结合之前所讲，实例的 \_\_proto\_\_ 指向构造函数的 prototype ，所以最终会查找到 Object.prototype.\_\_proto\_\_ 为止。那 Object.prototype.\_\_proto\_\_的值又是什么呢？

``` js
console.log(Object.prototype.__proto__) // null
```

盗一张图关系图... :
![原型链][原型链]

[原型链]:https://raw.githubusercontent.com/mqyqingfeng/Blog/master/Images/prototype5.png "原型链"

蓝色的线就是原型链

#### constructor

***

constructor 是什么鬼？

是原型对象上的一个属性。他会指向关联的构造函数。也就是说每一个构造函数上的 prototype 属性指向的原型对象上会有一个 constructor 属性指向关联的构造函数。

``` js
function Person() {

}
console.log(Person === Person.prototype.constructor); // true
```

***

参考：

* 你不知道的JavaScript（上卷）
* [JavaScript深入之从原型到原型链](https://github.com/mqyqingfeng/Blog/issues/2)

图片来源：

* [JavaScript深入之从原型到原型链](https://github.com/mqyqingfeng/Blog/issues/2)

说明：本文大量参考[冴羽大佬博客](https://github.com/mqyqingfeng/Blog)，本人出于复习和总结知识点的目的加入些许个人理解，如有冒犯，敬请谅解。
