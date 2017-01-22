# Js原型与原型链继承

------
    接触前端快一年了，说来惭愧，一直都是在埋头写写写，为了活动而活动，根本没时间好好对JavaScript这个有点特殊的编程语言做更深入的了解，所以今天就从JS最基础的原型和原型链开始讲起.....

------

## 什么是 JS的原型


我们创建的每个函数都有一个prototype属性，这个属性是一个指针，指向一个对象，这个对象的用途是包含可以由特定类型的所有实例共享的属性和方法。那么，prototype就是通过调用构造函数而创建的那个对象实例的原型对象。

这是什么鬼呢？简单的说，就是每个你定义的JS function对象(方法也是对象)都会有一个属性叫做prototype,它指向了一个对象,这个对象就是这个function的原型对象，简称原型。

可以写个function试一下:

    var f = functin() {}
    console.log(f.prototype);

输出如下结果：
![f.prototype][1]

可以看到，原型就是一个对象，他有两个属性，一个是叫constructor,一个叫\__proto__

据说JavaScript中，万物皆有原型，那我们来定义一个变量看看他的原型是什么
先从string开始：

    var s = '原型很牛逼';
    console.log(s.prototype);

输出结果：
![string.prototype][2]

哎，什么情况？
一定是string有问题，换个boolean试试

    console.log(false.prototype);

输出结果：
![此处输入图片的描述][3]

咋滴了...再试试

    var i = 1;
    console.log(i.prototype);
    var ar = [];
    console.log(ar.prototype);
    var jo = {};
    console.log(jo.prototype);

输出结果：
![undefined][4]

完了，世界观要崩塌了.....
prototype到底是不是原型？还是根本不是万物都有原型？
让我们来重塑下世界观...

先看看对于原型的定义：

> An Object's \__proto__ property references the same object as its internal [[Prototype]] (often referred to as "the prototype"), which may be an object or, as in the default case of Object.prototype.\__proto__, null . This property is an abstraction error, because a property with the same name, but some other value, could be defined on the object too. If there is a need to reference an object's prototype, the preferred method is to use Object.getPrototypeOf.


这段高大上的英文说了什么呢？总的来说，就是一个Object的\__proto__属性或者叫[[Prototype]]指向了他的原型，他的原型可以是一个对象或者是null

原来\__proto__属性才是指向对象原型的属性，回看本文开头引用的话

> 我们创建的每个__函数__都有一个prototype属性，这个属性是一个指针，指向一个对象，...

人家说的没错，函数才有prototype属性，那function f的\__proto__是什么呢？

    console.log(f.__proto__);


输出：
![f.__proto__][5]

貌似\__proto__属性才是指向对象原型的属性，那只有方法才有的prototype属性是干啥的呢？

下面有请我们久违的关键字__new__:

    var f = function() {
        this.aaa = 'yooooo';
        this.fxxk = function() { console.log('fxxk'); }
    }
    var f_obj = new f();
    console.log(f_obj.aaa);
    f_obj.fxxk();

输出如下:
![f_obj][6]

什么感觉，通过方法f创建了对象f_obj，class关键字呢？f难道是个类？
让我们看看f_obj的\__proto__是啥

    console.log(f_obj.__proto__);
输出如下：
![f_obj__proto__.jpg][7]

是不是很眼熟？让我们验证下:

    console.log(f_obj.__proto__ == f.prototype);
    //再来个强等于
    console.log(f_obj.__proto__ === f.prototype);

输出：
![f_equal][8]
一般我们称f为f_obj的构造函数，让我们来分解下当我们执行var f_obj = new f()的操作：

    var f_obj = new Object();
    f_obj.__proto__ = f.prototype;
    f.call(f_obj);
关键点就是f_obj.\__proto__ = f.prototype;这句
也就是说，\__proto__才是真正的对象的原型（指向原型），而function的prototype就是通过调用构造函数而创建的那个对象实例的原型对象。

顺道说一下__new__，在执行var f_obj = new f();时

> 1.一个新对象被创建。它继承自f.prototype.
  2.构造函数 f 被执行。执行的时候，相应的传参会被传入，同时上下文(this)会被指定为这个新实例。new f 等同于 new f(), 只能用在不传递任何参数的情况。
  3.如果构造函数返回了一个“对象”，那么这个对象会取代整个new出来的结果。如果构造函数没有返回对象，那么new出来的结果为步骤1创建的对象，ps：一般情况下构造函数不返回任何值，不过用户如果想覆盖这个返回值，可以自己选择返回一个普通对象来覆盖。当然，返回数组也会覆盖，因为数组也是对象。

