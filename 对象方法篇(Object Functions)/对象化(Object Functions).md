## 对象化
如果我们想要对一个数据按照对象进行操作，那么我们有必要先将其对象化。
对一个数据进行对象化，据我所知有两种方法：

1. `var myObj = obj || {}`
2. `var myObj = Object(obj)`

underscore中大量地使用了第二种方法
那么，这两种方式有什么区别呢，我们来分别看一下：
### 1. `var myObj = obj || {}`
这段代码的意思是：
1. 如果 obj 是 `undefined, null, 0, NaN, false, ""` 这类能转成 false 布尔值的数据时，那么就将 `{}` 赋值给 myObj
2. 如果 obj 是其它数据时，直接将其赋值给 myObj，obj 可以是数字，字符串等，不一定是一个严格意义上的对象（typeof obj === "object"）

### 2. `var myObj = Object(obj)`
这段代码的意思是：
1. 如果 obj 是一个严格意义上的对象（不含 null），那么直接将其返回
2. 如果 obj 是 undefined 或者 null，那么返回一个空对象 `{}`
3. 如果 obj 是其它原始类型的值，那么返回一个 **对象类型** 的被包裹的原始值，什么意思呢，如下所示：

```JavaScript
var obj = Object(2); // 相当于 new Number(2); 返回的是一个对象
// => Number {[[PrimitiveValue]]: 2}  

var obj = Object(NaN); // 相当于 new Number(NaN); 返回的是一个对象
// => Number {[[PrimitiveValue]]: NaN}  

var obj = Object(""); // 相当于 new String(""); 返回的是一个对象
// => String {length: 0, [[PrimitiveValue]]: ""}  
```
