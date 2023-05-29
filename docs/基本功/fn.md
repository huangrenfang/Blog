---
title: 功能性函数/类
path: /fn
---

## 功能性代码实现

### 节流函数

> 节流函数（throttle）是指在一定时间内只执行一次任务，即每隔一段时间执行一次任务，而不是每次触发事件都执行一次任务  
> 高频执行的函数，不需要每次都执行只需要在间隔内执行一次即可
> **应用场景：**

1. 页面滚动事件，每隔一段时间执行一次函数，避免频繁触发导致性能问题。
2. 窗口缩放事件，每隔一段时间执行一次函数，避免频繁触发导致性能问题。
3. 鼠标移动事件，每隔一段时间执行一次函数，避免频繁触发导致性能问题。
4. 搜索框输入事件，每隔一段时间执行一次搜索，避免频繁触发导致性能问题。

**实现：**

```js
function throttle(fn, delay) {
  let lastTime = 0;
  return function () {
    let nowTime = Date.now();
    if (nowTime - lastTime > delay) {
      fn.apply(this, arguments);
      lastTime = nowTime;
    }
  };
}
```

### 防抖函数

> 防抖函数（debounce）是指在一定时间内只执行最后一次操作，即在规定时间内多次触发同一事件，只执行最后一次操作.  
> 如果在这个规定的时间内执行多次函数，则防抖函数会执行最新的那个，之前的被覆盖了不会被执行.  
> 还是是担心触发多次的问题，导致发送多次请求或执行多次不必要的函数

**应用场景：**

1. 按钮点击事件，避免用户频繁点击导致多次请求。
2. 输入框输入事件，避免用户输入过快导致多次请求。
3. 窗口缩放事件，避免频繁触发导致性能问题。
4. 搜索框输入事件，避免频繁触发导致性能问题。

**实现：**

```js
function debounce(fn, delay) {
  let timer = null;
  return function () {
    if (timer) {
      clearTimeout(timer);
    }
    timer = setTimeout(() => {
      fn.apply(this, arguments);
      timer = null;
    }, delay);
  };
}
```

### 深拷贝

```js
function deepCopy(obj) {
  if (typeof obj !== 'object' || obj === null) {
    return obj;
  }
  let copy = Array.isArray(obj) ? [] : {};
  for (let key in obj) {
    if (Object.prototype.hasOwnProperty.call(obj, key)) {
      copy[key] = deepCopy(obj[key]);
    }
  }
  return copy;
}
```
