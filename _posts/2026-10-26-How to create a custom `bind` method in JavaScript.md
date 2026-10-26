---
layout: post
title: "Create a custom `bind` method in JavaScript"
subtitle: "How to create a custom `bind` method in JavaScript"
date: 2026-10-26 18:00
author: "Jeff"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
- Javascript
- bind
---

# Custom `bind` method code
``` js
Function.prototype.myBind = function (context, ...args) {
  const fn = this;
  if (new.target) {
    return new fn(...args);
  }
  return function (...newArgs) {
    return fn.apply(context, [...args, ...newArgs]);
  };
};

function fn(a, b, c, d, e) {
  console.log(this);
  // Expected output: 1
  console.log(a, b, c, d, e);
  // Expected output: 2 3 4 5 6
}

const newFn = fn.myBind(1, 2, 3);

newFn(4, 5, 6);
```

# Related Concepts
In JavaScript, the `bind` method can be used to specify what `this` refers to.
By default, `this` points to the caller; in the case of a top-level method, `this` points to `window`.

```js
window.x = 5
const module = {
  x: 42,
  getX() {
    let x = 80;
    return this.x;
  },
};

const unboundGetX = module.getX;
console.log(unboundGetX());
// Expected output: 5

console.log(module.getX());
// Expected output: 42

const boundGetX = unboundGetX.bind(module);
console.log(boundGetX());
// Expected output: 42

```