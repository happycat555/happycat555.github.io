---
layout: post
title: "TS 与 C# 单例写法对比：函数返回类与泛型静态缓存"
date: 2026-10-09
categories: [编程基础, 单例模式与泛型]
tags: [TypeScript, "C#", Unity, 单例模式, 泛型]
excerpt: "从 TS 的函数返回类出发，理解单例缓存为什么会共享，以及 C# 如何通过不同具体泛型类型隔离静态字段，并对照 Unity 官方包的实现。"
---

这份笔记围绕项目中的 `Singleton.ts`、自定义 `Singleton.cs`，以及 Unity Visual Scripting 包里的 `Singleton<T>` 展开。

## 1. 核心结论

单例需要解决两个问题：第一次访问时创建或找到实例，以后返回同一个实例；不同管理器各自保存自己的实例。

两种语言在第二个问题上的实现不同：

| 对比项 | 项目中的 TypeScript 写法 | 项目中的 C# 写法 |
| --- | --- | --- |
| 定义方式 | 函数返回一个类 | 直接定义泛型类 |
| 使用方式 | `extends Singleton<UIMgr>()` | `: Singleton<UIMgr>` |
| 缓存隔离方式 | 每次调用函数生成不同的类，各有静态缓存 | 不同具体泛型类型，各有静态缓存 |
| 创建实例 | getter 中的 `new this()` | `new T()` |
| 泛型的运行时行为 | 类型参数会被擦除 | 运行时保留泛型类型信息 |
| 构造限制 | 基类的 `protected constructor()` 限制外部创建 | `new()` 约束要求子类有公共无参构造函数 |

**不是 TS 必须用函数才能实现单例，而是这份 TS 代码通过函数创建类，来隔离各个管理器的缓存。**

## 2. TS 为什么看起来像在继承函数？

原代码如下：

```ts
export function Singleton<T>() {
    class SingletonE {
        protected constructor() {}
        private static _inst: SingletonE = null;

        public static get inst(): T {
            if (SingletonE._inst == null) {
                SingletonE._inst = new this();
            }
            return SingletonE._inst as T;
        }
    }

    return SingletonE;
}
```

使用：

```ts
class UIMgr extends Singleton<UIMgr>() {
    public showPanel() {
        console.log("显示面板");
    }
}

UIMgr.inst.showPanel();
```

这里不是继承函数，而是先执行函数，再继承它返回的类。可以拆开理解：

```ts
const BaseClass = Singleton<UIMgr>();

class UIMgr extends BaseClass {
    public showPanel() {
        console.log("显示面板");
    }
}
```

JS / TS 中的类也可以作为值存入变量、作为参数传递、作为函数返回值。

- `return SingletonE`：返回类本身，可以作为基类。
- 返回一个对象实例：不能直接用该实例作为这里的基类。
- `<UIMgr>`：类型参数，帮助编译器检查类型。
- `()`：实际执行函数，创建并返回类。

## 3. 为什么共享基类可能导致缓存混用？

问题不在于“谁抢到了缓存”，而在于所有访问是否指向同一个变量。

假设只有一个普通基类，内部始终通过 `Singleton._inst` 读写缓存：

```text
UIMgr  ──┐
         ├── Singleton._inst
AudioMgr ┘
```

访问过程如下：

1. 第一次访问 `UIMgr.inst`，缓存为空，创建 UIMgr 对象。
2. 把这个对象保存到 `Singleton._inst`。
3. 再访问 `AudioMgr.inst`，检查的仍然是同一个 `Singleton._inst`。
4. 缓存已经非空，直接返回之前的 UIMgr 对象，结果就错了。

**继承不会自动给每个子类复制一份基类的静态字段。** 尤其当代码明确写着 `Singleton._inst` 时，它始终访问该基类上的同一个字段。

这不意味着所有普通基类方案都会混用缓存。通过按构造函数存储的 Map 等方式，也能实现实例隔离；只是当前代码选择了类工厂方案。

## 4. TS 函数如何隔离缓存？

```ts
class UIMgr extends Singleton<UIMgr>() {}
class AudioMgr extends Singleton<AudioMgr>() {}
```

这里调用了两次函数。每次执行函数内部的类定义，都会创建一个新的类：

```text
第一次调用 → SingletonE 类① → _inst① → UIMgr 对象
第二次调用 → SingletonE 类② → _inst② → AudioMgr 对象
```

虽然源码中都叫 `SingletonE`，但它们是不同的运行时类，各自拥有自己的静态字段。

隔离的依据是“调用函数创建新类”，不是泛型参数本身。即使两次调用传入相同的类型参数，也仍然会生成两个不同的类。反过来，如果多个子类复用同一次调用返回的基类，这份实现仍会共享那个基类的缓存。

### `new this()` 创建谁？

访问 `UIMgr.inst` 时，继承来的静态 getter 中的 `this` 是 `UIMgr`，因此 `new this()` 创建 UIMgr 实例。

这里两个名字各有用途：

- `this`：决定创建哪个具体子类。
- `SingletonE._inst`：决定把实例存在哪个生成的基类缓存中。

### `as T` 做什么？

`as T` 是类型断言，让编译器把返回值视为 `T`。它不会转换对象，也不会在运行时验证对象是否真的是 `T`。

