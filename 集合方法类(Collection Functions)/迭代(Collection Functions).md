## underscore源码解析(8)--迭代(Collection Functions)

原生的 ES5 提供了很多关于的迭代方法，但是对于对象只能通过 for in 循环，或者用 Object.keys(obj) 提取出对象的属性后，再用数组的方法迭代这些提取出来的属性。underscore 针对这种情况定义了很多既可以迭代数组（包括类数组）也可以迭代对象的方法，这些方法被称为 Collection Functions（集合方法），这里的集合指的是可迭代的序列（数组或对象）。这篇文章起，我们将对这些方法进行一一解读。

### 1 \_.each(list, iteratee, context)
该方法与 ES5 中的 forEach 方法类似，会遍历 *list* 中的所有元素，并按顺序用 *iteratee* 去处理每个迭代的元素，*iteratee* 被绑定了 *context* 上下文。每次调用 *iteratee* 都会传递三个参数：*(element, index, list)*。如果 *list* 是一个对象，*iteratee* 的参数是：*(value, key, list)*。返回 *list* 以方便链式调用。

**示例**
```JavaScript
var obj = {
  name: "comma",
  age: 26,
  hobby: "Programing"
}

_.each(obj, function(value, key, obj) {
  console.log(key + "is " + value);
});
// name is comma
// age is 26
// hobby is Programing

_.each([1,2,3], function(element, index) {
  console.log("index " + index + " vs " + element);
});
// index 0 vs 1
// index 1 vs 2
// index 2 vs 3
```
**源码解读**
```JavaScript
// 该方法有两个名字 each 和 forEach
_.each = _.forEach = function(obj, iteratee, context) {

  // 对 iteratee 进行优化，主要是绑定上下文
  iteratee = optimizeCb(iteratee, context);
  var i, length;

  // 如果 obj 是数组或类数组
  if (isArrayLike(obj)) {
    for (i = 0, length = obj.length; i < length; i++) {

      // 数组和类数组的回调中传入参数分别为（迭代值，迭代索引，迭代的数组/类数组）
      iteratee(obj[i], i, obj);
    }
  } else {

    // 如果 obj 是对象
    // 获取对象的所有属性
    var keys = _.keys(obj);
    for (i = 0, length = keys.length; i < length; i++) {

      // 对象的回调中传入参数分别是（迭代属性值，迭代属性，迭代对象）
      iteratee(obj[keys[i]], keys[i], obj);
    }
  }

  // 返回传入的 obj 自身，便于链式调用
  return obj;
}
```
这个方法依赖了 *isArrayLike* 和 *optimizeCb* 两个内部函数，这两个内部函数在[之前的文章](https://changxiupeng.github.io/2016/12/12/underscore%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90%EF%BC%882%EF%BC%89/)中介绍过，有兴趣的可以点击看一下。

### 2 \_.map(list, iteratee, context)
该方法与 ES5 中的 map 方法类似，会遍历 *list* 中的所有元素，当 *iteratee* 按顺序去处理每个迭代的元素时会返回一个新元素，这个新元素被加入到该方法最后返回的数组中。 也就是说，该方法会将 *list* 中的每个元素映射为一个新元素，并最终构成一个新的数组返回。

**示例：**
```JavaScript
_.map([1,2,3], function(element) {
  return ++element;
});
// [2,3,4]

var obj = {
  name: "comma",
  age: 26,
  hobby: "Programing"
}
_.map(obj, function(value, key) {
  return value;
});
// ["comma", 26, "Programing"]
```
**源码解读：**
```JavaScript
_.map = _.collect = function(obj, iteratee, context) {

  iteratee = cb(iteratee, context);

  // 如果 obj 不是类数组，既是对象，那么就拿到它所有的属性
  var keys = !isArrayLike(obj) &&  _.keys(obj),

      // 属性集合 keys（如果 obj 不是对象，值为 false）或者 obj（数组）的长度
      length = (keys || obj).length,

      // 初始化一个长度固定的数组
      results = Array(length);

  for (var index = 0; index < length; index++) {

    // 获取当前迭代的属性（针对对象）或者索引（针对数组和类数组）
    var currentKey = keys ? keys[index] : index;

    // 将映射的新元素加入到要返回的数组中
    results[index] = iteratee(obj[currentKey], currentKey, obj);
  }
  return results;
}
```

### 3 \_.reduce(list, iteratee, memo, context)
该方法把 *list* 中的所有元素归结（折叠）为一个数值返回，*iteratee* 是迭代中使用的回调函数，*memo* 是折叠时的初始值，*context* 是回调函数的上下文。underscore 还提供了一个相似的方法 reduceRight，reduce 方法是从左开始折叠，reduceRight 是从右开始折叠。

**示例：**
```JavaScript
_.reduce([1, 2, 3], function(memo, num) {return memo + num;}, 0);
// => 6

var obj = {
  name: "comma",
  age: 26,
  hobby: "Programing"
}

_.reduce(obj, function(memo, value) {return memo + "@" + value;});
// => comma@26@Programing
```
**源码解读：**
```JavaScript
// 从左边开始折叠，该方法有三个名字 reduce、foldl、inject
_.reduce = _.foldl = _.inject = createReduce(1);

// 从右边开始折叠，该方法有两个名字 reduceRight、foldr
_.reduceRight = _.foldr = createReduce(-1);
```
这两个方法都严重依赖了一个内部函数 createReduce，我们来仔细看一下：
```JavaScript
// 创建一个折叠函数
// dir 为 1 时表示从左开始迭代
// dir 为 -1 表示从又开始迭代
function createReduce(dir) {

  /**
   * @param obj 要迭代的集合
   * @param iteratee 迭代时用的回调函数
   * @param memo 指定好的迭代初始值
   * @param keys 如果 obj 是对象，那么 keys 就是包含了 obj 所有属性的数组
   * @param length obj 的长度
   */
  function iterator(obj, iteratee, memo, keys, index, length) {

    // 对 obj 的元素进行迭代
    for (; index >= 0 && index < length; index += dir) {

      // 获取当前迭代的元素的属性或索引
      var currentKey = keys ? keys[index] : index;

      // 用当前迭代的返回值重写 memo
      memo = iteratee(memo, obj[currentKey], currentKey, obj);
    }

    //  返回迭代结束时的 memo 值，既最终结果
    return memo;
  }

  /**
   * @param obj 要迭代的集合
   * @param iteratee 迭代时用的回调函数
   * @param memo 是迭代的初始值，每次迭代都要有返回值，并保存到 memo 中
   * @param context 回调函数的上下文
   */
  return function(obj, iteratee, memo, context) {

    // 根据回调函数接收参数的个数（这里为4个）为其绑定上下文
    iteratee = optimizeCb(iteratee, context, 4);

    // 如果 obj 是对象，那么获取它所有的属性
    var keys = !isArrayLike(obj) && _.keys(obj)，

        // 获取集合的长度
        length = (keys || obj).length，

        // 根据迭代的方向获取迭代开始的索引值
        index = dir > 0 ? 0 : length - 1;

    // 如果没有传入初始值，那么就指定一个初始值
    if (arguments.length < 3) {

      // 根据 obj 的类型来获取第一个要迭代的元素，并将其赋值给 memo
      memo = obj[keys ? keys[index] : index];

      // 迭代开始的索引（属性）
      index += dir;
    }
    return iterator(obj, iteratee, memo, keys, index, length);
  }
}
```
这个方法之所以比 each 和 map 复杂很多，是因为要考虑到折叠的方向既从那边开始迭代，所以首先要获取迭代的初始索引和折叠的初始值。
