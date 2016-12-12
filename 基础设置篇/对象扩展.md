## 1. 初始化对象--Object(obj)
underscore 库的 API 中，很多函数内部都可以见到下面的一段代码：
```JavaScript
var obj = Object(obj);
```
这段代码的意思是：
- 如果 obj 是一个对象，那么 Object(obj) 直接将 obj 原样返回
- 如果 obj 是 undefined 或者 null，那么Object(obj) 返回一个 {}
- 如果 obj 是一个原始值，那么 Object(obj) 返回一个被包裹的原始值：

```JavaScript
var obj = Object(2); // 相当于 new Number(2); 返回的是一个对象
// => Number {[[PrimitiveValue]]: 2}  
console.log(obj.valueOf()); // 2

var obj = Object(NaN); // 相当于 new Number(NaN); 返回的是一个对象
// => Number {[[PrimitiveValue]]: NaN}  

var obj = Object(""); // 相当于 new String(""); 返回的是一个对象
// => String {length: 0, [[PrimitiveValue]]: ""}  
```

类似的对象初始化的方法还有：
```JavaScript
var obj = obj || {};
```
不同的是：
- 当 obj 为 undefined, null, 0, NaN, false, ""时，都会返回后面的 {}，而 Object(obj), 只有当 obj 等于 undefined 和 null 时才会返回 {}
- 当 obj 为其他原始数据类型时，直接将其返回，而不会像 Object(obj) 那样返回一个对象类型的被包裹的原始值

## 2. 操作属性
### 2.1 `_.has(obj, key)`
这个方法判断 obj 的自身属性（不包括原型链）中是否有 key 属性：

```JavaScript
_.has = function(obj, key) {
  return obj != null && hasOwnProperty.call(obj, key);
}

var obj = {a:1};
_.has(obj, a); // => true
_.has(obj, toString); // => false
```

### 2.2 `_.keys(obj)`
这个方法返回 obj 的所有自有属性

```JavaScript
_.keys = function(obj) {
  if(!_.isObject(obj)) return [];

  // 如果浏览器支持原生的 Object.keys 方法，直接调用 Object.keys
  if (nativeKeys) {
    return nativeKeys(obj);
  }

  // 如果不支持原生的 Object.keys() 方法，用 for···in 遍历对象属性
  var keys = [];
  for (var key in obj) {

    // 如果当前属性是 obj 的自身属性，则收入 keys 中
    if (_.has(obj, key)) {
      keys.push(key);
    }
  }

  // 如果自身属性有像 toString，valueOf 等这些覆盖属性，
  // 既在对象中重新定义了这些属性
  // IE9 之前的浏览器有 bug 不会遍历这些属性，需要校正
  if (hasEnumBug) {
    collectNonEnumProps(obj, keys);
  }

  return keys;
}
```

有两个依赖函数如下：
```JavaScript
// 检测重载的 toString 属性是否可遍历，
// 这里用 null 重写了 toString 属性
var hasEnumBug = !{toString: null}.
    propertyIsEnumerable('toString');

// 该 bug 不能遍历的重载属性如下
var nonEnumerableProps = ['value', 'isPrototypeOf', 'toString',
    'valueOf', 'propertyIsEnumerable', 'hasOwnProperty',
    'toLocaleString'];
var collectNonEnumProps = function(obj, keys) {
  // 获取不可遍历属性的长度
  var nonEnumIdx = nonEnumerableProps.length;

  // 取得对象的构造函数
  var constructo = obj.constructor;

  // 如果 constructo 是函数，而且有 prototype 属性（原型对象），
  // 那么取得就将 prototype 作为 obj 的原型
  // 否则，obj 的原型为 Object.prototype
  var proto = _.isFunction(constructo) && constructo.prototype || ObjProto;

  // 如果该对象有 constructor 属性，但是当前获得属性集合不存在这个属性
  // 那么就将它添加到当前属性集合中
  var prop = "constructor";  
  if (_.has(obj, prop) && _.contains(keys, prop )) {
    keys.push(prop);
  }

  // 将不可枚举的属性也添加到属性集合中
  while (nonEnumIdx--) {

    // prop 为当前遍历的不可枚举属性
    prop = nonEnumerableProps[nonEnumIdx];

    // 如果该属性是对象自身的属性，并且属性值与原型中属性值不一致
    // 并且当前的属性集合中没有这个属性，那么就将它加入到集合中
    if (prop in obj && obj[prop] !== proto[prop] &&
      !_.contains(keys, prop)) {
      keys.push(prop);      
    }
  }
};
```

### 2.3 `_.allKeys(obj)`
该方法获取 obj 的所有属性，包括原型中定义的属性
```JavaScript
_.allKeys = function(obj) {

  // 如果传入的参数不是对象，那么返回一个空数组
  if (!_.isObject(obj)) {
    return [];
  }

  // 将obj 的所有可遍历属性加入到一个数组中
  var keys = [];
  for (var key in obj) {
    keys.push(key);
  }

  // IE < 9，重写的属性不可遍历，将其修正
  if (hasEnumBug) {
    collectNonEnumProps(obj, keys);
  }

  return keys;
}
```

### 2.4 `_.value(obj)`
该方法可以获得 obj 的值的集合
```JavaScript
_.values = function(obj) {
  // 获取 obj 所有的自身属性
  // 并根据自身属性长度创建一个定长数组，提前分配内存空间
  var keys = _.keys(obj),
      length = keys.length,
      values = Array(length);

  for (var i = 0; i < length; i++) {
    values[i] = obj[keys[i]];
  }

  return values;
}
```

### 2.5 `_.invert(obj)`
该方法可以将对象的属性和属性值调换，既原来的属性变为属性值，原来的属性值变为属性
```JavaScript
_.invert = function(obj) {
  var result = {};
  var keys = _.keys(obj);
  for (var i = 0, length = keys.length; i < length; i++) {
    result[obj[keys[i]]] = keys[i];
  }

  return result;
};
```
### 2.6 `_.functions(obj)`
该方法可以获得 obj 所具有的方法列表
```JavaScript
_.functions = _.methods = function(obj) {
  var names = [];
  for (var key in obj) {
    if (_.isFunction(obj[key])) {
      names.push(key);
    }
  }
  return names.sort();
}
```
