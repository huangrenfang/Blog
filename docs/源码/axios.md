# axios 源码浅读

## 调试

> 为了方便调试拦截器部分相关内容，在本地启动一个 node 服务来模拟实际网络请求。  
> 自定义响应内容也可以满足响应拦截的需要

**调试步骤**

1. 本地启动 node 服务，服务实际网络请求
2. 将 axios npm link 生成软连接
3. 在项目中下载 axios 并根据自己的 ip 地址和请求路由进行网络请求调用

### node 服务代码

```javascript
const express = require('express');
const cors = require('cors');
const app = express();

app.use(cors());

app.get('/data', function (req, res) {
  const result = {
    a: 1,
    b: 2,
  };
  res.send(result);
});

const server = app.listen(8000, function (req, res) {
  console.log('8000 端口启动成功');
});
```

## 目录结构

```

├── /dist/                     # 项目输出目录
├── /lib/                      # 项目源码目录
│ ├── /cancel/                 # 定义取消功能
│ ├── /core/                   # 一些核心功能
│ │ ├── Axios.js               # axios的核心主类
│ │ ├── dispatchRequest.js     # 用来调用http请求适配器方法发送请求
│ │ ├── InterceptorManager.js  # 拦截器构造函数
│ │ └── settle.js              # 根据http响应状态，改变Promise的状态
│ ├── /helpers/                # 一些辅助方法
│ ├── /adapters/               # 定义请求的适配器 xhr、http
│ │ ├── http.js                # 实现http适配器
│ │ └── xhr.js                 # 实现xhr适配器
│ ├── axios.js                 # 对外暴露接口
│ ├── defaults.js              # 默认配置
│ └── utils.js                 # 公用工具
├── package.json               # 项目信息
├── index.d.ts                 # 配置TypeScript的声明文件
└── index.js                   # 入口文件

```

## 入口文件导读

> 默认导出的 axios 本质时内部`createInstance`创建的`Axios`主类上的`request`函数
>
> 即 axios 本质上是一个函数
>
> 此外还提供了 `create` 方法，`create `方法实际上为`createInstance`的调用，重新创建一个执行上下文为主类 `Axios` 的函数

```javascript
/**
 * Create an instance of Axios
 *
 * @param {Object} defaultConfig The default config for the instance
 *
 * @returns {Axios} A new instance of Axios
 */
function createInstance(defaultConfig) {
  const context = new Axios(defaultConfig);
  const instance = bind(Axios.prototype.request, context);

  // Copy axios.prototype to instance
  utils.extend(instance, Axios.prototype, context, { allOwnKeys: true });

  // Copy context to instance
  utils.extend(instance, context, null, { allOwnKeys: true });

  // Factory for creating new instances
  instance.create = function create(instanceConfig) {
    return createInstance(mergeConfig(defaultConfig, instanceConfig));
  };

  return instance;
}

// Create the default instance to be exported
const axios = createInstance(defaults);

export default axios;
```

### 工具方法解读

**bind**

```
export default function bind(fn, thisArg) {
  return function wrap() {
    return fn.apply(thisArg, arguments);
  };
}
```

简单的柯里化函数，指定第一个入参函数的执行上下文为第二个入参

**utils.extend**

```
const extend = (a, b, thisArg, {allOwnKeys}= {}) => {
  forEach(b, (val, key) => {
    if (thisArg && isFunction(val)) {
      a[key] = bind(val, thisArg);
    } else {
      a[key] = val;
    }
  }, {allOwnKeys});
  return a;
}
```

将第二个入参上的属性赋值到第一个入参上

## 主类 Axios 解读

### 拦截器实现原理

**拦截器类（请求和响应实现上是一致的）**

