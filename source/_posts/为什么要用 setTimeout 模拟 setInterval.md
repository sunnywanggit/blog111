---
title: 为什么要用 setTimeout 模拟 setInterval
date: 2023-01-01 00:00:00
---

<meta name="referrer" content="no-referrer" />

# 1. setTimeout
setTimeout() 方法用于在指定的毫秒数后调用函数或计算表达式。
## 1.1 参数

- 第一个参数function，必填的，回调函数，可以是一个函数，也可以是一个函数名。
- 第二个参数delay，可选的，延迟时间，单位是ms。
- 第三个参数param1,param2,param3...,可选的，是传递给回调函数的参数，比较不常用到，在IE9 及其更早版本不支持该参数。
## 1.2 返回值
返回一个 ID（数字），可以将这个ID传递给 clearTimeout() 来取消执行。
# 2. setInterval
setInterval() 方法可按照指定的周期（以毫秒计）来调用函数或计算表达式。
## 2.1 参数

- 第一个参数function，必填的，回调函数，可以是一个函数，也可以是一个函数名。
- 第二个参数delay，可选的，间隔时间，单位是ms。
- 第三个参数param1,param2,param3...,可选的，是传递给回调函数的参数，比较不常用到，在IE9 及其更早版本不支持该参数。
## 2.2 返回值
返回一个 ID（数字），可以将这个ID传递给clearInterval()以取消执行。
# 3. setTimeout 最短执行时间
很多人认为 setTimeout 是延时多久，那就应该是多久后执行。

其实这个观点是错误的，因为 JS 是单线程执行的，**如果前面的代码影响了性能，就会导致 setTimeout 不会按期执行。**当然了，可以通过代码去修正 setTimeout，从而使定时器相对准确：
```javascript
let period = 60 * 1000 * 60 * 2
let startTime = new Date().getTime()
let count = 0
let end = new Date().getTime() + period
let interval = 1000
let currentInterval = interval
function loop() {
  count++
  // 代码执行所消耗的时间
  let offset = new Date().getTime() - (startTime + count * interval);
  let diff = end - new Date().getTime()
  let h = Math.floor(diff / (60 * 1000 * 60))
  let hdiff = diff % (60 * 1000 * 60)
  let m = Math.floor(hdiff / (60 * 1000))
  let mdiff = hdiff % (60 * 1000)
  let s = mdiff / (1000)
  let sCeil = Math.ceil(s)
  let sFloor = Math.floor(s)
  // 得到下一次循环所消耗的时间
  currentInterval = interval - offset 
  console.log('时：'+h, '分：'+m, '毫秒：'+s, '秒向上取整：'+sCeil, '代码执行时间：'+offset, '下次循环间隔'+currentInterval) 
  setTimeout(loop, currentInterval)
}
setTimeout(loop, currentInterval)
```

第二个参数delay未设置的时候，默认为0，意味着“马上”执行，或者尽快执行。

但是有一个规定如下：
> If timeout is less than 0, then set timeout to 0. If nesting level is greater than 5, and timeout is less than 4, then set timeout to 4.


上面的意思是说,如果延迟时间短于0，则将延迟时间设置为0。如果嵌套级别大于5，延迟时间短于4ms，则将延迟时间设置为4ms。

还有另外一种情况。为了节电，对于那些不处于当前窗口的页面，浏览器会将最短延时限制扩大到1000ms。

**以上可以说是造成定时器不准时原因之一。**
# 4. setInterval 最短执行时间
在John Resig的新书《Javascript忍者的秘密》一书中提到
> Browsers all have a 10ms minimum delay on OSX and a(approximately) 15ms delay on Windows.


在苹果机上的最短间隔时间是10毫秒,在Windows系统上的最短间隔时间大约是15毫秒。

大多数电脑显示器的刷新频率是60HZ，大概相当于每秒钟重绘60次。因此，最平滑的动画效的最佳循环间隔是1000ms/60，约等于16.6ms。

综上所述，我认为setInterval的最短间隔时间应该为16.6ms。
# 5. setInterval 的缺点
再次强调，定时器指定的时间间隔，表示的是何时将定时器的代码添加到消息队列，而**不是何时执行代码**。区别在于，setTimeout直接推入，setInterval会检查任务队列中是否存在相同的回调函数（**未执行**），若有则跳过本次推入。