以下是不愿透露姓名的vjeux大牛实现的new关键字：
        
        function New (f) {
        /*1*/  var n = { '__proto__': f.prototype };
               return function () {
        /*2*/    f.apply(n, arguments);
        /*3*/    return n;
            };
        }




需要注意的是，f_obj与f其实并没有任何关系，例如：

    f.newparam = 'hehe';
    console.log(f_obj.newparam);
输出：
![hehe][9]
而是与f.prototype有关：

    f.prototype.newparam = 'haha';
    console.log(f_obj.newparam);
输出：
![haha][10]

结合不愿透露姓名的vjeux大牛实现的new就可以理解了

----
总结一下，\__proto__指向对象的原型，而prototype为对象的原型对象
来张图：
![link][11]

然后回答一下刚才提出的问题：
Q:prototype到底是不是原型？
A:应该说不是，\__proto__才是指向对象的原型，而prototype为对象的原型对象，创建对象是对象的\__proto__会指向构建函数的prototype

Q:万物都有原型？
A:不是，并不是万物都有原型，只有Object才有原型，非Object就没有原型，比如null就没有原型

    console.log(null.__proto__);
输出：
![null][12]


## 用原型实现继承

扯了半天，原型到底是用来干嘛的呢？
其实就是用来实现继承的

## 1.原型链
先上定义：

