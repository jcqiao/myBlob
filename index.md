# 类型转换

## 基础部分

### ToString

负责处理非字符串到字符串的强制类型转换。**toString** 是定义在 Object.prototype.toString 上，如果被处理的对象自身定义了toString则会覆盖其原型的toString，所以重写了原型的toString.

基本类型值的字符转换规则：null --> "null" , undefined --> "undefined" , true --> "true" , Number类型遵循通用规则(除了极大或绩效的数字使用指数形式)

Object.prototype.toString()返回内部属性[[class]]值，如[[object number]].

![avatar](https://github.com/jcqiao/myBlob/raw/gh-pages/code_Class.png)

以上介绍了基本类型的toString，`数组的toString经过重新定义`，将单元字符串化再用','拼接

``` javascript
let a = [1,2,3]
a.toString() // "1,2,3"
```

a.toString() 相当于是a.join()

### JSON字符串化 (不是强制类型转换 这里介绍是有关toString())

对于大多数简单值如bool,number,string,null其转为字符串的方法与toString基本相同。

所有安全的JSON值都可以用JSON.stringify()转化，不安全的JSON值如undefined,function,symbol,对象的循环引用，这些都不能正确呈现JSON格式，JSON在处理的时候回忽略掉输出undefined，若是在数组中则用null占位保证数组长度不变。

对于对象来说:
如果对象定义了toJSON()那么JSON.stringify()首先会调用toJSON()然后用他的返回值进行序列化

``` javascript
var obj = {
  name: 'a',
  age: 11,
  like: 'fdsfds',
  toJSON(){
    return { name: this.name}
  }
}
JSON.stringify(obj) //"{"name": "a"}"
```

很多人误认为toJSON()返回的是JSON字符串化后的值，其实不然，toJSON返回一个适当的值(安全的JSON值)，然后JSON.stringify()将toJSON()返回来的值进行字符串化。

- JSON.stringify(obj, params)
  他的第二的参数params可以是一个数组、函数。用来指定序列化的对象哪些属性需要被处理。其功能类似toJSON()

  ``` javascript
    var obj = {name: 'a', age: 11, like: [1,2,3]}
    JSON.stringify(obj, ["name","age"]) //"{"name": "a", "age": 11}"
    JSON.stringify(obj, function(k, v){ if(k !== "age") return v}) //"{"name": "a", "like": [1,2,3]}"
  ```

  字符串化是递归的，所以like数组也要进行序列化处理

### 总结

1. string,number,boolean,null的JSON.stringify()规则与toString()相同
2. 若对象中定义了toJSON()则在字符串化之前调用toJSON()，再进行字符串化

## toNumber

将非数值当做数值使用。true --> 1 false --> 0 undefined --> NaN null --> 0
toNumber不会将第一个0解析为八进制，遇到非数值就会停止。

对象、数组都会遵循一定规则先转为对应的基本类型值。该规则是先检查对象/数组是否有valueOf()，若有就返回基本类型(可能是Number,String...)；如果没有就找toString()方法，返回基本类型。再跟进基本类型进行数值转换

``` javascript
a = {valueOf() {return "ab"}}
Number(a) //NaN
```

1. 首先调用valueOf()将a转为String类型，值是"ab"
2. 对"ab"进行数值转换 NaN

数组的toString方法上文提到过，`数组重写了toString方法`，返回单元用','连接的字符串

## toBoolean

假值：
'' , false, +-0, undefined, null, NaN
真值
除假值以外

## 强制类型转换

### 字符串与数字

- 由String()和Number()两个内建函数实现的，不带new关键字，所以不创建包装对象。
- string遵循toString Number遵循toNumber。
- Object.prototype.toString()也可以显示实现转为字符串。

``` javascript
  var a = 123
  a.toString()
```

  a是个基本类型 所有js引擎自动为a创建了封装对象，然后该对象调用toString()方法

- +一元加也可以实现将字符串转为数字
  在js开源社区中 一元运算符'+'被认为是显示强制类型转换

  ``` javascript
  var a = +"123" //123
  ```

  使用场景：
  日期转为时间戳

  ``` javascript
  var d = +new Date()
  var d = Date.now() //es5 实现的机制也是+
  ```

  推荐使用Date.now()获取当前时间戳 使用new Date(d).getTime()获取指定日期的时间戳

- 奇怪的~运算符
  ~非 ~x 大致等同于 -(x+1) x=-1是一个“哨位值”，在js indexOf()方法中体现
  - 通常我们判断是否有指定值时if(a.indexOf("b") === -1)未找到，可以用if(~a.indexOf("b"))
  - 字位截除
    ~~ 相当于Math.floor 但在负数时不一样

    ``` javascript
    Math.floor(-49.6) // -50
    ~~-49.6 //-49
    ```

## 显示解析数字字符串

  解析字符串中的数字 vs 强制类型转换
  首先 两者是有区别的：

  ``` javascript
  var a = "42"
  var b = "42ab"
  Number(a) === parseInt(a) //true 42
  Number(b) //NaN
  parseInt(b) //42
  +a //42
  +b //NaN
  ```

  他们的区别在于 解析允许字符串中包含非数字(第一项非字母)，解析直到遇到非数字停止。转换不允许有非数字的出现，否则返回NaN。
  parseInt针对的是字符串解析，若传递非字符串首先隐式转换为字符串，然后在进行解析。第二个参数指定基数，也就是按照几进制来进行转换
  +运算符采用的parseInt

  ###解析非字符串
  parseInt的一个坑

  ``` javascript
  parseInt(1/0,19) //18
  ```

  1/0 --> Infinite / ∞，显然在js中选择了Infinite 但返回18未免也太匪夷所思了，是因为parseInt将Infinite转为字符串"Infinite"，然后在将字符串"Infinite"按照19进制进行解析I是18，n大于18所以解析停止。
  parseInt解析非字符串，若定义了toString()方法则调用定义的toString()方法。