所以真正何时执行代码的时间是不能保证的，取决于何时被主线程的事件循环取到，并执行。

![](https://gitee.com/sunnywanggitee/img-url/raw/master/img/20230723202234.png)

- 一开始执行 setInterval, 100 毫秒后将要执行的代码添加到队列。
- 100 毫秒时，执行代码进入队列，队列空闲，定时器内的代码执行。
- 200 毫秒时，第一次的定时器代码还在执行当中。第二次的定时器代码被推入事件队列，等待队列空闲，然后执行。
- 300 毫秒时，第一次的定时器代码还在执行中，第二次的定时器代码在事件队列末端等待执行。因为该定时器已经有第二次的代码在队列中等待了，所以**这一次的代码不会被推入队列，被忽略了**。
- 400 毫秒时，第一次的定时器代码执行完毕，队列空闲，下一个等待的代码执行，第二次的定时器代码开始执行。

捋一捋，这里的**第一次的代码和第二次代码的间隔并没有预期的 100 毫米，而是第一次的执行完，第二次的立马执行了。因为第一的代码还没执行完，第二次的代码就已经在队列中等待了。**

关于被忽略的第三次定时器代码，因为 300 毫秒时，这个定时器已经有第二次的代码在等待了，而只有当没有该定时器的代码在队列中时，该定时器新的代码才能去排队，所以第三次不会被添加到队列中。

由上可见，在这种极端情况下，暴露出来了 setInterval 的两个缺点：

1. **使用 setInterval 时，某些间隔会被跳过；**
2. **可能多个定时器会连续执行；**



可以这么理解：每个 setTimeout 产生的任务会直接 push 到任务队列中；而 setInterval 在每次把任务 push 到任务队列前，都要进行一下判断(看上次的任务是否仍在队列中，如果有则不添加，没有则添加)。

因而我们一般用 setTimeout 模拟 setInterval ，来规避掉上面的缺点。
# 6. 经典回顾
```javascript
for (var i = 0; i < 5; i++) {
  setTimeout(function() {
    console.log(i);
  }, 1000);
}
```
做过的朋友都知道：是一次输出了 5 个 5 ; 那么问题来了：是每隔 1 秒输出一个 5 ？还是一秒后立即输出 5 个 5 ？

答案是：一秒后立即输出 5 个 5因为 for 循环了五次，所以 setTimeout 被 5 次添加到时间循环中，等待一秒后全部执行。
# 7. 用 setTimeout 实现 setInterval

根据前面的分析，在某些情况下，setInterval 缺点是很明显的，为了解决这些弊端，可以使用 setTimeout() 代替来模拟实现 setInterval。

- 在前一个定时器执行完前，不会向队列插入新的定时器（解决缺点一）
- 保证定时器间隔（解决缺点二）
```javascript
function mySetInterval(fn, time = 1000) {
  let timer = null;
  let isClear = false;
  function interval() {
    if (isClear) {
      isClear = false;
      clearTimeout(timer);
      return;
    }
    fn();
    timer = setTimeout(interval, time);
  }
  timer = setTimeout(interval, time);
  return () => {
    isClear = true;
  };
}
```
# 8.用 requestAnimationFrame  实现 setInterval
```javascript
function setInterval(callback, interval) {
  let timer
  const now = Date.now
  let startTime = now()
  let endTime = startTime
  const loop = () => {
    timer = window.requestAnimationFrame(loop)
    endTime = now()
    if (endTime - startTime >= interval) {
      startTime = endTime = now()
      callback(timer)
    }
  }
  timer = window.requestAnimationFrame(loop)
  return timer
}

let a = 0
setInterval(timer => {
  console.log(1)
  a++
  if (a === 3) cancelAnimationFrame(timer)
}, 1000)

```

首先 requestAnimationFrame 自带函数节流功能，基本可以保证在 16.6 毫秒内只执行一次（不掉帧的情况下），并且该函数的延时效果是精确的，没有其他定时器时间不准的问题，当然你也可以通过该函数来实现 setTimeout。

参考文章：

- [高频前端面试题汇总之JavaScript篇（下）](https://juejin.cn/post/6941194115392634888)
- [定时器不准时☞带你揭秘setTimeout和setInterval](https://juejin.cn/post/6844904022051127310)
- [为什么要用 setTimeout 模拟 setInterval](https://juejin.cn/post/6989055903651594248)


















