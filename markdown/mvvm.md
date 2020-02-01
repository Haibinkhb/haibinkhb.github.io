当你把一个普通的 JavaScript 对象传入 Vue 实例作为 data 选项，Vue 将遍历此对象所有的属性，并使用 Object.defineProperty 把这些属性全部转为 getter/setter。

```xhtml
<div id="app">
    <div>{{person.name}}</div>
    <input type="text" v-modle="person.hobby">
</div>
<script>
    let vm = new MyVue({
        el:"#app",
        data:{
            person:{
                name: 'cxk',
                hobby: 'basketball'
            }
        }
    })
</script>
```

根据上面的 html 代码，我们从模板编译开始入手

```js
class MyVue {
    constructor(options) {
        // 实例属性
        this.$el = options.el;
        this.$data = options.data;

        if (this.$el) {
            // 编译模板
            new Compiler(this.$el, this)
        }
    }
}

class Compiler {
    constructor(el, vm) {
        // 判断 el 是否已经是一个元素节点
        this.el = this.isElementNode(el) ? el : document.querySelector(el);
        this.vm = vm;
        // 把根节点元素转成文档碎片放到内存中编译以提升性能
        let fragment = this.nodeFragment(this.el)
        // 编译文档碎片
        this.compile(fragment)
        // 把编译好的文档碎片重新放回元素
        this.el.appendChild(fragment)
    }
    compile(fragment) {
        // ...
    }
    nodeFragment(rootNode) {
        // 新建一个文碎片
        let fragment = document.createDocumentFragment();
        let firstChild;
        while (firstChild = rootNode.firstChild) {
            fragment.appendChild(firstChild)
        }
        return fragment
    }
    isElementNode(el) {
        return el.nodeType === 1;
    }
}
```

编译模板的整体思路大概就是这样，接下来重点是实现compile方法:

```js
class Compiler {
    // ...

    compile(fragment) {
        // 获取根元素节点的所有子元素节点
        let childNodes = fragment.childNodes;
        [...childNodes].forEach(node => {
            // 还是先判断是否是一个元素节点
            if (this.isElementNode(node)) {
                // 如果是元素
                this.compileElement(node)
                // 还需要递归调用
                this.compile(node)
            } else {
                // 如果是文本
                this.compileText(node)
            }
        })
    }
    compileElement(node) {
        // 需要判断元素节点是否有 v- 开头的属性
        let attributes = node.attributes; // 获取元素的所有属性
        [...attributes].forEach(attr => {
            let { name, value: expr } = attr
            if (name.startsWith('v-')) {
                // 元素有 v- 开头的属性，取出 v- 后面的指令
                let [, directive] = name.split('-')
                // 根据指令调用不同的方法
                CompilUtil[directive](node, this.vm, expr)
            }
        })
    }
    compileText(node) {
        // 需要判断文本内容是否有 {{}}
        let content = node.textContent
        if (/\{\{(.+?)\}\}/.test(content)) {
            CompilUtil['text'](node, this.vm, content)
        }
    }

    // ...
}
```

最后都是通过 CompilUtil 的方法来处理的，所以：

```js
CompilUtil = {
    getValue(vm, expr) {
        return expr.split('.').reduce((data, current) => {
            return data[current]
        }, vm.$data) // 从 vm 的 $data 上取对应表达式的值
    },
    modle(node, vm, expr) {
        /*
            node:元素节点 (input)
            vm: MyVue 实例
            expr 表达式 (person.hobby)
        */
        // 首先要通过表达式获取 vm 上对应的值
        let value = this.getValue(vm, expr);
        // 然后用获取到的值替换 input 元素的 value 属性
        let fn = this.update['modeUpdate']
        fn(node, value)
    },
    text(node, vm, expr) {
        /*
            node:文本节点
            vm: MyVue 实例
            expr 表达式 {{person.name}}
        */
        let reg = /\{\{(.+?)\}\}/g // 匹配 {{}}
        let value = expr.replace(reg, (...arg) => {
            // 通过表达式获取 vm 上对应的值
            return this.getValue(vm, arg[1]);
        })
        // 然后用获取到的值替换文本节点的 textContent 属性
        let fn = this.update['textUpdate']
        fn(node, value)
    },
    update: {
        modeUpdate(node, value) {
            node.value = value
        },
        textUpdate(node, value) {
            node.textContent = value
        }
    }
}
```

