## underscore源码解析(4)--是否值相等(Object Functions)

这篇文章接着上篇文章，继续学习 underscore 中关于数据判断的 API。上篇文章主要介绍了数据类型，以及数据是否为空，是否有限的判断。这篇文章只介绍一个方法，判断两个数据值是否相等。这个方法应该是 underscore 中代码量和注释最多的一个方法了。

### 1. 源码
```JavaScript
// 判断 a, b 是否值相等
_.isEqual = function (a, b) {

  // 调用内部函数 eq，返回一个布尔值
  return eq(a, b);
};
```

### 2. 需要考虑的情况
#### 2.1 `0 === -0`, 但是这里并不认为它们是相等的
```JavaScript
if (a === b) {
  return a !== 0 || 1 / a === 1 / b;
}
```
这段代码先判断 a，b 是否全等
如果全等再将 a 跟 0 相比较，如果它等于 0，那么就会返回 false（-0 === 0，所以用 a 还是 b 比较都一样），如果不等于 0，那么就可以排除这种情况
因为 `1 / 0 === Infinity` , `1 / -0 === -Infinity`，而 `Infinity !== -Infinity`，所以也可以用 `1 / a === 1 / b` 来排除这种情况

#### 2.2 `null == undefined` 但是这里并不认为它们相等
```JavaScript
if (a == null || b == null) {
  return a === b;
}
```
如果 a，b 中至少有一个是 null，那么就用全等判断它们是否相等

