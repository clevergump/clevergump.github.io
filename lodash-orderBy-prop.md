lodash.js源码分析--orderBy property

[TOC]

# 前言

本文不是 js工具库 Lodash的入门介绍文章。本文适合于对 Lodash 有一定使用经验的用户阅读，不适合于前端小白以及没有使用过该库的读者（如果是后者并且你有一定的js基础，建议先熟悉下该库的相关基础知识和使用方法再来阅读本文）。

# API

本文要介绍的是下面的 API：

```js
/**
 * This method is like `_.sortBy` except that it allows specifying the sort
 * orders of the iteratees to sort by. If `orders` is unspecified, all values
 * are sorted in ascending order. Otherwise, specify an order of "desc" for
 * descending or "asc" for ascending sort order of corresponding values.
 *
 * @static
 * @memberOf _
 * @since 4.0.0
 * @category Collection
 * @param {Array|Object} collection The collection to iterate over.
 * @param {Array[]|Function[]|Object[]|string[]} [iteratees=[_.identity]]
 *  The iteratees to sort by.
 * @param {string[]} [orders] The sort orders of `iteratees`.
 * @param- {Object} [guard] Enables use as an iteratee for methods like `_.reduce`.
 * @returns {Array} Returns the new sorted array.
 * @example
 *
 * var users = [
 *   { 'user': 'fred',   'age': 48 },
 *   { 'user': 'barney', 'age': 34 },
 *   { 'user': 'fred',   'age': 40 },
 *   { 'user': 'barney', 'age': 36 }
 * ];
 *
 * // Sort by `user` in ascending order and by `age` in descending order.
 * _.orderBy(users, ['user', 'age'], ['asc', 'desc']);
 * // => objects for [['barney', 36], ['barney', 34], ['fred', 48], ['fred', 40]]
 */
function orderBy(collection, iteratees, orders, guard)
```

# 文章内容说明

