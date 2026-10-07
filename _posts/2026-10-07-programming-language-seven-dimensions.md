---
layout: post
title: "编程语言的七大基础维度（以 TypeScript 为参照）"
date: 2026-10-07
categories: [学习笔记]
tags: [TypeScript, JavaScript, 编程基础]
excerpt: "从数据与类型、变量、运算符、控制流程、函数、代码组织到运行环境，用七个维度建立编程语言基础知识地图。"
---
> 定位：语言基础知识的完整地图。无论学什么语言，都可以套这个骨架自查。
> 配套：《Cocos-TypeScript-八股文》《TS核心突破-手写练习清单》

<details markdown="1">
<summary>文章目录 · 七个维度与自查总表</summary>

- [维度一：数据与类型](#data-types)
- [维度二：变量与常量](#variables)
- [维度三：运算符与表达式](#operators)
- [维度四：控制流程](#control-flow)
- [维度五：函数（JS/TS 的灵魂）](#functions)
- [维度六：代码组织](#organization)
- [维度七：运行环境常识](#runtime)
- [七维度自查总表](#checklist)

</details>

---

## 维度一：数据与类型 {#data-types}

数据是程序加工的对象，类型是数据的"规格说明书"。

### 1.1 基本类型（Primitive Types）

值直接存储、按值拷贝、不可变。

| 类型        | 示例                             | 说明                                       |
| --------- | ------------------------------ | ---------------------------------------- |
| number    | `42`、`3.14`、`NaN`、`Infinity`   | JS/TS 不区分 int 和 float，全部是 IEEE 754 双精度浮点 |
| string    | `'abc'`、`"abc"`、`` `模板${x}` `` | 不可变，模板字符串支持插值和多行                         |
| boolean   | `true` / `false`               |                                          |
| null      | `null`                         | 显式的"空"                                   |
| undefined | `undefined`                    | "未赋值"：声明未初始化、函数无 return、对象没有的属性          |
| symbol    | `Symbol('id')`                 | 唯一值，常用作对象私有键                             |
| bigint    | `100n`                         | 任意精度整数，解决 number 超过 2^53 精度丢失的问题         |

TS 在 JS 之上补充的特殊类型：

- `any`：放弃类型检查
- `unknown`：安全的 any，使用前必须收窄
- `void`：函数无返回值
- `never`：永不存在的值（抛错函数、死循环、穷尽检查）
- 字面量类型：`'up'`、`200` 这种"值本身就是类型"

### 1.2 复合类型（引用类型）

存的是引用（地址），按引用传递，可变。

- **数组**：`number[]`、`Array<string>`，还有定长定类型的**元组** `[string, number]`
- **对象**：`{ name: string, age: number }`，索引签名 `{ [key: string]: number }`
- **函数**：在 JS 里函数也是对象，可以赋值、传参、返回
- **Map / Set**：真正的哈希表和集合（区别于普通对象）
- **Date、RegExp、Error** 等内置对象

### 1.3 类型的规则（一门语言的"类型性格"）

- **静态 vs 动态**：TS 编译期检查（静态），JS 运行时才暴露（动态）
- **强类型 vs 弱类型**：JS 是弱类型——`'1' + 1 === '11'`、`[] + [] === ''` 这种隐式转换是著名坑区；TS 用编译器把这些坑提前拦住
- **结构类型（Structural Typing）**：TS 的特色——不看名字看形状，"像鸭子就是鸭子"。两个 interface 名字不同但结构一致就互相兼容

### 1.4 类型相关的核心操作

- **类型注解**：`let x: number = 1`
- **类型推断**：`let x = 1` 自动推断为 number
- **类型断言**：`value as string`（编译期"我保证"，无运行时检查）
- **类型收窄**：typeof / instanceof / in / 可辨识联合 / 自定义守卫
- **联合与交叉**：`string | number`（或）、`A & B`（且）
- **隐式转换规则**：JS 的 falsy 值只有 6 个——`false`、`0`、`''`、`null`、`undefined`、`NaN`，其余全为 truthy（包括 `[]`、`{}`、`'0'`）⭐面试高频

---

## 维度二：变量与常量 {#variables}

变量是数据的"名字"，这一维度的核心是：**名字在哪有效、活多久、什么时候创建。**

### 2.1 声明方式

| 关键字 | 可重新赋值 | 作用域 | 提升（Hoisting） | 重复声明 |
|---|---|---|---|---|
| `var` | 可以 | 函数作用域 | 提升并初始化为 undefined | 允许 |
| `let` | 可以 | 块级作用域 `{}` | 提升但处于暂时性死区（TDZ），提前访问报错 | 不允许 |
| `const` | 不可以（但对象内容可变！） | 块级作用域 | 同 let | 不允许 |

实践：**默认 const，需要重赋值才 let，永远不用 var**。

经典坑：`const obj = { a: 1 }; obj.a = 2;` 完全合法——const 锁的是"引用不能换"，不是"内容不能改"。真正冻结内容用 `Object.freeze()` 或 TS 的 `readonly` / `as const`。

### 2.2 作用域（Scope）

- **全局作用域**：顶层声明，污染全局是万恶之源
- **模块作用域**：ES Module 中每个文件天然隔离，不挂 window
- **函数作用域**：函数体内
- **块级作用域**：`{}` 内（if、for、while 的块）

**作用域链**：查找变量时从内层往外层逐层找，直到全局；找不到报 ReferenceError。这是闭包的底层原理。

### 2.3 提升与暂时性死区

- `var` 声明提升到函数顶部且初始化为 undefined——`console.log(a); var a = 1;` 输出 undefined
- `let`/`const` 也提升，但进入 TDZ——提前访问直接抛错（更安全）
- 函数声明整体提升；函数表达式按变量规则提升

### 2.4 生命周期

- 局部变量：函数调用时创建，调用结束可被回收（被闭包捕获的除外）
- 全局变量：与程序同寿
- JS 的销毁由垃圾回收器自动管理（见维度七）

---

## 维度三：运算符与表达式 {#operators}

### 3.1 算术运算符

`+` `-` `*` `/` `%`（取余） `**`（幂） `++` `--`

坑点：`+` 是唯一重载的运算符——两边有字符串就是拼接；`0.1 + 0.2 !== 0.3`（浮点精度，金融计算要用整数分或 decimal 库）。

### 3.2 比较运算符

- `==` / `!=`：**宽松相等，会做隐式转换**——`'1' == 1` 为 true，`null == undefined` 为 true
- `===` / `!==`：**严格相等，不转换**，类型不同直接 false
- 实践：**永远用 ===**，唯一例外是 `x == null` 同时判断 null 和 undefined 的惯用法
- `NaN !== NaN`，判断 NaN 用 `Number.isNaN()`
- 对象比较的是**引用**：`{} === {}` 为 false

### 3.3 逻辑运算符

- `&&`：短路——左边 falsy 直接返回左边的值，否则返回右边的值
- `||`：短路——左边 truthy 返回左边，否则返回右边
- `!`：取反
- **返回值是"操作数本身"而不是布尔值**：`a && b()` 是"条件执行"的惯用写法，`const x = a || 默认值` 是兜底惯用法

### 3.4 空值处理（现代写法，高频）

- `??`（空值合并）：**只有 null/undefined 才取右边**——`0 ?? 100` 是 0，而 `0 || 100` 是 100。这是 `??` 和 `||` 的本质区别 ⭐
- `?.`（可选链）：`obj?.a?.b?.[0]`，链条上任一为 null/undefined 就短路返回 undefined，不抛错。Cocos 里访问可能销毁的节点必用

### 3.5 位运算

`&` `|` `^` `~` `<<` `>>` `>>>`——游戏开发常用于**标志位/掩码**（如物理碰撞的 Group/Mask）、性能敏感的数学运算。

### 3.6 展开与解构

- 展开：`[...arr]`、`{ ...obj }`（浅拷贝！）
- 解构：`const { hp, atk = 10 } = role`（带默认值）、`const [first, ...rest] = arr`
- 交换变量：`[a, b] = [b, a]`

### 3.7 其他

- 三元：`cond ? a : b`
- 逗号运算符、typeof、instanceof、in、delete、void
- 赋值复合：`+=`、`|=`、`??=`、`&&=`、`||=`

---

## 维度四：控制流程 {#control-flow}

### 4.1 分支

```ts
if (cond) { ... } else if (cond2) { ... } else { ... }

switch (value) {           // 用 === 严格比较
  case 'a': ...; break;    // 忘写 break 会贯穿（fall-through）
  default: ...;
}
```

进阶：多个 if 链考虑换成 **表驱动**（对象映射 + 查找）或**策略模式**；TS 的 switch 配合可辨识联合 + never 穷尽检查是类型安全的最强形态。

### 4.2 循环

| 写法 | 适用 |
|---|---|
| `for (let i = 0; i < n; i++)` | 需要索引、需要逆序、性能敏感（游戏 update 里首选） |
| `for (const item of arr)` | 遍历**值**（数组、Set、Map、字符串） |
| `for (const key in obj)` | 遍历**键**（对象），会带上原型链上的可枚举键，慎用 |
| `while` / `do...while` | 次数未知、条件驱动 |
| `arr.forEach / map / filter / reduce` | 函数式遍历，语义清晰 |

高频坑：`for...in` 遍历数组得到的是**字符串索引**；`forEach` 里 `return` 只是跳过本次，要用 `break` 的场景换普通 for 或 `some/every`。

### 4.3 跳转

- `break`：跳出当前循环/switch
- `continue`：跳过本次迭代
- `return`：结束函数
- 标签（label）：`outer: for...` 配合 `break outer` 跳出多层循环（少用但要知道）

### 4.4 异常处理

```ts
try {
  // 可能抛错的代码
} catch (err) {
  // err 是 unknown 类型（TS 4.4+），用 instanceof 收窄
  if (err instanceof TypeError) { ... }
} finally {
  // 无论是否抛错都执行——释放资源的正确位置
}
```

- `throw` 可以抛任何值，但**只抛 Error 对象**（带堆栈信息）
- 自定义错误：`class NetError extends Error { ... }`
- 异步异常：Promise 的 reject 不会被 try/catch 捕获，要用 `.catch()` 或 async 函数里 await + try/catch ⭐

---

## 维度五：函数（JS/TS 的灵魂） {#functions}

### 5.1 定义方式

```ts
function add(a: number, b: number): number { return a + b; }  // 函数声明（整体提升）
const sub = function (a: number, b: number) { return a - b; }; // 函数表达式
const mul = (a: number, b: number) => a * b;                   // 箭头函数
```

### 5.2 参数体系

- **默认参数**：`function f(a = 10)`
- **可选参数**：`function f(a?: number)`（等价于 `a: number | undefined`）
- **剩余参数**：`function f(...args: number[])`
- **参数解构**：`function f({ x, y }: Point)`

### 5.3 函数是一等公民

函数可以：赋值给变量、作为参数传递（回调）、作为返回值返回。这是一切函数式技巧的地基。

```ts
// 回调：定时器、事件、数组方法的灵魂
setTimeout(() => console.log('tick'), 1000);
arr.map(x => x * 2).filter(x => x > 4);
```

### 5.4 闭包（Closure）⭐面试必考

**定义**：函数 + 它能访问的外层作用域变量，即使外层函数已执行完毕。

```ts
function counter() {
  let count = 0;              // 被闭包"捕获"，不会被回收
  return () => ++count;
}
const c = counter();
c(); // 1
c(); // 2 —— count 还活着
```

用途：私有化变量、柯里化、防抖节流、记忆化。
代价：捕获的变量不被 GC，滥用闭包是内存泄漏的来源之一。

### 5.5 this 指向 ⭐面试必考

JS 的 this **在调用时**才确定，规则优先级：

1. `new` 调用 → this 是新对象
2. `obj.fn()` → this 是 obj
3. `fn.call/apply/bind(that)` → 显式指定
4. 普通调用 `fn()` → 严格模式下 undefined，非严格模式是全局对象
5. **箭头函数没有自己的 this**，捕获定义处外层 this——这是它存在的最大意义

Cocos 经典坑：`node.on('event', this.onEvent)` 回调里 this 丢失——正确写法是 `node.on('event', this.onEvent, this)`（第三个参数绑 this）或箭头函数。

### 5.6 高阶函数与函数式技巧

- **高阶函数**：接收或返回函数的函数（map/filter/reduce、装饰器本质也是）
- **柯里化**：`f(a, b, c)` → `f(a)(b)(c)`
- **防抖（debounce）**：停止触发 N 毫秒后才执行（搜索输入框）
- **节流（throttle）**：固定频率执行（滚动、游戏摇杆上报）⭐手写题高频
- **纯函数**：相同输入恒定相同输出、无副作用——好测试、可缓存

### 5.7 生成器与迭代器

- `function*` 生成器：可以暂停和恢复的函数，`yield` 交出控制权
- 可迭代协议（Symbol.iterator）：让 for...of 能遍历自定义对象
- 应用场景：分帧加载大量资源、协程式流程控制

---

## 维度六：代码组织 {#organization}

### 6.1 模块（Module）

```ts
export const MAX_HP = 100;              // 命名导出
export default class Player { ... }     // 默认导出（一个模块一个）

import Player from './Player';
import { MAX_HP } from './const';
import * as Utils from './utils';
```

- ES Module：静态分析、tree-shaking（打包时剔除未用代码）、**现代唯一标准**
- CommonJS（require/module.exports）：Node.js 旧体系，了解即可
- 循环依赖：A 引 B、B 引 A——工程里要靠分层架构避免（底层不许依赖上层）

### 6.2 面向对象三件套

**封装**：把数据和操作收进类里，用访问修饰符控制暴露面。

```ts
class Role {
  public name: string;        // 默认，到处可见
  protected level: number;    // 子类可见
  private _exp: number;       // 仅本类（编译期约束）
  readonly id: number;        // 只能初始化一次
}
```

**继承与多态**：

```ts
abstract class Buff {                 // 抽象类：不能实例化，定义骨架
  abstract apply(role: Role): void;   // 抽象方法：子类必须实现
  onRemove() { /* 公共逻辑 */ }
}

class PoisonBuff extends Buff {
  apply(role: Role) { /* 掉血 */ }    // 多态：同一接口不同实现
}
```

- `extends` 单继承；`super` 访问父类
- **组合优于继承**：能力拆成小模块拼起来，比深继承树灵活（Cocos 的"节点 + 组件"就是组合思想的胜利——没有任何继承也能让节点拥有任意能力）⭐

### 6.3 interface：契约

- 定义"必须长什么样"而不关心实现
- `class X implements ISerializable` 强制类实现接口
- 游戏架构里接口用于解耦：`IPoolable`（可入池）、`ISave`（可存档）、`IDamageable`（可受伤）

### 6.4 泛型：代码的"类型参数化"

一份逻辑适配多种类型且保持类型安全：`class ObjectPool<T extends Node>`。详见《TS核心突破-手写练习清单》板块三。

### 6.5 装饰器：元编程

`@ccclass`、`@property`——编译期执行的函数，给类附加元数据/行为，Cocos 组件体系的基石。

### 6.6 规范与工程化

- 命名：变量函数 camelCase、类 PascalCase、常量 UPPER_SNAKE、私有 _prefix（约定）
- ESLint（规则检查）+ Prettier（格式化）+ 类型即文档
- 分层：数据层 / 逻辑层 / 视图层，单向依赖

---

## 维度七：运行环境常识 {#runtime}

这一维度决定你"懂语法"还是"懂语言"。

### 7.1 内存模型：栈与堆

- **栈（Stack）**：基本类型的值、函数调用帧，自动分配释放，快
- **堆（Heap）**：对象/数组/函数的实体，动态分配，靠引用访问

```ts
let a = 1;
let b = a;      // 值拷贝：改 b 不影响 a

let o1 = { x: 1 };
let o2 = o1;    // 引用拷贝：o2.x = 2 会把 o1.x 也改掉 ⭐
```

函数传参同理：基本类型传值，对象传引用——**函数内修改对象参数会影响外部**，这是最常见的 bug 来源之一。防御：`{ ...obj }` 浅拷贝、`structuredClone(obj)` 深拷贝。

### 7.2 垃圾回收（GC）

- 主流算法：**标记-清除**——从根（全局对象、当前调用栈）出发，走不到的对象回收
- 引用计数：旧算法，循环引用会漏（`a.b = b; b.a = a`）
- **内存泄漏的本质**：不再需要的对象仍被引用着，GC 收不走。典型场景：忘注销的事件监听、全局缓存无限增长、闭包捕获大对象、游离的 DOM/节点
- GC 有停顿（stop-the-world）——游戏里频繁 new/destroy 会引发 GC 抖动造成掉帧，这就是**对象池**存在的理由 ⭐

### 7.3 执行模型：事件循环（Event Loop）⭐面试必考

JS 是**单线程**的，靠事件循环处理并发：

```
同步代码 → 清空微任务队列 → 渲染（浏览器）→ 取一个宏任务 → 清空微任务 → …
```

- **宏任务（macrotask）**：setTimeout、setInterval、I/O、setImmediate
- **微任务（microtask）**：Promise.then/catch/finally、queueMicrotask、await 之后的代码
- 铁律：**每执行完一个宏任务，清空所有微任务**，微任务里产生的微任务也当轮执行

```js
console.log(1);
setTimeout(() => console.log(2));
Promise.resolve().then(() => console.log(3));
console.log(4);
// 输出：1 4 3 2 —— 同步 → 微任务 → 宏任务
```

### 7.4 异步编程演进史 ⭐

1. **回调**：`load(url, cb)`——嵌套深了成"回调地狱"
2. **Promise**：链式 `.then().catch()`，三种状态 pending/fulfilled/rejected，**状态不可逆**
3. **async/await**：Promise 的语法糖，用同步写法写异步

```ts
async function loadAll() {
  try {
    const a = await loadRes('a');                    // 串行
    const [b, c] = await Promise.all([loadB(), loadC()]); // 并行 ⭐
  } catch (err) { /* 统一捕获 */ }
}
```

- 并行工具：`Promise.all`（全成功才成功）、`allSettled`（等全部结束）、`race`（先到先得）、`any`（首个成功）
- Cocos 实践：资源加载全封装成 Promise，加载队列用 async 顺序控制
- 手写 Promise 是经典手写题（状态机 + then 链 + 微任务调度）

### 7.5 标准库（自带的工具箱）

| 类别 | 常用 API |
|---|---|
| 字符串 | slice、split、replace、includes、trim、padStart、模板字符串 |
| 数组 | push/pop/shift/unshift、splice/slice、map/filter/reduce/find/includes、sort（默认按字符串排！数字排序要传比较函数）⭐、flat |
| 对象 | Object.keys/values/entries/assign/freeze、解构、hasOwn |
| Map/Set | 键可以是任意类型、去重、O(1) 查找 |
| Math | floor/ceil/round/random/max/min/pow |
| JSON | stringify/parse（深拷贝乞丐版，注意循环引用和 undefined 丢失） |
| Date | 时间戳、格式化 |
| 正则 | test/match/replace |
| Error | Error/TypeError/RangeError + 自定义错误 |

### 7.6 其他环境常识

- **严格模式** `'use strict'`：禁止隐式全局变量等危险行为，ES Module 默认严格
- **深拷贝 vs 浅拷贝**：浅拷贝只拷第一层；深拷贝方法——`structuredClone`（推荐）、JSON 法（有局限）、递归手写（手写题）
- **判等三兄弟**：`==`（转换）、`===`（严格）、`Object.is`（能区分 +0/-0，认为 NaN 等于 NaN）
- **精度问题**：浮点存储天然有误差，`Number.MAX_SAFE_INTEGER`（2^53-1）、bigint 兜底

---

## 七维度自查总表 {#checklist}

| 维度 | 核心问题 | 不掌握的表现 |
|---|---|---|
| 1 数据与类型 | 这个值是什么类型？规则怎么约束它？ | 隐式转换坑、any 满天飞 |
| 2 变量与常量 | 这个名字在哪有效、活多久？ | 作用域混乱、var 提升坑 |
| 3 运算符与表达式 | 怎么对数据做计算和判断？ | `==` 坑、<code>&#124;&#124;</code>/`??` 混用 |
| 4 控制流程 | 代码按什么顺序、什么条件执行？ | 逻辑绕圈、异常漏处理 |
| 5 函数 | 代码怎么复用和抽象？ | this 丢失、不会闭包 |
| 6 代码组织 | 大项目怎么拆、怎么管？ | 文件互相引、没有分层 |
| 7 运行环境 | 代码实际是怎么跑起来的？ | 内存泄漏、异步时序猜谜 |
