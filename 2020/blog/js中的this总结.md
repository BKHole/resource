# js中的this总结

1. 默认this指向global 对象

```
console.log(this === window); // true
```

2.在一个function内，没有使用strict模式， 指向global

```
function f() {
  return this;
}

console.log(f() === window); // true
```
3.在一个function内，使用了strict模式，this是undefined
```
'use strict';

function f() {
  return this;
}

console.log(f()); // undefined
```
4.在箭头函数内，this是上下文
```
const f = () => this;

console.log(f() === window); // true

const obj = {
  foo: function() {
    const baz = () => this;
    return baz();
  },
  bar: () => this
};

console.log(obj.foo()); // { foo, bar }
console.log(obj.bar() === window); // true
```
5.对象内的方法，this指向该方法被调用的对象
```
const obj = {
  f: function() {
    return this;
  }
};

const myObj = Object.create(obj);
myObj.foo = 1;

console.log(myObj.f()); // { foo: 1 }
```
6.在构造函数内，this指向被新构造的对象
```
class C {
  constructor() {
    this.x = 10;
  }
}

const obj = new C();
console.log(obj.x); // 10
```
7.在事件处理程序中，this指向被监听的元素
```
const el = document.getElementById('my-el');

el.addEventListener('click', function() {
  console.log(this === el);
})// true});
```

## Binding

使用function.prototype.bind()，可以从现有函数返回一个新函数，其中它永久地绑定到bind()的第一个参数。

```
function f() {
  return this.foo;
}

var x = f.bind({foo: 'hello'});
console.log(x()); // 'hello'
```
更简便的，可以直接使用

Function.prototype.call()

Function.prototype.apply()

this将会被指向第一个参数
```
function f() {
  return this.foo;
}

console.log(f.call({foo: 'hi'})); // 'hi'
```
