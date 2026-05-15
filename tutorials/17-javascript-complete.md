# 第17章：JavaScript 从零到高阶

## 本章学习目标

本章将从零开始学习 JavaScript，涵盖 ES6+ 全部核心语法，每个知识点包含完整示例和算法证明。

**学习路径：**

1. 执行模型与事件循环
2. 类型系统与强制转换
3. ES6+ 核心特性
4. 函数深入（闭包/this/柯里化）
5. 原型链与继承
6. Promise 与异步编程
7. 模块化
8. DOM API
9. Fetch 与网络请求
10. 设计模式

---

## 1. 执行模型

### 1.1 调用栈（Call Stack）

JavaScript 是单线程语言，所有代码在**调用栈**中执行：

```javascript
function multiply(a, b) {
    return a * b;
}

function square(n) {
    return multiply(n, n);
}

function main() {
    const result = square(5);
    console.log(result);
}

main();
```text

**调用栈变化：**

```

步骤 1: main() 入栈
栈: [main]

步骤 2: main() 调用 square(5), square 入栈
栈: [main, square]

步骤 3: square(5) 调用 multiply(5,5), multiply 入栈
栈: [main, square, multiply]

步骤 4: multiply 返回结果, multiply 出栈
栈: [main, square]

步骤 5: square 返回结果, square 出栈
栈: [main]

步骤 6: main 执行完毕, main 出栈
栈: []

```text

**栈溢出（Stack Overflow）：**

```javascript
function infinite() {
    return infinite();
}
infinite(); // RangeError: Maximum call stack size exceeded
```

### 1.2 事件循环（Event Loop）

```javascript
console.log('1');  // 同步代码

setTimeout(() => {
    console.log('2');  // 宏任务
}, 0);

Promise.resolve().then(() => {
    console.log('3');  // 微任务
});

console.log('4');  // 同步代码

// 输出顺序: 1 → 4 → 3 → 2
```text

**执行优先级证明：**

```

同步代码 → 微任务队列 → 宏任务队列

1. 执行所有同步代码（调用栈清空）
2. 检查微任务队列，全部执行
3. 取一个宏任务执行
4. 回到步骤 2

因此: 微任务(3) 总是优先于 宏任务(2)

```text

**微任务 vs 宏任务：**

| 类型 |  API | 优先级 |
| ------ | ------ | -------- |
| 同步 | 普通代码 | 最高 |
| 微任务 | Promise.then/catch/finally, MutationObserver, queueMicrotask | 中 |
| 宏任务 | setTimeout, setInterval, I/O, UI 渲染 | 低 |

---

## 2. 类型系统

### 2.1 7 种原始类型 + Object

```javascript
// 原始类型（不可变）
typeof undefined     // "undefined"
typeof null          // "object"  (语言 bug)
typeof true          // "boolean"
typeof 42            // "number"
typeof "hello"       // "string"
typeof Symbol()      // "symbol"
typeof 9007199254740991n  // "bigint"

// 引用类型
typeof {}            // "object"
typeof []            // "object"
typeof function(){}  // "function"
```

### 2.2 类型强制转换

```javascript
// 字符串拼接优先
'5' + 3      // '53'  (数字转字符串)
'5' - 3      // 2    (字符串转数字)
'5' * '2'    // 10

// == 的强制转换规则
null == undefined   // true
'1' == 1            // true  (字符串转数字)
true == 1           // true  (布尔转数字)
[] == false         // true  (空数组转0)

// 始终使用 === 代替 ==
```text

**类型转换规则表：**

