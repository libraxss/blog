# axios源码阅读笔记

> 为啥要读axios源码  
> axios 是什么  
> axios 怎么写的

写了几年前端，却没有完整的阅读过任一知名js框架源码，这样不好，虽然目前没有跳槽的打算，但是出去面试的确爱问这个

最近闲来无事，那就从头来读一读axios的源码

至于为什么要读axios，主要是因为其他知名库比如vue/react的源码太长，而且解读的人太多，另外平时工作中，网络请求一般都是自己封装一个简单XMLHttpRequest，所以想看看这个牛逼哄哄的axios为什么这么dio

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

    "name": "axios",
    "version": "0.19.0",
    "description": "Promise based HTTP client for the browser and node.js",
    "main": "index.js",

看源码，其实主要找到main这行就行了，不过也可以顺便看看还有哪些

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

不愧是知名库，脚本这叫一个全，讲究

    "devDependencies": {
        "bundlesize": "^0.17.0",
        ...... x 12
        "webpack-dev-server": "^1.14.1"
    },
    "dependencies": {
        "follow-redirects": "1.5.10",
        "is-buffer": "^2.0.2"
    }

我擦，这么多依赖，不愧是知名库，讲究

顺便一提devDependencies和dependencies的区别，顾名思义，

> devDependencies是在开发中用到的，可能是一些单元测试或者代码检查工具之类的  
> dependencies是发布时用到的，包含所有发布时会用到的依赖

简单的说就是发布时用不到的全部扔到devDependencies里，来来来，大家可以立刻检查检查自己之前写的工程是不是直接npm install --save安装依赖，然后dependencies里一坨(笔者就是...)

    "bundlesize": [
        {
        "path": "./dist/axios.min.js",
        "threshold": "5kB"
        }
    ]