此时模板就能正确的显示我们想要的效果了，不过要想实现响应式，还远远不够。开头就引用了 Vue 官方文档的描述 Vue 实例会遍历 data 对象，并使用 Object.defineProperty 把这些属性全部转为 getter/setter。因此我们需要一个
监听者 Observer

```js
class MyVue {
    constructor(options) {
        // 实例属性
        this.$el = options.el;
        this.$data = options.data;

        if (this.$el) {
            // 监听者 Observer
            new Observer(this.$data)
            // 编译模板
            new Compiler(this.$el, this)
        }
    }
}

class Observer {
    constructor(data) {
        this.observe(data)
    }
    observe(data) {
        if (typeof data === 'object') {
            // 遍历 data 对象，并使用 Object.defineProperty 把这些属性全部转为 getter / setter。
            for (let key in data) {
                this.defineReactive(data, key, data[key])
            }
        }
    }
    defineReactive(obj, key, value) {
        this.observe(value) // 如果值是对象，递归
        Object.defineProperty(obj, key, {
            enumerable: true,
            configurable: true,
            get: () => {
                return value
            },
            set: (newValue) => {
                if (newValue !== value) {
                    // 如果新值是一个对象，为新对象也添加 getter, setter
                    this.observe(newValue)
                    value = newValue
                }
            }
        })
    }
}

// ...
```

在 Observer 中我们用 Object.defineProperty 劫持 data 对象的属性,我们称 Observer 为 监听者。虽然 Vue 2.x 是基于数据劫持的双向绑定，但是依然离不开发布订阅的模式。所以我们要实现一个订阅发布中心，即消息管理员（Dep）,它负责储存订阅者和消息的分发,不管是订阅者还是发布者都需要依赖于它。

```js
class Dep {
    constructor() {
        // 储存订阅者的数组
        this.subs = [];
    }
    // 添加订阅者
    addSubs(watcher) {
        this.subs.push(watcher)
    }
    notify() {
        // 通知所有的订阅者(Watcher)，触发订阅者的相应逻辑处理(更新操作)
        this.subs.forEach(watcher => {
            watcher.update()
        })
    }
}
```

我们还需要实现一个订阅者(Watcher)。

```js
class Watcher {
    constructor(vm, expr, cb) {
        this.vm = vm; // 被订阅的数据一定来自于当前Vue实例
        this.cb = cb; // 当数据更新时想要做的事情
        this.expr = expr; // 被订阅的数据
        this.val = this.get(); // 维护更新之前的数据
    }
    get() {
        // 通过表达式获取 vm 上对应的值
        const val = CompilUtil.getValue(vm, expr)
        return val;
    }
    update() {
        // 数据更新了，获取新的值并调用回调函数
        let newValue = CompileUtil.getVal(this.expr, this.vm)
        if (newValue !== this.value) {
            this.cb(newValue)
        }
    }
}
```

现在万事具备了，我们需要在指令中或需要编译的模板中安插订阅者(Watcher)

```js
CompilUtil = {
    getValue(vm, expr) {
        return expr.split('.').reduce((data, current) => {
            return data[current]
        }, vm.$data) // 从 vm 的 $data 上取对应表达式的值
    },
    setValue(vm, expr, value){
         expr.split('.').reduce((data, current, index, arr) => {
            if(index === arr.length -1){
                data[current] = value
            }
            return data[current]
        }, vm.$data) // 从 vm 的 $data 上取对应表达式的值
    },
    modle(node, vm, expr) {
        /*
            node:元素节点 (input)
            vm: MyVue 实例
            expr 表达式 (person.hobby)
        */
        // 首先要通过表达式获取 vm 上对应的值
        let value = this.getValue(vm, expr);
        // 然后用获取到的值替换 input 元素的 value 属性
        let fn = this.update['modeUpdate']
        // 安插订阅者，传入参数，当数据更新时 Dep 会通知它调用回调更新数据
        new Watcher(vm, expr, (newValue)=>{
            fn(node, newValue)
        })
        // 为输入框添加 input 事件
        node.addEventListener('input',(e)=>{
            this.setValue(vm, expr, e.target.value)
        })
        fn(node, value)
    },
    getAllExprValue(vm, expr){
        return expr.replace(/\{\{(.+?)\}\}/g, (...arg) => {
            return this.getValue(vm, arg[1])
        })
    },
    text(node, vm, expr) {
        /*
            node:文本节点
            vm: MyVue 实例
            expr 表达式 {{person.name}}
        */
        let fn = this.update['textUpdate']
        let reg = /\{\{(.+?)\}\}/g // 匹配 {{}}
        let value = expr.replace(reg, (...arg) => {
            // arg[1] 才是我们需要的表达式
            new Watcher(vm, arg[1], () => {
                // 表达式可能会有多个，当只更改一个的时候也需要重新获取所有的表示式对应的值进行整体的替换
                fn(node, this.getAllExprValue(vm, expr))
            })
            // 通过表达式获取 vm 上对应的值
            return this.getValue(vm, arg[1]);
        })
        // 然后用获取到的值替换文本节点的 textContent 属性
        fn(node, value)
    },
    update: {
        modeUpdate(node, value) {
            node.value = value
        },
        textUpdate(node, value) {
            node.textContent = value
        }
    }
}
```