| 值 | 转 Number | 转 String | 转 Boolean |
| ---- | ----------- | ----------- | ------------ |
| `undefined` | `NaN` | `"undefined"` | `false` |
| `null` | `0` | `"null"` | `false` |
| `true` | `1` | `"true"` | `true` |
| `false` | `0` | `"false"` | `false` |
| `""` | `0` | `""` | `false` |
| `"123"` | `123` | `"123"` | `true` |
| `"abc"` | `NaN` | `"abc"` | `true` |
| `0` | `0` | `"0"` | `false` |
| `[]` | `0` | `""` | `true` |
| `[1]` | `1` | `"1"` | `true` |
| `{}` | `NaN` | `"[object Object]"` | `true` |

---

## 3. ES6+ 核心特性

### 3.1 let / const

```javascript
// var: 函数作用域
if (true) {
    var x = 1;
}
console.log(x); // 1 (var 没有块级作用域)

// let/const: 块级作用域
if (true) {
    let y = 2;
    const z = 3;
}
console.log(y); // ReferenceError: y is not defined

// const: 必须初始化，不能重新赋值
const PI = 3.14;
PI = 3; // TypeError: Assignment to constant variable

// const 对象属性可以修改
const obj = { a: 1 };
obj.a = 2;  // OK
obj = {};   // TypeError
```

**TDZ（Temporal Dead Zone）证明：**

```javascript
console.log(a); // undefined (var 提升)
var a = 1;

console.log(b); // ReferenceError (let 的 TDZ)
let b = 2;

// let 存在提升，但处于"暂时性死区"直到初始化
```text

### 3.2 箭头函数

```javascript
// 传统函数
function add(a, b) {
    return a + b;
}

// 箭头函数
const add = (a, b) => a + b;

// 一个参数可省略括号
const square = x => x * x;

// 无参数需保留括号
const hello = () => 'hello';

// 返回对象需加括号
const getUser = () => ({ id: 1, name: 'Alice' });
```

**箭头函数没有自己的 this：**

```javascript
const obj = {
    name: 'obj',
    traditional: function() {
        console.log(this.name); // 'obj' (调用时的 this)
    },
    arrow: () => {
        console.log(this.name); // undefined (定义时的 this)
    }
};

obj.traditional(); // obj
obj.arrow();       // undefined (箭头函数的 this 来自外层作用域)
```text

### 3.3 解构赋值

```javascript
// 数组解构
const [a, b, ...rest] = [1, 2, 3, 4, 5];
// a = 1, b = 2, rest = [3, 4, 5]

// 对象解构
const user = { id: 1, name: 'Alice', role: 'admin' };
const { name, role, age = 18 } = user;
// name = 'Alice', role = 'admin', age = 18 (默认值)

// 重命名
const { name: userName } = user;

// 嵌套解构
const data = { user: { address: { city: 'Beijing' } } };
const { user: { address: { city } } } = data;
```

### 3.4 展开运算符

```javascript
// 数组展开
const arr1 = [1, 2, 3];
const arr2 = [...arr1, 4, 5];  // [1, 2, 3, 4, 5]
const max = Math.max(...arr1); // 3

// 对象展开
const base = { a: 1, b: 2 };
const extended = { ...base, c: 3 };  // { a: 1, b: 2, c: 3 }

// 浅拷贝
const copy = { ...base };
const arrCopy = [...arr1];
```text

### 3.5 模板字符串

```javascript
const name = 'World';
const greeting = `Hello, ${name}!`;  // "Hello, World!"

// 多行字符串
const html = `
    <div>
        <h1>${title}</h1>
        <p>${content}</p>
    </div>
`;

// 带表达式
const price = 100;
const msg = `总价: ¥${price * 1.13}`;  // "总价: ¥113"
```

---

## 4. 函数深入

### 4.1 闭包（Closure）

**定义：** 函数 + 其词法环境的组合。内部函数可以访问外部函数的变量，即使外部函数已经返回。

```javascript
function createCounter() {
    let count = 0;  // 私有变量
    
    return {
        increment: () => ++count,
        decrement: () => --count,
        getCount: () => count,
    };
}

const counter = createCounter();
counter.increment();  // 1
counter.increment();  // 2
counter.decrement();  // 1
console.log(counter.getCount()); // 1
```text

