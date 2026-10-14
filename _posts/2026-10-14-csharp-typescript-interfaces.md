---
layout: post
title: "C# 与 TypeScript 的接口有什么区别？从类型匹配到运行时判断"
date: 2026-10-14
categories: [编程基础, 类型系统与参数传递]
tags: ["C#", TypeScript, 接口, 类型系统]
excerpt: "用同一个受伤害能力示例，对比 C# 的显式接口实现与 TypeScript 的结构兼容，并解释运行时检查、可选属性和默认实现的差异。"
---

从 TypeScript 转向 C# 时，很容易产生一个疑问：一个类已经拥有接口要求的全部方法，为什么还是不能赋值给接口变量？

两种语言的 `interface` 都能描述调用方需要的能力，但判断“一个对象是否符合接口”的规则不同。C# 的普通接口转换依赖已声明的实现关系；TypeScript 主要检查成员结构是否兼容。

本文以“可以受到伤害”为例。代码是相互独立的教学片段；C# 调用语句放在方法体内，TS 示例默认启用 `strict`。涉及可空引用和默认接口实现的部分需要 C# 8.0 或更新版本，默认接口实现还需要运行时支持。

## 1. C# 看实现关系，TS 看结构兼容

在 C# 中，先定义能力，再让类声明实现该接口：

```csharp
public interface IDamageable
{
    void TakeDamage(int amount);
}

public class Enemy : IDamageable
{
    public void TakeDamage(int amount)
    {
        System.Console.WriteLine($"受到 {amount} 点伤害");
    }
}
```

调用时可以只依赖接口：

```csharp
IDamageable target = new Enemy();
target.TakeDamage(10);
```

如果删除 `Enemy` 声明中的 `: IDamageable`，即使方法完全不变，上面的赋值也不能通过编译。类型需要声明实现关系，或者从基类继承已有的实现关系。这通常称为**名义类型**的特征。

这里的“声明实现关系”不要与 C# 专门的“显式接口成员实现”语法混淆；后者会写成 `void IDamageable.TakeDamage(...)`，属于另一个概念。普通公开方法实现接口即可满足本例需求。[C# 接口规范](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/language-specification/interfaces)

TypeScript 的对应代码则可以不写 `implements`：

```ts
interface IDamageable {
    takeDamage(amount: number): void;
}

class Enemy {
    takeDamage(amount: number): void {
        console.log(`受到 ${amount} 点伤害`);
    }
}

const target: IDamageable = new Enemy();
target.takeDamage(10);

const trainingDummy: IDamageable = {
    takeDamage(amount: number) {
        console.log(`木桩受到 ${amount} 点伤害`);
    }
};
```

