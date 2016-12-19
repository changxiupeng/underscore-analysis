## underscore源码解析(9)--最大最小元素(Collection Functions)

这篇文章介绍 underscore 库集合部分的两个方法，max 和 min，这两个方法可以分别获取到一个集合中的最大和最小元素；也可以传入一个迭代函数，该函数对集合中的每个元素进行一次计算，并返回计算结果，然后对计算结果进行比较，最后返回计算结果最大或最小时对应的那个集合元素。

### 1 \_.max(obj, iretatee, context)

**示例：**
```JavaScript
_.max([6, 2, 3, 10]);
// => 10

_.max({num1: 2, num2: 5, num3: 50});
// => 50

_.max([6, 2, 3, 10], function(elem) { return elem % 2; });
// => 3

_.max({num1: 2, num2: 5, num3: 50}, function(num) { return num % 2; });
// => 5

var stooges = [{name: 'moe', age: 40}, {name: 'larry', age: 50},
    {name: 'curly', age: 60}];
_.max(stooges, function(stooge){ return stooge.age; });
// => {name: 'curly', age: 60}
```

**源码解读：**
```JavaScript
_.max = function(obj, iteratee, context) {

  // 设置循环中比较的初始值以及要用到的变量
  var result = -Infinity, lastComputed = -Infinity,
      value, computed;

  // 如果传入了 obj 并且没有传入 iteratee 那么执行下面代码
  // 按说没有传入实参时，形参自动赋值 undefined，这里用 null 可能是因为
  // undefined == null 以及防止真的传入 null
  if (iteratee == null && obj != null) {

    // 如果 obj 是对象，那么将 obj 重写为包含了 obj 所有自身属性的数组
    obj = isArrayLike(obj) ? obj : _.values(obj);

    // 遍历数组的所有元素或者对象的所有属性值，将其中最大的值保存到 result 中
    for (var i = 0, length = obj.length; i < length; i++) {
      value = obj[i];
      if (value > result) {
        result = value;
      }
    }
  } else {
    // 如果传入了 iretatee 或者传入的 obj 为 null 执行下面逻辑
    // 对 iteratee 进行优化
    iteratee = cb(iteratee, context);

    // 对集合的每个元素执行一次 iteratee，将计算结果排序，并将得到最大计算结果
    // 的元素保存到 result 中
    _.each(obj, function(value, index, list) {

      // 得到执行后的结果
      computed = iteratee(value, index, list);

      // 如果执行后的结果大于上次保存的计算结果
      if (computed > lastComputed || computed === - Infinity &&
          result === -Infinity) {

        // 将当前遍历的元素保存到 result（保存的是最大值） 中
        result = value;

        // 将当前计算结果保存到 lastComputed（保存的是最大值）
        lastComputed = computed;      
      }
    });
  }

  // 返回得到的最大值
  return result;
}
```

### 2 \_.min(obj, iretatee, context)

**示例：**
```JavaScript
_.min([6, 2, 3, 10]);
// => 2

_.min({num1: 2, num2: 5, num3: 50});
// => 50

_.min([6, 2, 3, 10], function(elem) { return elem % 2; });
// => 6
// 只取第一个

_.min({num1: 2, num2: 5, num3: 50}, function(num) { return num % 2; });
// => 2

var stooges = [{name: 'moe', age: 40}, {name: 'larry', age: 50},
    {name: 'curly', age: 60}];
_.min(stooges, function(stooge){ return stooge.age; });
// => {name: 'moe', age: 40}
```

**源码解读：**
min 方法跟 max 方法的实现非常相似，代码中只修改了几处地方
```JavaScript
_.mix(obj, iteratee, context) {
  var result = Infinity, lastComputed = Infinity, value, computed;
  if (iteratee == null && obj != null) {
    obj = isArrayLike(obj) ? obj : _.values(obj);
    for (var i = 0, length = obj.length; i < length; i++) {
      value = obj[i];
      if (value < result) {
        result = value;
      }
    }
  } else {
    iteratee = cb(iteratee, context);
    _.each(obj, function(value, index, list) {
      computed = iteratee(value, index, list);
      if (computed > lastComputed || computed === Infinity &&
          result === Infinity) {
            result = value;
            lastComputed = computed;
          }
    });
  }
  return result;
}
```