**闭包内存模型：**

```

createCounter() 执行时:
  栈: count = 0
  返回对象 { increment, decrement, getCount }
        ↓（这些函数保持对 count 的引用）
  即使 createCounter 执行完毕，count 不会被 GC 回收
  因为闭包仍然引用着它

```text

**实际应用：**

```javascript
// 防抖（debounce）
function debounce(fn, delay) {
    let timer = null;
    return function(...args) {
        clearTimeout(timer);
        timer = setTimeout(() => fn.apply(this, args), delay);
    };
}

// 节流（throttle）
function throttle(fn, limit) {
    let inThrottle = false;
    return function(...args) {
        if (!inThrottle) {
            fn.apply(this, args);
            inThrottle = true;
            setTimeout(() => inThrottle = false, limit);
        }
    };
}
```

### 4.2 this 的 4 条规则

```javascript
// 规则 1: 默认绑定 → 全局对象（非严格模式）/ undefined（严格模式）
function show() {
    console.log(this); // 非严格模式: window/global
}

// 规则 2: 隐式绑定 → 调用方法的对象
const obj = { name: 'obj', show() { console.log(this.name); } };
obj.show(); // "obj"

// 规则 3: 显式绑定 → call/apply/bind 指定的对象
function greet() { console.log(`Hello, ${this.name}`); }
greet.call({ name: 'Alice' }); // "Hello, Alice"

// 规则 4: new 绑定 → 新创建的对象
function Person(name) {
    this.name = name;
}
const p = new Person('Bob');
console.log(p.name); // "Bob"
```text

**优先级：** new > 显式 > 隐式 > 默认

### 4.3 call / apply / bind

```javascript
// call: 立即执行，参数逐个传入
func.call(thisArg, arg1, arg2, arg3);

// apply: 立即执行，参数数组传入
func.apply(thisArg, [arg1, arg2, arg3]);

// bind: 返回新函数，不执行
const bound = func.bind(thisArg, arg1, arg2);
bound(); // 执行

// 示例
const numbers = [5, 2, 8, 1, 9];
Math.max.apply(null, numbers);  // 9 (apply 展开数组)
Math.max(...numbers);           // 9 (ES6 展开更简洁)
```

### 4.4 柯里化（Currying）

```javascript
// 将多参数函数转换为一系列单参数函数
function curry(fn) {
    return function curried(...args) {
        if (args.length >= fn.length) {
            return fn.apply(this, args);
        }
        return function(...args2) {
            return curried.apply(this, args.concat(args2));
        };
    };
}

// 使用
function add(a, b, c) {
    return a + b + c;
}

const curriedAdd = curry(add);
curriedAdd(1)(2)(3);  // 6
curriedAdd(1, 2)(3);  // 6
```text

---

## 5. 原型链

### 5.1 基础概念

```javascript
// 每个对象都有一个 __proto__（内部 [[Prototype]]）
// 函数对象还有 prototype 属性

function Animal(name) {
    this.name = name;
}

Animal.prototype.speak = function() {
    return `${this.name} makes a sound`;
};

const dog = new Animal('Dog');
dog.speak(); // "Dog makes a sound"
```

**原型链查找算法：**

```javascript
// 访问 obj.property 时的查找步骤
function getProperty(obj, prop) {
    let current = obj;
    while (current !== null) {
        if (current.hasOwnProperty(prop)) {
            return current[prop];
        }
        current = Object.getPrototypeOf(current);  // 沿着 __proto__ 链上移
    }
    return undefined;
}

// 原型链:
// dog → Animal.prototype → Object.prototype → null
//           ↑                   ↑
//       Animal.speak      Object.toString
//                         Object.hasOwnProperty
```text

### 5.2 原型继承

