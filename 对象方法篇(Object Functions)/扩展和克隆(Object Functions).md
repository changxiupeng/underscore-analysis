## underscore源码解析(7)--扩展和克隆(Object Functions)

该项目的 **GitHub** 地址为 **[underscore-analysis](https://github.com/changxiupeng/underscore-analysis)**，所有文章都在这里，并将不断更新。如果你觉得我的解读还可以，对你学习 js 有一定的帮助，欢迎 **Watch && Star**，你的关注和肯定必定会促使我投入更多的时间和经历做好这个系列。如果你愿意的话，也欢迎 **Fork**，我们一起来将这个系列做好做大，一起成长。

---
### 扩展
underscore 提供了三个方法来扩展对象的属性和方法：
1. **`_.extend(destination, objs)`**
2. **`_.defaults(destination, objs)`**
3. **`_.extendOwn(destination, objs)`**

这三个方法很相似，都是将 objs （一个或多个对象）中的属性浅复制到 destination 对象中。
不同的是：
-  `_.extend()` 会将 objs 的所有属性（包括继承的）一起复制，如果出现同名属性，前面的会被覆盖，既属性值取最后一次重复出现时的属性值。
- `_.defaults()` 也是复制 objs 的所有属性，与 `_.extend()` 唯一一点区别是后面的同名属性不会覆盖前面的，最终该属性的属性值取 **第一次** 出现时的值。
- `_.extendOwn()` 与 `_.extend()` 的唯一一点区别是只复制 objs 自身的属性。

**示例：**
```JavaScript
var a = {
  name: "comma",
  age: 26
}

var b = {
  name: "cxp",
  hobby: "Programming"
}
b.prototype.city = "Wuhan";

_.extend(a, b);
// => {name: "cxp", age: 26, hobby: "Programming", city: "Wuhan"}

_.defaults(a, b);
// => {name: "comma", age: 26, hobby: "Programming", city: "Wuhan"}

_.extendOwn(a, b);
// => {name: "comma", age: 26, hobby: "Programming"}
```

**源码解读：**

```JavaScript
_.extend = createAssigner(_.allKeys);

_.defaults = createAssigner(_.allKeys, true);

// extendOwn 也可以写成 assign，可能是照顾到 ES6 中 Object.assign(objs)方法
// extendOwn 与 Object.assign(objs) 得到同样的结果
_.extendOwn = _.assign = createAssigner(_.keys);
```
可以看出来两个方法都严重依赖了一个 underscore 的内部函数 `createAssigner`
这个函数根据不同的要求（只复制自身属性还是包括继承属性），生成一个属性分配器函数并返回，然后由这个属性分配器函数接收具体的对象，实现属性扩展的具体逻辑



```JavaScript
/**
 * @param keysFunc 指定获取属性的方式
 * @param undefinedOnly 同名属性是否取第一次出现时的属性值
 */
var createAssigner = function(keysFunc, undefinedOnly) {

  // 将属性分配器函数返回，这里只设了一个形参，表示要扩展的对象
  // 其它被复制的对象虽然没有指定形参，但是也可以传入，通过arguments 获取
  return function(obj) {

    // 获取传入的对象的个数
    var length = arguments.length;

    // 要扩展的对象没有被传入、传入一个，或者传入一个但是值为 null，将其返回
    if (length < 2 && obj == null) return obj;

    // 遍历每一个要被复制的对象
    for (var index = 1; index < length; index++) {

      // 获取当前被遍历对象的属性
      var source = arguments[index],
          keys = keysFunc(source),
          l = keys.length;

        // 对当前遍历对象的所有属性进行遍历
        for (var i = 0; i < l; i++) {
          var key = keys[i];

          // 当 undefinedOnly 为 false，既同名属性不取第一个属性值
          // 或者 obj 中没有这个属性时，将当前对象中的属性属性值添加到 obj 中
          // ---------------------------------------------------------
          // _.extend 和 _.extendOwn 方法没有传入 undefinedOnly 参数，
          // undefinedOnly === undefined, 所以 !undefinedOnly === true
          // 所以，无论 obj 中是否有 key，后面的同名 key 都会将其重写
          // ---------------------------------------------------------
          // 而 _.defaults 中 undefinedOnly 被设置为 true
          // !undefinedOnly === false 只有当 obj 中没有 key 时才会被添加
          if (!undefinedOnly || obj[key] === void 0) obj[key] =
              source[key];
        }
    }

    return obj;
  };
};
```

### 克隆
underscore 提供了一个用于对象克隆（浅）的方法:
**`_clone(obj)`**
该方法接收一个对象为参数，然后返回一个此对象副本，作为参数的对象中任何嵌套的对象和数组都只被复制了对应的引用给返回的对象副本。

**示例：**
```JavaScript
var a = {
  name: "comma",
  gf: {
    name: "kxt"  
  }
}

var b = _.clone(a);
console.log(b.name); // comma

// 因为是浅复制，所以引用类型共享一个内存地址
console.log(b.gf === a.gf); // true
b.gf.age = 26;
console.log(a.gf.age); // 26
```
**注**：
无论是数组还是对象，虽然 clone 返回的数组/对象与原来的数组/对象不在同一个地址了，比直接赋值的浅复制深，但它所执行的仍然是浅复制，返回数组/对象内部嵌套的数组和对象仍然只是引用。

**源码解读：**
```JavaScript
_.clone = function(obj) {

  // 如果 obj 不是对象，则直接将其返回
  // 这里的对象指的是函数和 typeof 为 “object” （不包括 null）的数据类型
  if (!_.isObject(obj) return obj);

  // 如果 obj 是数组，那么就将其调用 slice 方法后返回
  // 如果 obj 是对象，那么就调用 extend 方法将其复制到一个空对象中并返回
  return _.isArray(obj) ? obj.slice() : _.extend({}, obj);
}
```
