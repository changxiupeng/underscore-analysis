## underscore源码解析(6)--查找和筛选(Object Functions)

### 1 \_.findKey(obj, predicate, context)
遍历 obj 的属性，返回第一个满足 predicate 条件的属性的属性名

**示例：**
```JavaScript
var obj = {name: "comma", age: 26};
_.findKey(obj, function(value) {
  return value === 26;
}); // age
```

**源码解读：**
```JavaScript
/**
 * @param obj 待迭代的集合
 * @param predicate 每个迭代的属性需要进行的检测，可能是一个函数，字符串或者对象等
 * @param context 如果 predicate 是函数，那么 context 就是它的上下文
 */
_.findKey = function (obj, predicate, context) {

  // cb 根据 predicate 类型的不同返回不同的函数
  predicate = cb(predicate, context);
  var keys = _.keys(obj), key;
  for (var i = 0; i < keys.length; i++) {
    key = keys[i];
    if (predicate(obj[key], key, obj)) return key;
  }
};
```
cb 是 underscore 的一个内部函数，在之前的文章[有关迭代的两个内部函数](https://changxiupeng.github.io/2016/12/12/underscore%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90%EF%BC%882%EF%BC%89/)中解读过

### 2 \_.pick(object, oiteratee, context)
该方法会对传入的对象的键值对进行筛选，然后返回一个该对象的副本，这个副本中只包含 oiteratee 指定的键值对（白名单），而 oiteratee 可以是并列的几个字符串，也可以是包含字符串元素的一个数组，还可以是一个函数，看起来有点蒙，我们先来看例子：

**示例：**
```JavaScript
var obj = {
  name: "comma",
  age: 26,
  hobby: "Programming"
}
_.pick(obj, "name", "age");
// => {name: "comma", age: 26}

_.pick(obj, ["name", "age"]);
// => {name: "comma", age: 26}

_.pick(obj, function(value, key, obj) {
  return value === "comma" || 26;
});
// => {name: "comma", age: 26}
```

**源码解读：**
```JavaScript
_.pick(object, oiteratee, context) {
  var result = {}, obj = object, iteratee, keys;

  // 如果没有传入对象，则返回空对象
  if (obj == null) return result;

  // 如果 oteratee 是函数
  // keys 变量保存的是 obj 中的所有属性
  // iteratee 保存的是将 oiteratee 绑定了 context 上下文后的函数
  if (_.isFunction(oiteratee)) {
    keys = _.allKeys(obj);
    iteratee = optimizeCb(oiteratee, context);
  } else {

    // 如果 oiteratee 不是函数
    // 将该方法传入的参数拍平到一个数组中，去掉第一个参数 object
    // keys 变量中保存的是拍平的数组
    // iteratee 变量保存的是一个新定义的函数，判断拍平的数组元素是否在 obj中
    keys = flatten(arguments, false, false, 1);
    iteratee = function(value, key, obj) {
      return key in obj;
    }
    obj = Object(obj);
  }

  // 作者非常巧妙地将两种情况合二为一，一起进行判断删选
  // 两种情况的 keys 和 iteratee 有不同的含义，但是却完美地融入到一个循环中
  for (var i = 0, length = keys.length; i < length; i++) {
    var key = keys[i];
    var value = obj[key];
    if (iteratee(value, key, obj)) return result[key] = value;
  }
  return result;
}
```
flatten 是 underscore 的一个内部函数，接受四个参数，分别是要 _拍平的（类）数组_、_是否浅拍平_、_是否在严格模式下_，_从第几个参数开始_。这个函数会在数组方法那部分解读。

### 3 \_.omit(obj, oiretaree, context)
该方法返回一个 obj 对象的副本，这个对象副本中保存的是 **除去** oiteratee 指定的键值对后剩余的所有键值对，此时 oiteratee 指定的是 **黑名单**
有了 `_.pick` , `_.omit` 的实现就变得很简单，通过 `_.negate` 方法对传入的 oiretaree 白名单函数取反，构成黑名单函数，然后再执行 `_.pick`

**示例：**
```JavaScript
var obj = {
  name: "comma",
  age: 26,
  hobby: "Programming"
}
_.omit(obj, "name", "age");
// => {hobby: "Programming"}

_.omit(obj, ["name", "age"]);
// => {hobby: "Programming"}

_.omit(obj, function(value, key, obj) {
  return value === "comma" || 26;
});
// => {hobby: "Programming"}
```

**源码解读：**
```JavaScript
_.omit = function (obj, iteratee, context) {

  //如果 iteratee 是函数那么就对它取反
  if (_.isFunction(iteratee)) {
    iteratee = _.negate(iteratee);
  } else {

    // 如果 iteratee 不是函数，将其转成一个函数
    // 将参数拍平并将元素转成字符串形式，转成字符串是因为对象的属性是字符串
    var keys = _.map(flatten(arguments, false, false, 1), String);
    iteratee = function (value, key) {

      // 如果 key 不在拍平的数组中，那么就返回 true
      return !_.contains(keys, key);
    };
  }
  return _.pick(obj, iteratee, context);
}
```