```javascript
function Mammal(name) {
    this.name = name;
}

Mammal.prototype.breathe = function() {
    return `${this.name} is breathing`;
};

function Cat(name) {
    Mammal.call(this, name);  // 调用父类构造器
}

// 设置原型链: Cat.prototype → Mammal.prototype
Cat.prototype = Object.create(Mammal.prototype);
Cat.prototype.constructor = Cat;

Cat.prototype.meow = function() {
    return `${this.name} says meow`;
};

const kitty = new Cat('Kitty');
kitty.breathe(); // "Kitty is breathing"
kitty.meow();    // "Kitty says meow"

// 原型链: kitty → Cat.prototype → Mammal.prototype → Object.prototype → null
```

---

## 6. 类和面向对象

### 6.1 ES6 class

```javascript
class Animal {
    constructor(name) {
        this.name = name;
    }
    
    speak() {
        return `${this.name} makes a sound`;
    }
    
    // 静态方法
    static isAnimal(obj) {
        return obj instanceof Animal;
    }
    
    // getter/setter
    get description() {
        return `Animal: ${this.name}`;
    }
}

const a = new Animal('Dog');
console.log(Animal.isAnimal(a)); // true
```text

### 6.2 extends / super

```javascript
class Dog extends Animal {
    constructor(name, breed) {
        super(name);  // 调用父类构造器（必须在使用 this 之前）
        this.breed = breed;
    }
    
    // 重写方法
    speak() {
        return `${super.speak()} and barks`;  // 调用父类方法
    }
    
    static isDog(obj) {
        return obj instanceof Dog;
    }
}

const dog = new Dog('Rex', 'Husky');
dog.speak(); // "Rex makes a sound and barks"
```

### 6.3 私有字段

```javascript
class BankAccount {
    #balance = 0;  // 私有字段（ES2022）
    
    constructor(owner) {
        this.owner = owner;
    }
    
    deposit(amount) {
        if (amount > 0) {
            this.#balance += amount;
        }
    }
    
    getBalance() {
        return this.#balance;
    }
}

const account = new BankAccount('Alice');
account.deposit(100);
account.#balance;  // SyntaxError: Private field
```text

---

## 7. Promise 与异步编程

### 7.1 Promise 三种状态

```javascript
// pending → fulfilled (resolved)
// pending → rejected

const promise = new Promise((resolve, reject) => {
    // 异步操作
    setTimeout(() => {
        const success = true;
        if (success) {
            resolve('数据加载成功');
        } else {
            reject(new Error('加载失败'));
        }
    }, 1000);
});

promise
    .then(data => console.log(data))
    .catch(err => console.error(err))
    .finally(() => console.log('完成'));
```

### 7.2 Promise 链式调用

```javascript
fetch('/api/user')
    .then(res => res.json())
    .then(user => fetch(`/api/orders/${user.id}`))
    .then(res => res.json())
    .then(orders => console.log(orders))
    .catch(err => console.error('请求失败:', err));
```text

### 7.3 Promise 静态方法

```javascript
// Promise.all: 全部成功才成功，一个失败就失败
const p1 = fetch('/api/events');
const p2 = fetch('/api/users');
Promise.all([p1, p2])
    .then(([events, users]) => {
        console.log('全部完成');
    });

// Promise.allSettled: 等待全部完成（无论成功失败）
Promise.allSettled([p1, p2])
    .then(results => {
        results.forEach(r => console.log(r.status, r.value || r.reason));
    });

// Promise.race: 返回第一个完成的
Promise.race([
    fetch('/api/events'),
    new Promise((_, reject) => 
        setTimeout(() => reject(new Error('超时')), 5000)
    )
]);

// Promise.any: 返回第一个成功的
Promise.any([p1, p2])
    .then(firstSuccess => console.log(firstSuccess));
```

### 7.4 async/await

