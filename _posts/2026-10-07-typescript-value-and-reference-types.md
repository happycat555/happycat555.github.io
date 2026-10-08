---
layout: post
title: "TypeScript 中的值类型与引用类型"
date: 2026-10-07
categories: [编程基础, 类型系统与参数传递]
tags: [TypeScript, JavaScript, 类型系统, 参数传递]
excerpt: "从基本类型与对象出发，理解赋值、函数传参、对象比较和深浅拷贝，并分清 const、readonly 与 Object.freeze 的边界。"
---

> 整理自与「小蓝」的讨论（2026-10-07），补充概念边界与代码示例。

## 一、先明确：讨论的是运行时的值

TypeScript 沿用 JavaScript 的运行时行为。理解赋值和传参时，可以把值分为**基本类型的值（primitive）**和**对象（object）**。

日常所说的“值类型与引用类型”，在这里指基本值与对象的行为差异。TypeScript 没有像 C# `struct` 那样可由开发者定义、赋值时自动进行值复制的结构体机制。

这不是 TypeScript 全部类型的分类：`any`、`unknown`、`never`、联合类型等属于静态类型系统，不是额外的 JavaScript 运行时值种类。

## 二、基本类型：七种，值本身不可变

| 类型 | 例子与说明 |
| --- | --- |
| `string` | `"hello"`，字符串 |
| `number` | `42`、`3.14`，也包括 `NaN` 和 `Infinity` |
| `bigint` | `123n`，大整数；使用时需考虑目标运行环境支持 |
| `boolean` | `true`、`false` |
| `undefined` | 常见于未赋值变量、缺失属性或没有返回值的函数 |
| `null` | 通常由程序显式用来表示“没有值” |
| `symbol` | `Symbol("id")`，每次调用都会产生新的唯一值 |

赋值后，重新修改一个变量的绑定，不会影响另一个变量：

```typescript
let a = 10;
let b = a;
b = 20;
console.log(a); // 10

let s = "abc";
const upper = s.toUpperCase();
console.log(s, upper); // abc ABC
s = "xyz"; // 让 s 绑定另一个值，没有修改原来的字符串
```

**值不可变，不等于变量不能重新赋值。** `let` 可以重新绑定，`const` 不可以。

