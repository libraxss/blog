万物始于无
=========
ECMAScript的Null类型介绍
================================

> 简单介绍下js中的原始类型Null及与Undefine的区别

> 请允许我使用ECMAScript作为标题，下文中则统一简写为js

> 标题只是用来装逼的


不久前，某人问了我一个问题，是关于js中null的，<br/>
当时我就一个反应，这&#8482;还要问！？<br/>
但是在查阅了资料后，发现还是有点说头的，尤其我原先的一些理解也是错误的<br/>
那今天就来说一说这个null

<br/>

## js中的原始类型 ##
>原始类型/基本数据类型/简单数据类型/primitive value

其实这四种称呼的是一种东西，这里统一称呼为**原始类型**（因为感觉比较厉害）<br/>
众所周知，主流编程语言都会有自己的原始类型，js也有其原始类型，<br>
一般来说，有一下6种：
- null
- undefined
- boolean
- string
- number
- symbol（ECMAScript 6 新定义）

当然js还有一种类型
- object

但是个人理解，这个应该归类为引用类型

关于原始类型的概念，可参照MDN的定义:
>原始数据 （原始值、原始数据类型）不是一种 object 类型并且没有自己的方法的。在 JavaScript 中，有六种原始数据类型：string，number，boolean，null，undefined，symbol (new in ECMAScript 2015)。<br/><br/>
大多数时候，原始值是由最底层的语言直接表现的。<br/><br/>
所有的原始数据都是不变的（即不能被改变）。


同时，关于原始类型与引用类型的区别，可参照stackoverflow的如下回答：

>友情提示: 手工翻译，错了不管<br/>
>Primitive values are data that are stored on the stack.
Primitive value is stored directly in the location that the variable accesses.<br/>
原始类型的数据是存储在栈中的数据，原始值可直接通过变量进行访问
<br/>
<br/>
Reference values are objects that are stored in the heap.<br/>
Reference value stored in the variable location is a pointer to a location in memory where the object is stored.<br/>
引用类型的数据是存储在堆中，变量通过内存中存储的指针来访问
<br/>
<br/>
Primitive types include Undefined, Null, Boolean, Number, or String.<br/>
原始类型包括Undefined, Null, Boolean, Number, 与 String
<br/>
<br/>
The Basics:<br/>
Objects are aggregations of properties. A property can reference an object or a primitive. Primitives are values, they have no properties.<br/>
对象是各种属性的聚合。属性可以是一个对象或者一个原始类型。原始类型都是值，并不具有属性
<br/>
<br/>
Updated:<br/>
JavaScript has 5 primitive data types: string, number, boolean, null, undefined. With the exception of null and undefined, all primitives values have object equivalents which wrap around the primitive values, e.g. a String object wraps around a string primitive. All primitives are immutable
（这部分跟上面有点重复，挑重点说，就是原始类型是不可变的）.



其他类型暂且不表，今天就主要介绍下null与undefined

## Null/null Undefined/undefined##
关于null的定义，ECMA文档中写的很简单：

>4.3.12 null value
<br/>primitive value that represents the intentional absence of any object value
<br/>表示有意缺少任何对象值的原始值
<br/>
4.3.13 Null type
<br/>
type whose sole value is the null value
<br/>Null类型只有一个值，就是null.......


[ecma-262 6.0](http://www.ecma-international.org/ecma-262/6.0/#sec-null-value)


>6.1.2The Null Type#
>
>The Null type has exactly one value, called null.
><br/>Null类型只有一个值，就是null.......

[ecma-262 7.0](http://www.ecma-international.org/ecma-262/6.0/#sec-null-value)

而undefiled的定义也不复杂：

>4.3.10 undefined value
<br/>
primitive value used when a variable has not been assigned a value
<br/>
一种原始值，用来作为没有赋值的变量的默认值
<br/>
<br/>
4.3.11 Undefined type
<br/>
type whose sole value is the undefined value
<br/>Undefined类型也只有一个值，就是undefined.......


[ecma-262 6.0](http://www.ecma-international.org/ecma-262/6.0/#sec-null-value)




## 异同 ##
### 同 ###
大多数的使用场景下，他们其实没有太大的区别：

    var a = undefined;

    var a = null;

使用上似乎没什么太大区别，作为boolean使用他们都会转换为false

    var exist = undefined;
    if (exist) {
        console.log("exist!");
    } else {
        console.log("not exist!");
    }//输出"not exist!"

    var exist = null;
    if (exist) {
        console.log("exist!");
    } else {
        console.log("not exist!");
    }//输出"not exist!"

甚至用"=="连接两者，他们是相等的

    console.log(null == undefined);// 输出true


### 异 ###
#### 1.从定义中可以看出，他们两者是不同，至少从设计用途来看，####

null表示**有意缺少**任何对象值的原始值,

undefined用来作为**没有赋值**的变量的默认值


从使用场景看，undefined是作为声明未赋值变量的默认值，例如：

    var a;
    console.log(typeof a);//输出"undefined"

而null是用来表示一个空的值，而这个空值是使用时赋予的：

    var a = null;
    console.log(typeof a);//输出"null"


#### 2.表示类型不同  ####

这不是句废话.....

从含义上来说，

null是一个空的值(不是对象，只是空值)，是一个值

undefined则是一个全局的属性，甚至在ES3中， 可以为这个属性赋值

>undefined is a property of the global object (and thus a global variable; see The Global Object).
<br/> Under ECMAScript 3, you had to take precautions when reading undefined, because it was easy to accidentally change its value.
<br/> Under ECMAScript 5, that is not necessary, because undefined is read-only.

[Speaking JavaScript: occurrences_of_undefined_and_null](http://speakingjs.com/es5/ch08.html#_occurrences_of_undefined_and_null)



## 小结 ##
Null类型是ECMAScript的一种原始类型，用来表示空这个值，<br/>
在使用时，可以用来为变量赋值，以表示该变量指向的是一个空值


## 误区/解释 ##
通常在使用中会有如下的误区/疑惑，让我们来一一解释：

### 1. typeof null == "object"? ###
A: 这是个比较经典的问题了，几乎所有人在初次接触js都会遇到这个问题：

    console.log(typeof null);//结果输出"object"
这是为什么捏？
<br/>
有两种主流的解释：
- null值表示一个空对象指针，而这也正是使用typeof操作符检测null值会返回object的原因

- 这是一个bug

个人更倾向于后者，其实，在ECMA的修订过程中，曾有一版是将输出的值改为"null",但之后为了兼容性，又改回来了(具体待查)....


### 2. null + 1 = 1? ###
A: 这又是个比较神奇的地方了，如果换成undefined,你又会发现：

    console.log(null + 1);//输出1
    console.log(undefined + 1);//报错












</end>