```javascript
// async/await = Promise 的语法糖
async function loadUserData(userId) {
    try {
        const response = await fetch(`/api/users/${userId}`);
        const user = await response.json();
        const orders = await fetch(`/api/orders/${user.id}`)
            .then(r => r.json());
        
        return { user, orders };
    } catch (error) {
        console.error('加载失败:', error);
        throw error;
    }
}

// 并行等待
async function loadMultiple() {
    const [events, users] = await Promise.all([
        fetch('/api/events').then(r => r.json()),
        fetch('/api/users').then(r => r.json()),
    ]);
    return { events, users };
}
```text

---

## 7.5 ES6+ 数据结构与方法

### 7.5.1 Array 核心方法（日常开发最常用）

```javascript
const numbers = [1, 2, 3, 4, 5];

// map — 映射每个元素为新值（返回等长新数组）
const doubled = numbers.map(n => n * 2);        // [2, 4, 6, 8, 10]
const users = [{name:'小明'}, {name:'小红'}];
const names = users.map(u => u.name);            // ['小明', '小红']

// filter — 过滤元素（返回满足条件的子集）
const evens = numbers.filter(n => n % 2 === 0); // [2, 4]
const adults = users.filter(u => u.age >= 18);

// reduce — 归约为单个值（最灵活）
const sum = numbers.reduce((acc, n) => acc + n, 0);  // 15
// 高级用法：分组
const items = [
  { cat: 'fruit', name: '苹果' },
  { cat: 'fruit', name: '香蕉' },
  { cat: 'tool', name: '锤子' },
];
const grouped = items.reduce((acc, item) => {
  (acc[item.cat] = acc[item.cat] || []).push(item.name);
  return acc;
}, {});  // { fruit: ['苹果', '香蕉'], tool: ['锤子'] }

// find — 查找第一个匹配元素
const found = numbers.find(n => n > 3);          // 4（不是数组！）
// some — 是否有任意元素匹配
const hasEven = numbers.some(n => n % 2 === 0);  // true
// every — 是否全部元素匹配
const allPositive = numbers.every(n => n > 0);   // true

// flat/flatMap — 展平嵌套数组
[1, [2, 3], [4, [5]]].flat(1);   // [1, 2, 3, 4, [5]]
[1, [2, 3], [4, [5]]].flat(2);   // [1, 2, 3, 4, 5]
['hello', 'world'].flatMap(s => s.split(''));  // ['h','e','l','l','o','w','o','r','l','d']
```

> **实际项目用法**：Vue 中经常用 `map` 转换 API 返回的数据，用 `filter` 搜索和筛选列表，用 `find` 查找单条记录。

### 7.5.2 Map 和 Set

```javascript
// Map — 键可以是任意类型（比 Object 更灵活）
const userMap = new Map();
userMap.set(1, { name: '小明' });        // 数字键
userMap.set('key', 'value');              // 字符串键
const obj = { id: 1 };
userMap.set(obj, '对象键');               // 对象键 ✅（Object 不行）

// Map 常用方法
userMap.get(1);          // 取值
userMap.has(1);          // 检查存在
userMap.delete(1);       // 删除
userMap.size;            // 大小（Object 没有直接的方法）

// Map 的遍历（有序！按插入顺序）
for (const [key, value] of userMap) { /* ... */ }
[...userMap.keys()];     // 所有键
[...userMap.values()];   // 所有值

// Set — 唯一值集合
const tags = new Set(['vue', 'react', 'vue']);
tags.size;               // 2（自动去重）

// Set 常用操作
tags.add('angular');
tags.has('vue');         // true
tags.delete('react');
tags.clear();            // 清空

// Set 转数组
[...tags];
Array.from(tags);

// WeakMap / WeakSet — 弱引用版本（不阻止 GC）
// 适合存储 DOM 节点的元数据
const wm = new WeakMap();
const element = document.querySelector('.card');
wm.set(element, { clickCount: 0 });  // element 被移除后，元数据自动 GC
```text

### 7.5.3 Object 静态方法

