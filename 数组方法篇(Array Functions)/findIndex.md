```JavaScript
_.findIndex = createPredicateIndexFinder(1);
_.findLastIndex = createPredicateIndexFinder(-1);

// 之所以又定义一个函数，是因为可以默认传入一个表示方向的参数
// 这样在需要从左开始查找的时候就可以直接调用 findIndex，
// 而从右开始查找直接调用 fingLastIndex
var createPredicateIndexFinder = function(dir) {

  // 返回了在调用方法时真正接收参数的函数
  // 参数分别为数组、检测回调、检测回调的上下文
  return function(array, predicate, context) {

    // 为检测回调绑定上下文
    predicate = cb(predicate, context);

    // 获取 array 的长度，getLength 是 underscore 的一个内部函数，
    // 接收一个数据，如果这个数据有 length 属性，那么就返回其 length 值
    var length = getLength(array);

    // 下面要对 array 中的元素进行遍历了
    // 因为有左右两个方向，所以获取开始遍历的起始索引
    var index = dir > 0 ? 0 : length - 1;

    // index 要么是递增要么是递减，但是只要它在 array 的长度范围内就还没遍历完
    for (; index >= 0 && index < length; index += dir) {

      // 如果遍历中的元素能通过 predicate 的检验那么就返回这个元素的索引
      if (predicate(array[index], index, array)) return index;
    }

    // 如果遍历完了还没有返回，那么就说明没有元素符合 predicate 的条件，返回 -1
    return -1;
  }
}
```