> ECMAScript does not contain proper classes such as those in C++, Smalltalk, or Java, but rather, supports constructors which create objects by executing code that allocates storage for the objects and initialises all or part of them by assigning initial values to their properties. All constructors are objects, but not all objects are constructors. Each constructor has a Prototype property that is used to implement prototype-based inheritance and shared properties. Objects are created by using constructors in new expressions; for example, new String("A String") creates a new String object. Invoking a constructor without using new has consequences that depend on the constructor. For example, String("A String") produces a primitive string, not an object.
ECMAScript supports prototype-based inheritance. Every constructor has an associated prototype, and every object created by that constructor has an implicit reference to the prototype (called the object's prototype) associated with its constructor. Furthermore, a prototype may have a non-null implicit reference to its prototype, and so on; this is called the prototype chain. When a reference is made to a property in an object, that reference is to the property of that name in the first object in the prototype chain that contains a property of that name. In other words, first the object mentioned directly is examined for such a property; if that object contains the named property, that is the property to which the reference refers; if that object does not contain the named property, the prototype for that object is examined next; and so on.


太长不看，直接上他人总结的重点：

> JavaScrip可以采用构造器(constructor)生成一个新的对象,每个构造器都拥有一个prototype属性,而每个通过此构造器生成的对象都有一个指向该构造器原型(prototype)的内部私有的链接(proto),而这个prototype因为是个对象,它也拥有自己的原型,这么一级一级指导原型为null,这就构成了原型链.

眼熟么，不就是对象的\__proto__属性链接到构造函数的prototype么...
然后因为prototype也是个对象，那自然也会有\__proto__属性，有该属性那也一定有构建他的prototype...该proto链的最终指向Object.prototype，~~因为Object.prototype个空对象~~，因为Object.prototype的\__proto__为null，所以链条终止

来张图：
![proto chain][13]


----------


那么现在可以开始搞继承了，个人总结了下，继承大概有三种：
1.实例对象继承自构造器

    var f = function() {
        this.aaa = 'yooooo';
        this.fxxk = function() { console.log('fxxk'); }
    }
    f_obj = new f();
    f_obj.fxxk();

2.对象继承自对象

    var obj_father = {
        name:'father',
        sex:'male'
    }
    var obj_son = {}
    obj_son.__proto__ = obj_father;
    console.log(obj_son.name);


3.方法继承自方法

    var f_father = function() {
        this.name = 'father';
        this.sex = 'male';
        this.f_showoff = function() {
            console.log('I am Ur Father!!');
        }
    }
    obj_father = new f_father();
    console.log(obj_father.name);
    console.log(obj_father.sex);
    obj_father.f_showoff();
    
    var f_son = function() {
        this.name = 'son';
        this.s_showoff = function() {
        console.log('Nooooooo!');
        }
    }
    f_son.prototype = new f_father();
    f_son.prototype.constructor = f_son;
    obj_son = new f_son();
    console.log(obj_son.name);
    console.log(obj_son.sex);
    obj_son.f_showoff();
    obj_son.s_showoff();

    
__注意:以上均为个人总结，属于拍脑袋乱写，绝对不保证正确！！__

来详细说一下上面的三种方法：
第一种其实不能算是继承，应该算是__类似__于实例化类...
但是由于可以这么做：

    var f = function() {
        this.aaa = 'yooooo';
        this.fxxk = function() { console.log('fxxk'); }
    }
    f_obj = new f();
    f_obj2 = new f();
    f_obj.__proto__.newparam = 'yaoyaoyao';
    console.log(f_obj.newparam);
    console.log(f_obj2.newparam);
    
输出：
![yaoyaoyao][14]

不过这种方法是极度不推荐的，因为一旦改动了某个实例的\__proto__属性，会导致所有实例的属性都被修改


第二种很少被提到，因为有个关键原因，就是伟大的IE9之前是不支持\__proto__属性，当然，ECMA也是不推荐直接修改这个属性，不过ECMAScript V5开始支持更叼的Object.create(),你可以这么写：

    var obj_daddy = {
        name:'daddy',
        sex:'male',
        showoff: function() {
            console.log('I am Your Father!!');
        }
    }
    
    var obj_kid = Object.create(obj_daddy);
    console.log(obj_kid.name);
    obj_kid.showoff();



第三种就比较常用了，而且也很类型于类继承的体验，不过需要注意一点：

    f_son.prototype.constructor = f_son;

 > 每个函数都有默认的prototype，这个prototype.constructor默认指向的就是这个函数本身。在未给f_son指定f_father作为原型之前，f_son.prototype.constructor或者f_son._proto_.constructor指向的就是f_son函数。但是当这样做之后,f_son函数的prototype被覆盖成了一个f_father对象。 我们知道，constructor始终指向的是创建本身的构造函数，因此f_son.prototype.constructor自然就指向了创建f_son.prototype这个f_father对象的函数，也就是f_father函数。这样一来，f_son对象的constructor指向就不对了
 
 当然，这也没有任何影响，除非你这么做：

    var f_son2 = f_son.constructor();

    
##注意事项:
1.\__proto__属性应尽量不使用，因为ECMA不推荐这么做，而且会有兼容问题，如果你非要用，请使用那个什么来不及写了

    
    
  [1]: http://dialer.cdn.cootekservice.com/web/images/share_doc/f_prototype.jpg
  [2]: http://dialer.cdn.cootekservice.com/web/images/share_doc/obj_prototype.jpg
  [3]: http://dialer.cdn.cootekservice.com/web/images/share_doc/obj_prototype.jpg
  [4]: http://dialer.cdn.cootekservice.com/web/images/share_doc/undefind.jpg
  [5]: http://dialer.cdn.cootekservice.com/web/images/share_doc/f__proto__.jpg
  [6]: http://dialer.cdn.cootekservice.com/web/images/share_doc/f_obj.jpg
  [7]: http://dialer.cdn.cootekservice.com/web/images/share_doc/f_obj__proto__.jpg
  [8]: http://dialer.cdn.cootekservice.com/web/images/share_doc/f_equal.jpg
  [9]: http://dialer.cdn.cootekservice.com/web/images/share_doc/hehe.jpg
  [10]: http://dialer.cdn.cootekservice.com/web/images/share_doc/haha.jpg
  [11]: http://upload-images.jianshu.io/upload_images/1577855-c8f351fdf047bb9c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240
  [12]: http://dialer.cdn.cootekservice.com/web/images/share_doc/null.jpg
  [13]: http://img.bbs.csdn.net/upload/201307/17/1374070786_409876.jpg
  [14]: http://dialer.cdn.cootekservice.com/web/images/share_doc/yaoyaoyao.jpg
  [15]: https://www.zybuluo.com/mdeditor?url=https://www.zybuluo.com/static/editor/md-help.markdown
  [16]: https://www.zybuluo.com/mdeditor?url=https://www.zybuluo.com/static/editor/md-help.markdown#cmd-markdown-高阶语法手册
  [17]: http://weibo.com/ghosert
  [18]: http://meta.math.stackexchange.com/questions/5020/mathjax-basic-tutorial-and-quick-reference
 