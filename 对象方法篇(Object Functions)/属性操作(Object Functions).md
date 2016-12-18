## underscore源码解析(5)--属性操作(Object Functions)

### 1 \_.has(obj, key)
该方法用来判断 obj 的 **自身属性** 中是否有 key 这个属性
```JavaScript
_.has = function (obj, key) {

  // 如果 obj 不为 null就调用原生方法 Object.prototype.hasOwnProperty
  // 在 underscore 中该方法的引用被保存到 hasOwnProperty 变量中
  return obj !== null && hasOwnProperty.call(obj, key);
}
```
示例：
```JavaScript
var foo = {a: 1}

_.has(foo, a) // true

_.has(foo, toString) // false
```

### 2 \_.keys(obj)
该方法返回一个数组，数组包括了 obj 的所有 **自身属性**

```JavaScript
_.keys = function (obj) {

  // 如果 obj 不是对象类型则返回一个空数组
  // underscore 的 isObject 方法，当 obj 是函数或对象类型时（不含 null）
  // 返回 true
  if (!_.isObject(obj)) return [];

  // 如果浏览器支持原始的 Object.keys() 方法，则返回调用的结果
  // underscore 中变量 nativeKeys 保存了 Object.keys()的引用
  if (nativeKeys) return nativeKeys(obj);

  // 如果浏览器不支持原生的方法，那么就采用 for in 循环的方法
  var keys = [];
  for (var key in obj) {
    if (hasOwnProperty(obj)) {
      keys.push(key);
    }
  }

  // IE < 9的浏览器在此有 bug，如果对象在自身属性中重写了
  // 像 toString valueOf 这些不可遍历的原型属性，它们在 for in
  // 循环中是不会被遍历到的，下面的代码将其修复
  if (hasEnumBug) {
    collectNonEnumProps(obj, keys);
  }
  return keys;
}

// 依赖一 hasEnumBug
// 手动将 toString 重写后，判断这个属性是否可枚举
var hasEnumBug = !{toString: null}.
    propertyIsEnumerable("toString")

// 依赖二 collectNonEnumProps
// 该 bug 不可遍历的属性如下
var nonEnumerableProps = ['value', 'isPrototypeOf', 'toString',
    'valueOf', 'propertyIsEnumerable', 'hasOwnProperty',
    'toLocaleString'];

var collectNonEnumProps = function (obj, keys) {

  // 获取不可遍历属性的长度
  var nonEnumIdx = nonEnumerableProps.length;

  // 获取 obj 的原型对象，由于不能直接从 obj 上获取其原型
  // 所以需要通过它的构造器获取
  var constructo = obj.constructor;

  // 如果 obj 的构造器是一个函数并且有自己的原型属性，那么就取这个原型
  // 如果上面的条件不符合那么直接取 Object.prototype 作为其原型对象
  // 变量 ObjProto 中保存了 Object.prototype 的引用
  var proto = _.isFunction(constructo) && constructo.prototype ||
      ObjProto;

  // constructor 是一个特例
  // 如果 obj 中有，但是属性列表 keys 中没有这个属性，那么就将它加到 keys 中
  // contains 是数组部分定义的一个方法，检测数组 keys 中是否包含 prop
  var prop = "constructor";
  if (_.has(obj, prop) && !_.contains(keys, prop)) {
    keys.push(prop);
  }

  while (nonEnumIdx--) {
    prop = nonEnumerableProps[nonEnumIdx];

    // 如果当前遍历到的不可枚举属性存在于 obj 中，既被重新定义为自身属性
    // 并且属性值与在 obj 原型对象中的值不一样
    // 并且还没有被收到要返回的属性集合中，就将它加入到集合中
    if (prop in obj && obj[prop] !== proto[prop] &&
        !_.contains(keys, prop)) {
      keys.push(prop);
    }
  }
};
```
示例：
```JavaScript
var obj = {
  a: "a",
  toString: "i've been changed"
}

_.keys(obj) // ["a", "toString"]
```
### 3 \_.allKeys(obj)
该方法返回 obj 中的所有属性，包括原型中的可枚举属性

```JavaScript
_.allKeys = function(obj) {

  // obj 不是对象则返回一个空数组
  if (_.isObject(obj)) return [];

  // 直接用 for in 循环遍历所有属性
  var keys = [];
  for (var key in obj) {
    keys.push(key);
  }

  // IE < 9 在自身属性中重写的原型属性不能被枚举
  // 跟 _.keys() 一样将其修复
  if (hasEnumBug) {
    collectNonEnumProps(obj, keys);
  }

  return keys;
}
```

示例：
```JavaScript
var obj = {
  a: "a",
  toString: "i've been changed"
}
obj.prototype.bar = "bar";

_.allKeys(obj); // ["a", "toString", "bar"]
```
### 4 \_.values(obj)
该方法返回一个数组，数组中包含了 obj 的所有自身属性的属性值

```JavaScript
_.values = function (obj) {

  // 获取 obj 的属性集合
  // 根据该集合的长度初始化一个等长的数组
  var keys = _.keys(obj),
      length = keys.length,
      results = Array(length);

  // 将属性值加入到初始化的数组中
  for (var i = 0; i < length; i++) {
    results[i] = obj[keys[i]];
  }

  return results;
}
```
示例：
```JavaScript
var obj = {a: "A", b: "B", c: "C"};
_.values(obj); // ["A", "B", "C"]
```

### 5 \_.functions(obj)
该方法返回一个数组，数组中包含了 obj 所有方法的方法名

```JavaScript
_.functions = function (obj) {
  var names = [];

  // 如果属性是方法，那么将属性名加入 names 中
  for (var key in obj) {
    if (_.isFunction(obj[key])) names.push(key);
  }

  // 最后对获取的方法名进行排序
  return names.sort();
}
```

示例：

```JavaScript
var obj = {
  a: "a",
  say: function () {
    alert("i'm a function")
  },
  sing: function () {
    alert("i'm also a function")
  }
}

_.functions(obj); // ["say", "sing"]
```

### 6 \_.invert(obj)
该方法将 obj 内的属性和其对应的属性值对调（键值对调）
```JavaScript
_.invert = function (obj) {
  var results = {};
  var keys = _.keys(obj);
  for (var i = 0; i < keys.length; i++) {
    results[obj[keys[i]]] = keys[i];
  }
  return results;
}
```

示例：
```JavaScript
var obj = {a: "A", b: "B", c: "C"};
_.invert(obj); // {A: "a", B: "b", C: "c"}
```

### 7 \_.pairs(obj)
该方法将 obj 中的每对键值对都转成 `[key, value]` 数组，然后返回包含了所有这些键值对数组的列表数组

```JavaScript
_.pairs = function (obj) {
  var keys = _.keys(obj),
      length = keys.length,
      pairs = Array(length);

  for (var i = 0; i < length; i++) {
    pairs[i] = [keys[i], obj[keys[i]]];
  }

  return pairs;
}
```
