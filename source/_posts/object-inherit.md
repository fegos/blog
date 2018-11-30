---
title: Javascript中的对象、函数以及类的继承
tags: javascript
author: henry
date: 2018-11-27 00:00:00
---


## **JS的类**
在研究继承之前, 我们先看一下Javascript中的类是如何实现的

**ES6**
```javascript
class A {
  constructor() {
    this.name = 'class A';
  }
  toString() {
    console.log(this.name)
  }
}
```
**ES5**
```javascript
/* ..._createClass... */
/* ..._classCallCheck... */

var A = (function() {
  function A() {
    _classCallCheck(this, A);

    this.name = "class A";
  }

  _createClass(A, [
    {
      key: "toString",
      value: function toString() {
        console.log(this.name);
      }
    }
  ]);

  return A;
})();
```

使用`new`操作符生成类的实例对象
```javascript
const a = new A();

a.toString()    // class A
a.__proto__ === A.prototype   // true
```
**类是一种特殊的函数, 通过new关键字调用函数产生新实例对象, 通过this访问对象的属性和方法, 最后返回该对象**

## **原型链**
Javascript中的继承是通过原型链实现的。

**ES6**
```javascript
class A {}

class B extends A {}
```
**ES5**
```javascript
var A = function A() {
  _classCallCheck(this, A);
};

var B = (function(_A) {
  _inherits(B, _A);   // 继承的实现

  function B() {
    _classCallCheck(this, B);

    return _possibleConstructorReturn(
      this,
      (B.__proto__ || Object.getPrototypeOf(B)).apply(this, arguments)
    );
  }

  return B;
})(A);

/* _possibleConstructorReturn */

function _classCallCheck(instance, Constructor) {
  if (!(instance instanceof Constructor)) {
    throw new TypeError("Cannot call a class as a function");
  }
}

function _inherits(subClass, superClass) {
  // 原型的继承 (私有方法和属性)
  subClass.prototype = Object.create(superClass && superClass.prototype, {
    constructor: {    // constructor指向subClass构造函数
      value: subClass,
      enumerable: false,
      writable: true,
      configurable: true
    }
  });
  // 类的继承 (静态方法和属性)
  if (superClass)
    Object.setPrototypeOf
      ? Object.setPrototypeOf(subClass, superClass)
      : (subClass.__proto__ = superClass);
}

```
在`_inherits`函数的代码中可以看到, 类的继承是通过两条原型链实现的
1. 私有方法的原型链
```
const a = new A();
const b = new B();

b.__proto__ === B.prototype   // true

B.prototype instanceof A    // true
B.prototype.__proto__ === A.prototype   // true
b.__proto__.__proto__ === A.prototype   // true

b instanceof B    // true
b instanceof A    // ???
```
2. 静态方法的原型链
```
A.print = () => { console.log('print, static in A') }
A.test = () => { console.log('test, static in A') }
B.test = () => { console.log('test, static in B') }

B.test()    // test, static in B
B.print()   // ???
```
**类继承的实现方式已经比较清楚了**
* 通过类的`prototype`属性, 构造出类实例的原型链, 实现私有属性的继承
* 通过类的`__proto__`属性, 构造出类的原型链, 实现静态属性的继承

**继承关系（B extends A）**
![image](/images/object-inherit.png)

## instanceof的原理

引用MDN文档中关于`instanceof`操作符的解释

`instanceof运算符用于测试构造函数的prototype属性是否出现在对象的原型链中的任何位置`

`A.prototype`存在于b的`__proto__`原型链上, 所以

```
b instanceof A    // true

// B.prototype赋值为A的新实例
B.prototype = Object.create(A.prototype, {
  constructor: {
    value: B,
    enumerable: false,
    writable: true,
    configurable: true
  }
});

b instanceof B    // ???
b instanceof A    // ???
```

## Object & Function

**Javascript中一切都是对象, 包括函数**, 更准确的说法是:

1. 除了原始类型外, 都是对象（Array, Object, Function等）
2. 除了`null`和`undefined`, 原始类型在使用时可以被包装成对象（String, Number, Boolean等）