额...bundlesize是啥属性？[Npm doc](https://docs.npmjs.com/files/package.json#browser)里没查到，网上搜了下，原来是个插件，主要用来指定具体文件的大小上限，一旦超过就会向你报警，不愧是知名库啊，真讲究

### index.js

好，让我们进入主题吧，既然main指向了/index.js，那我们直接看下index.js写了啥

    // index.js
    module.exports = require('./lib/axios');

就一行，入口嘛，正常操作，那我们就顺着脉络看看./lib/axios.js

    // Create the default instance to be exported
    var axios = createInstance(defaults);

    module.exports = axios;

    // Allow use of default import syntax in TypeScript
    module.exports.default = axios;

js模块，当然先从export部分来看，export的axios是一个由createInstance方法创建的对象，嗯，从创建方法的名字来看这就是个单例对象[狗头]

那createInstance干了些啥呢？

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

1. 先通过构造方法Axios创建context
2. 再通过bind.js中的bind方法，将Axios原型方法request映射到instance对象，即可以直接通过Axios()的方式调用，这段有点绕，之后会详细说说
3. 调用utils.js中的extends方法，将Axios的原型绑定到instance对象，注意，这里的context是作为调用的属性参数用，这个extends方法挺别致的，之后会详细读一下
4. 调用utils.js中的extends方法，将context的各个属性复制到instance对象
5. return instance对象

这个单例构造方法虽然只有短短的五句话，但是&#8482;的非常的绕，看的时候都有点晕，接下来我们慢慢的来阅读

### Axios.js

首先是Axios.js，这个文件无论从文件名字还是文件位置(core/Axios.js)，都毫无疑问的是整个库的核心部分了，先看这段

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

这是个es5的类的构造方法，class写多了看到这段还真有点怀念，不得不说**es6大法好**

这里定义了Axios类的两个属性，defaults和interceptors，

- defaults应该是构造方法的参数  
- interceptors则声明了两个属性request和response，这两个属性是InterceptorManager类的两个实例，这应该是处理请求中间件用的  

既然request和response都是InterceptorManager的实例，那就先看看InterceptorManager到底是组撒的

#### InterceptorManager.js

先看构造方法

    function InterceptorManager() {
        this.handlers = [];
    }

就定义了一个handlers数组，接着往下看

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

在InterceptorManager的原型上声明了一个use方法，先不看注释不看内容，光看这两个参数fulfilled和rejected，眼熟不？这不就是promise吗？莫非又是自己实现了一个promise？额...我为什么要说又.....

注释说的清楚明白，use方法就是将一个中间件push到handlers数组中，传入中间件的fulfilled用来then的，rejected用来catch(reject)，然后返回这个中间件的id(其实就是它的index)

接着看

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

这个reject很简单，就是根据id将handlers中的中间件置空，但是搜索了下，没有找到调用点，这个之后看实际调用的时候再看

接着看

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

这个forEach看上去挺简单的，其实不然，不信？让我们层层抽丝剥茧的来看

1. 首先，它接收的参数是一个方法fn，头等公民嘛，你懂的；

2. 然后，他调用了utils中的forEach方法，这个utils是一个工具集合，里边集成了几乎所有axios库用到的工具，让我们看看有多少：

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

    简直就是百宝箱有木有，重新定义了js有木有，真心讲究有木有

3. 注意，接收的参数fn是在utils的forEach的回调方法里才会用到....md闭包大法好

utils.js中的各个方法会在之后用到它们的地方再聊，这里先看看utils里的forEach到底干了什么

##### utils.js@forEach

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

utils的forEach接收两个参数，

- 要迭代的对象obj，可以是数组，也可以是对象，如果是数组，则回调时带上值，下标和完整的数组，如果是对象，则回调时带上值，属性名和完整的对象
- 回调方法fn

首先是一段js式的判空，注意！null，undefinedは违うのだよ、undefinedは！(null和undefined是不同的！和undefined！)

然后判断如果obj不是对象，则强制转换为数组...方便迭代

    // Force an array if not already something iterable
    if (typeof obj !== 'object') {
        /*eslint no-param-reassign:0*/
        obj = [obj];
    }

obj = [obj]; 对，就是这么暴力，暴力到要加eslint注释来防止警告，顺便一提，no-param-reassign是禁止对函数参数再赋值

接下来遍历数组/对象属性

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

先调用isArray判断是否是数组，这个isArray方法是怎么判断的？

    function isArray(val) {
        return toString.call(val) === '[object Array]';
    }

toString就是Object原型上的toString方法，

    var toString = Object.prototype.toString;

如果是Array，那调用Object的toString应该会打印出'[object Array]'，这样判断是否是数组真的是js的特色了，PS：es6可以用Array.isArray来判断是否是数组，哎，**es6大法好**

回到forEach部分，

- 如果obj是数组，那么就遍历该数组，然后依次通过回调方法调用数组的内容，传递的参数依次是值，下标和数组本身
- 如果obj是对象，则遍历该对象的自身属性，并通过回调方法调用该属性，传递的参数依次是值，属性名称和obj对象本身

        for (var key in obj) {
            if (Object.prototype.hasOwnProperty.call(obj, key)) {
                fn.call(null, obj[key], key, obj);
            }
        }

    这里要注意下for in和hasOwnProperty，for in是es6之前的遍历对象属性的方法，循环只遍历可枚举属性，遍历的是对象的属性名，一般都和hasOwnProperty配合使用，以枚举自身的属性而不包含继承的属性，当然es6之后一般都使用for of或者Object.keys()/Object.values()，**es6大法好**

总之，utils的forEach就做一件事情，遍历obj参数的值，并通过回调调用遍历的值

然后回到InterceptorManager，我们继续看InterceptorManager中的forEach

    InterceptorManager.prototype.forEach = function forEach(fn) {
        utils.forEach(this.handlers, function forEachHandler(h) {
            if (h !== null) {
                fn(h);
            }
        });
    };

调用utils.forEach，传递的参数是InterceptorManager的handlers数组，回调方法forEachHandler则在utils.forEach遍历并回调时，先判断h(obj)是否为空，不为空的话则执行fn方法，参数即遍历的值

这tm回调回调再回调的过程，很晕，只能在实际调用的时候再解析了，现在先大致了解整个流程

好，现在可以回到Axios.js了(一种递归要完成的感觉)，先看两段forEach

    // Provide aliases for supported request methods
    utils.forEach(['delete', 'get', 'head', 'options'], function forEachMethodNoData(method) {
        /*eslint func-names:0*/
        Axios.prototype[method] = function(url, config) {
            return this.request(utils.merge(config || {}, {
                method: method,
                url: url
            }));
        };
    });

这段里的['delete', 'get', 'head', 'options']很清楚，就是一些http的[request method](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods)，从forEachMethodNoData来看，这些应该都是不带data的method，而这段代码是什么意思呢？那就顺便让我们来走一遍utils.forEach的流程吧

- 这里调用了utils.forEach，参数是['delete', 'get', 'head', 'options']和 function forEachMethodNoData(method)

- 进入forEach函数体，对应的obj即['delete', 'get', 'head', 'options']，fn即forEachMethodNoData

    1. 判断obj是否为空，这里obj非空，是一个数组
    2. 判断obj是否为object，这里obj不是对象
    3. 判断obj是否为数组，obj是数组，则进行数组遍历

            for (var i = 0, l = obj.length; i < l; i++) {
                fn.call(null, obj[i], i, obj);
            }

    4. 遍历数组，数组第一个值为'delete'，调用fn即forEachMethodNoData，传入参数'delete', 0, ['delete', 'get', 'head', 'options']
    5. 执行forEachMethodNoData

            function forEachMethodNoData(method) {
                /*eslint func-names:0*/
                Axios.prototype[method] = function(url, config) {
                    return this.request(utils.merge(config || {}, {
                        method: method,
                        url: url
                    }));
                };
            });
    6. 因为回调方法只接收一个参数method，则method的值为'delete'
    7. 执行语句Axios.prototype[method] = function(url, config){..}，即Axios.prototype['delete'] = function(url, config){...}
    8. 继续遍历，调用fn，传入参数'get', 1, ['delete', 'get', 'head', 'options']
    9.如此反复直到遍历结束