1. 本文使用的源码版本是 4.17.15, 下载地址: [https://github.com/lodash/lodash/releases](https://github.com/lodash/lodash/releases)
2. 这篇文章只讨论参数collection 是数组，并且参数 iteratees 为 string 或 string数组时的调用方式和源码解析。源码解析只讨论大致脉络，不需要深入到极端细节。
3. 由于本文的例子采用的是基于 `chain` 方法的链式调用，所以调用时会由于collection参数提前导致省略掉 collection 的后续传参，所以实参会相对于实际API的方法签名定义集体错位。

# API调用方式

如果我们要对一个类型相同的对象所组成的数组进行排序，我们期望的排序规则是：按照对象的某一个属性数值的ASCII码大小，对数组进行升序或降序排列，则可以调用 

```js
orderBy(数组, 属性名, 'asc'或'desc')
```

或者 

```js
orderBy(数组, [属性名], ['asc'或'desc'])
```

即：参数 iteratees 和 orders 既可以传入一个 string，也可以传入一个string数组。

如果要按照对象的多个属性先后排序，属性有优先级顺序，对于按照同一个属性排序时分不出先后顺序的，再按照下一个属性排序， 则可以调用

```js
orderBy(数组, [属性名，属性名...], ['asc'或'desc'，'asc'或'desc'...]);
```

即：参数 iteratees 和 orders 只能传入string数组。

例如：

以API 注释中举例的数组 users为例，

```js
var users = [
    { 'user': 'fred',   'age': 48 },
    { 'user': 'barney', 'age': 34 },
    { 'user': 'fred',   'age': 40 },
    { 'user': 'barney', 'age': 36 }
];
```

如果要按照数组中各对象的user属性数值的 ASCII码对数组进行升序排序，则调用方式如下：

```js
// orderBy的部分，也可以写作如下数组的形式：
// .orderBy(['user'], ['asc'])
var orderedArr = _.chain(users).cloneDeep().orderBy('user', 'asc').value();
console.table(orderedArr);
```

**注意：orderBy的部分，既可以直接写字符串（属性名字符串，排序方式字符串），也可以写成这些字符串的数组。**

排序后的数组名为 orderedArr，其内容如下：

```
[
	{user: "barney", age: 34},
	{user: "barney", age: 36},
	{user: "fred", age: 48},
	{user: "fred", age: 40}
]
```

对于排序时顺序相同的对象，将默认按照他们原始的先后顺序保持不变。例如：上例中，有2个对象的user属性都叫 ”barney“, 那么二者之间根据 user 属性就无法排序，那就保持他们原有的先后顺序不变。另外，也有2个对象的user属性都叫 “fred”，对他们的处理策略也是同理。对于user属性数值相同的对象，如果我们也想按照另外一个属性的数值来对他们进行进一步的排序，例如：想按照 age从大到小排序，那么我们可以这样调用：

```js
var orderedArr = _.chain(users).cloneDeep().orderBy(['user', 'age'], ['asc', 'desc']).value();
console.table(orderedArr);
```

则排序后的结果如下：

```js
[
	{user: "barney", age: 36},
	{user: "barney", age: 34},
	{user: "fred", age: 48},
	{user: "fred", age: 40}
]
```

如果 users数组中包含更多数量的对象时，就很可能存在user及age数值都相同的对象，如果数组中的对象还包含更多的属性，那么我们就可以再继续指定按照更多的属性对上述相同的对象进行更进一步的排序。

# 源码解析

上源码：

1. <span id="1">分析栈： orderBy</span>

```js
function orderBy(collection, iteratees, orders, guard) {
  if (collection == null) {
    return [];
  }
   // 如果 iteratees 不是数组，例如: 'user', 则 iteratees 会被转换为一个数组，数组内只有一个元素，就是我们传递
   // 的形参, 即： 'user'会转换为 ['user']
  if (!isArray(iteratees)) {
    iteratees = iteratees == null ? [] : [iteratees];
  }
  // 本文讨论的重点不在 guard，所以讨论的都是 guard都不传参的情况，即：guard为undefined，所以经过下面这句代码
  // 处理后的 orders数值不变
  orders = guard ? undefined : orders;
  // 如果 orders 不是数组(例如: 'asc')并且guard没有传参，则 orders 会被转换为一个数组，数组内只有一个元素，
  // 就是我们传递的形参，即： 'asc'会转换为 ['asc']
  if (!isArray(orders)) {
    orders = orders == null ? [] : [orders];
  }
  return baseOrderBy(collection, iteratees, orders);
}
```

由于本文分析的着重点不在参数 guard 这部分内容，所以分析时，都默认该参数不传参，即：该参数传递的是undefined 数值。

代码加了注释，从注释来看，我们给 iteratees 及 orders 传递的字符串数值，都将被包装为一个数组。另外，我们一般都是给 orders 传递 ’asc‘ 或 ’desc‘ 这两种数值中的一种，但从上述源码来看，orders并没有要求一定要传这两种数值，我们就有了疑问：可以传其他数值吗？例如：我传 “abc”作为排序规则行不行？答案见后面的分析。

接下来就进入了 baseOrderBy(collection, iteratees, orders) 阶段，此时 iteratees 和 orders 都已被封装成了数组。我们可以以具体调用的例子来辅助我们的源码分析，以上面的例子为例，此时的实参情况如下：

`baseOrderBy(['user', 'age'], ['asc', 'desc'])`

2. <span id="2">分析栈： orderBy->baseOrderBy</span>

```js
/**
 * The base implementation of `_.orderBy` without param guards.
 *
 * @private
 * @param {Array|Object} collection The collection to iterate over.
 * @param {Function[]|Object[]|string[]} iteratees The iteratees to sort by.
 * @param {string[]} orders The sort orders of `iteratees`.
 * @returns {Array} Returns the new sorted array.
 */
// 按照例子中的实参：baseOrderBy(['user', 'age'], ['asc', 'desc'])
function baseOrderBy(collection, iteratees, orders) {
  var index = -1;
  // 需要注意的是, iteratee 这个单词在lodash中经常用来表示迭代用的函数, 而在此处语境中, iteratees 
  // 是一个字符串数组(对象属性名称的数组), 并不是函数数组.
    
  // 经常用来表示迭代函数的
  // 既然是分析按照属性来排序，那么 iteratees 数组就一定不为空数组，至少含有一个属性名称，所以
  //   iteratees.length ? iteratees : [identity] 这部分内容的结果就是 iteratees
  // 经后边3的分析，getIteratee() === baseIteratee
  // 经后边4的分析，baseUnary(getIteratee()) 就是一个函数：
  //    var func1 = function(value) {
  //    	return baseIteratee(value);
  //    };
  // 综上所述，下面这句 arrayMap 的代码在此次源码分析的特定假设前提下，就等效于：
  //   iteratees = arrayMap(iteratees, func1);
  // 经后边5,6,7,8的分析, 下面的 arrayMap 代码就是将 iteratees 这个数组中每个元素的数值分别替换为如下
  // 函数func2, 
  //   	var func2 = function(object) {
  // 		return object == null ? undefined : object[key];
  // 	};
  // 每个 func2最终执行时的内部变量 key分别是各元素先前的数值. 由于 func2本质上是一个闭包, 它内部所
  // 引用的参数 key定义在该闭包的作用域中, 在后续执行时其作用域能继续保持在内存中不被回收, 使得
  // func2 具有了记忆功能, 能够记住变量 key的数值并在 func2自己执行时能够调用到它所记住的 key的数值. 
  // 由于每个 func2记住的都是属于自己的那个变量 key的数值, 这就使得当多个 func2被调用时, 各自调用的变量
  // key 的数值并不会发生张冠李戴的现象. 
  // 总结来说, 在本文的语境下, iteratees 表示属性名称的数组, 下面的 arrayMap的执行就是要将
  // iteratees 数组中的每个元素都替换为 func2, 并且让每个替换后的 func2分别记住自己对应的元素数值作为其
  // 作用域中参数 key的数值.
  // 	[prop1, prop2, prop3...] ==> [func2, func2, func2]  
  iteratees = arrayMap(iteratees.length ? iteratees : [identity], baseUnary(getIteratee())); // ------- (1)

  var result = baseMap(collection, function(value, key, collection) {
    var criteria = arrayMap(iteratees, function(iteratee) {
      return iteratee(value);
    });
    return { 'criteria': criteria, 'index': ++index, 'value': value };
  }); // ------- (2)

  return baseSortBy(result, function(object, other) {
    return compareMultiple(object, other, orders);
  }); // ------- (3)
}
```

需要格外注意的是, 此处语境中的参数 iteratees 这个复数单词, 和我们在 lodash 各种常用 API 中的 iteratee 这个单数单词, 二者虽然只是单复数的区别, 但数据类型和实际含义却有重大区别.  在 lodash 的常用 API 中, iteratee 常用来表示作为迭代器使用的 function类型的变量, 而此处的 iteratees 却并不是一个 function 数组,  而是一个string 数组 (表示对象属性名称的数组).  所以, 如果你想避免后续思路被误导, 你可以将此处的 iteratees 变量名更改为 props 等名称.

我们将上述源码按照语句的顺序做了(1), (2), (3)...这样的序号标识, 后面就按这些序号来分析每一句代码所执行的逻辑.

## 语句(1)的解析

在本文中, 既然是探讨按照属性来排序，那么 iteratees 这个表示属性名称的数组中就至少会含有一个属性名称, 即 iteratees 一定不是空数组，所以 `iteratees.length ? iteratees : [identity]` 这句代码的结果就是 `iteratees`. 所以，

```js
iteratees = arrayMap(iteratees.length ? iteratees : [identity], baseUnary(getIteratee()));
```

这句代码就可以简化为

```js
iteratees = arrayMap(iteratees, baseUnary(getIteratee()));
```

此后分析的重点就是  `baseUnary(getIteratee())` 和 `arrayMap` 这两部分内容。

2.1 <span id="2.1">分析栈： orderBy->baseOrderBy->getIteratee()</span>

先从getIteratee() 入手

```js
/**
 * Gets the appropriate "iteratee" function. If `_.iteratee` is customized,
 * this function returns the custom method, otherwise it returns `baseIteratee`.
 * If arguments are provided, the chosen function is invoked with them and
 * its result is returned.
 *
 * @private
 * @param {*} [value] The value to convert to an iteratee.
 * @param {number} [arity] The arity of the created iteratee.
 * @returns {Function} Returns the chosen function or its result.
 */
function getIteratee() {
 // 我们可能对 lodash.iteratee 进行了重写，如果没有重写，则 lodash.iteratee 和 iteratee都指向相同的
 // 方法，result相当于是对该方法的又一个引用。 如果重写了lodash.iteratee的数值，并且由于js的动态类型
 // 特性，重写可以改变数据类型，所以如果重写后的数值是 truthy的，例如：仍然是function，或者仍然是非''的字
 // 符串、非0的number、非null/undefined等等，则 result就取该重写后的 lodash.iteratee。
 // 反之，如果重写后的 lodash.iteratee 是个falsy的数值，例如：null/undefined/0/''等，那么 result
 // 就取Lodash库原先定义的 iteratee 方法。
  var result = lodash.iteratee || iteratee;
  result = result === iteratee ? baseIteratee : result;
  // 由于getIteratee()方法是在 baseOrderBy()方法内部调用的，调用时没有传参，所以 arguments.length === 0
  return arguments.length ? result(arguments[0], arguments[1]) : result;
}
```

看该API本身的注释，“If `_.iteratee` is customized, this function returns the custom method, otherwise it returns `baseIteratee`.”  这句话差不多就能明白该方法的含义。由于整个 Lodash 库只对外暴露了一个叫做 lodash 的对象作为调用入口，该对象的别名是 _ 。既然对外暴露，那么我们就有可能对定义在该对象上的公有方法进行重写，其中就包括 lodash.iteratee方法。不过一般没有特别需求的情况下我们不会重写该方法。

结合该API本身的注释，以及我们额外添加的注释，概括一下该方法的含义是，如果我们重写了 `lodash.iteratee`方法，则返回我们重写的方法，否则返回 `baseIteratee` 方法。

分析完该方法，我们再回到 [2. 分析栈： orderBy->baseOrderBy][2] 去分析。

2.2 <span id="2.2">分析栈: orderBy->baseUnary</span>

```js
/**
 * The base implementation of `_.unary` without support for storing metadata.
 *
 * @private
 * @param {Function} func The function to cap arguments for.
 * @returns {Function} Returns the new capped function.
 */
function baseUnary(func) {
  return function(value) {
    return func(value);
  };
}
```

baseUnary(xxx) 换句话说就是一个函数了。简单起见，我们就认为我们不会重写 lodash 库的方法，这样，3 中的 getIteratee() 其实就是 `baseIteratee`方法，那么，[2. 分析栈： orderBy->baseOrderBy][2] 中的 `baseUnary(getIteratee())` 就可以认为是如下函数

```js
var func1 = function(value) {
  return baseIteratee(value);
}
```

而在 [2. 分析栈： orderBy->baseOrderBy][2] 分析的最后，我们得出结论：

```js
iteratees = arrayMap(iteratees.length ? iteratees : [identity], baseUnary(getIteratee()));
```

这句代码可以简化为

```js
iteratees = arrayMap(iteratees, baseUnary(getIteratee()));
```

而`baseUnary(getIteratee())` 又是上述函数 func1，那么，上述代码就可以进一步等效为：

```js
iteratees = arrayMap(iteratees, func1);
```

此时我们将分析栈再切回到  [2. 分析栈： orderBy->baseOrderBy][2] 去进行下一个分析。

2.3 <span id="2.3">分析栈: orderBy->arrayMap</span>

```js
/**
 * A specialized version of `_.map` for arrays without support for iteratee
 * shorthands.
 *
 * @private
 * @param {Array} [array] The array to iterate over.
 * @param {Function} iteratee The function invoked per iteration.
 * @returns {Array} Returns the new mapped array.
 */
// 经过前面的分析，此时的实参是： (iteratees, func1)
// 对于我们举的例子，就是：arrayMap(['user', 'age'], func1), func1 如下：
//    var func1 = function(value) {
//    	return baseIteratee(value);
//    };
function arrayMap(array, iteratee) {
  var index = -1,
      length = array == null ? 0 : array.length,
      result = Array(length);

  while (++index < length) {
    result[index] = iteratee(array[index], index, array);
  }
  return result;
}
```

源码本身很好理解, 对数组中的每个元素分别进行转换,  各自转换后的新数值就是 `iteratee(array[index], index, array)` 执行的结果,  下面是进一步的推导过程:

```js
因为
result[index] = iteratee(array[index], index, array);
iteratee = func1 = function(value) {
    return baseIteratee(value);
}
所以, 得出结论:
result[index] = baseIteratee(result[index])
```

调用 arrayMap 函数的作用, 概括来说就是: 对一个数组 array,  我们需要对它里面的每个元素分别执行一次 baseIteratee 变换, 并将变换后的结果作为对应元素的新数值. 

结合本文分析的主题--数组依据其包含的对象的某些属性名称进行排序,  那么这里的数组其实就是属性名称的数组, 所以此处就是对属性名称数组中的每个属性名称执行 baseIteratee 变换.  如果结合前文的 users 例子来说,  数组 array 在变换前的内容是 ['user', 'age'],  执行arrayMap() 变换后的结果是 [baseIteratee('user'), baseIteratee('age')].  所以接下来的分析重心将转向 baseIteratee 这个函数本身的逻辑.

2.4 <span id="2.4">分析栈: orderBy->arrayMap->baseIteratee</span>

```js
/**
 * The base implementation of `_.iteratee`.
 *
 * @private
 * @param {*} [value=_.identity] The value to convert to an iteratee.
 * @returns {Function} Returns the iteratee.
 */
// 在本文的语境中, value 就是数组中对象属性名称的字符串, 例如: users 例子中的 'user', 'age'等属性名称
function baseIteratee(value) {
  // Don't store the `typeof` result in a variable to avoid a JIT bug in Safari 9.
  // See https://bugs.webkit.org/show_bug.cgi?id=156034 for more details.
  if (typeof value == 'function') {
    return value;
  }
  if (value == null) {
    return identity;
  }
  if (typeof value == 'object') {
    return isArray(value)
      ? baseMatchesProperty(value[0], value[1])
      : baseMatches(value);
  }
  // value 是string类型, 所以最终会执行下面这句代码
  return property(value);
}
```

在本文的语境中, 该方法的参数 value 是string类型, 表示数组中对象的属性名称, 例如: users 例子中的 'user', 'age'等属性名称, 那么上述逻辑最终会执行 `return property(value);` 这句.  

2.5 <span id="2.5">分析栈: orderBy->arrayMap->baseIteratee->property</span>

```js
/**
 * Creates a function that returns the value at `path` of a given object.
 *
 * @static
 * @memberOf _
 * @since 2.4.0
 * @category Util
 * @param {Array|string} path The path of the property to get.
 * @returns {Function} Returns the new accessor function.
 * @example
 *
 * var objects = [
 *   { 'a': { 'b': 2 } },
 *   { 'a': { 'b': 1 } }
 * ];
 *
 * _.map(objects, _.property('a.b'));
 * // => [2, 1]
 *
 * _.map(_.sortBy(objects, _.property(['a', 'b'])), 'a.b');
 * // => [1, 2]
 */
// 在本文的语境中, path 就是数组中对象属性名称的字符串, 例如: users 例子中的 'user', 'age'等属性名称
function property(path) {
    // isKey(path)为true, toKey(path)仍为path, 所以最终返回值将是 baseProperty(path)的执行结果
  return isKey(path) ? baseProperty(toKey(path)) : basePropertyDeep(path);
}
```

如注释, 在本文语境中, 参数 path 就是数组中对象属性名称的字符串, 例如: users 例子中的 'user', 'age'等属性名称, 上述方法的返回值将是 baseProperty(path)的执行结果, 

2.6 <span id="2.6">分析栈: orderBy->arrayMap->baseIteratee->property->baseProperty</span>

```js
/**
 * The base implementation of `_.property` without support for deep paths.
 *
 * @private
 * @param {string} key The key of the property to get.
 * @returns {Function} Returns the new accessor function.
 */
function baseProperty(key) {
  return function(object) {
    return object == null ? undefined : object[key];
  };
}
```

可以将 baseProperty 函数换成如下写法:

```js
function baseProperty(key) {
  var func2 = function(object) {
    return object == null ? undefined : object[key];
  };
  return func2;
}
```

所以, `baseProperty`函数调用的结果, 其实是一个函数 func2, 这个新函数 func2 要想执行, 还需要给它传入一个参数, 这个参数看起来既可以是一个数组, 也可以是一个普通对象.

总结 2.1~2.6 的分析,  [2. 分析栈： orderBy->baseOrderBy][2] 中的如下代码 

```js
iteratees = arrayMap(iteratees.length ? iteratees : [identity], baseUnary(getIteratee()));
```

就是将 iteratees 这个数组中每个元素的数值分别替换为如下函数 func2

```js
var func2 = function(object) {
	return object == null ? undefined : object[key];
};
```

每个 func2 最终执行时的内部变量 key分别是各元素先前的数值. 由于 func2本质上是一个闭包, 它内部所引用的参数定义在该闭包的作用域中, 在后续执行时其作用域能继续保持在内存中不被回收, 使得 func2 具有了记忆功能, 能够记住变量 key的数值并在 func2自己执行时能够调用到它所记住的 key的数值. 由于每个 func2记住的都是属于自己的那个变量 key的数值, 这就使得当多个 func2被调用时, 各自调用的变量 key 的数值并不会发生张冠李戴的现象.  所以在本文的语境下, 这句 arrayMap 的执行就是要将 iteratees 数组中的每个元素都替换为 func2, 并且让每个替换后的 func2分别记住自己对应的元素数值作为其作用域中参数 key的数值.

> [prop1, prop2, prop3...] ==> [func2, func2, func2]

分析完这句代码后, 我们又回到了 [2. 分析栈： orderBy->baseOrderBy][2] 去分析下一句代码.









[1]: #1
[2]: #2
[3]: #3