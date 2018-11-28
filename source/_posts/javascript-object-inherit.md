---
title: JS对象继承
date: 2017-08-11
tags: javascript
author: lucky3mvp
---

## 构造函数

**对象 B 继承对象 A：让对象 B 可以拥有对象 A 的属性和方法**

+ 假设现有 Father 类，拥有属性 name, age
+ 同时，Father 类原型有方法 getAge, sayHi

```js
function Father(name, age) {
    this.name = name;
    this.age = age;
}

Father.prototype.getAge = function() {
    return this.age; 
}

Father.prototype.sayHi = function() {
    return `Hi ${this.name}`
}
```

## Prototype

**类的原型**

```js
类的 prototype 属性指向类原型对象
类原型的 constructor 属性指向类自身
A.prototype.constructor === A
```
![obj&it's prototype](https://attachments.tower.im/tower/1c24e97d5440461fb8c74714546c0741?filename=obj+and+its+prototype.png)

实例可以拥有 prototype 对象里的属性和方法。

```js
// 变量 arr 继承了 Array 原型的 indexOf 方法
// 变量 obj 继承了 Object 原型的 hasOwnProperty 方法
let arr = ['I', 'have', 'indexOf', 'method'];
let obj = {
    msg: "I'm object, I have hasOwnProperty method"
};

arr.indexOf('I')            // 0
obj.hasOwnProperty('msg')   // true
obj.hasOwnProperty('name')  // false
```

```js
function ObjGenFunc(name) {
    this.name = name;
}
// 类上的方法，不可以被实例继承
ObjGenFunc.saySorry = function() {
    return 'sorry'
}
// 原型上的方法，可以被实例继承
ObjGenFunc.prototype.sayHi = function() {
    return `hi ${this.name}!`;
}
// 原型上的属性，可以被实例继承
ObjGenFunc.prototype.age = 18;

let instance = new ObjGenFunc('asy');
instance.name;          // "asy"
instance.age;           // 18
instance.sayHi();       // "hi asy!"
instance.saySorry();    // error: instance.saySorry is not a function
```

![obj&it's inherit](https://attachments.tower.im/tower/f4f7314829c84127b5ed75ba93f9c3c7?filename=obj%26pro%26ins.png)

**原型链**

JS 实例对象有一个指向原型对象的指针(例如：浏览器显示的`__proto__`)。当访问一个对象的属性时：
1. 先在该对象上搜寻，若找到则结束，若找不到进入第二步；
2. 在该对象的`__proto__`指向的对象上继续寻找，找到则结束，找不到重复第二步；
3. 依次层层向上搜索，直到找到一个匹配的属性，或者到达原型链的末尾(`Object.prototype.__proto__ === null`)

当创建一个实例时:
+ 实例对象的原型，用 `__proto__` 表示，指向其构造函数的 prototype 属性;
+ 同时实例的 constructor 属性，指向其构造函数自身

结合对象和其原型的关系，有以下关系和图示：

```js
a.constructor === A
a.__proto__ === A.prototype
a.__proto__.constructor === A
```

**子类与父类**

将子类的原型当做父类的实例来让子类原型继承父类的属性和方法，从而子类的实例能继承父类的实例和方法

```js
function Child(name, age, school) {
    this.name = name;
    this.age = age;
    this.school = school;
}

// 将 Child 的原型视为父类 Father 的实例，因此 Child.prototype 就拥有了 Father 的原型方法和原型属性
Child.prototype = new Father();
// 继承链修正
Child.prototype.constructor = Child;

let childIns = new Child('child', 18, 'RUC');
childIns.name/age/school;       // 'child' 18 'RUC'
childIns.sayHi();               // 'Hi child'
childIns.getAge();              // 18

let fatherIns = new Child('father', 50);
childIns.name/age;              // 'father' 50
childIns.sayHi();               // 'Hi father'
childIns.getAge();              // 50
```

```js
// Father 和 Father 实例的关系
fatherIns.constructor === Father
fatherIns.__proto__ === Father.prototype
fatherIns.__proto__.constructor === Father

// Child 和 Child 实例的关系
childIns.constructor === Child
childIns.__proto__ === Child.prototype
childIns.__proto__.constructor === Child

// Child 和 Father 的关系
Child.prototype.__proto__ === Father.prototype
```

![prototype inherit](https://attachments.tower.im/tower/1301acd2c59e4f4abdff1333425be1f2?filename=ES5.png)

## **ES6 继承**

其实质是先创造父类的实例对象 this（也因此子类的 constructor 必须先调用 super 方法），然后再用子类的构造函数修改 this

```js
// 父类
class Father {
    constructor(name, age) {
        this.name = name;
        this.age = age;
    }
    
    _sayHi() {
        return `Hi ${this.name}`
    }
    
    _getAge() {
        return this.age;
    }
}

// 子类通过关键字 extends 实现对父类的继承
class Child extends Father {
    constructor(name, age, school) {
        super();
        this.name = name;
        this.age = age;
        this.school = school;
    }
    
    _getSchool() {
        return this.school;
    }
}

let childIns = new Child('child', 18, 'RUC');
childIns.name/age/school;       // 'child', 18, 'RUC'
childIns._getSchool();          // 'RUC'
childIns._sayHi();              // 'Hi child'
childIns._getAge();             // 18

let fatherIns = new Father('father', 50);
fatherIns._sayHi();             // 'Hi father'
fatherIns._getAge();            // 50
```

ES6 的继承语法更符合程序员以往接触的继承方式，并且 ES6 继承依然拥有 ES5 继承的特性：

```js
// Father 和 Father 实例的关系
fatherIns.constructor === Father
fatherIns.__proto__ === Father.prototype
fatherIns.__proto__.constructor === Father

// Child 和 Child 实例的关系
childIns.constructor === Child
childIns.__proto__ === Child.prototype
childIns.__proto__.constructor === Child

// Child 和 Father 的关系
Child.prototype.__proto__ === Father.prototype
```

此外，es6 继承，还有另一条继承链
```js
Child.__proto__ === Father
```

![ES6 inherit](https://attachments.tower.im/tower/0f82d45ff5424ccf9fc20127f03f557e?filename=inheritComp.png)

**两条继承链**
+ 原型链，子类原型的 `__proto__` 属性指向父类原型
+ 类链，子类的 `__proto__` 属性直接指向了父类

```js
Child.prototype.__proto__ === Father.prototype
Child.__proto__ === Father
```
```js
// ES6 继承语法
class Child extends Father {}

// 等同于
Object.setPrototypeOf(Child.prototype, Father.prototype);
Object.setPrototypeOf(Child, Father);
```

Object.setPrototypeOf 方法的作用与 `__proto__` 相同，用来设置一个对象的 prototype 对象
```js
// setPrototypeOf 方法实现
Object.setPrototypeOf = function (obj, proto) {
    obj.__proto__ = proto;
    return obj;
}
```

### 三类特殊的继承
+ 继承 Object 类

```js
class A extends Object {}

// A 是构造函数 Object 的复制，A 的实例就是 Object 的实例
A.__proto__ === Object;                      // true
A.prototype.__proto__ === Object.prototype;  // true
```

+ 不继承：

```js
class A {}

// A 是普通函数，直接继承 Function.prototype
A.__proto__ === Function.prototype;          // true
// new A() 返回一个空对象, 所以 A.prototype.__proto__ 指向构造函数 Object 的 prototype 属性
A.prototype.__proto__ === Object.prototype;  // true
```

+ 继承 null：

```js
class A extends null {}

A.__proto__ === Function.prototype;          // true
A.prototype.__proto__ === undefined;         // true
```

### 补充1：Function & Object 的关系

+ Function 和 Object 与其各自的原型的关系

![origin](https://attachments.tower.im/tower/2fca4209d654444fb020cd31fc21a389?filename=of1.png)

+ Object 和 Function 都是构造函数，而所有的构造函数的都是 Function 的实例对象，因此 Object 和 Function 都是 Function 的实例对象

![Function.prototype](https://attachments.tower.im/tower/cc25fb9d30194547b44a3f3fab61634a?filename=of2.png)

+ Function.prototype 是 Object 的实例对象

![Object.prototype](https://attachments.tower.im/tower/321031acdc604c9c8219abf17e37525b?filename=of4.png)

### 补充2：instanceof 运算符
```js
Child instanceof Function;   // true
childIns instanceof Child;   // true
childIns instanceof Father;  // true
childIns instanceof Object;  // true

// ???
Object instanceof Function;
Function instanceof Object;
```

instanceof 运算符的实现

```js
// instance instanceof Class
function instance_of(instance, Class) {
    while (true) { 
        if (instance.__proto__ === null )
            return false;
        else if (instance.__proto__ === Class.prototype) 
            return true;

        instance = instance.__proto__; 
            
    }
}
```