这里的主题是**函数**与**对象**的关系

**Object是对象的构造函数, Function是函数的构造函数**

* Object是一个函数, 那它是Function的一个实例吗?
* Object也是一个对象, 那它会不会是自己的一个实例？
* Function是函数也是对象, 那么它又是谁的实例呢？是Function？还是Object？

我们从原型和继承的角度, 分析一下Object和Function之间的关系。

**Object与Function的关系**
```javascript
Object instanceof Function    // true
Function instanceof Object    // true

Object instanceof Object    // true
Function instanceof Function    // true

/* 原型 */
ORIGIN_OBJECT = Object.prototype    // {constructor: ƒ, __defineGetter__: ƒ, __defineSetter__: ƒ, hasOwnProperty: ƒ, …}
ORIGIN_FUNCTION = Function.prototype    // ƒ () { [native code] }

ORIGIN_OBJECT.__proto__   // null
ORIGIN_FUNCTION.__proto__   // ORIGIN_OBJECT

ORIGIN_FUNCTION instaceof Function    // false
ORIGIN_FUNCTION instaceof Object    // true

ORIGIN_FUNCTION()   // undefined

new ORIGIN_FUNCTION()   // Uncaught TypeError: Function.prototype is not a constructor
ORIGIN_FUNCTION.prototype   // undefined

/* 原型链 */
Object.__proto__ === Function.prototype   // ORIGIN_FUNCTION
Function.__proto__ === Function.prototype   // ORIGIN_FUNCTION
Function.prototype.__proto__ === Object.prototype   // ORIGIN_OBJECT
Object.prototype.__proto__ === null   // null
```
我们得到了两个最原始的**对象**类型:
* `{constructor: ƒ, __defineGetter__: ƒ, …}`, 原始对象, 处于原型链上描述Object类型, 所有原型链的终点
* `ƒ () { [native code] }`, 原始函数, 处于原型链上描述Function类型, 与关键字new的构造函数实例化有关

**构造函数实例化**

这里有两个比较重要的属性
* `prototype`, 作为构造函数, 用于实例化时生成原型链`__proto__`
* `__proto__`, 作为函数实例, 指向Function.prototype, 形成原型链

```javascript
/* Object实例化 */
const o = new Object();
o.__proto__ === Object.prototype

/* Function实例化 */
const fn = new Function()
fn.__proto__ === Function.prototype
fn.prototype.__proto__ === Object.prototype   // fn.prototype = Object.create(Object.prototype, { constructor: fn })

/* fn实例化 */
const t = new fn()
t.__proto__ === fn.prototype
t.__proto__.__proto__ === Object.prototype
```

Object的实例化较为简单:
* 生成新实例对象, 其`__proto__`指向`Object.prototype`
* 实例对象为Object类型

Function的实例化稍有不同:
* 生成新实例对象, 其`__proto__`指向`Function.prototype`
* 实例对象（fn）为Function类型, 可执行实例化生成新对象（Object类型）
* 将fn的`prototype`属性赋值为Object类型的对象, 并将`prototype.constructor`指向fn自身

**函数是一类特殊的Object, 它的实例是一个对象; Function是一个特殊的函数, 它的实例是一个函数类型的对象**
* `Object.prototype`和`Function.prototype`是两段原生代码（原生对象类型）, 都位于原型链上, 分别作为`Object`和`Function`的类型判断依据
* `Object`是个函数类型的对象, 可以看做`Function`的一个实例, 那么`Object.__proto__ = Function.prototype`符合自身为函数类型
* `Function`是个函数类型的对象, 可以看做`Function`的一个实例, 那么`Function.__proto__ = Function.prototype`符合自身为函数类型
* `Object`和`Function`都是对象, `Object.prototype`应该存在于原型链上, 那么`Function.prototype.__proto__ = Object.prototype`符合自身为对象类型
* `Function`的实例为`fn`, 那么`Function.prototype`和`Object.prototype`都在`fn`的原型链上, 符合`Function`的实例是函数也是对象

通过一张图来说明Object、Function及其实例的关系
![image](/images/object-function.png)
