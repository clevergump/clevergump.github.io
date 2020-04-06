lodash源码分析--orderBy property

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

1. 这篇文章只讨论参数collection 是数组，并且参数 iteratees 为 string 或 string数组时的调用方式和源码解析。源码解析只讨论大致脉络，不需要深入到极端细节。
2. 由于本文的例子采用的是基于 `chain` 方法的链式调用，所以调用时会由于collection参数提前导致省略掉 collection 的后续传参，所以实参会相对于实际API的方法签名定义集体错位。

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
var orderedArr = _.chain(users).cloneDeep()
	.orderBy(['user', 'age'], ['asc', 'desc']).value();
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

1. 分析栈： orderBy

```js
function orderBy(collection, iteratees, orders, guard) {
  if (collection == null) {
    return [];
  }
   // 如果 iteratees 不是数组，例如: 'user', 则 iteratees 会被转换为一个数组，数组内只有一个元素，就是我们传递的形参, 即： 'user'会转换为 ['user']
  if (!isArray(iteratees)) {
    iteratees = iteratees == null ? [] : [iteratees];
  }
  // 本文讨论的重点不在 guard，所以讨论的都是 guard都不传参的情况，即：guard为undefined，所以经过下面这句代码处理后的 orders数值不变
  orders = guard ? undefined : orders;
  // 如果 orders 不是数组(例如: 'asc')并且guard没有传参，则 orders 会被转换为一个数组，数组内只有一个元素，就是我们传递的形参，即： 'asc'会转换为 ['asc']
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

2. 分析栈： orderBy->baseOrderBy

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
  iteratees = arrayMap(iteratees.length ? iteratees : [identity], baseUnary(getIteratee()));

  var result = baseMap(collection, function(value, key, collection) {
    var criteria = arrayMap(iteratees, function(iteratee) {
      return iteratee(value);
    });
    return { 'criteria': criteria, 'index': ++index, 'value': value };
  });

  return baseSortBy(result, function(object, other) {
    return compareMultiple(object, other, orders);
  });
}
```

为了分析上述源码，我们必须先分析它调用的其他API源码。

3. 分析栈： orderBy->baseOrderBy->getIteratee()

先从getIteratee() 入手

```
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
  return arguments.length ? result(arguments[0], arguments[1]) : result;
}
```

看注释，“If `_.iteratee` is customized, this function returns the custom method, otherwise it returns `baseIteratee`.”  这句话差不多就能明白该方法的含义，由于整个 Lodash 库只对外暴露了一个叫做 lodash 的对象作为调用入口，该对象的别名是 _ 。既然对外暴露，那么我们就有可能对定义在该对象上的公有方法进行重写，其中就包括 lodash.iteratee （即: _.iteratee）。

