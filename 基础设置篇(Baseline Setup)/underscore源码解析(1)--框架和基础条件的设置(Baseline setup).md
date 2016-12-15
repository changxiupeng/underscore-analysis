---
title: underscore源码解析(1)--框架和基础条件的设置(Baseline setup)
---
## 结构概括
underscore 的设计逻辑如下：
1. 所有的业务逻辑都包裹在一个匿名的自执行的函数当中
2. 定义了一个下划线变量`_`来标识自身，`_`是一个 **函数对象**，所有的 API 都挂载到该对象上

underscore 由七个部分组成：
1. Baseline setup （基础设置）
2. Collection Functions （集合）
3. Array Functions （数组）
4. Function Functions （函数）
5. Object Functions （对象）
6. Utility Functions （工具）
7. OOP （链式调用）

## 源码
下面我们通过源码来解释一下上面的概括：

```JavaScript
// Underscore.js 1.8.3

(function() {

  // Baseline setup
  // --------------

  // 创建一个 root 对象，在浏览器端指的是 “window”， 在服务器端指的是 “exports”
  var root = this;

  /*
   * 默认情况下，underscore 对象 “_” 会覆盖全局对象 （window、exports）上同名的 “_” 属性
   * 但是，underscore 将全局对象中的 “_” 属性值先保存到一个 previousUnderscore 的变量中
   * 但用户已经在全局中绑定了 “_” 属性，用户可以调用 underscore 提供的 noConflict 函数来
   * 重命名 underscore 对象，避免与之前的全局 “_” 冲突
   */

  // 保存全局变量 "_" 的值，因为是在最上面，所以在全局 “_” 被覆盖之前它的值就被保存了
  var previousUnderscore = root._;

  // 将被保存的之前的全局 “_” 中的值在赋值给它原来的拥有者（window，其他库等），这段代码是
  //我从 Utility 部分提上来的
  _.noConflict = function() {
    root._ = previousUnderscore;

    // 返回 underscore 对象
    return this;
  }

  // 用户通过调用 noConflict 函数给 underscore 重起名，下面的代码不是 underscore 中的，
  // 是应该用户写的，那么之后就可以用 under 来调用库中的 API 了
  // var under = _.noConflict();

  // 将原型赋值给一个变量，便于之后的压缩，（not gzipped）
  var ArrayProto = Array.prototype, ObjProto = Object.prototype, FuncProto = Function.prototype;

  // 将原型中常用的方法赋值给变量，便于之后更方便的引用
  var
    push             = ArrayProto.push,
    slice            = ArrayProto.slice,
    toString         = ObjProto.toString,
    hasOwnProperty   = ObjProto.hasOwnProperty;

  // 将库中常用的一些 ES5 原生方法保存到变量中
  var
    nativeIsArray      = Array.isArray,
    nativeKeys         = Object.keys,
    nativeBind         = FuncProto.bind,
    nativeCreate       = Object.create;

  // 创建一个 Ctor 局部变量，用来重复创建匿名函数
  var Ctor = function(){};

  // 定义 “_” 构造函数，之所以定义成一个函数对象而不是普通对象是因为可以用来生产 “_” 实例对象
  // 比如 var a = _(obj);
  var _ = function(obj) {
    if (obj instanceof _) return obj;
    if (!(this instanceof _)) return new _(obj);
    this._wrapped = obj;
  };

  // 针对不同的宿主环境，将 underscore 的命名变量存放在不同的宿主全局对象中
  if (typeof exports !== 'undefined') { // Node.js
    if (typeof module !== 'undefined' && module.exports) {
      exports = module.exports = _;
    }
    exports._ = _;
  } else { // 浏览器
    root._ = _;
  }

  // 将当前的版本号存放在 VERSION 属性中
  _.VERSION = '1.8.3';

  // Collection Functions
  // --------------------


  // Array Functions
  // ---------------


  // Function (ahem) Functions
  // ------------------


  // Object Functions
  // ----------------


  // Utility Functions
  // -----------------


  // OOP
  // ---------------

}.call(this));
```

### 局部变量的妙用
```JavaScript
var ArrayProto = Array.prototype, ObjProto = Object.prototype, FuncProto = Function.prototype;

var
  push             = ArrayProto.push,
  slice            = ArrayProto.slice,
  toString         = ObjProto.toString,
  hasOwnProperty   = ObjProto.hasOwnProperty;

var
  nativeIsArray      = Array.isArray,
  nativeKeys         = Object.keys,
  nativeBind         = FuncProto.bind,
  nativeCreate       = Object.create;
```
underscore 本身也依赖了一些 js 的原生方法，underscore 会通过一些局部变量来保存一些它经常使用的方法或属性，这样做有如下两点好处：
- 在后续使用到这些地方或属性时，避免了冗长的代码书写
- 减少了对象成员的访问深度，这样做可以带来一定的性能提升，比如 `Array.prototype.push` 可以直接保存到 `push`变量中

### 下划线 `_` 构造函数
```JavaScript
var _ = function(obj) {
  if (obj instanceof _) return obj;
  if (!(this instanceof _)) {
    return new _(obj);
  }
  this._wrapped = obj;
}
```
第一句比较简单，如果传入的 obj 是 `_` 的实例，则直接返回将其返回
主要看第二句，它主要针对的是不加 `new` 关键字就可以创建实例的情况，比如我们生成数组：
```JavaScript
var a = new Array();
var b = Array();
```
当我们不加 new 关键字创建下划线实例时，应为它不像 Array 那样是内建的构造函数，所以这里它仅仅相当于一个普通的函数调用，它内部的 this 指向的是全局对象，而不是下划线对象，所以我们有必要将通过 new 关键字创建的下划线实例返回。将传入的 obj 保存到实例对象的 `_wrapped` 属性中。
```JavaScript
var a = _();
```
总结：
这个构造函数的真正意图是创建一个下划线构造函数的实例，创建的过程中为这个实例添加一个`_wrapped`实例属性，将传入的参数作为这个实例属性的值。
在进行这些操作之前，还需要进行一些检测
- 如果传入的参数是下划线构造函数的实例，那么直接将这个参数作为结构返回
- 如果是没有加 new 关键字的情况下调用的该构造函数，那么就返回加了 new 关键字的调用结果，这里相当于是递归调用
