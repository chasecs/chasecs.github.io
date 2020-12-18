---
layout: post
title:  "翻译：JavaScript Promises 实现原理详解（JavaScript Promises ... In Wicked Detail）"
categories: ["javascript"]
date: 2017-10-07
description: "
Promise 是异步编程的一种解决方案，让 javascript 可以从杂乱回调函数中解脱出来。后来 ES6 标准把 Promise 纳入其中，原生提供了 Promise 对象。Promise 也成为 ES6 最主要的特性之一。网上介绍 promise 使用方法的文章很多，解释其原理却很少。这篇文章循序渐进地实现了一遍 promise，分析透彻，对了解 promise 的工作原理很有帮助。为了加深印象，所以把原文翻译了一遍。"
---
>
**译按：**
*Promise 是异步编程的一种解决方案，让 javascript 可以从杂乱回调函数中解脱出来。后来 ES6 标准把 Promise 纳入其中，原生提供了 Promise 对象。Promise 也成为 ES6 最主要的特性之一。*  
*网上介绍 promise 使用方法的文章很多，解释其原理却很少。这篇文章循序渐进地实现了一遍 promise，分析透彻，对了解 promise 的工作原理很有帮助。为了加深印象，所以把原文翻译了一遍。*
<!--more-->

>
**原文：** [JavaScript Promises ... In Wicked Detail](http://www.mattgreer.org/articles/promises-in-wicked-detail/)
>


我在自己的 JavaScript 代码里使用 promises 有一段时间了。刚开始使用的时候确实有些费解，但现在已经驾轻就熟。不过仔细想起来，我并不是完全明白它的原理。写这篇文章的目的就是为了弄明白它。如果你坚持看完这篇文章，你也能很好地理解 promise。

我们将渐进式地实现一个 promise，最终版本会基本符合 [Promises/A+ 规范](http://promises-aplus.github.io/promises-spec/)，在这个过程中，你也会明白 promise 是怎样去满足异步编程的需求。本文假定你已经对 promises 有一定的了解。如果你还什么都不知道，可以先去 [promisejs.org](http://promisejs.org/) 看看。

## 更新日志
- 2017-10-07 Added a fiddle to the [This Code is Brittle and Bad](#当前的代码既脆弱又不好) section to demonstrate how it can fail.
- 2014-12-23: Added [Recovering from Rejection](#从Reject中挽救回来) section. The article was a bit ambiguous on handling rejection, this new section should clear things up.

## 目录
1. [为什么](#为什么)
2. [最简单的例子](#最简单的例子)
    - [定义 Promise 类型](#定义Promise类型)
    - [当前的代码既脆弱又不好](#当前的代码既脆弱又不好)
3. [Promises 的状态](#Promises的状态)
4. [实现 Promises 链](#实现Promises链)
    - [回调是可选项](#回调是可选项)
    - [在 Promise 链中返回 Promise](#在Promise链中返回Promise)
5. [Promise 的 Reject](#Promises的Reject)
    - [未知错误应该归纳到 Reject](#未知错误也应该归纳到Reject)
    - [Promises 可能会“吃掉”错误](#Promises可能会“吃掉”错误)
    - [使用 `done()` 来挽救](#使用done来挽救)
    - [从 Reject 中挽救回来](#从Reject中挽救回来)
6. [Promise 必须是异步的](#Promise必须是异步的)
  - [为什么 Promises/A+ 规范要求异步](#为什么Promises/A+规范要求异步)
7. [准备封装 then/promise 之前](#准备封装then/promise之前)
8. [结论](#结论)
9. [延伸阅读](#延伸阅读)
10. [翻译](#翻译)

## 为什么

为什么要费心思去深入理解 promise 呢？弄清楚它的工作机制，你可以更好地运用它，并在出现问题的时候更好地排查故障。我写这篇文章的缘由，是有一回和同事被一个棘手的 promise 问题给难住了，如果我当时像现在这么熟悉 promise，就不会一筹莫展。

## 最简单的例子

我们先实现一个最简单的 promise 。

我们要把这段代码：
```js
doSomething(function(value) {
  console.log('Got a value:' + value);
});
```
改造成这样:
```js
doSomething().then(function(value) {
  console.log('Got a value:' + value);
});
```

为此，需要把 `doSomething()` 从：
```js
function doSomething(callback) {
  var value = 42;
  callback(value);
}
```
改成如下基于 "promise" 的方案：
```js
function doSomething() {
  return {
    then: function(callback) {
      var value = 42;
      callback(value);
    }
  };
}
```
[fiddle](http://jsfiddle.net/city41/zdgrC/1/)

当前的做法仅仅是给回调模式（callback pattern）的加了一层糖衣，还没有什么实际作用。不过，我们才刚起步，而且也触及到了 promise 的核心思想。

> Promises 捕获一个概念上称作终值(eventual value)的东西，并把它保存到一个对象(object)

这是为什么 promises 值得玩味的原因。如果“最终的东西”可以被捕获，我们就可以着手实现很强大的功能。我们将在稍后的文章中探讨此事。

### <a name="定义Promise类型"/>定义 Promise 类型

上述代码只是返回了一个简单的对象。让我们定义一个真正的 Promise 类型，并以它为基础渐进扩展：
```js
function Promise(fn) {
  var callback = null;
  this.then = function(cb) {
    callback = cb;
  };

  function resolve(value) {
    callback(value);
  }

  fn(resolve);
}
```
使用这个 Promise 类型来实现 `doSomething()`：
```js
function doSomething() {
  return new Promise(function(resolve) {
    var value = 42;
    resolve(value);
  });
}
```

当前的代码存在一个问题。如果你追溯整个执行的过程，你会发现 `resolve()` 会在 `then()` 之前被调用，这意味着此时的 `callback` 还是 null 。我们暂时先用 `setTimeout` 来处理这个问题：
```js
function Promise(fn) {
  var callback = null;
  this.then = function(cb) {
    callback = cb;
  };

  function resolve(value) {
    // force callback to be called in the next
    // iteration of the event loop, giving
    // callback a chance to be set by then()
    setTimeout(function() {
      callback(value);
    }, 1);
  }

  fn(resolve);
}
```
[fiddle](http://jsfiddle.net/city41/uQrza/1/)

通过这个办法，这段代码还是可以运行的，虽然有些勉强。

### 当前的代码既脆弱又不好

目前这个简陋的 promise 实现版本，只能在同步执行的代码中运行（译注：原文用的词是 asynchronicity，个人认为是写错了，应该是synchronicity）。只要 `then()` 也是异步的，这段程序就会再次失败，`callback` 又会变成 null。为什么我要做这个错误的示例呢，那是因为这个实现版本的好处是浅显易懂，让你先了解个大概。`then()` 和 `resolve()` 下来也会继续保留着，它们是 promise 的核心概念。

下面这个例子能表明我的意思：

[fiddle](http://jsfiddle.net/3kku32vp/)

*译注：START👇  
```js
var promise = doSomething()

setTimeout(function() {
    promise.then(function(value) {
    	log("got a value", value);
		});
}, 2);

//setTimeout 延迟 `then()` 的调用，因此 promise 先执行，此时 promise 的 callback 还是 null
```
*译注：END👆

如果你打开 console，你会发现一个错误提示：callback is not a function 。因为 `then()` 在 `setTimeout` 里调用.

## <a name="Promises的状态"/>Promises 的状态

上面那个不可靠的实现版本提醒我们，Promise 应该有状态标志。我们需要在程序执行前知道，当前处于哪种状态，并保证状态的正确地变更。这样程序才会变得健壮起来。

>- promise 等待一个值时，它的状态是 pending，获取一个值之后，状态是 resolved 。
>- 当 promise resolve 一个值（value）之后，它会锁定那个值，不会再次 resolve。

（promise 还需要一个 rejected 的状态，用于处理发生错误的情况，我们稍后再涉及这方面的内容）

让我们在 promise 里加入状态标志，替换掉原来的暂行办法：
```js
function Promise(fn) {
  var state = 'pending';
  var value;
  var deferred;

  function resolve(newValue) {
    value = newValue;
    state = 'resolved';

    if(deferred) {
      handle(deferred);
    }
  }

  function handle(onResolved) {
    if(state === 'pending') {
      deferred = onResolved;
      return;
    }

    onResolved(value);
  }

  this.then = function(onResolved) {
    handle(onResolved);
  };

  fn(resolve);
}
```
[fiddle](http://jsfiddle.net/city41/QX85J/1/)

虽然代码变得比原来复杂了，不过现在任何时候都可以调用 `then()`，`resolve()` 也可以在随时被调用，两者不必考虑谁先谁后。这样就可以同时支持同步和异步的代码。

这都是因为有了 state 这个标志，`then()` 和 `resolve()` 都把原来的工作交给了 `handle()`。`handle()` 将根据不同 state 作出以下处理：
- 如果 `then()` 先于 `resolve()` 被调用，当前还没有 value 可以交付。在这种情况下，状态将是 pending，回调方法 `onResolved` 将被被暂时保存到 `deferred`。直到 `resolve()` 被调用后，再交付 value 并触发寄存在 `deferred` 的 `onResolved`。
- 如果 `resolve()` 先于 `then()` 被调用，value 会被保存下来，当 `then()` 被调用时，value 就可以交付了。

注意到没有，现在已经不需要用到 `setTimeout` 了，不过这只是暂时的，等会儿还要再用到。

>
在 promises 里，并不讲究程序执行的先后次序，我们可以根据自己的需要决定先调用 `then()` 还是 `resolve()`。这也是“”捕获一个终值并保存到对象里”的强大之处。
>

按 Promises/A+ 规范，我们的 promises 还有不少要完善的地方，不过当前已经挺强大的了。我们可以多次调用 `then()`，每次都会得到相同的返回值：
```js
var promise = doSomething();

promise.then(function(value) {
  console.log('Got a value:', value);
});

promise.then(function(value) {
  console.log('Got the same value again:', value);
});
```
>>
当然，对于目前的 promise 实现版本，这种说法也不完全是对的。可能会有相反的情况: 比如，在 `resolve()` 调用之前，多次调用 `then()`，只会成功兑现最后一次调用的 `then()`。对于这种情况有一个解决方案：把 promise 里所有的 deferreds 都保存在一个列表里，而不是像现在只保存在一个变量里。为了不让本文变得更复杂，我就不在这里实现这个特性。
>>

## <a name="实现Promises链" />实现 Promises链

promise 可以异步地捕获一个概念并保存到对象里（capture the notion of asynchronicity in an object)，所以，我们可以
实现 promise 链，映射 promises，让它们依序或者平行地运行……以下的代码在 promise 中很常见：
```js
getSomeData()
.then(filterTheData)
.then(processTheData)
.then(displayTheData);
```
`getSomeData` 返回一个 promise，并调用 `then()`，而第一个 then 返回的，也必须是一个 promise，然后才可以不断地调用 `then()`。

> `then()` 总是返回一个 promise

实现了链的 promise ：
```js
function Promise(fn) {
  var state = 'pending';
  var value;
  var deferred = null;

  function resolve(newValue) {
    value = newValue;
    state = 'resolved';

    if(deferred) {
      handle(deferred);
    }
  }

  function handle(handler) {
    if(state === 'pending') {
      deferred = handler;
      return;
    }

    if(!handler.onResolved) {
      handler.resolve(value);
      return;
    }

    var ret = handler.onResolved(value);
    handler.resolve(ret);
  }

  this.then = function(onResolved) {
    return new Promise(function(resolve) {
      handle({
        onResolved: onResolved,
        resolve: resolve
      });
    });
  };

  fn(resolve);
}
```
[fiddle](http://jsfiddle.net/city41/HdzLv/2/)

嗯，代码开始有点古怪了，你应该庆幸我们慢慢地构建这一切。这个环节最关键就是，`then()` 返回一个新的 promise 。
>>
由于 `then()` 总是会返回一个新的 promise 对象，这意味着至少有一个 promise 对象被创建(created)、处理(resolved)，最终却被无视(ignored)。这种情况也被视为一种浪费，回调模式就不存在这种问题。这是 promise 的一个弊端，想必现在你也能理解为什么 Javascript 圈子里有人会不喜欢 promise。
>>

那么，第二个 promise 的要 resolve 的值在哪里呢？它来自第一个 promise 的返回值，在 `handle()` 方法的最后部分实现。 `handler` 对象包含了一个 `onResolved` 回调方法和一个 `resolve()` 的引用（reference）。因为有不只一个 `resolve()` 方法，每个 promise 都会复制属于自身的 `resolve()` 方法，以及一个运行该方法的闭包。这也是从第一个 promise 过渡到第二个 promise 的桥梁。

我们把第一个 promise 归结到这行代码：
```js
var ret = handler.onResolved(value);
```

在这个例子中，`handler.onResolved` 相当于：
```js
function(value) {
  console.log("Got a value:", value);
}
```

换句话说，先有一个 value 被传到第一个 `then()`，然后第一个 `handler` 的返回值则用于 resolve 第二个 promise 的值，这样就实现了 promise 链 ：
```js
doSomething().then(function(result) {
  console.log('first result', result);
  return 88;
}).then(function(secondResult) {
  console.log('second result', secondResult);
});

// the output is
//
// first result 42
// second result 88

doSomething().then(function(result) {
  console.log('first result', result);
  // not explicitly returning anything
}).then(function(secondResult) {
  console.log('second result', secondResult);
});

// now the output is
//
// first result 42
// second result undefined
```

因为 `then()` 总是会返回一个 promise，所以这个 promise 链可以无限延长：

```js
doSomething().then(function(result) {
  console.log('first result', result);
  return 88;
}).then(function(secondResult) {
  console.log('second result', secondResult);
  return 99;
}).then(function(thirdResult) {
  console.log('third result', thirdResult);
  return 200;
}).then(function(fourthResult) {
  // on and on...
});
```
假如在上述例子中，我们要在最后收集整个 promise 链的每一个返回值，该怎么办？利用这段 promise 链，我们可以自己构建返回的结果：
```
doSomething().then(function(result) {
  var results = [result];
  results.push(88);
  return results;
}).then(function(results) {
  results.push(99);
  return results;
}).then(function(results) {
  console.log(results.join(',');
});
// the output is
//
// 42, 88, 99
```
>
Promises 只会 resolve 一个值 value 。如果你需要传递超过一个值，就需要把多个值改头换面包装起来（比如用数组、对象，合并字符串，等等）。
>

更好的办法是使用 promise 的 `all()` 方法，或者其它的工具方法，它们拓展了 promise 的可用性，下来也将会探讨这部分内容。

### 回调是可选项

`then()` 方法里的回调并不是必须的，如果你省略它，随后的 promise 会接着 resolve 上一个 promise 返回的值：
```js
doSomething().then().then(function(result) {
  console.log('got a result', result);
});

// the output is
//
// got a result 42
```
个中原因可以在 `handle()`里发现，假如没有回调，它就会 resolve 上一个 promise 留下的值：
```js
if(!handler.onResolved) {
  handler.resolve(value);
  return;
}
```

### <a name="在Promise链中返回Promise" />在 Promise 链里返回 Promise 类型

当前的 promise 链还是有些初级，它只会机械地把 resolve 之后的值传递下去。假如其中一个被 resolve 的值是 promise ——就像下面这段代码——会出现什么情况：
```js
doSomething().then(function(result) {
  // doSomethingElse returns a promise
  return doSomethingElse(result);
}).then(function(finalResult) {
  console.log("the final result is", finalResult);
});
```

如代码执行的结果所示，`finalResult` 不是一个 resolve 之后的值，它是一个 promise，如果要得到预期的结构，需要这样修改：
```js
doSomething().then(function(result) {
  // doSomethingElse returns a promise
  return doSomethingElse(result);
}).then(function(anotherPromise) {
  anotherPromise.then(function(finalResult) {
    console.log("the final result is", finalResult);
  });
});
```
但谁会希望代码这样乱糟糟的？再改进一下实现 promise 的代码，就可以优雅地处理这个问题。
在 `resolve()` 方法里增加一个判断条件，处理 promise 的情况：
```js
function resolve(newValue) {
  if(newValue && typeof newValue.then === 'function') {
    newValue.then(resolve);
    return;
  }
  state = 'resolved';
  value = newValue;

  if(deferred) {
    handle(deferred);
  }
}
```
[fiddle](http://jsfiddle.net/city41/38CCb/2/)

如果我们得到的返回值是一个 promise，就会循环地调用 `resolve()`。当返回值不是 promise 时，就像之前那样执行。

>>这里也有可能会出现无限循环的情况，Promises/A+ 规范建议实现 promise 时要检测无限循环的情况，但不是必须的。

>>
还有需要指出的一点，这里上述判断返回值是否为 promise 的方法并不符合 Promises/A+ 规范，这篇文章的内容也没有完全符合 Promises/A+ 规范。如果你对这方面的内容感到好奇，建议你阅读这一节 [promise resolution procedure](https://promisesaplus.com/#the-promise-resolution-procedure).
>>

需要注意的是，我们判断 `newValue` 是否为 promise 的方法是很宽松的，只判断返回值是否有一个 `then()` 方法。我这种 [duck typing](https://en.wikipedia.org/wiki/Duck_typing) 的做法其实是故意的，这样一来，不同的 promise 实现版本就可以协同使用（interopt）。实际上，不同 promise 库之前混用是很常见的，你在使用不同的第三方库时，它们也可能使用不同的 promise 实现方法。

> 不同的 promise 实现方法可以协同使用（interopt），只要它们都恰如其分地遵照 Promises/A+ 规范

实现了 promise 链之后，当前的版本已经相对比较完善了，但还缺少错误处理机制。

## <a name="Promises的Reject" />Promises 的 Reject

当 promise 执行过程中出现错误时，需要有一个原因来 reject 。我们可以在 `then()` 里传入第二个回调方法处理这一过程：
```js
doSomething().then(function(value) {
  console.log('Success!', value);
}, function(error) {
  console.log('Uh oh', error);
});
```
>如先前所述，promise 的状态将从 pending 转变到 resolved 或 rejected 之一。因此，`then()` 的两个回调方法只有一个会被调用。

Promises 通过 `reject()` 方法来实现拒绝，以下是实现了错误处理的 `doSomething()` ：
```js
function doSomething() {
  return new Promise(function(resolve, reject) {
    var result = somehowGetTheValue();
    if(result.error) {
      reject(result.error);
    } else {
      resolve(result.value);
    }
  });
}
```

在 promise 的实现方法里，我们也需要加入 reject ：
```js
function Promise(fn) {
  var state = 'pending';
  var value;
  var deferred = null;

  function resolve(newValue) {
    if(newValue && typeof newValue.then === 'function') {
      newValue.then(resolve, reject);
      return;
    }
    state = 'resolved';
    value = newValue;

    if(deferred) {
      handle(deferred);
    }
  }

  function reject(reason) {
    state = 'rejected';
    value = reason;

    if(deferred) {
      handle(deferred);
    }
  }

  function handle(handler) {
    if(state === 'pending') {
      deferred = handler;
      return;
    }

    var handlerCallback;

    if(state === 'resolved') {
      handlerCallback = handler.onResolved;
    } else {
      handlerCallback = handler.onRejected;
    }

    if(!handlerCallback) {
      if(state === 'resolved') {
        handler.resolve(value);
      } else {
        handler.reject(value);
      }

      return;
    }

    var ret = handlerCallback(value);
    handler.resolve(ret);
  }

  this.then = function(onResolved, onRejected) {
    return new Promise(function(resolve, reject) {
      handle({
        onResolved: onResolved,
        onRejected: onRejected,
        resolve: resolve,
        reject: reject
      });
    });
  };

  fn(resolve, reject);
}
```

[fiddle](http://jsfiddle.net/city41/rLXsL/2/)

除了增加了 `reject()` 方法，`handle()` 方法也要对 rejection 作出处理。在 `handle()` 方法里，将根据 `state` 的状态分别作出 reject 或 resolve 。而当前的 `state` 的也会被传递到下一个 promise ，下一个 promise 的 `resolve()` 或 `reject()` 方法同样也会改变它自身的 `state`。

>>
使用 promise 时，很容易忽略处理错误的回调，这样你就不会得到错误提示。至少你应该在你的最后一个 promise 里加上错误回调。关于这个问题你可以查看下文“ promise可能会‘吃掉’错误”一节的内容
>>

### <a name="未知错误应该归纳到Reject" />未知错误应该归纳到 Reject

目前，我们的错误处理只针对已知错误。可能有未知的异常出现，就会导致程序崩溃，所以有必要在实现 promise 时对意外作出正确的 reject，即在 `resolve()` 方法里加入 `try/catch` ：
```js
function resolve(newValue) {
  try {
    // ... as before
  } catch(e) {
    reject(e);
  }
}
```
此外，还得确保传入 `handle()` 回调方法也不会抛出异常：
```js
function handle(deferred) {
  // ... as before

  var ret;
  try {
    ret = handlerCallback(value);
  } catch(e) {
    handler.reject(e);
    return;
  }

  handler.resolve(ret);
}
```

### <a name="Promises可能会“吃掉”错误" />Promises 可能会“吃掉”错误

 >>promise 也有可能因为误解，导致错误被吃掉，很多人就被这种情况坑过。

请看以下例子：
 ```js

 function getSomeJson() {
   return new Promise(function(resolve, reject) {
     var badJson = "<div>uh oh, this is not JSON at all!</div>";
     resolve(badJson);
   });
 }

 getSomeJson().then(function(json) {
   var obj = JSON.parse(json);
   console.log(obj);
 }, function(error) {
   console.log('uh oh', error);
 });
 ```
[fiddle](http://jsfiddle.net/city41/M7SRM/3/)

这段代码将会发生什么？`then()` 方法里的回调期望得到的是一个 JSON 对象，就天真地把结果当作 JSON 来解析，导致发生异常。虽然我们已经有了处理错误的回调，但管用吗？

>>
实际上，错误回调并没有被调用，如果你在 fiddle 上运行这段代码，你不会得到任何输出，没有错误，什么都没有。
>>

为什么会出现这种情况？因为未处理的异常发生在 `then()` 的回调方法里，它是在 `handle()` 方法里被捕获的。这导致 `hande()` 方法 reject 了 `then()` 返回的 promise，而不是正在处理的 promise，整个过程就像 promise 被正确的 resolve 一样。

>
记住，在 `then()` 的回调里，你正在响应的 promise 已经被 resolve，所以你的回调方法对这个 promise 没有影响
>

如果你想捕获上面的那个错误，需要在下游加一个错误处理：
```js
getSomeJson().then(function(json) {
  var obj = JSON.parse(json);
  console.log(obj);
}).then(null, function(error) {
  console.log("an error occured: ", error);
});
```
这样就可以打印处错误日志了。

>> 按我的个人经验，这是 promise 最大的缺陷。下一节介绍了一个更好的办法——`done()`——来解决这个问题。

### <a name="使用done来挽救" />使用 `done()` 来挽救

大部分（并非所有）promise 库都有一个 `done()` 方法。它和 `then()` 很像，并且没有上面提到的缺陷。

任何可以使用 `then()` 的地方，都可以使用 `done()`，它的主要不同点在于不会返回一个 promise，而且任何发生在 `done()` 发生的异常，不会被 promise 的实现机制捕获。换句话说，`done()` 表示整个 promise 链都被 resolve 了。上面 `getSomeJson()` 如果使用 ` done()` 会更加健壮:
```js
getSomeJson().done(function(json) {
  // when this throws, it won't be swallowed
  var obj = JSON.parse(json);
  console.log(obj);
});
```
`done()` 也可以增加一个处理错误的回调——`done(callback, errback)`，因为所有的 promise 都被 resovle 了，所以如果有异常发生，你也会得到通知。

>>`done()` 不属于 Promises/A+ 规范（至少现在不是），所以你使用的 promise 库可能没有这个方法。

### <a name="从Reject中挽救回来" />从 Reject 中挽救回来

把一个被 reject 的 promise 挽救回来是有可能的。如果 `then()` 的错误回调被调用了，之后的 promise 就会以 resolve 来响应，而不是 reject ：
```js
aMethodThatRejects().then(function(result) {
  // won't get here
}, function(err) {
  // since aMethodThatRejects calls reject()
  // we end up here in the errback
  return "recovered!";
}).then(function(result) {
  console.log("after recovery: ", result);
}, function(err) {
  // we won't actually get here
  // since the rejected promise had an errback
});

// the output is
// after recovery: recovered!
```

如果 `then()` 没有传入处理错误的回调，reject 就会被传递到下一个 promise ：
```js
// notice the two calls to then()
aMethodThatRejects().then().then(function(result) {
  // we won't get here
}, function(err) {
  console.log("error propagated");
});

// the output is
// error propagated
```

## <a name="Promise必须是异步的" />Promise 必须是异步的

本文开始的时候，我们用到 `setTimeout`，但很快又移除了。
按 Promises/A+ 规范，promise 必须是异步的。要符合这个要求也不难，我们只要把 `handle()` 的主要代码封装在 `setTimeout` 里面：
```js
function handle(handler) {
  if(state === 'pending') {
    deferred = handler;
    return;
  }
  setTimeout(function() {
    // ... as before
  }, 1);
}
```
实际上，真正的 promise 库并不倾向使用 `setTimeout`。如果是面向 NodeJS 的库，它倾向使用 `process.nextTick`，如果是针对浏览器的，可能用 `setImmediate` 或者 [setImmediate shim](https://github.com/NobleJS/setImmediate) (目前只有 IE 支持 setImmediate)，又或者是 Kris Kowal 的异步工具库 [asap](https://github.com/kriskowal/asap)（Kris Kowal 还写了 [Q](https://github.com/kriskowal/q) ，一个很受欢迎的 promise 库）

### <a name="为什么Promises/A+规范要求异步" />为什么 Promises/A+ 规范要求异步？

因为这样可以保证代码在执行过程保持一致，并且更加可靠，请看下面例子：
```js
var promise = doAnOperation();
invokeSomething();
promise.then(wrapItAllUp);
invokeSomethingElse();
```
这段代码程序调用的流程是怎样的？从字面理解，你可能认为是 `invokeSomething()` -> `invokeSomethingElse()` -> `wrapItAllUp()`。实际上却是取决于 promise 的 resolve 是同步还是异步的。如果 `doAnOperation()` 是异步的，流程和预想的一样。但是，如果 `doAnOperation()` 是同步的，流程就会变成：`invokeSomething()` -> `wrapItAllUp()` -> `invokeSomethingElse()`，这样就不好了。

为了避开这种情况，promise 必须异步地 resolve，即使是你并不需要。这样会减少意外发生，别人在使用 promise 的时候也不用考虑同步还是异步。

>>
Promises always require at least one more iteration of the event loop to resolve. This is not necessarily true of the standard callback approach.
>>

## <a name="准备封装then/promise之前" />准备封装 then/promise 之前

目前市面上已经有许多特性齐全的 promise 库。[then](https://github.com/then) 组织的 promise 库是最简单的。它的目的就是，以最简省的方法实现一个符合 Promises/A+ 规范的 promise 库，如果你看看它的实现方法，会发现看起来很眼熟。

>> 本文完成时，最终的 promise 版本看起来就很像 then/promise 的版本。以后可能就不像了，因为它们正推到重写 promise 实现方法。

本文的 promise 版本和实际的 promise 库有些不同，因为有许多 Promises/A+ 规范里的细节我并没有涉及。我建议阅读 Promises/A+ 规范，它很简短，而且通俗直白。

## 结论

如果你看到这，真要谢谢你花时间的读下来了。我们已经触及到 promises 的核心，这也是Promises/A+ 规范唯一提及的内容。大部分的 promise 实现方法会提供更多功能，比如 `all()`, `spread()`, `race()`, `denodeify()` 等待。我建议阅读 [API docs for Bluebird] 了解更多关于 promise 的内容。

当我理解了 promise 的工作原理和注意事项后，我更喜欢它了，它让我的代码变得更加整洁、优雅。它还有许多可以谈论的话题，本文只是抛砖引玉。

## 延伸阅读

更多关于 promises 的文章  
- [promisejs.org](promisejs.org) – great tutorial on promises (already mentioned it a few times)
- [Q’s Design Rationale](Q’s Design Rationale) – an article much like this one, but goes into even more detail. By Kris Kowal, creator of Q
- [Some debate over whether done() is a good thing](https://github.com/domenic/promises-unwrapping/issues/19)
- [Flattening Promise Chains](http://solutionoptimist.com/2013/12/27/javascript-promise-chains-2/) by Thomas Burleson. A nice article that goes into more advanced usage of promises. If my article is the “what”, then Thomas’s is a nice look at the “why”.

**Found a mistake?** if I made an error and you want to let me know, please [email me](matt.e.greer@gmail.com) or [file an issue](https://github.com/city41/blog/issues). Thanks!

## 翻译

- [日文翻译](http://p-baleine.hatenablog.com/entry/2014/03/12/190000), translated by Junpei Tajima
- [繁体中文翻译](http://p-baleine.hatenablog.com/entry/2014/03/12/190000), translated by Jih-Chi Lee