`string` 与 `new String("abc")` 也不同：后者创建包装对象。日常应使用基本值和小写类型名 `string`、`number`、`boolean`。[TypeScript：常用类型](https://www.typescriptlang.org/docs/handbook/2/everyday-types.html)

## 三、对象：赋值后共享同一个对象

普通对象、数组、函数、类实例，以及 `Map`、`Set`、`Date`、`RegExp`、`Promise`、类型化数组等，都是对象。

```typescript
const obj1 = { x: 1 };
let obj2 = obj1;
obj2.x = 99;
console.log(obj1.x); // 99：两者访问的是同一个对象

obj2 = { x: 2 };
console.log(obj1.x); // 99：重新绑定 obj2，不会改掉 obj1 的绑定

const arr1 = [1, 2, 3];
const arr2 = arr1;
arr2.push(4);
console.log(arr1.length); // 4
```

可以把第一次赋值理解为“复制引用”，没有自动复制整个对象：

```text
obj1 ─┐
      ├──→ { x: 99 }
obj2 ─┘

obj2 重新赋值后：
obj1 ────→ { x: 99 }
obj2 ────→ { x: 2 }
```

这里的“引用”用于解释对象身份与共享关系，并不意味着 JavaScript 暴露了可操作的内存地址。

## 四、函数参数一律按值传递

传入对象时，函数与调用方可以访问同一个对象；但函数不能通过重新给形参赋值，替换调用方变量的绑定。

```typescript
function update(num: number, obj: { v: number }) {
  num = 100;       // 只改变局部参数
  obj.v = 100;     // 修改共享对象
  obj = { v: 200 }; // 只让局部参数指向新对象
  return obj;
}

const n = 1;
const original = { v: 1 };
const returned = update(n, original);
console.log(n);             // 1
console.log(original.v);    // 100，不是 200
console.log(returned.v);    // 200
console.log(original === returned); // false
```

因此，“对象按引用传递”容易造成误解。更准确的表述是：**参数按值传递；传入对象时，共享该对象的访问关系。** [MDN：函数参数](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Functions)

## 五、比较：对象看身份，基本值也有边界

```typescript
const first = { x: 1 };
const second = { x: 1 };
const alias = first;
console.log(first === second); // false：内容相同，对象不同
console.log(first === alias);  // true：同一个对象

const missingNumber = Number("not a number");
console.log(missingNumber === missingNumber); // false：NaN 不等于自身
console.log(Number.isNaN(missingNumber));     // true
console.log(Object.is(NaN, NaN));             // true
console.log(0 === -0);                       // true
console.log(Object.is(0, -0));               // false

const id1: symbol = Symbol("id");
const id2: symbol = Symbol("id");
console.log(id1 === id2); // false：描述相同，不代表同一个 symbol
```

不要把基本类型的比较全部概括为“比内容”；`NaN`、正负零和 `symbol` 值得单独记忆。`Object.is` 也不是对象的深比较工具。[MDN：严格相等](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Strict_equality)

## 六、浅拷贝与深拷贝

### 浅拷贝只创建新的外层容器

```typescript
const source = { hp: 100, position: { x: 1 } };
const shallow = { ...source };
shallow.hp = 80;
shallow.position.x = 99;
console.log(source.hp);         // 100：外层属性独立
console.log(source.position.x); // 99：嵌套对象仍然共享
```

对象展开、`Object.assign({}, source)`、数组展开和 `slice()` 都不会递归复制嵌套对象。对象展开也不会把类实例的原型方法复制成一个新实例。

只需要修改某个分支时，可以按需复制：

```typescript
const source = { hp: 100, position: { x: 1 } };
const next = {
  ...source,
  position: { ...source.position, x: 2 },
};
console.log(source.position.x, next.position.x); // 1 2
```

### structuredClone 有适用范围

在支持此 API 的运行环境中，可以克隆支持结构化克隆的数据：

```typescript
const source = { position: { x: 1 }, createdAt: new Date(0) };
const copy = structuredClone(source);
copy.position.x = 20;
console.log(source.position.x);           // 1
console.log(copy.createdAt instanceof Date); // true
```

它支持循环引用以及 `Date`、`Map`、`Set` 等数据，但不是任意对象的通用复制器：函数、symbol 值、DOM 节点等不能这样克隆；自定义类实例的原型链和方法也不能指望被完整保留。对于带行为或外部资源的对象，应使用显式构造、项目约定的 `clone` 方法或相应框架 API。[MDN：结构化克隆](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Structured_clone_algorithm)

### JSON 往返不是通用深拷贝

`JSON.parse(JSON.stringify(value))` 只适用于明确接受 JSON 数据语义的场景：

- 对象属性中的 `undefined`、函数和 symbol 值会被省略；数组中的这些值会变为 `null`。
- `Date` 通常变成字符串，`NaN` 和正负 `Infinity` 变成 `null`。
- 循环引用会报错；默认情况下，包含 `bigint` 的序列化也会报错。
- `Map`、`Set` 的条目不会自动按原类型保留。

选择复制方式前，先确认数据里有哪些值，以及是否真的需要复制整个对象图。[MDN：JSON.stringify](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/JSON/stringify)

## 七、const、readonly、Object.freeze 各管什么

| 机制 | 作用 | 边界 |
| --- | --- | --- |
| `const` | 禁止变量重新绑定 | 对象属性仍可修改 |
| `readonly` / `Readonly<T>` | TypeScript 类型检查限制写入 | 不提供运行时冻结，通常只约束一层 |
| `Object.freeze` | 运行时冻结对象自身的属性 | 是浅冻结，不自动冻结嵌套对象 |

```typescript
const state = { hp: 100 };
state.hp = 80; // 合法：没有给 state 重新赋值

const config: Readonly<{ nested: { speed: number } }> = {
  nested: { speed: 1 },
};
config.nested.speed = 2; // 合法：嵌套对象并没有被声明为只读

const frozen = Object.freeze({ nested: { speed: 1 } });
frozen.nested.speed = 2; // 合法：只冻结外层
console.log(frozen.nested.speed); // 2
```

`readonly` 也不会阻止其他可写别名修改同一个对象；`as const` 不会在运行时调用冻结。`Object.freeze` 对普通对象的自身属性生效，也不能据此断言 `Map`、`Set` 的内部条目被冻结。[TypeScript：只读属性](https://www.typescriptlang.org/docs/handbook/2/objects.html)、[MDN：Object.freeze](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/freeze)

## 八、类型声明与内存实现不要混在一起

`interface`、类型别名和类型注解通常在编译时被擦除；类则有运行时构造函数，实例仍是对象。

```typescript
interface User { name: string }
type UserId = string;
class Player { name = ""; }

const first = new Player();
const second = first;
second.name = "HappyCat";
console.log(first.name); // HappyCat
```

“类型信息被擦除”不等于所有 TypeScript 语法都会消失，例如普通 `enum` 通常会生成运行时代码。

也不要用“基本类型一定在栈上、对象一定在堆上”来判断程序行为。具体存储由引擎和优化策略决定；赋值、共享、比较等语言层面的行为才是这里需要掌握的内容。

## 九、速查与自测

| 操作 | 基本值 | 对象 |
| --- | --- | --- |
| 赋值或传参 | 传递值，变量绑定独立 | 不复制对象，可能共享同一对象 |
| 重新给变量或参数赋值 | 不修改其他绑定 | 同样不修改其他绑定 |
| 修改对象属性 | 基本值本身不可变 | 可影响所有访问同一对象的代码，受只读和冻结等约束 |
| `===` | 按类型和值的规则比较，注意特殊值 | 比较是否为同一个对象 |

看下面的例子，先猜输出，再运行：

```typescript
const players = [{ hp: 100 }];
const copied = [...players];
copied[0].hp = 50;
copied.push({ hp: 80 });
console.log(players.length, players[0].hp);
```

答案是 `1 50`：数组容器复制了，但第一个元素仍是共享对象。

继续阅读：[编程语言的七大基础维度（以 TypeScript 为参照）]({% post_url 2026-10-07-programming-language-seven-dimensions %})。