```javascript
const user = { name: '小明', age: 25, role: 'admin' };

Object.keys(user);    // ['name', 'age', 'role']
Object.values(user);  // ['小明', 25, 'admin']
Object.entries(user); // [['name', '小明'], ['age', 25], ['role', 'admin']]

// 遍历对象（现代方式）
for (const [key, value] of Object.entries(user)) {
  console.log(`${key}: ${value}`);
}

// 合并对象
const defaults = { theme: 'light', lang: 'zh' };
const config = Object.assign({}, defaults, user);  // 合并到新对象
// 或使用展开运算符（更常用）
const merged = { ...defaults, ...user };

// 冻结对象（防止修改）
const CONSTANTS = Object.freeze({ MAX_RETRY: 3, TIMEOUT: 5000 });
// CONSTANTS.MAX_RETRY = 5;  // ❌ 严格模式下会报错
```

### 7.5.4 JSON 处理

```javascript
// 序列化（对象 → 字符串）
const data = { name: '小明', age: 25, role: 'admin' };
JSON.stringify(data);
// '{"name":"小明","age":25,"role":"admin"}'

// 格式化输出
JSON.stringify(data, null, 2);
// {
//   "name": "小明",
//   "age": 25,
//   "role": "admin"
// }

// 反序列化（字符串 → 对象）
const json = '{"name":"小明","age":25}';
const parsed = JSON.parse(json);   // { name: '小明', age: 25 }

// 安全解析（防止 JSON.parse 抛异常）
function safeJSONParse(str) {
  try {
    return JSON.parse(str);
  } catch {
    return null;
  }
}
```text

### 7.5.5 classList 和 dataset（DOM 操作必备）

```javascript
const card = document.querySelector('.card');

// classList — 更安全的类名操作
card.classList.add('active');         // 添加类
card.classList.remove('active');      // 移除类
card.classList.toggle('expanded');    // 切换类
card.classList.contains('active');    // 检查是否存在

// 批量添加/移除
card.classList.add('highlight', 'selected');

// dataset — 访问 data-* 属性
// <div data-user-id="42" data-role="admin"></div>
const div = document.querySelector('[data-user-id]');
div.dataset.userId;     // '42'（驼峰命名）
div.dataset.role;       // 'admin'
div.dataset.status = 'active';  // 设置 data-status="active"
```

---

## 8. 模块化

### 8.1 ESM (ES Modules)

```javascript
// math.js
export const PI = 3.14159;
export function add(a, b) { return a + b; }
export default class Calculator { ... }

// app.js
import Calculator, { PI, add as sum } from './math.js';
```text

### 8.2 CommonJS (Node.js)

```javascript
// math.js
const PI = 3.14159;
function add(a, b) { return a + b; }
module.exports = { PI, add };

// app.js
const { PI, add } = require('./math');
```

### 8.3 动态 import

```javascript
// 按需加载
button.addEventListener('click', async () => {
    const module = await import('./heavy-module.js');
    module.default.init();
});
```text

---

## 9. DOM API

### 9.1 选择器

```javascript
// 选择单个元素
const el = document.querySelector('.my-class');
const byId = document.getElementById('my-id');

// 选择多个元素
const list = document.querySelectorAll('.item');  // NodeList
const byClass = document.getElementsByClassName('item');  // HTMLCollection
```

### 9.2 DOM 操作

```javascript
// 创建
const div = document.createElement('div');
div.textContent = 'Hello';
div.className = 'card';

// 插入
parent.appendChild(div);
parent.prepend(div);
parent.insertBefore(div, referenceNode);
parent.insertAdjacentHTML('beforeend', '<span>new</span>');

// 删除
div.remove();
parent.removeChild(div);
```text

### 9.3 事件三阶段

