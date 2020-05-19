# axios源码阅读笔记

> 为啥要读axios源码  
> axios 是什么  
> axios 怎么写的

写了几年前端，却没有完整的阅读过任一知名js框架源码，这样不好，虽然目前没有跳槽的打算，但是出去面试的确爱问这个

最近闲来无事，那就从头来读一读axios的源码

至于为什么要读axios，主要是因为其他知名库比如vue/react的源码太长，而且解读的人太多(诶，好像解读axios的人也不少啊)，另外平时工作中，网络请求一般都是自己封装一个简单XMLHttpRequest，所以想看看这个牛逼哄哄的axios为什么这么dio

---

## axios是什么

axios，是当前非常著名的前端库，主要用于处理封装前端的请求，是一款非常好用的前端库，顺便一提查到axios的读音读作艾克C欧丝

---

## 让我们开始吧

### 先搞到axios源码

搞源码比搞钱容易的多的多的多的多...上 ~~gayhub~~ github上一搜就搜到了

[axios - Promise based HTTP client for the browser and node.js](https://github.com/axios/axios.git)

看看这清晰的介绍，看看这华丽的布局，看看这详尽的用法，一看就知道是大师的手笔，比一些个人作坊的连readme都懒得写(比如在下)的库不知道高到哪里去啊~

### 源码先看package.json

下好源码，先看什么？readme？？No，都说了一些库连&#8482; readme都不写(比如在下)，所以当然必须先看工程户口本package.json

```json
"name": "axios",
"version": "0.19.0",
"description": "Promise based HTTP client for the browser and node.js",
"main": "index.js",
```

看源码，其实主要找到main这行就行了，不过也可以顺便看看还有哪些

```json
"scripts": {
    "test": "grunt test && bundlesize",
    "start": "node ./sandbox/server.js",
    "build": "NODE_ENV=production grunt build",
    "preversion": "npm test",
    "version": "npm run build && grunt version && git add -A dist && git add CHANGELOG.md bower.json package.json",
    "postversion": "git push && git push --tags",
    "examples": "node ./examples/server.js",
    "coveralls": "cat coverage/lcov.info | ./node_modules/coveralls/bin/coveralls.js",
    "fix": "eslint --fix lib/**/*.js"
},
```

不愧是知名库，脚本这叫一个全，讲究

```json
    "devDependencies": {
        "bundlesize": "^0.17.0",
        ...... x 12
        "webpack-dev-server": "^1.14.1"
    },
    "dependencies": {
        "follow-redirects": "1.5.10",
        "is-buffer": "^2.0.2"
    }
    
```

我擦，这么多依赖，不愧是知名库，讲究

顺便一提devDependencies和dependencies的区别，顾名思义，

```text
devDependencies是在开发中用到的，可能是一些单元测试或者代码检查工具之类的  
dependencies是发布时用到的，包含所有发布时会用到的依赖
```

简单的说就是发布时用不到的全部扔到devDependencies里，来来来，大家可以立刻检查检查自己之前写的工程是不是直接npm install --save安装依赖，然后dependencies里一坨(笔者就是...)

```json
"bundlesize": [
    {
        "path": "./dist/axios.min.js",
        "threshold": "5kB"
    }
]

```

额...bundlesize是啥属性？[Npm doc](https://docs.npmjs.com/files/package.json#browser)里没查到，网上搜了下，原来是个插件，主要用来指定具体文件的大小上限，一旦超过就会向你报警，不愧是知名库啊，真讲究

### index.js

好，让我们进入主题吧，既然main指向了/index.js，那我们直接看下index.js写了啥

```javascript
// index.js
module.exports = require('./lib/axios');
```

就一行，入口嘛，正常操作，那我们就顺着脉络看看./lib/axios.js

```javascript

// Create the default instance to be exported
var axios = createInstance(defaults);

module.exports = axios;

// Allow use of default import syntax in TypeScript
module.exports.default = axios;

```


那createInstance干了些啥呢？

```javascript

/**
* Create an instance of Axios
*
* @param {Object} defaultConfig The default config for the instance
* @return {Axios} A new instance of Axios
*/
function createInstance(defaultConfig) {
    var context = new Axios(defaultConfig);
    var instance = bind(Axios.prototype.request, context);

    // Copy axios.prototype to instance
    utils.extend(instance, Axios.prototype, context);

    // Copy context to instance
    utils.extend(instance, context);

    return instance;
}

```

1. 先通过构造方法Axios创建context
2. 再通过bind.js中的bind方法，将Axios原型方法request映射到instance对象，即可以直接通过Axios()的方式调用，这段有点绕，之后会详细说说
3. 调用utils.js中的extends方法，将Axios的原型绑定到instance对象，注意，这里的context是作为调用的属性参数用，这个extends方法挺别致的，之后会详细读一下
4. 调用utils.js中的extends方法，将context的各个属性复制到instance对象
5. return instance对象

这个构造方法虽然只有短短的五句话，但是&#8482;的非常的绕，看的时候都有点晕，接下来我们慢慢的来阅读

### Axios.js

首先是Axios.js，这个文件无论从文件名字还是文件位置(core/Axios.js)，都毫无疑问的是整个库的核心部分了，先看这段

```javascript

/**
* Create a new instance of Axios
*
* @param {Object} instanceConfig The default config for the instance
*/
function Axios(instanceConfig) {
    this.defaults = instanceConfig;
    this.interceptors = {
        request: new InterceptorManager(),
        response: new InterceptorManager()
    };
}

```

这是个es5的类的构造方法，class写多了看到这段还真有点怀念，不得不说**es6大法好**

这里定义了Axios类的两个属性，defaults和interceptors，

- defaults应该是构造方法的参数  
- interceptors则声明了两个属性request和response，这两个属性是InterceptorManager类的两个实例，这应该是处理请求中间件用的  

既然request和response都是InterceptorManager的实例，那就先看看InterceptorManager到底是组撒的

#### InterceptorManager.js

先看构造方法

```javascript

function InterceptorManager() {
    this.handlers = [];
}

```

就定义了一个handlers数组，接着往下看

```javascript
/**
 * Add a new interceptor to the stack
 *
 * @param {Function} fulfilled The function to handle `then` for a `Promise`
 * @param {Function} rejected The function to handle `reject` for a `Promise`
 *
 * @return {Number} An ID used to remove interceptor later
 */
InterceptorManager.prototype.use = function use(fulfilled, rejected) {
  this.handlers.push({
    fulfilled: fulfilled,
    rejected: rejected
  });
  return this.handlers.length - 1;
};
```

在InterceptorManager的原型上声明了一个use方法，先不看注释不看内容，光看这两个参数fulfilled和rejected，眼熟不？这不就是promise吗？莫非又是自己实现了一个promise？额...我为什么要说又.....

注释说的清楚明白，use方法就是将一个中间件push到handlers数组中，传入中间件的fulfilled用来then的，rejected用来catch(reject)，然后返回这个中间件的id(其实就是它的index)

接着看

```javascript

/**
* Remove an interceptor from the stack
*
* @param {Number} id The ID that was returned by `use`
*/
InterceptorManager.prototype.eject = function eject(id) {
    if (this.handlers[id]) {
        this.handlers[id] = null;
    }
};

```

这个eject很简单，就是根据id将handlers中的中间件置空，但是搜索了下，没有找到调用点，这个之后看实际调用的时候再看

接着看

```javascript

/**
* Iterate over all the registered interceptors
*
* This method is particularly useful for skipping over any
* interceptors that may have become `null` calling `eject`.
*
* @param {Function} fn The function to call for each interceptor
*/
InterceptorManager.prototype.forEach = function forEach(fn) {
    utils.forEach(this.handlers, function forEachHandler(h) {
        if (h !== null) {
            fn(h);
        }
    });
};

```

这个forEach看上去挺简单的，其实不然，不信？让我们层层抽丝剥茧的来看

1. 首先，它接收的参数是一个方法fn，头等公民嘛，你懂的；

2. 然后，他调用了utils中的forEach方法，这个utils是一个工具集合，里边集成了几乎所有axios库用到的工具，让我们看看有多少：

```javascript

    module.exports = {
        isArray: isArray,
        isArrayBuffer: isArrayBuffer,
        isBuffer: isBuffer,
        isFormData: isFormData,
        isArrayBufferView: isArrayBufferView,
        isString: isString,
        isNumber: isNumber,
        isObject: isObject,
        isUndefined: isUndefined,
        isDate: isDate,
        isFile: isFile,
        isBlob: isBlob,
        isFunction: isFunction,
        isStream: isStream,
        isURLSearchParams: isURLSearchParams,
        isStandardBrowserEnv: isStandardBrowserEnv,
        forEach: forEach,
        merge: merge,
        deepMerge: deepMerge,
        extend: extend,
        trim: trim
    };

```

简直就是百宝箱有木有，重新定义了js有木有，真心讲究有木有

3. 注意，接收的参数fn是在utils的forEach的回调方法里才会用到....md闭包大法好

utils.js中的各个方法会在之后用到它们的地方再聊，这里先看看utils里的forEach到底干了什么

##### utils.js@forEach

```javascript

    /**
    * Iterate over an Array or an Object invoking a function for each item.
    *
    * If `obj` is an Array callback will be called passing
    * the value, index, and complete array for each item.
    *
    * If 'obj' is an Object callback will be called passing
    * the value, key, and complete object for each property.
    *
    * @param {Object|Array} obj The object to iterate
    * @param {Function} fn The callback to invoke for each item
    */

    function forEach(obj, fn) {
        // Don't bother if no value provided
        if (obj === null || typeof obj === 'undefined') {
            return;
        }

        // Force an array if not already something iterable
        if (typeof obj !== 'object') {
            /*eslint no-param-reassign:0*/
            obj = [obj];
        }

        if (isArray(obj)) {
            // Iterate over array values
            for (var i = 0, l = obj.length; i < l; i++) {
                fn.call(null, obj[i], i, obj);
            }
        } else {
            // Iterate over object keys
            for (var key in obj) {
                if (Object.prototype.hasOwnProperty.call(obj, key)) {
                    fn.call(null, obj[key], key, obj);
                }
            }
        }
    }
```

utils的forEach接收两个参数，

- 要迭代的对象obj，可以是数组，也可以是对象，如果是数组，则回调时带上值，下标和完整的数组，如果是对象，则回调时带上值，属性名和完整的对象
- 回调方法fn

首先是一段js式的判空，注意！null，undefinedは违うのだよ、undefinedは！(null和undefined是不同的！和undefined！)

然后判断如果obj不是对象，则强制转换为数组...方便迭代

```javascript

    // Force an array if not already something iterable
    if (typeof obj !== 'object') {
        /*eslint no-param-reassign:0*/
        obj = [obj];
    }
```

obj = [obj]; 对，就是这么暴力，暴力到要加eslint注释来防止警告，顺便一提，no-param-reassign是禁止对函数参数再赋值

接下来遍历数组/对象属性

```javascript

    if (isArray(obj)) {
        // Iterate over array values
        for (var i = 0, l = obj.length; i < l; i++) {
            fn.call(null, obj[i], i, obj);
        }
    } else {
        // Iterate over object keys
        for (var key in obj) {
            if (Object.prototype.hasOwnProperty.call(obj, key)) {
                fn.call(null, obj[key], key, obj);
            }
        }
    }
```

先调用isArray判断是否是数组，这个isArray方法是怎么判断的？

```javascript

    function isArray(val) {
        return toString.call(val) === '[object Array]';
    }
```

toString就是Object原型上的toString方法，

```javascript
    var toString = Object.prototype.toString;
```

如果是Array，那调用Object的toString应该会打印出'[object Array]'，这样判断是否是数组真的是js的特色了，PS：es6可以用Array.isArray来判断是否是数组，哎，**es6大法好**

回到forEach部分，

- 如果obj是数组，那么就遍历该数组，然后依次通过回调方法调用数组的内容，传递的参数依次是值，下标和数组本身
- 如果obj是对象，则遍历该对象的自身属性，并通过回调方法调用该属性，传递的参数依次是值，属性名称和obj对象本身

```javascript

        for (var key in obj) {
            if (Object.prototype.hasOwnProperty.call(obj, key)) {
                fn.call(null, obj[key], key, obj);
            }
        }
```

```text
这里要注意下for in和hasOwnProperty，for in是es6之前的遍历对象属性的方法，循环只遍历可枚举属性，遍历的是对象的属性名，一般都和hasOwnProperty配合使用，以枚举自身的属性而不包含继承的属性，当然es6之后一般都使用for of或者Object.keys()/Object.values()，es6大法好
```

**总之，utils.forEach就做一件事情，遍历传入的obj参数的值，并通过传入的回调方法fn处理遍历的值，与es6的Array.forEach类似，而且同时还能处理object的属性，这个方法挺重要，你将会在整个axios库里一直见到它。**

然后回到InterceptorManager，我们继续看InterceptorManager中的forEach

```javascript
    InterceptorManager.prototype.forEach = function forEach(fn) {
        utils.forEach(this.handlers, function forEachHandler(h) {
            if (h !== null) {
                fn(h);
            }
        });
    };
```

调用utils.forEach，传递的参数是InterceptorManager的handlers数组，回调方法forEachHandler则在utils.forEach遍历并回调时，先判断h(obj)是否为空，不为空的话则执行fn方法，参数即遍历的值

这tm回调回调再回调的过程，很晕，只能在实际调用的时候再解析了，现在先大致了解整个流程

好，现在可以回到Axios.js了(一种递归要完成的感觉)，先看两段forEach

```javascript

    // Provide aliases for supported request methods
    utils.forEach(['delete', 'get', 'head', 'options'], function forEachMethodNoData(method) {...});

```

这段里的['delete', 'get', 'head', 'options']很清楚，就是一些http的[request method](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods)，从处理方法forEachMethodNoData来看，这些应该都是不带data的method，而这段代码是什么意思呢？那就让我们顺便来走一遍utils.forEach的流程

- 这里调用了utils.forEach，参数是['delete', 'get', 'head', 'options']和 function forEachMethodNoData(method)

- 进入forEach循环，对应的obj即['delete', 'get', 'head', 'options']，fn即forEachMethodNoData

    1. 判断obj是否为空，这里obj非空，是一个数组
    2. 判断obj是否为object，这里obj不是对象
    3. 判断obj是否为数组，obj是数组，则进行数组遍历

        ```javascript
        for (var i = 0, l = obj.length; i < l; i++) {
            fn.call(null, obj[i], i, obj);
        }
        ```

    4. 遍历数组，数组第一个值为'delete'，调用fn即forEachMethodNoData，传入参数'delete'（当前项）, 0（index，位置）, ['delete', 'get', 'head', 'options']（数组本身）
    5. 执行forEachMethodNoData，定义如下：

        ```javascript

        function forEachMethodNoData(method) {
            /*eslint func-names:0*/
            Axios.prototype[method] = function(url, config) {
                return this.request(utils.merge(config || {}, {
                    method: method,
                    url: url
                }));
            };
        });
        ```

    6. 因为方法forEachMethodNoData只接收一个参数method，则method的值为'delete'
    7. 执行语句Axios.prototype[method] = function(url, config){..}，即Axios.prototype['delete'] = function(url, config){...}
    8. 继续遍历，调用fn，传入参数'get', 1, ['delete', 'get', 'head', 'options']
    9. 如此反复直到遍历结束

可见，这步就是将这些method声明成Axios原型上的方法，即可以直接通过Axios实例axios.get(), axios.delete()的方式发起请求，循环结束后，Axios.prototype上则包含了'delete', 'get', 'head', 'options'这些方法

```javascript
    utils.forEach(['post', 'put', 'patch'], function forEachMethodWithData(method) {
        /*eslint func-names:0*/
        Axios.prototype[method] = function(url, data, config) {
            return this.request(utils.merge(config || {}, {
                method: method,
                url: url,
                data: data
            }));
        };
    });
```

这段里的['post', 'put', 'patch']也是一些http的[request method](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods)，不需要从forEachMethodWithData来看，直接看post，put，你就知道这些应该都是带data的method，处理同上，将这些method声明成Axios原型上的方法，顺便提一下patch，这个不常用，在HTTP协议中，请求方法 [PATCH](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Methods/PATCH)  用于对资源进行部分修改。然后需要注意的是这个method是非幂等的。

再顺便一提http的method有哪些：

```text

GET

The GET method requests a representation of the specified resource. Requests using GET should only retrieve data.

PUT

The PUT method replaces all current representations of the target resource with the request payload.

DELETE

The DELETE method deletes the specified resource.

CONNECT

The CONNECT method establishes a tunnel to the server identified by the target resource.

OPTIONS

The OPTIONS method is used to describe the communication options for the target resource.

```

如果面试时面试官问你这个，你就说一些基本的就行，比如get,post,put,delete,options这几个，然后注意下哪些是幂等(idempotent)，哪些非幂等(no-idempotent side effects)，比如get,put,delete,options都是幂等的，post是非幂等的，其实这些method里只有post和patch是非幂等的......[幂等](https://developer.mozilla.org/zh-CN/docs/Glossary/%E5%B9%82%E7%AD%89)这个名词在RESTful中时常被提起，这里稍微说道说道

- 幂等： ~~请求之后不会改变服务器的值，~~ 同样的请求被执行一次与连续执行多次的效果是一样的，服务器的状态也是一样的
  - 比如get请求获取你的昵称，你的昵称并不会因为你get而改变，你get多次都不会改变你的昵称  
  - 一般来说，get请求应该是幂等的，但是也见过有很多使用get请求来传递参数并导致服务器的资源多次修改，这当然不能说是错的，但是至少是不RESTful的

- 非幂等： 同样的请求被执行一次与连续执行多次的效果是不同的，服务器的状态也是会变化的
  - 比如post请求添加你的联系方式，你post提交新的联系方式后，你的联系方式就会新增，你post多次，会新增多次联系地址
  - 同样的，也见过很多以幂等方式使用post方法的，比如为了隐藏掉get请求暴露的url参数，所有的请求都使用post，以传递token字段或一些自定义信息，这当然也不能说是错的或者不合理的，但是至少也是不RESTful的

**值得注意的是，所谓的幂等性所指的究竟是什么？**

**按MDN的说法，“幂等性只与后端服务器的实际状态有关”，这实在是有点语焉不详...**

**我的理解很简单，既然RESTful定义了一切都是资源，那么幂等性指的就是该操作对资源的操作造成的结果是否一致的。**

> 这样就能理解为什么post是非幂等而put却是幂等的，post更类似于新增(Create)资源的操作，多次post，则会新增多个资源，而put就更类似更新(Update)的操作，当修改的值(例如put带有的参数)是确定的，那多次修改对资源造成的结果是一致的，当然put也是可以新增资源的，当要修改的资源不存在，服务器应该新增该put的资源，之后再进行多次put，对该资源造成的结果是一致的

> 同样，也能很好的理解为什么delete也是幂等操作，因为按照定义，delete操作只能删除(Delete)对应的资源，就像MDN上所说“开发者不应该使用DELETE方法实现具有删除最后条目功能的 RESTful API。”所以delete操作应该设计成只能删除对应的资源，比如delete("abc")，只能删除"abc"这个条目，不应该设计成deleteNext()，每执行一次就删除下一条条目

从[HTTP协议](https://tools.ietf.org/html/rfc7231#section-4.2.2)对幂等的定义，就能更好的理解为什么要设计幂等方法，

```text
Idempotent methods are distinguished because the request can be repeated automatically if a communication failure occurs before the client is able to read the server's response.
```

幂等方法可以在链接发送故障时，客户端能够读取服务器资源前，自动重复请求，而不会造成任何副作用

```text
For example, if a
client sends a PUT request and the underlying connection is closed
before any response is received, then the client can establish a new
connection and retry the idempotent request.  It knows that repeating
the request will have the same intended effect, even if the original
request succeeded, though the response might differ.
```

例如通过put方法更新服务器某个资源，即使服务器挂了/网线被人拔了/wifi被屏蔽了，再连接重新建立后，可以重试幂等请求，无论怎么请求，请求几次，只要是幂等的，就不会有副作用

然后让我们回到Axios.js(又扯远了...)，通过两段forEach，Axios的原型上已经有了一堆方法了，然而这些method对应的方法，最终都会调用到this.request，另一个声明在Axios原型上的方法

```javascript
    /**
    * Dispatch a request
    *
    * @param {Object} config The config specific for this request (merged with this.defaults)
    */
    Axios.prototype.request = function request(config) {
        /*eslint no-param-reassign:0*/
        // Allow for axios('example/url'[, config]) a la fetch API
        if (typeof config === 'string') {
            config = arguments[1] || {};
            config.url = arguments[0];
        } else {
            config = config || {};
        }

        config = mergeConfig(this.defaults, config);

        // Set config.method
        if (config.method) {
            config.method = config.method.toLowerCase();
        } else if (this.defaults.method) {
            config.method = this.defaults.method.toLowerCase();
        } else {
            config.method = 'get';
        }

        // Hook up interceptors middleware
        var chain = [dispatchRequest, undefined];
        var promise = Promise.resolve(config);

        this.interceptors.request.forEach(function unshiftRequestInterceptors(interceptor) {
            chain.unshift(interceptor.fulfilled, interceptor.rejected);
        });

        this.interceptors.response.forEach(function pushResponseInterceptors(interceptor) {
            chain.push(interceptor.fulfilled, interceptor.rejected);
        });

        while (chain.length) {
            promise = promise.then(chain.shift(), chain.shift());
        }

        return promise;
    };
```

这块代码涉极到的部分特别多，看的时候真是头大，没办法，抽丝剥茧的康康

首先接受的参数是config，这个config是整个请求发送的重要参数，而且它灵活多变，可以是各种类型(伟大的js)，所以处理起来也是各种情况都要考虑

1. 判断config是否是string，因为config允许以axios('example/url'[, config])的方式调用，即可以只是url，也可以是带参数的
2. 如果是string，那么将config.url赋值为第一个调用参数(arguments[0])，即例子中的'example/url'，同时将config赋值为第二个调用参数(arguments[1])，如果没第二个调用参数，则赋值为新对象{}；

    注意这两个语句的顺序，先尝试构建config对象，再将config.url赋值为第一个调用参数
3. 如果config不是string，那么就尝试构建config对象，config = config || {}; 标准的js对象初始化语句
4. 经过初始处理过的config，要再经过一次mergeConfig操作，WTF，先康康mergeConfig是什么操作

    > /lib/core/mergeConfig.js

    ```javascript

            /**
            * Config-specific merge-function which creates a new config-object
            * by merging two configuration objects together.
            *
            * @param {Object} config1
            * @param {Object} config2
            * @returns {Object} New object resulting from merging config2 to config1
            */
            module.exports = function mergeConfig(config1, config2) {
                // eslint-disable-next-line no-param-reassign
                config2 = config2 || {};
                var config = {};

                var valueFromConfig2Keys = ['url', 'method', 'params', 'data'];
                var mergeDeepPropertiesKeys = ['headers', 'auth', 'proxy'];
                var defaultToConfig2Keys = [
                    'baseURL', 'url', 'transformRequest', 'transformResponse', 'paramsSerializer',
                    'timeout', 'withCredentials', 'adapter', 'responseType', 'xsrfCookieName',
                    'xsrfHeaderName', 'onUploadProgress', 'onDownloadProgress',
                    'maxContentLength', 'validateStatus', 'maxRedirects', 'httpAgent',
                    'httpsAgent', 'cancelToken', 'socketPath'
                ];

                utils.forEach(valueFromConfig2Keys, function valueFromConfig2(prop) {
                    if (typeof config2[prop] !== 'undefined') {
                        config[prop] = config2[prop];
                    }
                });

                utils.forEach(mergeDeepPropertiesKeys, function mergeDeepProperties(prop) {
                    if (utils.isObject(config2[prop])) {
                        config[prop] = utils.deepMerge(config1[prop], config2[prop]);
                    } else if (typeof config2[prop] !== 'undefined') {
                        config[prop] = config2[prop];
                    } else if (utils.isObject(config1[prop])) {
                        config[prop] = utils.deepMerge(config1[prop]);
                    } else if (typeof config1[prop] !== 'undefined') {
                        config[prop] = config1[prop];
                    }
                });

                utils.forEach(defaultToConfig2Keys, function defaultToConfig2(prop) {
                    if (typeof config2[prop] !== 'undefined') {
                        config[prop] = config2[prop];
                    } else if (typeof config1[prop] !== 'undefined') {
                        config[prop] = config1[prop];
                    }
                });

                var axiosKeys = valueFromConfig2Keys
                    .concat(mergeDeepPropertiesKeys)
                    .concat(defaultToConfig2Keys);

                var otherKeys = Object
                    .keys(config2)
                    .filter(function filterAxiosKeys(key) {
                        return axiosKeys.indexOf(key) === -1;
                    });

                utils.forEach(otherKeys, function otherKeysDefaultToConfig2(prop) {
                    if (typeof config2[prop] !== 'undefined') {
                        config[prop] = config2[prop];
                    } else if (typeof config1[prop] !== 'undefined') {
                        config[prop] = config1[prop];
                    }
                });

                return config;
            };
    ```

    我***[脏话]，太长，不看....

    好吧，我开玩笑的，这里总结下，调用mergeConfig，
    - 会先遍历http请求可能涉及到的参数名
    - 尝试以这些参数名为基础合并传入的config1和config2参数
    - 如果config1和config2同时包含的参数，则优先使用config2的值
    - 当config2存在其他的参数名时，也同样会进行赋值，赋值规则同上，优先使用config2的值

    我这边举个例子：

    ```javascript
            config1: {
                url: "aaa/aaa",
                method: "get",
                others1: {}
            }
            config2: {
                url: "bbb/bbb",
                method: "post",
                data: {},
                others2: { value: "others" },
            }
    ```

    合并后：

    ```javascript
        config: {
            url: "bbb/bbb",
            method: "post",
            data: {},
            others1: {},
            others2: { value: "others" }
        }
    ```

    好，继续让我们回到Axios.prototype.request....

5. 得到merge之后的config后，设置config.method
    - 如果config带有method参数，转成小写后直接用
    - 如果没有，则使用this.defaults.method
    - 如果defaults里也没method，那么直接用'get'
6. 接下来是重头戏了，处理中间件
    - 首先定义 chain数组, 默认有两个值, dispatchRequest和undefined....  
        undefined先不管，我们先看看dispatchRequest

        dispatchRequest.js

        ```javascript

            /**
            * Dispatch a request to the server using the configured adapter.
            *
            * @param {object} config The config that is to be used for the request
            * @returns {Promise} The Promise to be fulfilled
            */
            module.exports = function dispatchRequest(config) {
                throwIfCancellationRequested(config);

                // Ensure headers exist
                config.headers = config.headers || {};

                // Transform request data
                config.data = transformData(
                    config.data,
                    config.headers,
                    config.transformRequest
                );

                // Flatten headers
                config.headers = utils.merge(
                    config.headers.common || {},
                    config.headers[config.method] || {},
                    config.headers || {}
                );

                utils.forEach(
                    ['delete', 'get', 'head', 'post', 'put', 'patch', 'common'],
                    function cleanHeaderConfig(method) {
                        delete config.headers[method];
                    }
                );

                var adapter = config.adapter || defaults.adapter;

                return adapter(config).then(function onAdapterResolution(response) {
                    throwIfCancellationRequested(config);

                    // Transform response data
                    response.data = transformData(
                        response.data,
                        response.headers,
                        config.transformResponse
                    );

                    return response;
                }, function onAdapterRejection(reason) {
                    if (!isCancel(reason)) {
                        throwIfCancellationRequested(config);

                        // Transform response data
                        if (reason && reason.response) {
                            reason.response.data = transformData(
                            reason.response.data,
                            reason.response.headers,
                            config.transformResponse
                            );
                        }
                    }

                    return Promise.reject(reason);
                });
            };
        ```

        我靠...又那么长....不过这个比较重要，我们一句句看
        dispatchRequest接受参数config，就是之前经过了重重处理的config，

        先调用throwIfCancellationRequested

        ```javascript
            /**
            * Throws a `Cancel` if cancellation has been requested.
            */
            function throwIfCancellationRequested(config) {
                if (config.cancelToken) {
                    config.cancelToken.throwIfRequested();
                }
            }
        ```

        简单的说就是如果config带有cancelToken，就调用cancelToken的throwIfRequested方法取消发送过的请求

    - 接下来，处理相对路径的url，如果config.url是绝对路径，则直接使用，反之若存在baseURL且config.url不是绝对路径，则将两个url进行合并操作

        ```javascript
                // Support baseURL config
                if (config.baseURL && !isAbsoluteURL(config.url)) {
                    config.url = combineURLs(config.baseURL, config.url);
                }
        ```

        其中isAbsoluteURL是用来判断是否是绝对路径，如果是以```<scheme>//```或者```//```开头的，则是绝对路径，反之则是不是，判断是用了如下正则表达式

        ```javascript
            /^([a-z][a-z\d\+\-\.]*:)?\/\//i.test(url)
        ```

        combineURLs的操作方式则是去掉baseURL的最后的```/```，去掉relativeURL的第一个```/```，然后用```/``` 将两个字符串合并起来，orz

    - 接着处理header

        ```javascript
            // Ensure headers exist
            config.headers = config.headers || {};
        ```

        标准的js处理方式，如果没有定义header，则声明header为一个空对象