```javascript
class InterceptorManager {
  constructor() {
    this.handlers = [];
  }

  /**
   * Add a new interceptor to the stack
   *
   * @param {Function} fulfilled The function to handle `then` for a `Promise`
   * @param {Function} rejected The function to handle `reject` for a `Promise`
   *
   * @return {Number} An ID used to remove interceptor later
   */
  use(fulfilled, rejected, options) {
    this.handlers.push({
      fulfilled,
      rejected,
      synchronous: options ? options.synchronous : false,
      runWhen: options ? options.runWhen : null,
    });
    return this.handlers.length - 1;
  }

  /**
   * Remove an interceptor from the stack
   *
   * @param {Number} id The ID that was returned by `use`
   *
   * @returns {Boolean} `true` if the interceptor was removed, `false` otherwise
   */
  eject(id) {
    if (this.handlers[id]) {
      this.handlers[id] = null;
    }
  }

  /**
   * Clear all interceptors from the stack
   *
   * @returns {void}
   */
  clear() {
    if (this.handlers) {
      this.handlers = [];
    }
  }

  /**
   * Iterate over all the registered interceptors
   *
   * This method is particularly useful for skipping over any
   * interceptors that may have become `null` calling `eject`.
   *
   * @param {Function} fn The function to call for each interceptor
   *
   * @returns {void}
   */
  forEach(fn) {
    utils.forEach(this.handlers, function forEachHandler(h) {
      if (h !== null) {
        fn(h);
      }
    });
  }
}
```

**核心函数看 use 和 forEach 即可**

#### use 函数

使用拦截器时，通过 use 方法传入拦截函数.`InterceptorManager`将拦截函数存入自身维护的拦截函数数组 handlers 中，最后调用时使用。

#### forEach 函数

`InterceptorManager`类本身实现的 forEach 方法，容易和 Array.prototype.forEach 搞混.

函数的作用和遍历时一样的，为遍历 handlers 数组，并将数组元素传入到入参的函数中

#### 主类 Axios 拦截器使用逻辑

> Axios 主类上的代码做删减，只保留拦截器相关

```javascript
class Axios {
  constructor(instanceConfig) {
    this.defaults = instanceConfig;
    this.interceptors = {
      request: new InterceptorManager(),
      response: new InterceptorManager(),
    };
  }

  request(configOrUrl, config) {
    const requestInterceptorChain = [];
    let synchronousRequestInterceptors = true;
    this.interceptors.request.forEach(function unshiftRequestInterceptors(
      interceptor,
    ) {
      if (
        typeof interceptor.runWhen === 'function' &&
        interceptor.runWhen(config) === false
      ) {
        return;
      }

      synchronousRequestInterceptors =
        synchronousRequestInterceptors && interceptor.synchronous;

      requestInterceptorChain.unshift(
        interceptor.fulfilled,
        interceptor.rejected,
      );
    });

    len = requestInterceptorChain.length;

    let newConfig = config;

    i = 0;

    while (i < len) {
      const onFulfilled = requestInterceptorChain[i++];
      const onRejected = requestInterceptorChain[i++];
      try {
        newConfig = onFulfilled(newConfig);
      } catch (error) {
        onRejected.call(this, error);
        break;
      }
    }

    try {
      promise = dispatchRequest.call(this, newConfig);
    } catch (error) {
      return Promise.reject(error);
    }

    i = 0;
    len = responseInterceptorChain.length;

    while (i < len) {
      promise = promise.then(
        responseInterceptorChain[i++],
        responseInterceptorChain[i++],
      );
    }

    return promise;
  }
}
```

因为请求拦截和响应拦截基本是差不多的，除了各自维护的拦截函数数组插入顺序不同外。

请求拦截是通过遍历拦截函数素组将函数 unshift 插入数组，因此先定义的将会后执行。而响应拦截是通过 push 方法，所以时先定义的会先执行

获取到对应的拦截数组后，会遍历将配置传入到 fullfilled 状态的函数中。

通过 fullfilled 执行修改后返回的最新的 config 传递给真正执行网络请求的`dispatchRequest`中

`dispatchRequest`执行真正的网络请求并返回 promise 对象。在 promise 对象中，将 fulfilled 状态获取到的数据传入响应拦截中

`dispatchRequest`中有 httpAdapter 和 xhrAdapter，对 node 环境和浏览器环境做了适配，而我们常关注的 xhrAdapter 则是基于 XMLHttpRequest 实现的 源码就不展示

内容为一些状态处理、XMLHttpRequest 封装和一些异常处理
