# JS实现Ajax并发请求控制

## 场景

假设现在有这么一种场景：现有 30 个异步请求需要发送，但由于某些原因，我们必须将同一时刻并发请求数量控制在 5 个以内，同时还要尽可能快速的拿到响应结果。

应该怎么做？

首先我们来了解一下 `Ajax`的串行和并行。

## 基于 Promise.all 实现 Ajax 的串行和并行

我们平时都是基于`promise`来封装异步请求的，这里也主要是针对异步请求来展开。

- 串行：一个异步请求完了之后在进行下一个请求
- 并行：多个异步请求同时进行

通过定义一些`promise实例`来具体演示串行/并行。

### 串行

```
var p = function () {
  return new Promise(function (resolve, reject) {
    setTimeout(() => {
      console.log('1000')
      resolve()
    }, 1000)
  })
}
var p1 = function () {
  return new Promise(function (resolve, reject) {
    setTimeout(() => {
      console.log('2000')
      resolve()
    }, 2000)
  })
}
var p2 = function () {
  return new Promise(function (resolve, reject) {
    setTimeout(() => {
      console.log('3000')
      resolve()
    }, 3000)
  })
}


p().then(() => {
  return p1()
}).then(() => {
  return p2()
}).then(() => {
  console.log('end')
})
```

如示例，串行会从上到下依次执行对应接口请求。

### 并行

通常，我们在需要保证代码在多个异步处理之后执行，会用到：

```
Promise.all(promises: []).then(fun: function);
```

`Promise.all`可以保证，`promises`数组中所有`promise`对象都达到`resolve`状态，才执行`then`回调。

```
var promises = function () {
  return [1000, 2000, 3000].map(current => {
    return new Promise(function (resolve, reject) {
      setTimeout(() => {
        console.log(current)
      }, current)
    })
  })
}

Promise.all(promises()).then(() => {
  console.log('end')
})
```

## Promise.all 并发限制

这时候考虑一个场景：如果你的`promises`数组中每个对象都是`http请求`，而这样的对象有几十万个。

那么会出现的情况是，你在瞬间发出几十万个`http请求`，这样很有可能导致堆积了无数调用栈导致内存溢出。

这时候，我们就需要考虑对`Promise.all`做并发限制。

`Promise.all并发限制`指的是，每个时刻并发执行的`promise`数量是固定的，最终的执行结果还是保持与原来的`Promise.all`一致。

## 题目实现

### 思路分析

整体采用递归调用来实现：最初发送的请求数量上限为允许的最大值，并且这些请求中的每一个都应该在完成时继续递归发送，通过传入的索引来确定了`urls`里面具体是那个`URL`，保证最后输出的顺序不会乱，而是依次输出。

### 代码实现

```
function multiRequest(urls = [], maxNum) {
  // 请求总数量
  const len = urls.length;
  // 根据请求数量创建一个数组来保存请求的结果
  const result = new Array(len).fill(false);
  // 当前完成的数量
  let count = 0;

  return new Promise((resolve, reject) => {
    // 请求maxNum个
    while (count < maxNum) {
      next();
    }
    function next() {
      let current = count++;
      // 处理边界条件
      if (current >= len) {
        // 请求全部完成就将promise置为成功状态, 然后将result作为promise值返回
        !result.includes(false) && resolve(result);
        return;
      }
      const url = urls[current];
      console.log(`开始 ${current}`, new Date().toLocaleString());
      fetch(url)
        .then((res) => {
          // 保存请求结果
          result[current] = res;
          console.log(`完成 ${current}`, new Date().toLocaleString());
          // 请求没有全部完成, 就递归
          if (current < len) {
            next();
          }
        })
        .catch((err) => {
          console.log(`结束 ${current}`, new Date().toLocaleString());
          result[current] = err;
          // 请求没有全部完成, 就递归
          if (current < len) {
            next();
          }
        });
    }
  });
}
```