类实例和普通对象都可以满足接口。编译器关心成员名称和类型能否兼容，这叫**结构类型**。它不是要求对象恰好只有接口列出的成员；不过，新建对象字面量会受到额外属性检查，不能据此认为任意多余字段都能直接写进赋值表达式。[TS 类型兼容规则](https://www.typescriptlang.org/docs/handbook/type-compatibility)

## 2. TS 的 implements 有什么用？

既然结构符合就能使用，为什么还要写下面这段代码？

```ts
class Enemy implements IDamageable {
    takeDamage(amount: number): void {
        console.log(amount);
    }
}
```

`implements` 可以明确表达设计意图，并让编译器在类声明处检查契约是否满足。如果后来接口新增必需成员，相关类就能及时暴露错误。

但它不会给类注入方法，也不会把接口的可选成员自动添加到类上，更不会创建运行时的接口身份。类方法的参数类型也不能指望仅靠 `implements` 自动补齐。[TS 类与 implements 检查](https://www.typescriptlang.org/docs/handbook/2/classes.html#implements-clauses)

两种语言都允许一个类实现多个接口。接口让调用方围绕能力组织代码，不要求所有对象属于同一个业务父类。[C# 接口基础](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/types/interfaces)

## 3. 运行时：C# 可以判断接口，TS 接口会被擦除

C# 运行时保留接口类型信息，因此可以这样检查：

```csharp
object candidate = new Enemy();

if (candidate is IDamageable damageable)
{
    damageable.TakeDamage(10);
}
```

这里判断的是对象的运行时类型是否满足接口实现关系，不是临时搜索一个同名方法。`null` 不会通过这个判断。[C# 类型测试运算符](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/type-testing-and-cast)

TS 的接口声明在生成 JavaScript 时被擦除，不会生成可供运行时使用的接口对象。因此下面的写法不成立：

```ts
// 错误示例：IDamageable 只存在于类型层面
// candidate instanceof IDamageable
```

`instanceof Enemy` 可以用于检查类实例的原型链，但它判断的是类，不代表所有结构上满足 `IDamageable` 的对象都能通过。[TS 类型擦除](https://www.typescriptlang.org/docs/handbook/2/basic-types.html#erased-types)、[TS instanceof 收窄](https://www.typescriptlang.org/docs/handbook/2/narrowing.html#instanceof-narrowing)

对未知值可以自行编写类型守卫。下面的片段使用 TS 4.9+ 对 `in` 检查的支持：

```ts
function isDamageable(value: unknown): value is IDamageable {
    return typeof value === "object"
        && value !== null
        && "takeDamage" in value
        && typeof value.takeDamage === "function";
}
```

这只验证“有一个可调用的成员”，不能证明该函数真的接受数值或遵守伤害计算语义。类型谓词是开发者向编译器作出的承诺；外部数据需要按实际格式校验，写一个接口或类型断言不会自动完成验证。[TS 类型谓词](https://www.typescriptlang.org/docs/handbook/2/narrowing.html#using-type-predicates)

## 4. 可选属性不等于可空属性

TS 可以声明一个属性允许不存在：

```ts
interface IUser {
    name?: string;
}

const guest: IUser = {};
```

在严格空值检查下，读取 `guest.name` 需要考虑 `undefined`。开启 `exactOptionalPropertyTypes` 后，显式写 `name: undefined` 与完全省略 `name` 也存在区别；是否允许显式赋 `undefined`，取决于声明的属性类型。[TS 对象的可选属性](https://www.typescriptlang.org/docs/handbook/2/objects.html#optional-properties)

C# 的可空引用属性表达的是另一件事：

```csharp
#nullable enable

public interface IUser
{
    string? Name { get; }
}

public class Guest : IUser
{
    public string? Name => null;
}
```

`Name` 这个属性必须由实现提供，或由适用的默认实现提供；`string?` 表示它的值允许为 `null`。它不表示成员可以不存在。可空引用注解主要帮助编译器进行静态分析，不会自动添加运行时空值检查。[C# 可空引用类型](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/null-safety/nullable-reference-types)

## 5. 接口能否包含实现？

不能把“接口永远只能声明方法，不能写方法体”当作两种语言共同的规则。

TS 接口本身是类型描述，不能承载运行时方法实现。行为需要放在类、函数或对象中。

C# 从 8.0 起支持默认接口实现。例如下面是独立的 C# 接口示例：

```csharp
public interface IDamageable
{
    void TakeDamage(int amount);

    void TakeSmallHit()
    {
        TakeDamage(1);
    }
}
```

实现类可以只提供 `TakeDamage`，通过 `IDamageable` 类型的变量调用默认的 `TakeSmallHit()`。默认成员不会因此自动变成类自身可直接访问的成员。它适合在一定条件下演进接口，但使用前需要确认目标运行时支持；不能只凭项目接受某些 C# 8 语法就推断所有特性可用。[默认接口实现](https://learn.microsoft.com/en-us/dotnet/csharp/advanced-topics/interface-implementation/default-interface-methods-versions)

## 6. 声明合并与 partial 不是同一机制

TS 在同一作用域中可以合并同名接口声明：

```ts
interface UserInfo {
    name: string;
}

interface UserInfo {
    level: number;
}

const user: UserInfo = { name: "Guest", level: 1 };
```

合并后两个成员都属于 `UserInfo`，重复成员的声明必须遵守兼容规则。这也常用于已有类型声明的扩展。[TS 声明合并](https://www.typescriptlang.org/docs/handbook/declaration-merging.html)

C# 可以使用 `partial interface` 将同一个接口分散到多个声明中，但需要显式标记 `partial`，并在同一次编译中组成该类型。不能把它当成 TS 式的任意声明合并，更不能用它向已经编译的第三方接口随意添加成员。[C# 接口规范中的 partial 声明](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/language-specification/interfaces)

## 7. 如何把这些区别用到实际代码里？

| 问题 | C# | TypeScript |
| --- | --- | --- |
| 有同名同类型的方法就满足接口吗？ | 还需要实现关系 | 普通接口主要按结构兼容判断 |
| 普通对象字面量能满足接口吗？ | 不像 TS 那样凭结构直接满足 | 可以 |
| 能在运行时判断接口类型吗？ | 可以使用 `is` | 接口被擦除，需要自行验证结构 |
| 成员可以声明为可选吗？ | 没有直接对应 `name?` 的接口成员语法 | 支持可选属性和方法 |
| 能提供默认方法体吗？ | 支持，需要语言与运行时配合 | 接口本身不能提供实现 |
| 同名声明会自动合并吗？ | 需要显式 `partial` 等条件 | 同一作用域下支持接口声明合并 |

从 TS 转向 C# 时，先检查类型是否建立了实现关系，再检查成员签名；不要只看“方法明明已经存在”。从 C# 转向 TS 时，则要把编译期契约和运行时验证分开：`implements`、类型注解和 `as` 都不会自动验证外部输入。

设计接口时，可以从调用方真正需要的能力出发。伤害处理逻辑只依赖 `IDamageable`，敌人、木桩或可破坏道具各自提供实现，就能避免把调用逻辑绑定到某一种具体对象。接口负责约定能力，具体类和对象负责实现行为。
