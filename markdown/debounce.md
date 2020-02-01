#### 防抖

***

对于连续触发的事件，只在事件停止触发的 n 秒后再执行事件。也就是说在一个事件触发的 n 秒内又触发了这个事件，那就以新的事件的时间为准，n 秒后才执行，就是要等你触发完事件的 n 秒内不再触发事件，才执行。

根据描述，实现第一版代码：

```js
function debounce(fun, wait){
    var timeout;
    return function(){
        clearTimeout(timeout);
        timeout = setTimeout(fun, wait)
    }
}
container.onmousemove = debounce(getUserAction, 1000);
```

现在随你怎么移动，反正你移动完 1000ms 内不再触发，我才执行事件。看看使用效果：

![s](https://raw.githubusercontent.com/mqyqingfeng/Blog/master/Images/debounce/debounce-1.gif)

如果我们在 getUserAction 函数中 console.log(this)，在不使用 debounce 函数的时候，this 的值为：

```html
<div id="container"></div>
```

但是如果使用我们的 debounce 函数，this 就会指向 Window 对象（由setTimeout()调用的代码运行在与所在函数完全分离的执行环境上。这会导致，这些代码中包含的 this 关键字在非严格模式会指向 window (或全局)对象，严格模式下为 undefined，这和所期望的this的值是不一样的。）！所以要将 this 指向正确的对象。

```js
function debounce(fun, wait){
    var timeout;
    return function(){
        var self = this;
        clearTimeout(timeout)
        timeout = setTimeout(function(){
            fun.apply(self);
        }, wait)
    }
}
```

现在 this 已经可以正确指向了。但JavaScript 在事件处理函数中会提供事件对象 event，所以还要为要防抖的函数提供 event 对象：

```js
function debounce(fun, wait){
    var timeout;
    return function(){
        var self = this,
            args = arguments;
        clearTimeout(timeout);
        timeout = setTimeout(function(){
            fun.apply(self, args)
        }, wait)
    }
}
```

有时候会有希望立刻执行函数，然后等到停止触发 n 秒后，才可以重新触发执行的需求，所以再改进下：

```js
function debounce(fun, wait, immediate){
    var timeout;
    return function(){
        var self = this,
            args = arguments;
        if(timeout) clearTimeout(timeout);
        if(immediate){
            var callNow = !timeout;// clearTimeout(timeout) 后 timeout 依然是 true
            timeout = setTimeout(function(){
                timeout = null;
            }, wait)
            if(callNow) fun.apply(self, arguments);
        }else{
            timeout = setTimeout(function(){
                fun.apply(self, arguments)
            }, wait)
        }
    }
}
```

效果：

![立刻执行](https://raw.githubusercontent.com/mqyqingfeng/Blog/master/Images/debounce/debounce-4.gif)

第一次触发会立即执行，之后会等到停止触发 n 秒后，才可以继续触发执行。

需要防抖的函数可能会有返回值，所以也要返回函数的执行结果，但是当 immediate 为 false 的时候，因为使用了 setTimeout ，我们将 func.apply(context, args) 的返回值赋给变量，最后再 return 的时候，值将会一直是 undefined，所以我们只在 immediate 为 true 的时候返回函数的执行结果。

```js
function debounce(fun, wait, immediate){
    var timeout, result;
    return function(){
        var self = this,
            args = arguments;
        if(timeout) clearTimeout(timeout);
        if(immediate){
            var callNow = !timeout;// clearTimeout(timeout) 后 timeout 依然是 true
            timeout = setTimeout(function(){
                timeout = null;
            }, wait)
            if(callNow) result = fun.apply(self, arguments);
        }else{
            timeout = setTimeout(function(){
                fun.apply(self, arguments)
            }, wait)
        }
        return result;
    }
}
```

最后希望能取消 debounce 函数，比如说我 debounce 的时间间隔是 10 秒钟，immediate 为 true，这样的话，我只有等 10 秒后才能重新触发事件，现在我希望有一个按钮，点击后，取消防抖，这样我再去触发，就可以又立刻执行啦

```js
function debounce(fun, wait, immediate){
    var timeout, result;
    var debounced = function(){
        var self = this,
            args = arguments;
        if(timeout) clearTimeout(timeout);
        if(immediate){
            var callNow = !timeout;// clearTimeout(timeout) 后 timeout 依然是 true
            timeout = setTimeout(function(){
                timeout = null;
            }, wait)
            if(callNow) result = fun.apply(self, arguments);
        }else{
            timeout = setTimeout(function(){
                fun.apply(self, arguments)
            }, wait)
        }
         return result;
    }
    debounced.cancel = function(){
        clearTimeout(timeout);
        timeout = null;
    }
}
```

#### 节流

***

节流就是对于持续触发的事件，每隔一段事件，只执行一次事件。

根据首次是否执行以及结束后是否执行，效果有所不同，实现的方式也有所不同。我们用 leading 代表首次是否执行，trailing 代表结束后是否再执行一次。

节流有两种主流的实现方式，一种是使用时间戳，一种是设置定时器。

使用时间戳：触发事件的时候，获取当前的时间戳，然后减去之前的时间戳（一开始设置为0），如果大于设置的时间周期就执行函数，然后更新时间戳为当前时间戳，如果小于就不执行。

```js
function throttle(fun, wait){
    var self, args,
        previous = 0;
    return function(){
        var now = +new Date(); // 先获取时间戳再给self，args赋值相对会准一点..
        self = this;
        args = arguments;
        if(now - previous > wait){
            fun.apply(self, args);
            previous = now;
        }
    }
}
```

效果：

![时间戳](https://raw.githubusercontent.com/mqyqingfeng/Blog/master/Images/throttle/throttle1.gif)

鼠标移入的时候，事件立刻执行，每过 1s 会执行一次，如果在 5.2s 停止触发，以后不会再执行事件。

使用定时器：当触发事件的时候，设置一个定时器，再触发时间的时候如果已经有定时器了就不执行，知道定时器执行，然后再执行函数，清空定时器，这样就可以设置下个定时器。

```js
function throttle(fun, wait){
    var timeout,args,self;
    return function(){
        self = this;
        args = arguments;
        if(!timeout){
            timeout = setTimeout(function(){
                timeout = null;
                fun.apply(self, args)
            }, wait)
        }
    }
}
```

为了让效果更加明显，我们设置 wait 的时间为 3s，效果演示如下：

![定时器](https://github.com/mqyqingfeng/Blog/raw/master/Images/throttle/throttle2.gif)

可以看到：当鼠标移入的时候，事件不会立刻执行，晃了 3s 后终于执行了一次，此后每 3s 执行一次，当数字显示为 3 的时候，立刻移出鼠标，相当于大约 9.2s 的时候停止触发，但是依然会在第 12s 的时候执行一次事件。

所以比较两个方法：

1. 第一种事件会立刻执行，第二种事件会在 n 秒后第一次执行
2. 第一种事件停止触发后没有办法再执行事件，第二种事件停止触发后依然会再执行一次事件

能不能双剑合璧，鼠标移入能立刻执行，停止触发的时候还能再执行一次：

```js
function throttle(fun, wait){
    var timeout,args,self,now,remaining,
        previous = 0;
    var later = function(){
        previous = +new Date();
        timeout = null;
        fun.apply(self, args)
    }
    var throttled = function(){
        now = +new Date();
        self = this;
        args = arguments;
        remainning = wait - (now - previous);
        if(remainning <= 0 || remaining > wait){
            if(timeout){
                clearTimeout(timeout);
                timeout = null;
            }
            previous = now;
            fun.apply(self, args);
        }else if(!timeout){
            timeout = setTimeout(later, ramaining);
        }
    }
    return throttled;
}
```

***

参考：

* [JavaScript专题之跟着underscore学防抖](https://github.com/mqyqingfeng/Blog/issues/22)
* [avaScript专题之跟着 underscore 学节流](https://github.com/mqyqingfeng/Blog/issues/26)

说明：本文全部参考[冴羽大佬博客](https://github.com/mqyqingfeng/Blog)，本人出于复习和总结知识点的目的加入些许个人理解，如有冒犯，敬请谅解。