只是安插订阅者(Watcher)还是不够，在数据更新时得有人通知它更新，所以当读取 Observer 监听的属性的时候，判断Dep类是否存在target属性，如果有就将对应的订阅者(Watcher)添加到 dep 实例的 subs 数组中，如果数据发生变化，dep 实例就会调用 notify 通知所有订阅者更新数据。

```js
class Watcher {
    // ...
    get() {
        // 当前订阅者(Watcher)读取被订阅数据的最新更新后的值时，通知订阅者管理员收集当前订阅者
        Dep.target = this // 关键一步
        // 通过表达式获取 vm 上对应的值
        const val = CompilUtil.getValue(this.vm, this.expr)
        // 置空，用于下一个Watcher使用
        Dep.target = null
        return val;
    }
    // ...
}

class Observer {
    constructor(data) {
        this.observe(data)
    }
    observe(data) {
        if (typeof data === 'object') {
            // 遍历 data 对象，并使用 Object.defineProperty 把这些属性全部转为 getter / setter。
            for (let key in data) {
                this.defineReactive(data, key, data[key])
            }
        }
    }
    defineReactive(obj, key, value) {
        this.observe(value) // 如果值是对象，递归
        let dep = new Dep()
        Object.defineProperty(obj, key, {
            enumerable: true,
            configurable: true,
            get: () => {
                /*
                    如果Dep类存在target属性，将其添加到dep实例的subs数组中
                    target指向一个Watcher实例，每个Watcher都是一个订阅者
                    Watcher实例在实例化过程中，会读取data中的某个属性，从而触发当前get方法
                */
                Dep.target && dep.addSubs(Dep.target)
                return value
            },
            set: (newValue) => {
                if (newValue !== value) {
                    // 如果新值是一个对象，为新对象也添加 getter, setter
                    this.observe(newValue)
                    value = newValue
                    // 数值被改变了,通知所有订阅者
                    dep.notify();
                }
            }
        })
    }
}
```

至此,一个简单的双向绑定算是被我们实现了。

我们还可以试着简单的实现一下 computed 和 methods

```js
class MyVue {
    constructor(options) {
        // 实例属性
        this.$el = options.el;
        this.$data = options.data;
        // 将所有data最外层属性代理到Vue实例上
        Object.keys(this.$data).forEach(key => this._proxy(key));
        let computed = options.computed;
        let methods = options.methods;
        if (this.$el) {
            // 监听者 Observer
            new Observer(this.$data)
            for (let key in computed) {
                // 通过 this.key 获取 computed 属性是，实际上是调用了 computed[key]
                Object.defineProperty(this.$data, key, {
                    get: () => {
                        return computed[key].call(this)
                    }
                })
            }

            for (let key in methods) {
                // 通过 this.key 获取 methods 属性是，实际上是调用了 methods[key]
                Object.defineProperty(this, key, {
                    get: () => {
                        return methods[key]
                    }
                })
            }
            // 编译模板
            new Compiler(this.$el, this)
        }
    }
    _proxy(key) {
        Object.defineProperty(this, key, {
            configurable: true,
            enumerable: true,
            get: () => this.$data[key],
            set: val => {
                this.$data[key] = val;
            }
        })
    }
}

class Compiler {
   // ...
    compileElement(node) {
        // 需要判断元素节点是否有 v- 开头的属性
        let attributes = node.attributes; // 获取元素的所有属性
        [...attributes].forEach(attr => {
            let { name, value: expr } = attr
            if (name.startsWith('v-')) {
                // 元素有 v- 开头的属性，取出 v- 后面的指令
                let [, directive] = name.split('-')
                let [directiveName, enentName] = directive.split(':') // 如果是事件(v-on:click)用 : 分割
                // 根据指令调用不同的方法
                CompilUtil[directiveName](node, this.vm, expr, enentName)
            }
        })
    }
   // ...
CompilUtil = {
   // ...
    on(node, vm, expr, enentName){
         /*
            node:元素节点
            vm: MyVue 实例
            expr: 表达式 事件函数名
            enentName: 事件名
        */
        // 给元素节点添加事件
        node.addEventListener(enentName, (e)=>{
            // 根据表达式调用对应函数，并传入事件对象
            vm[expr].call(vm, e)
        })
    },
    // ...
}
```

