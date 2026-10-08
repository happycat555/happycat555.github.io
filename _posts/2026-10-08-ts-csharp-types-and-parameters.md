---
layout: post
title: "TS 与 C# 类型系统 & 传参笔记"
date: 2026-10-08
categories: [编程基础, 类型系统与参数传递]
tags: [TypeScript, JavaScript, "C#", 字符串, 类型系统, 参数传递]
excerpt: "对比 TypeScript 与 C# 的类型系统、字符串、对象共享及参数传递，梳理按值传递、引用拷贝与 ref / out 的区别。"
---

> 2026-10-08 学习记录

<details markdown="1">
<summary>文章目录 · 类型系统与参数传递</summary>

- [1. TypeScript 的类型 vs JavaScript 的运行时值](#section-1)
- [2. C# 的 string：引用类型，但表现得像值类型](#section-2)
- [3. TS 的 string：原始类型（primitive）](#section-3)
- [4. JS 引擎（V8）如何实现 string](#section-4)
- [5. TS 函数参数传递：按值，但对对象"值即引用"（call-by-sharing）](#section-5)
- [6. C# 引用类型传参与 JS 的区别](#section-6)

</details>

## 1. TypeScript 的类型 vs JavaScript 的运行时值 {#section-1}

`any`、`unknown`、`never`、联合类型（`A | B`）等都属于**静态类型系统**，只在编译/类型检查阶段存在；编译成 JS 后类型注解全部擦除，运行时只剩 JS 原本的值种类：`string`、`number`、`boolean`、`object`、`undefined`、`symbol`、`bigint`。

常见误解：

- `unknown` 不是某种特殊的值，只是"类型上可以是任何东西"
- `never` 不是值，表示"不可能有这个值"（如总会抛异常的函数，返回值类型为 `never`）
- `string | number` 不是运行时的"联合值"，运行时该值要么是 string 要么是 number，联合只是类型层面的"或"

**例外**：`enum`（普通枚举）和 `class` 编译后会生成真实运行时对象；装饰器等也有运行时行为。

一句话：类型是编译器检查代码的工具，不是 JS 引擎在运行时实际区分的东西。

## 2. C# 的 string：引用类型，但表现得像值类型 {#section-2}

`string` 是引用类型（基类 `System.Object`），但因为**不可变（immutable）**，表现像值类型：

```csharp
string a = "hello";
string b = a;        // b 和 a 指向同一对象（堆上）
b = b + "!";         // 产生新对象，a 不变
```

- 变量存的是引用（地址），赋值拷贝引用
- 任何"修改"（拼接、替换）都创建新对象，原字符串不被改
- 可以为 `null`；`==` 比较的是**内容**而不是引用（String 重载了 `==`）

**高频面试题**：string 是引用类型，为什么有值类型的表现？答案：**不可变性**。

## 3. TS 的 string：原始类型（primitive） {#section-3}

JS/TS 的类型二分：

| 类别 | 类型 |
|---|---|
| 原始类型（值类型） | `string`、`number`、`boolean`、`undefined`、`null`、`symbol`、`bigint` |
| 引用类型 | `object`（`{}`、`[]`、`function`、class 实例、`Date` 等） |

TS 的 `string` 直接存值本身，没有"指向对象"的概念：

```typescript
let a = "hello";
let b = a;      // 值的拷贝，完全独立
b += "!";       // a 仍是 "hello"

let x = { s: "hello" };
let y = x;      // y 和 x 指向同一个对象
y.s = "hi";     // x.s 也变成 "hi"
```

**佐证**：JS 访问 `"abc".length` 时，引擎临时把原始字符串**装箱**为 `String` 对象，调用完即丢弃——说明原始 string 本身没有属性和方法。

**C# vs TS 的 string 对比**：

- C#：引用类型 + 不可变 → 拷贝引用，但因不可变，表现像值类型
- TS：天生是值类型，拷贝的就是值本身

两者殊途同归：所有"修改字符串"操作都产生新字符串。

## 4. JS 引擎（V8）如何实现 string {#section-4}

TS 不实现 string，运行时由 JS 引擎实现。三个层面：

**规范层面（ECMAScript）**：string 是不可变的 16 位无符号整数序列（UTF-16 code unit），按值比较。`.length` 返回 code unit 数量，不是字符数（`"𠮷".length === 2`）。

**V8 内部表示**（按需选择，惰性优化）：

| 内部类型 | 适用场景 | 原理 |
|---|---|---|
| SeqString | 短字符串 | 扁平 char 数组 |
| ConsString | 拼接结果 | 只存左右两个子串引用（rope），`a + b` 为 O(1) |
| SlicedString | 子串 | 存父串指针 + 偏移量，`slice()` 为 O(1) |
| ThinString | 去重后 | 指向内联化的唯一一份 |
| ExternalString | 长字符串 | 字符数据放在 V8 堆外 |

rope 结构示例：`for (let i = 0; i < 10000; i++) s += i;` 不是 O(n²)，每轮只建一个 ConsString 节点，需要连续内存时才一次性 flatten。

**内存与内联化（interning）**：内容相同的字面量字符串在堆中只存一份，多个变量指向它。这也是 `new String("a") === new String("a")` 为 false 的原因——`new String` 强制装箱成对象，跳过内联，是两个不同的堆对象。

## 5. TS 函数参数传递：按值，但对对象"值即引用"（call-by-sharing） {#section-5}

JS/TS 参数只有按值传递，没有按引用：

```typescript
function f(x: number) { x = 99; }
let a = 1;
f(a);
console.log(a);  // 1，没变
```

对象传的是**引用的拷贝**：

```typescript
function g(obj: { n: number }) { obj.n = 99; }
let o = { n: 1 };
g(o);
console.log(o.n);  // 99，变了！
```

但替换引用无效：

```typescript
function h(obj: { n: number }) { obj = { n: 99 }; }
h(o);
console.log(o.n);  // 不受影响
```

**规则**：能通过引用**修改**对象内部，不能**替换**调用者那边的引用指向。

### 为什么 `obj.n = 99` 会生效（内存图解）

```
let o = { n: 1 };
// 栈：o 存引用 0xA100；堆：0xA100 处是对象 { n: 1 }

g(o)  // 参数按值拷贝：obj = 0xA100（和 o 指向同一对象）

obj.n = 99  // 顺着引用找到对象，改的是对象本身
// o ──► 0xA100: { n: 99 }   ← 同一对象，o 看到的也变了

// 而 obj = { n: 99 } 是给局部变量重新赋值：
// o   ──► 0xA100: { n: 1 }   ← 原封不动
// obj ──► 0xB200: { n: 99 }   ← 指向新对象
```

区分两个"地址"：

- 变量槽位本身的地址（如 `0xB100`）
- 槽位里存的值 = 堆对象地址（如 `0xA100`）

JS 不暴露真实地址（无指针运算），可用 `Object.is(a, b)` 判断两变量是否指向同一对象。

### 其他语言对比

| 语言 | 机制 |
|---|---|
| C++ | 默认按值；`void f(T& x)` 真正按引用，可替换调用者变量 |
| C# | 默认按值；`ref`/`out` 按引用，可替换 |
| Java | 和 JS 一样：基本类型按值，对象传引用拷贝（call-by-sharing） |
| Python | 和 JS 完全一样的 call-by-sharing |
| Go | slice/map/interface 传引用拷贝；替换需指针参数 |
| Rust | 默认按值 move；`&T` 借用、`&mut T` 可变借用，编译期检查 |

**实践建议**（TS）：函数内不要原地改传入对象；想改返回新对象（React/Redux 不可变数据流）。防御性拷贝：`{...obj}` / `[...arr]`，深拷贝用 `structuredClone(obj)`。

## 6. C# 引用类型传参与 JS 的区别 {#section-6}

**默认情况：和 JS 完全相同**——拷贝引用、共享对象。

真正的区别：C# 多了 `ref` / `out` 关键字。

**`ref`：传变量本身（栈槽位）**

```csharp
void Swap(ref string a, ref string b) => (a, b) = (b, a);

string x = "hello", y = "world";
Swap(ref x, ref y);
// x == "world", y == "hello"  ← 调用者的变量被替换！（JS 做不到）
```

原理：`ref` 把变量槽位本身传入，函数对参数的赋值直接写调用者的槽位，没有拷贝层。

**`out`：`ref` 变体，强制赋值**

```csharp
bool TryParse(string s, out int result)
{
    result = 42;  // 必须赋值，否则编译错误
    return true;
}

if (int.TryParse("123", out int n))
    Console.WriteLine(n);  // 123
```

`ref` 要求调用前必须初始化；`out` 不要求，但函数内必须赋值（编译器强制）。

**C# 值类型（struct）**：默认**整体拷贝**，函数内改拷贝不可见；`ref struct` 可直接改原变量。JS 没有 struct 概念。

### 总结对比表

| 场景 | JS/TS | C#（引用类型） | C#（值类型 struct） |
|---|---|---|---|
| 传参默认行为 | 拷贝引用，共享对象 | 拷贝引用，共享对象 | 整体拷贝 |
| 函数内改对象内部 | 调用者可见 ✅ | 调用者可见 ✅ | 改的是拷贝，不可见 |
| 函数内替换整个参数 | 调用者不可见 ❌ | 不可见 ❌ | 不可见 ❌ |
| 替换调用者的变量 | 做不到 | `ref` / `out` ✅ | `ref` / `out` ✅ |

**一句话**：JS 永远只有一种规则，简单但无选择权；C# 默认安全（无副作用），需要时用 `ref` 显式开启"按引用"，`out` 的强制赋值把错误挡在编译期。
