Vue不同组件之间是怎么通信的?

* 父子组件用props/$emit
* 非父子组件用Event Bus通信
* 如果项目够复杂,可能需要Vuex等全局状态管理库通信

不过在很多情况之下，我们的应用程序不需要类似Vuex这样的库来处理组件之间的数据通讯，而可以考虑Vue中的 事件总线 ，即 *EventBus*。

##### EventBus的简介

EventBus 又称为事件总线。在Vue中可以使用 EventBus 来作为沟通桥梁的概念，就像是所有组件共用相同的事件中心，可以向该中心注册发送事件或接收事件，所以组件都可以上下平行地通知其他组件，但也就是太方便所以若使用不慎，就会造成难以维护的灾难，因此才需要更完善的Vuex作为状态管理中心，将通知的概念上升到共享状态层次。

##### 全局使用EventBus

在用 vue-cli 创建的项目中可以 main.js 文件中添加如下代码：

```js
const EventBus = new Vue();
Vue.prototype.$bus = EventBus
```

现在假设有两个如下的兄弟组件：

```vue
<!-- UpdateMessage.vue -->
<template>
  <div>
      <input type="text" v-model="message">
      <button @click="updateMessage">update</button>
  </div>
</template>

<script>
export default {
    data () {
        return {
            message:''
        }
    },
    methods: {
        updateMessage(){
            this.$bus.$emit('updateMessage',this.message)
        }
    }
}
</script>
```

在 UpdateMessage 组件中触发 updateMessage 事件,同时在 ShowMessage 组件中监听该事件:

```vue
<!-- ShowMessage.vue -->
<template>
  <h1>
      {{message}}
  </h1>
</template>

<script>
export default {
    data () {
        return {
            message:''
        }
    },
    created () {
        this.$bus.$on('updateMessage',(value)=>{
            this.message = value
        })
    }
}
</script>
```

从上面的代码中，我们可以看到 ShowMessage 组件侦听一个名为 updateMessage 的特定事件，这个事件在组件实例化时被触发，或者你可以在创建组件时触发。另一方面，我们有另一个组件 UpdateMessage，它有一个按钮，当有人点击它时，使用 Vue 提供的 api 非常容易就实现了 EventBus。

上面主要是使用了 Vue 提供的 api 很容易就实现了，我们也可与试着自己实现一个 Event 类来模拟 JavaScript 的 Event

##### 初始化

我们利用 ES6 的 class 关键字对 Event 进行初始化Event的事件清单。

我们选择了 Map 作为储存事件的结构,因为作为键值对的储存方式 Map 比一般对象更加适合,我们操作起来也更加简洁。

```js
class Event {
    constructor() {
        this.events = this.events || new Map()
    }
}
```

##### 监听和触发

我触发监听函数我们可以用apply与call两种方法确保 this 的指向

```js
class Event {
    constructor() {
        this.events = this.events || new Map()
    }
    // 触发名为 eventType 的事件
    emit(eventType, ...args){
        let handler,
            argsLen = args.length
        ;
        // 从存储事件的 Map 中获取事件对应的回调函数
        handler = this.events.get(eventType)
        if(argsLen > 0){
            handler.apply(this, args)
        }else{
            handler.call(this)
        }
        return true
    }
    // 监听名为 eventType 的事件
    on(eventType, cb) {
        // 将事件及对应的回调函数存入 this.events
        if(!this.events.get(eventType)){
            this.events.set(eventType, cb)
        }
    }
}
```

我们实现了触发事件的emit方法和监听事件的addListener方法,至此我们就可以进行简单的实践了。

```js
// 创建一个 Event 实例
const event = new Event();

// 监听名为 print 的事件，并为这个事件添加一个回调函数
event.on('print',e => {
    console.log(`print ${e}`);
})

// 触发 print 事件
event.emit('print','事件被触发了') // 回调函数成功执行输出： print 事件被触发了
```

似乎不错,我们实现了基本的触发/监听,但是如果有多个监听者呢?

```js
// 重复监听同于一个事件
event.on('print',e => {
    console.log(`print1 ${e}`);
})
event.on('print',e => {
    console.log(`print2 ${e}`);
})
// 触发 print 事件
event.emit('print','事件被触发了') // print1 事件被触发了
```

只会触发一次，而且是只会触发第一个。

##### 改进

on 方法确实是很不健全，在绑定第一个监听后就无法对后续的进行绑定了，因此：

```js
class Event {
    constructor() {
        this.events = this.events || new Map()
    }
    // 触发名为 eventType 的事件
    emit(eventType, ...args) {
        // 对应的 emit 方法也需要改进
        let handler,
            argsLen = args.length
            ;
        // 从存储事件的 Map 中获取事件对应的回调函数
        handler = this.events.get(eventType)
        if (Array.isArray(handler)) {
            // 如果是一个数组说明有多个监听者,需要依次此触发里面的函数
            handler.forEach(item => {
                if (argsLen > 0) {
                    item.apply(this, args)
                } else {
                    item.call(this)
                }
            })
        } else {
            if (argsLen > 0) {
                handler.apply(this, args);
            } else {
                handler.call(this);
            }
        }
        return true
    }
    // 监听名为 eventType 的事件
    on(eventType, cb) {
        // 先获取对应事件名的回调函数清单
        let handler = this.events.get(eventType);
        if (!handler) {
            // 将事件及对应的回调函数存入 this.events
            this.events.set(eventType, cb)
        } else if (handler && typeof handler === 'function') {
            // 如果事件清单中已经有一个回调函数了,多个监听者我们需要用数组储存
            this.events.set(eventType, [handler, cb])
        } else {
            handler.push(cb) // 已经有多个监听者,那么直接往数组里push函数即可
        }
    }
}
```

至此，可以愉快的触发多个函数了

```js
// 重复监听同于一个事件
event.on('print',e => {
    console.log(`print1 ${e}`);
})
event.on('print',e => {
    console.log(`print2 ${e}`);
})
event.on('print',e => {
    console.log(`print3 ${e}`);
})
// 触发 print 事件
event.emit('print','事件被触发了')
// print1 事件被触发了
// print2 事件被触发了
// print3 事件被触发了
```

还需要一个移除监听函数的方法

```js
class Event {
    constructor() {
        this.events = this.events || new Map()
    }
    // ...
    remove(eventType, cb) {
        // 同样先获取事件名对应的回调函数清单
        let handler = this.events.get(eventType)
        if (handler && typeof handler === 'function') {
            // 如果是函数,说明只被监听了一次, 直接删除函数
            this.events.delete(eventType, cb)
        } else {
            let postion;
            // 如果 handler 是数组,说明被监听多次要找到对应的函数
            for (let i = 0; i < handler.length; i++) {
                if (handler[i] === fn) {
                    postion = i;
                } else {
                    postion = -1;
                }
            }
            // 如果找到匹配的函数,从数组中清除
            if (postion !== -1) {
                // 找到数组对应的位置,直接清除此回调
                handler.splice(postion, 1);
                // 如果清除后只有一个函数,那么取消数组,以函数形式保存
                if (handler.length === 1) {
                    this._events.set(type, handler[0]);
                }
            } else {
                return this;
            }
        }
    }
}
```

至此，已经基本完成了Event最重要的几个方法，可以说一个Event的骨架是被我们开发出来了,但是它仍然有不足和需要补充的地方。

* 没有对参数进行充分的判断,没有完善的报错机制
* 没有监听者上限
* ...

***

说明：本文是本人对 Event 的一些简单的理解，如有错误还望指出。