#### 2.3 如果 a，b 是下划线对象的实例
```JavaScript
if (a instanceof _) {
  a = a._wrapped;
}
if (b instanceof _) {
  b = b._wrapped;
}
```
这种情况下，只需要比较 a,b 两个对象中的 `_wrapped` 属性值
因为调用下划线构造函数时，值是存在实例的 `_wrapped`属性中的，具体可以看[第一篇解读文章](https://changxiupeng.github.io/2016/12/11/underscore%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90%EF%BC%881%EF%BC%89/)
也就是说可以不用比较 ab 从下划线对象中继承的属性和方法，直接比较实例化时添加的属性值即可，所以，就需要对 ab 重新赋值为它们各自的 `_wrapped` 属性值

#### 2.4 如果 a，b 是 `RegExg，String，Boolean，Number，Date` 类型的值
```JavaScript
var className = toString.call(a);

//如果 a，b 的数据类型都不相同，那么返回 false
if (className !== toString.call(b)) {
  return false;
}

// 数据类型相同，再来根据数据类型的不同分别进行判断
switch (className) {

  // 因为简单的运算就可以将正则转成字符串
  // "" + /a/i === "/a/i"
  // 所以正则可以跟字符串一起判断
  case "[object RegExp]":
  case "[object String]":
    return "" + a === "" + b;

  case "[object Number]":

    // 数字类型首先要解决 NaN 这个特殊情况
    // NaN 的特点很明显，它是数字类型并且不等于自身
    // 而且，NaN 是唯一一个不等于自身的数字，如果 ab 都不等于自身
    // 则认为它们相等
    // 在 ab 前面加 “+”表示将其转化成数字
    if (+a !== +a) {
      return +b !== +b;
    }

    // 判断除了 NaN 之外的数字，这里要考虑到 0 的情况
    // 如果是零就需要用 1 除后比较，如果不是就直接全等比较
    return +a === 0 ? 1 / +a === 1 / b : +a === +b;

  // 用加法运算符将 Date 和 Boolean 值转成数字进行比较
  // Date 转成数字后变成 UTC 的毫秒数
  case "[object Date]":
  case "[object Boolean]":
    return +a === +b;
}
```

#### 2.5 如果 a，b 是 Array 或 Object 类型
```JavaScript
var areArrays = className === "[object Array]";

// 如果 a 不是数组类型
// 在进入数组和对象两选一的判断之前，先排除一些特殊情况
if (!areArrays) {

  // 只要a和b有一个不是 object 类型，则认为 ab 不相等
  if (typeof a != "object" || typeof b != "object") {
    return false;
  }

  // 拥有不用构造器的对象是不相等的，但是不同 iframes 下的 Objects 和 Arrays 是相等的
  var aCtor = a.construtor, bCtor = b.construtor;
  if (aCtor !== bCtor && !(_.isFunction(aCtor) && aCtor instanceof aCtor &&
                           _.isFunction(bCtor) && bCtor instanceof bCtor)
                      && ('constructor' in a && 'constructor' in b)) {
    return false;
  }
}

// 采用栈数据结构来进行递归
// 如果 ab 内部有多层的数组或对象的嵌套，那么最外层会先进栈
// 然后层层依次进栈，最内层在栈顶
aStack = aStack || [];
bStack = bStack || [];

var length = aStack.length;

// 这里可能是为了检测对象是否有自调用
while(length--) {
  if (aStack[length] === a) {
    return bStack[length] === b;
  }
}

// 如果上面 while 中没有进入 if 中，这里就把 a 和 b 放到栈里
aStack.push(a);
aStack.push(b);

// 如果是数组
if (areArrays) {
  length = a.length;

  // 如果 ab 长度不相等，直接返回 false
  if (length !== b.length) return false;

  // 如果 ab 长度相等，那么递归比较每个元素
  while (length--) {

    // 如果有一个元素不相等，就返回false
    if (!eq(a[length], b[length], aStack, bStack)) return false;
  }
} else { // 如果是对象
  var keys = _.keys(a), key;
  length = keys.length;

  // 如果两个对象的属性数量不一样，那么返回 false
  if (_.keys(b).length !== length) return false;

  // 递归比较每个属性值是否相等
  while (length--) {
    key = keys[length];
    if (!(_.has(b, key) && eq(a[key], b[key], aStack, bStack)))
      return false;
  }
}

// 如果上面的代码都没有返回 false，那说明这层的比较是相等的，将这一层从栈顶弹出
aStack.pop();
bStack.pop();

// 将这层值匹配正确的结果返回给上一层
return true;
```
#### 2.6 综合后源码
```JavaScript
var eq = function(a, b, aStack, bStack) {
  if (a === b) return a !== 0 || 1 / a === 1 / b;
  if (a == null || b == null) return a === b;
  if (a instanceof _) a = a._wrapped;
  if (b instanceof _) b = b._wrapped;
  var className = toString.call(a);
  if (className !== toString.call(b)) return false;
  switch (className) {
    case '[object RegExp]':
    case '[object String]':
      return '' + a === '' + b;
    case '[object Number]':
      if (+a !== +a) return +b !== +b;
      return +a === 0 ? 1 / +a === 1 / b : +a === +b;
    case '[object Date]':
    case '[object Boolean]':
      return +a === +b;
  }

  var areArrays = className === '[object Array]';
  if (!areArrays) {
    if (typeof a != 'object' || typeof b != 'object') return false;

    var aCtor = a.constructor, bCtor = b.constructor;
    if (aCtor !== bCtor && !(_.isFunction(aCtor) && aCtor instanceof aCtor &&
                             _.isFunction(bCtor) && bCtor instanceof bCtor)
                        && ('constructor' in a && 'constructor' in b)) {
      return false;
    }
  }

  aStack = aStack || [];
  bStack = bStack || [];
  var length = aStack.length;
  while (length--) {
    if (aStack[length] === a) return bStack[length] === b;
  }

  aStack.push(a);
  bStack.push(b);

  if (areArrays) {
    length = a.length;
    if (length !== b.length) return false;
    while (length--) {
      if (!eq(a[length], b[length], aStack, bStack))
        return false;
    }
  } else {
    var keys = _.keys(a), key;
    length = keys.length;
    if (_.keys(b).length !== length) return false;
    while (length--) {
      key = keys[length];
      if (!(_.has(b, key) && eq(a[key], b[key], aStack, bStack)))
        return false;
    }
  }


  aStack.pop();
  bStack.pop();
  return true;
};

```