另外，原代码若开启 `strictNullChecks`，字段应声明为 `SingletonE | null`，才能用 `null` 初始化。

## 5. C# 为什么不需要用函数返回类？

当前的 C# 基类如下：

```csharp
public abstract class Singleton<T>
    where T : Singleton<T>, new()
{
    private static T _instance;

    public static T Instance
    {
        get
        {
            if (_instance == null)
            {
                _instance = new T();
            }

            return _instance;
        }
    }

    protected Singleton() { }
}
```

使用：

```csharp
public sealed class UIMgr : Singleton<UIMgr>
{
    public void ShowPanel() { }
}

public sealed class AudioMgr : Singleton<AudioMgr>
{
}
```

在 C# 中，`Singleton<UIMgr>` 和 `Singleton<AudioMgr>` 是不同的具体泛型类型，各自拥有静态字段：

```text
Singleton<UIMgr>   → _instance①
Singleton<AudioMgr> → _instance②
```

因此源码只需要定义一个泛型类，就能做到缓存隔离。这里保证隔离的是具体泛型类型不同，而不是继承动作本身。

TS 则不能直接这样写：

```ts
class Singleton<T> {
    static inst: T; // 错误：静态成员不能引用类的类型参数
}
```

TS 的类型参数在运行时被擦除，不会因为使用了不同的类型参数，就自动生成不同的类和静态存储。

## 6. 当前 C# 写法需要注意什么？

### `new()` 和私有构造函数不能同时使用

`where T : Singleton<T>, new()` 中的 `new()` 要求 T 提供公共无参构造函数。

因此下面的子类与当前基类不兼容，会导致编译错误：

```csharp
public sealed class UIMgr : Singleton<UIMgr>
{
    private UIMgr() { }
}
```

应改成公共无参构造函数，或者省略构造函数，让编译器生成默认公共无参构造函数：

```csharp
public sealed class UIMgr : Singleton<UIMgr>
{
    public UIMgr() { }
}
```

这样也意味着外部仍能 `new UIMgr()`。这个方案靠使用约定保证统一访问 `UIMgr.Instance`，并不能严格禁止额外实例。

若希望子类保留私有构造函数，可以采用之前讨论过的 `Lazy<T> + Activator.CreateInstance` 方案，但它引入了反射，且缺少无参构造函数等问题会在首次创建时才暴露。`Lazy<T>` 保证的初始化线程安全，也不等于整个管理器的数据操作都线程安全。

### 普通类与 Unity 组件要区分

当前模板是普通 C# 类单例：

- 适合不依赖组件生命周期的管理器或数据模型。
- 不能挂到 GameObject 上。
- 不会自动调用 Unity 的 Awake、Update 等生命周期方法。
- 当前的判空后 `new T()` 写法不是线程安全的，适合仅在主线程访问。
- 静态缓存不会因为切换场景就自动清空，存档和单局数据需要自行设计重置逻辑。

## 7. Unity Visual Scripting 包如何处理？

查看的文件位于：

```text
Library/PackageCache/com.unity.visualscripting@4edfcc09512f/
Runtime/VisualScripting.Core/Unity/Singleton.cs
```

其中的核心结构是：

```csharp
public static class Singleton<T> where T : MonoBehaviour, ISingleton
{
    private static T _instance;
}
```

它同样利用 C# 泛型静态字段按具体类型隔离的规则，不同 T 不会共用 `_instance`。

不过它是静态工具类，不能作为继承基类。组件需满足其接口、特性和生命周期配合要求，通过 `Singleton<T>.instance` 获取实例。

在运行状态下，其主要流程是：

1. 有缓存就返回缓存。
2. 没有缓存就查找场景中的 T 组件。
3. 找到一个就保存为实例。
4. 找不到且允许自动创建，就创建 GameObject 并添加 T 组件；不允许则报错。
5. 找到多个则报错。

它还处理持久化、Awake 注册、OnDestroy 清理等。非运行状态下访问实例会重新走查找流程，而不是直接使用运行时缓存。代码里的锁也不意味着可以在后台线程调用 Unity 场景查找或组件创建 API。

自定义基类与包里的实现，在缓存隔离上用的是同一个 C# 机制，只是实例的创建方式和生命周期处理不同。不需要修改 PackageCache 中的代码来实现自己的单例。

## 8. MVC 中 Model 一定要单例吗？

不一定。MVC 描述职责划分，并不规定 Model 必须是单例。

- UIMgr 等全局管理入口可以使用单例。
- 背包、角色、关卡等具体数据模型可以是普通对象。
- 玩家背包、仓库、敌人背包可能同时存在，允许创建多个模型更灵活。
- 可以让一个单例数据入口持有多个普通 Model，也可以把 Model 通过构造函数传给 Controller。

选择单例前，先确定对象是否确实全局唯一，以及什么时候创建、清空或替换。

## 9. 最容易记住的区别

**TS 这份写法：调用函数生成不同的类，每个类保存自己的实例。**

**C# 这份写法：使用不同的具体泛型类型，每个类型保存自己的实例。**

两者都在确保：UI 管理器和音频管理器，不会把实例存进同一个缓存变量。