可见，这步就是将这些method声明成Axios原型上的方法，即可以直接通过Axios实例axios.get(), axios.delete()的方式发起请求

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

这段里的['post', 'put', 'patch']也是一些http的[request method](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods)，不需要从forEachMethodWithData来看，直接看post，put，你就知道这些应该都是带data的method，处理同上，将这些method声明成Axios原型上的方法，顺便提一下patch，这个不常用，在HTTP协议中，请求方法 [PATCH](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Methods/PATCH)  用于对资源进行部分修改。然后需要注意的是这个method是非幂等的。

再顺便一提http的method有哪些：

    GET
    The GET method requests a representation of the specified resource. Requests using GET should only retrieve data.

    HEAD
    The HEAD method asks for a response identical to that of a GET request, but without the response body.

    POST
    The POST method is used to submit an entity to the specified resource, often causing a change in state or side effects on the server.

    PUT
    The PUT method replaces all current representations of the target resource with the request payload.

    DELETE
    The DELETE method deletes the specified resource.

    CONNECT
    The CONNECT method establishes a tunnel to the server identified by the target resource.

    OPTIONS
    The OPTIONS method is used to describe the communication options for the target resource.

    TRACE
    The TRACE method performs a message loop-back test along the path to the target resource.

    PATCH
    The PATCH method is used to apply partial modifications to a resource.

如果面试时面试官问你这个，你就说一些基本的就行，比如get,post,put,delete,options这几个，然后注意下哪些是幂等(no side effect)，哪些非幂等(side effect)，比如get,put,delete,options都是幂等的，post是非幂等的，其实这些method里只有post和patch都是非幂等的......额，如果都看到这里你还是不知道[幂等](https://developer.mozilla.org/zh-CN/docs/Glossary/%E5%B9%82%E7%AD%89)的意思并且没有上网查....你可以的，

- 幂等 请求之后不会改变服务器的值，请求被执行一次与连续执行多次的效果是一样的
    > 比如get请求获取你的昵称，你的昵称并不会因为你get而改变，你get多次都不会改变你的昵称  
    > 但是万一如果你get请求之后导致你的昵称被改了，那说明你请求的这个服务器错误地打破幂等性约束，你可以骂它一句垃圾，但是别被它听见了

- 非幂等 请求之后会改变服务器的值，多次请求会导致值多次改变
    > 比如post请求更新你的昵称，你post提交新的昵称后，你的昵称就会因为你的post而改变，你post多次，会多次改变昵称  
    > 同样的，如果你post之后你的昵称没变，那说明你请求的这个服务器可能已经挂了....赶紧联系网站管理员吧

然后让我们回到Axios.js(又扯远了...)，通过两段forEach，Axios的原型上已经有了一堆方法了，然而这些method对应的方法，都会调用到this.request，另一个声明在Axios原型上的方法

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