```javascript
// 1. 捕获阶段 (window → document → ... → target)
// 2. 目标阶段
// 3. 冒泡阶段 (target → ... → document → window)

parent.addEventListener('click', () => {
    console.log('parent capture');
}, true);  // capture=true

child.addEventListener('click', (e) => {
    console.log('child bubble');
    e.stopPropagation();  // 阻止冒泡
});  // capture=false (默认)
```

**事件委托：**

```javascript
// 不需要给每个 li 绑定事件
document.querySelector('ul').addEventListener('click', (e) => {
    const li = e.target.closest('li');
    if (li) {
        console.log('点击了:', li.textContent);
    }
});
```text

---

## 10. Fetch API

```javascript
// 基本用法
const response = await fetch('/api/events', {
    method: 'GET',
    headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${token}`,
    },
});

if (!response.ok) {
    throw new Error(`HTTP ${response.status}`);
}

const data = await response.json();

// POST 请求
const createResponse = await fetch('/api/orders', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ event_id: 1, quantity: 2 }),
});

// 请求拦截器封装
async function apiRequest(url, options = {}) {
    const config = {
        headers: {
            'Content-Type': 'application/json',
            ...options.headers,
        },
        ...options,
    };
    
    const response = await fetch(url, config);
    if (!response.ok) {
        const error = await response.json().catch(() => ({}));
        throw new Error(error.detail || `HTTP ${response.status}`);
    }
    return response.json();
}
```

---

## 11. 设计模式

### 11.1 单例模式

```javascript
class Store {
    static #instance;
    
    constructor() {
        if (Store.#instance) {
            return Store.#instance;
        }
        this.state = {};
        Store.#instance = this;
    }
}

const s1 = new Store();
const s2 = new Store();
console.log(s1 === s2); // true
```text

### 11.2 观察者模式

```javascript
class EventBus {
    constructor() {
        this._handlers = {};
    }
    
    on(event, handler) {
        if (!this._handlers[event]) {
            this._handlers[event] = [];
        }
        this._handlers[event].push(handler);
    }
    
    emit(event, data) {
        const handlers = this._handlers[event] || [];
        handlers.forEach(handler => handler(data));
    }
    
    off(event, handler) {
        const handlers = this._handlers[event];
        if (handlers) {
            this._handlers[event] = handlers.filter(h => h !== handler);
        }
    }
}

const bus = new EventBus();
bus.on('order:created', (order) => console.log('新订单:', order.order_no));
```

### 11.3 发布-订阅模式

```javascript
// 与观察者模式的区别: 发布-订阅通过消息代理解耦
class PubSub {
    constructor() {
        this._channels = {};
    }
    
    subscribe(channel, callback) {
        if (!this._channels[channel]) {
            this._channels[channel] = [];
        }
        this._channels[channel].push(callback);
        return () => this.unsubscribe(channel, callback);
    }
    
    publish(channel, message) {
        (this._channels[channel] || []).forEach(cb => cb(message));
    }
    
    unsubscribe(channel, callback) {
        const cbs = this._channels[channel];
        if (cbs) {
            this._channels[channel] = cbs.filter(cb => cb !== callback);
        }
    }
}
```text

---

## 本章总结

| 知识点 | 掌握程度 |
| -------- | --------- |
| 事件循环和调用栈 | 理解执行模型 |
| 类型系统和强制转换 | 了解所有类型 |
| ES6+ 特性 | 熟练使用 let/const/箭头函数/解构/展开 |
| 闭包和 this | 理解四规则和闭包应用 |
| 原型链 | 理解查找算法 |
| Promise/async/await | 能写出正确的异步代码 |
| 模块化 | 了解 ESM 和 CommonJS |
| DOM 操作 | 能操作页面元素 |

---

## 安全防护

### XSS（跨站脚本攻击）防护

JavaScript 中 XSS 是最常见的安全漏洞——攻击者注入恶意脚本窃取用户数据。

#### 危险操作

```javascript
// ❌ 直接用 innerHTML 插入用户内容
document.getElementById('output').innerHTML = userInput