最终代码：

```js
class MyVue {
    constructor(options) {
        // 实例属性
        this.$el = options.el;
        this.$data = options.data;
        let computed = options.computed;
        let methods = options.methods;
        // 将所有data最外层属性代理到Vue实例上
        Object.keys(this.$data).forEach(key => this._proxy(key));
        if (this.$el) {
            // 监听者 Observer
            new Observer(this.$data)

            for (let key in computed) {
                // 通过 this.key 获取 computed 属性是，实际上是调用了 computed[key]
                Object.defineProperty(this, key, {
                    get: () => {
                        return computed[key].call()
                    }
                })
            }

            for (let key in methods) {
                // 通过 this.key 获取 methods 属性是，实际上是调用了 methods[key]
                Object.defineProperty(this, key, {
                    get: () => {
                        return methods[key]
                    }
                })
            }
            // 编译模板
            new Compiler(this.$el, this)
        }
    }
    _proxy(key) {
        Object.defineProperty(this, key, {
            configurable: true,
            enumerable: true,
            get: () => this.$data[key],
            set: val => {
                this.$data[key] = val;
            }
        })
    }
}

class Compiler {
    constructor(el, vm) {
        // 判断 el 是否已经是一个元素节点
        this.el = this.isElementNode(el) ? el : document.querySelector(el);
        this.vm = vm;
        // 把根节点元素转成文档碎片放到内存中编译提升性能
        let fragment = this.nodeFragment(this.el)
        // 编译文档碎片
        this.compile(fragment)
        // 把编译好的文档碎片重新放回元素
        this.el.appendChild(fragment)
    }
    compile(fragment) {
        // 获取根元素节点的所有子元素节点
        let childNodes = fragment.childNodes;
        [...childNodes].forEach(node => {
            // 还是先判断是否是一个元素节点
            if (this.isElementNode(node)) {
                // 如果是元素
                this.compileElement(node)
                // 还需要递归调用
                this.compile(node)
            } else {
                // 如果是文本
                this.compileText(node)
            }
        })
    }
    compileElement(node) {
        // 需要判断元素节点是否有 v- 开头的属性
        let attributes = node.attributes; // 获取元素的所有属性
        [...attributes].forEach(attr => {
            let { name, value: expr } = attr
            if (name.startsWith('v-')) {
                // 元素有 v- 开头的属性，取出 v- 后面的指令
                let [, directive] = name.split('-')
                let [directiveName, enentName] = directive.split(':')
                // 根据指令调用不同的方法
                CompilUtil[directiveName](node, this.vm, expr, enentName)
            }
        })
    }
    compileText(node) {
        // 需要判断文本内容是否有 {{}}
        let content = node.textContent
        if (/\{\{(.+?)\}\}/.test(content)) {
            CompilUtil['text'](node, this.vm, content)
        }
    }
    nodeFragment(rootNode) {
        // 新建一个文碎片
        let fragment = document.createDocumentFragment();
        let firstChild;
        while (firstChild = rootNode.firstChild) {
            fragment.appendChild(firstChild)
        }
        return fragment
    }
    isElementNode(el) {
        return el.nodeType === 1;
    }
}

class Observer {
    constructor(data) {
        this.observe(data)
    }
    observe(data) {
        if (typeof data === 'object') {
            // 遍历 data 对象，并使用 Object.defineProperty 把这些属性全部转为 getter / setter。
            for (let key in data) {
                this.defineReactive(data, key, data[key])
            }
        }
    }
    defineReactive(obj, key, value) {
        this.observe(value) // 如果值是对象，递归
        let dep = new Dep()
        Object.defineProperty(obj, key, {
            enumerable: true,
            configurable: true,
            get: () => {
                /*
                    如果Dep类存在target属性，将其添加到dep实例的subs数组中
                    target指向一个Watcher实例，每个Watcher都是一个订阅者
                    Watcher实例在实例化过程中，会读取data中的某个属性，从而触发当前get方法
                */
                Dep.target && dep.addSubs(Dep.target)
                return value
            },
            set: (newValue) => {
                if (newValue !== value) {
                    // 如果新值是一个对象，为新对象也添加 getter, setter
                    this.observe(newValue)
                    value = newValue
                    // 数值被改变了,通知所有订阅者
                    dep.notify();
                }
            }
        })
    }
}

class Dep {
    constructor() {
        // 储存订阅者的数组
        this.subs = [];
    }
    // 添加订阅者
    addSubs(watcher) {
        this.subs.push(watcher)
    }
    notify() {
        // 通知所有的订阅者(Watcher)，触发订阅者的相应逻辑处理(更新操作)
        this.subs.forEach(watcher => {
            watcher.update()
        })
    }
}

class Watcher {
    constructor(vm, expr, cb) {
        this.vm = vm; // 被订阅的数据一定来自于当前Vue实例
        this.cb = cb; // 当数据更新时想要做的事情
        this.expr = expr; // 被订阅的数据
        this.val = this.get(); // 维护更新之前的数据
    }
    get() {

        // 当前订阅者(Watcher)读取被订阅数据的最新更新后的值时，通知订阅者管理员收集当前订阅者
        Dep.target = this
        // 通过表达式获取 vm 上对应的值
        const val = CompilUtil.getValue(this.vm, this.expr)
        // 置空，用于下一个Watcher使用
        Dep.target = null
        return val;
    }
    update() {
        // 数据更新了，获取新的值并调用回调函数
        let newValue = CompilUtil.getValue(this.vm, this.expr)
        if (newValue !== this.value) {
            this.cb(newValue)
        }
    }
}

CompilUtil = {
    getValue(vm, expr) {
        return expr.split('.').reduce((data, current) => {
            return data[current]
        }, vm.$data) // 从 vm 的 $data 上取对应表达式的值
    },
    setValue(vm, expr, value) {
        expr.split('.').reduce((data, current, index, arr) => {
            if (index === arr.length - 1) {
                data[current] = value
            }
            return data[current]
        }, vm.$data) // 从 vm 的 $data 上取对应表达式的值
    },
    modle(node, vm, expr) {
        /*
            node:元素节点 (input)
            vm: MyVue 实例
            expr 表达式 (person.hobby)
        */
        // 首先要通过表达式获取 vm 上对应的值
        let value = this.getValue(vm, expr);
        // 然后用获取到的值替换 input 元素的 value 属性
        let fn = this.update['modeUpdate']
        // 安插订阅者，传入参数，当数据更新时 Dep 会通知它调用回调更新数据
        new Watcher(vm, expr, (newValue) => {
            fn(node, newValue)
        })
        // 为输入框添加 input 事件
        node.addEventListener('input', (e) => {
            this.setValue(vm, expr, e.target.value)
        })
        fn(node, value)
    },
    on(node, vm, expr, enentName) {
        /*
            node:元素节点 
            vm: MyVue 实例
            expr 表达式 事件函数名
            enentName： 事件名
        */
        // 给元素节点添加事件
        node.addEventListener(enentName, (e) => {
            // 根据表达式调用对应函数，并传入事件对象
            vm[expr].call(vm, e)
        })
    },
    text(node, vm, expr) {
        /*
            node:文本节点
            vm: MyVue 实例
            expr 表达式 {{person.name}}
        */
        let fn = this.update['textUpdate']
        let reg = /\{\{(.+?)\}\}/g // 匹配 {{}}
        let value = expr.replace(reg, (...arg) => {
            // arg[1] 才是我们需要的表达式
            new Watcher(vm, arg[1], () => {
                // 表达式可能会有多个，当只更改一个的时候也需要重新获取所有的表示式对应的值进行整体的替换
                fn(node, this.getAllExprValue(vm, expr))
            })
            // 通过表达式获取 vm 上对应的值
            return this.getValue(vm, arg[1]);
        })
        // 然后用获取到的值替换文本节点的 textContent 属性
        fn(node, value)
    },
    getAllExprValue(vm, expr) {
        return expr.replace(/\{\{(.+?)\}\}/g, (...arg) => {
            return this.getValue(vm, arg[1])
        })
    },
    update: {
        modeUpdate(node, value) {
            node.value = value
        },
        textUpdate(node, value) {
            node.textContent = value
        }
    }
}
```

***

说明：本文是本人对 Promise 的一些简单的理解，如有错误还望指出。
