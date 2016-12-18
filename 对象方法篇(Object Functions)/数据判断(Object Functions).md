## underscore源码解析(3)--数据判断(Object Functions)


前面两篇文章我们了解了 underscore 的大致框架，以及它为了后面更好地定义方法做的一些基础设置。从这篇文献开始，我将深入学习它的 API。 首先学习的是它的对象方法部分，这篇文章将主要解读这部分关于数据判断的方法。

### 1\. 数据类型判断

#### 1.1 常规判断

先来看一下 underscore 的源码：

```javascript
_.each(['Arguments', 'Function', 'String', 'Number',
    'Date', 'RegExp', 'Error'], function(name) {
  _['is' + name] = function(obj) {
    return toString.call(obj) === '[object ' + name + ']';
  };
});
```

这里调用了 underscore 的 each 方法（这个方法在 [第二篇文章](https://changxiupeng.github.io/2016/12/12/underscore%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90%EF%BC%882%EF%BC%89/) 中有过说明），这个方法为第一个数组参数中的每个字符串元素添加一个 "is" 前缀后，将这个字符串作为 underscore 库的方法保存，每个方法的函数体内都调用了 toString （`var toString = Object.prototype.toString`）方法。

为什么要调用这个方法呢，我们首先来看一下 MDN 关于 Object.prototype.toString() 描述：

> Every object has a toString() method that is automatically called when the object is to be represented as a text value or when an object is referred to in a manner in which a string is expected. By default, the toString() method is inherited by every object descended from Object. If this method is not overridden in a custom object, toString() returns "[object _type_]", where type is the object type.

翻译成中文大致是： js 中每个对象都有一个 toString 方法，当对象需要被转成字符串时，就会自动调用它。 默认情况下，每个对象的 toString 方法都是从 Object 上继承的，如果当前对象没有重新定义 toString 方法，既没有覆盖从 Object 上继承的 toString 方法时，那么调用该对象的方法时，会返回 "[object _type_]", 这里的字符串 _type_ 表示当前对象的类型

我们可以在每个对象上调用默认 toString 方法，来判断对象的类型：

```javascript
var toString = Object.prototype.toString;
toString.call(new Date); // [object Date]
toString.call(new String); // [object String]
```

总结： 对于 Arguments, Function, String, Number, Date, RegExp, Error 这些数据类型，underscore 都是通过 `Obect.prototype.toString` 方法来判断其类型的

#### 1.2 补充 \_.isArguments(obj)

上面虽然已经定义了该方法，但是由于 IE9 之前的版本在此处有 bug，对于 Arguments 类型，通过 toString 方法返回的是 "[object object]"， 而不是 "[object Arguments]"，所以还要进行修正

```javascript
if (!_.isArguments(arguments)) {
  _.isArguments = function (obj) {
    // 只有arguments 才会有 callee 属性
    return _.has(obj, "callee");
  }
}
```

#### 1.3 补充 \_.isFunction(obj)

修正了早期 V8 存在的一些 bug

```javascript
if (typeof /./ != 'function' && typeof Int8Array != 'object') {
  _.isFunction = function (obj) {
    return typeof obj === "function" || false;
  }
}
```

PS: 这也是 _编写可维护的JavaScript_ 一书中推荐的判断函数类型的方式（没有 if 条件句判断），直接用 typeof 操作符判断

#### 1.4 \_.isArray(obj)
判断 obj 是否是数组类型
```JavaScript
// 优先使用 ES5中的 Array.isArray 方法来判断
// 不支持原生方法的可以用 1.1 中的常规方法判断 (IE 9 之前不支持)
_.isArray = nativeIsArray || function (obj) {
  return toString.call(obj) === "[object Array]";
};
```

PS: 判断一个数据是不是数组还有一种老式方法，既检查该数据是否有 sort 方法，因为只有数组才有这个方法：
```JavaScript
function isFunction(obj) {
  return typeof obj.sort === "function" || false;
}
```
#### 1.5 \_.isObject(obj)
判断 obj 是否是对象类型
```JavaScript
_.isObject = function (obj) {
  var type = typeof obj;
  return type === "function" || type === "object" && !!obj;
};
```
从代码中我们可以看出来，underscore 将函数也视为对象，但是像将 null 排除在外，即使 `typeof null = "object"`

#### 1.6 \_.isBoolean(obj)
判断 obj 是否是布尔类型
```JavaScript
_.isBoolean = function (obj) {
  return obj === true || obj === false ||
      toString(obj) === "[object Boolean]";
};
```
#### 1.7 \_.isNull(obj)
判断 obj 是否是 null
```JavaScript
_.isNull = function (obj) {

  // 只需要判断 obj 是否与 null 全等即可
  return obj === null;
};
```
#### 1.8 \_.isUndefined(obj)
判断 obj 是否是 undefined
```JavaScript
_.isUndefined = function (obj) {
  return obj === void 0;
}
```
这里将 obj 与 void 0 全等对比是一种 hack 的方法。

- 为什么要这样做呢，为什么不能像判断 null 那样直接全等比较呢。这是因为 undefined 在 ES3 中可以被用作全局和函数变量，在 ES5 中可以被用作函数变量，也就是说它在一些情况下是可以被赋值操作，被改写的，所以直接全等比较是不靠谱的。
- 既然 undefined 并不能真正靠谱地表示 “未定义”，那么我们就需要找替代品，而 void 运算符可以完美胜任。该运算符后面跟一个表达式，而无论这个表达式是什么，运算结果都是 “undefined”。那干脆在后面加个 0，运算开销最小。

PS: 当函数没有返回结果的时候，默认返回的就是 “undefined”，所以也可以利用一个匿名空函数。
```JavaScript
var Ctor = function(){};

_.isUndefined = function (obj) {
  return obj === Ctor();
}
```

jQuery 采用了另外一种方式，为整体的匿名函数定义两个形参，但是只传入一个实参，那么第二个形参就会被赋值 “undefined”
```JavaScript
(function(window, undefined)) {
  //...
})(window)
```
#### 1.9 \_.isNaN(obj)
判断 obj 是否是 NaN
```JavaScript
_.isNaN = function (obj) {
  // obj 是数字并且不等于自身（只有 NaN 这个值符合）
  return _.isNumber(obj) && obj !== +obj;
}
```
underscore 只希望判断 obj 是不是等于 NaN 这个特殊的值（NaN 本质上是一个数字类型的值，像 1，2，3 一样它也代表了一个数值），而不是希望判断 obj 到底是不是个数字。原生的 isNaN(obj) 方法，只要 obj 不是数字类型的值就会返回 true：
```JavaScript
isNaN("ddd"); // true
isNaN(undefined); // true

_.isNaN("ddd"); // false
_.isNaN(undefined); // false
_.isNaN(NaN); // true
```
### 2. 其他判断
#### 2.1 \_.isElement(obj)
判断 obj 是否是一个 DOM 元素节点
```JavaScript
_.isElement = function (obj) {
  return !!(obj && obj.nodeType === 1;)
}
```
元素节点的 nodeType 值为 1
之所以会加上两个非 “!” 运算符，是因为如果 obj 是 undefined，null, NaN, 0, 这几个可以转成 false 的值是，返回的是这几个值本身，而不是 false，所以要将它们强制转成 false
#### 2.2 \_.isFinite(obj)
判断 obj 是不是有穷数字
```JavaScript
_.isFinite = function (obj) {
  return isFinite(obj) && !isNaN(parseFloat(obj));
};
```
这里依赖了原生的 isFinite 方法，原生的方法在进行判断的时候会先将 obj 转成数字，然后再判断该数字是否有限，但是，true 和 null 会被转成1 和 0，就导致将 true 和 null 判断成有限，而它们根本就不是数字，`!isNaN(parseFloat(obj))` 可以将排除这种情况

#### 2.3 \_.isEmpty(obj)
判断 obj 是不是为空
```JavaScript
_.isEmpty = function (obj) {
  // 如果 obj 值是 null 则认为它是空的
  if (obj == null) {
    return true;
  }

  // 如果 obj 是数字，字符串或者参数类型（类数组），并且长度为 0 ，则为空
  if (isArrayLike(obj) && (_.isArray(obj) || _.isString(obj) ||
      _.isArguments(obj))) {
    return obj.length === 0;
  }

  // 如果 obj 是对象，那么取得它所有的自身属性，并判断长度，长度为 0，则为空
  return _.keys(obj).length === 0;
}
```