// ❌ 用 document.write
document.write(userInput)

// ❌ 用 eval 执行动态代码
eval(userInput)

// ❌ 用 dangerouslySetInnerHTML（React）
<div dangerouslySetInnerHTML={{__html: userInput}} />
```

#### 安全做法

```javascript
// ✅ 用 textContent 替代 innerHTML
document.getElementById('output').textContent = userInput

// ✅ 用 createTextNode
document.getElementById('output').appendChild(document.createTextNode(userInput))

// ✅ 需要对 HTML 编码
function escapeHtml(str) {
    const div = document.createElement('div')
    div.textContent = str
    return div.innerHTML
}
```text

### localStorage 安全风险

本系统将 JWT 存储在 localStorage 中。这意味着任何 XSS 攻击都能窃取令牌。缓解措施：

1. **CSP 限制脚本来源**（`script-src 'self'`）
2. **令牌时效短**（access_token 30 分钟过期）
3. **令牌轮换**（每次刷新生成新令牌，旧令牌失效）

### Fetch API 安全

```javascript
// ✅ 安全请求：通过拦截器自动注入 Authorization 头
const response = await fetch('/api/v1/orders', {
    headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${token}`
    }
})

// ❌ 避免：将令牌放在 URL 查询参数中（会被记录到日志）
// fetch('/api/v1/orders?token=' + token)
```

---

## 常见问题与排错

| 问题 | 原因 | 解决 |
| ------ | ------ | ------ |
| `this` 为 undefined（在回调中） | 普通函数的 `this` 由调用者决定 | 使用箭头函数或在构造函数中 `.bind(this)` |
| `0.1 + 0.2 !== 0.3` | IEEE 754 浮点精度 | 涉及金额必须用 `Decimal` 或整数（分）计算 |
| 异步代码未按预期顺序执行 | 忘记 `await` | 检查 Promise 是否漏掉了 `await` 关键字 |
| `const` 声明的对象可以修改属性 | `const` 只保证引用不变 | 用 `Object.freeze()` 实现深度不可变 |
| `==` 意外为 true | 类型强制转换 | 始终使用 `===` 严格相等 |
| `array.sort()` 排序错误 | 默认按字符串排序 | `array.sort((a, b) => a - b)` |
| forEach 中 `await` 不生效 | forEach 不等待 Promise | 使用 `for...of` 或 `Promise.all` |
| 箭头函数不能用作构造函数 | 箭头函数没有 `[[Construct]]` | 方法定义用普通函数 |
| 闭包中循环变量始终是最后一个值 | var 的函数作用域 | 使用 `let`（块级作用域）或用 IIFE 捕获 |
| `typeof null === 'object'` | JavaScript 的历史遗留 bug | 用 `=== null` 判断空值 |

---

## 本章练习

1. **手写 Promise.all**：实现一个函数 `myPromiseAll(promises)`，行为与 `Promise.all` 完全相同
2. **实现防抖与节流**：实现 `debounce(fn, delay)` 和 `throttle(fn, interval)`，并说明各自适用场景
3. **深拷贝函数**：实现 `deepClone(obj)`，能处理对象、数组、嵌套结构、循环引用
4. **事件总线**：对照本项目 `frontend/src/utils/api.js`，实现一个 Axios 封装，包含请求拦截器自动注入 token 和响应拦截器处理 401
5. **WebSocket 客户端封装**：对照 `frontend/src/utils/websocket.js`，实现一个带自动重连的 WebSocket 客户端
6. **手写 reduce**：实现 `Array.prototype.myReduce(callback, initialValue)`，处理空数组和无初始值的情况

---

## 下一章预告

第 18 章将深入学习 Vue 3 框架，包括组合式 API 的完整用法、Vue Router 路由配置、Pinia 状态管理和组件通信模式，实际结合本项目的 Vue 组件代码进行讲解。
