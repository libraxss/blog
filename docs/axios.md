# axios源码阅读笔记

> 为啥要读axios源码  
> axios 是什么  
> axios 怎么写的

写了4年前端，却没有完整的阅读过任一知名js框架源码，这样不好，虽然目前没有跳槽的打算，但是出去面试的确爱问这个

最近闲来无事，那就从头来读一读axios的源码

至于为什么要读axios，主要是因为其他知名库比如vue/react的源码太长，而且解读的人太多，另外平时工作中，网络请求一般都是自己封装一个简单XMLHttpRequest，所以想看看这个牛逼哄哄的axios为什么这么dio

## axios是什么

axios，是当前非常著名的前端库，主要用于处理封装前端的请求，是一款非常好用的前端库，顺便一提查到axios的读音读作艾克C欧丝

## 让我们开始吧

### 先搞到axios

搞源码比搞钱容易的多的多的多的多...上gayhub上一搜就搜到了

[axios - Promise based HTTP client for the browser and node.js](https://github.com/axios/axios.git)

看看这清晰的介绍，看看这华丽的布局，看看这详尽的用法，一看就知道是大师的手笔，比一些个人作坊的连readme都懒得写的库不知道高到哪里去啊~

哎，以后牛逼了一定也要自己照这样整一个

### 源码先看package.json

下好源码，先看什么？readme？？No，都说了一些库连&#8482; readme都不写，所以当然必须先看工程户口本package.json

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
        ......
        "webpack-dev-server": "^1.14.1"
    },
    "dependencies": {
        "follow-redirects": "1.5.10",
        "is-buffer": "^2.0.2"
    }

我擦，这么多依赖，不愧是知名库，讲究

顺便一提devDependencies和dependencies的区别，顾名思义，

devDependencies是在开发中用到的，可能是一些单元测试或者代码检查工具之类的

dependencies是发布时用到的，包含所有发布时会用到的依赖

简单的说就是发布时用不到的全部扔到devDependencies里，来来来，大家可以立刻检查检查是不是自己之前写的工程直接npm install --save安装依赖，然后dependencies里一坨(笔者就是...)

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

构造方法createInstance干了些啥呢？

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

首先是Axios.js，这个从名字还是位置，都毫无疑问的是整个库的核心部分了，先看这段

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

这是个es5的类的构造方法，class写多了看到这段还真有点怀念，不得不说es6大法好

