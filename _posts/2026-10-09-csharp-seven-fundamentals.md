---
layout: post
title: "编程语言七大基础维度 · C# 版"
date: 2026-10-09
categories: [编程基础, 基础知识地图]
tags: ["C#", ".NET", 编程基础, 面向对象, 异步编程]
excerpt: "以 C# 为例，从类型系统、控制流、函数方法、集合、面向对象、资源管理到并发异步，建立七个基础维度的学习地图。"
---

> 无论学习哪门编程语言，都可以从七个基础维度去拆解它。本文以 C#（.NET 8+ 语法）为例，逐维度讲解核心概念并配以代码示例。

---

<details markdown="1">
<summary>文章目录 · C# 七大基础维度</summary>

1. [维度一：类型系统与变量](#types-and-variables)
2. [维度二：控制流](#control-flow)
3. [维度三：函数与方法](#functions-and-methods)
4. [维度四：集合与数据结构](#collections)
5. [维度五：面向对象与抽象](#object-oriented)
6. [维度六：错误处理与资源管理](#errors-and-resources)
7. [维度七：并发与异步编程](#concurrency-and-async)
8. [总结](#summary)

</details>

---

## 维度一：类型系统与变量 {#types-and-variables}

类型系统决定了一门语言如何描述和操作数据。C# 是**强类型、静态类型**语言，类型在编译期确定。

### 值类型 vs 引用类型

- **值类型**（`int`、`double`、`bool`、`struct`、`enum`）：存储在栈上，赋值即复制。
- **引用类型**（`class`、`string`、`interface`、`delegate`）：存储在托管堆上，赋值复制的是引用。

```csharp
int a = 10;
int b = a;        // 值复制，b 修改不影响 a
b = 20;
Console.WriteLine(a); // 输出 10

var list1 = new List<int> { 1, 2 };
var list2 = list1;    // 引用复制，指向同一对象
list2.Add(3);
Console.WriteLine(list1.Count); // 输出 3
```

### 常用类型与可空

```csharp
// 基本类型
int age = 25;
double price = 19.99;
bool isActive = true;
char grade = 'A';
string name = "C#";

// 可空值类型（Nullable）
int? score = null;
if (score.HasValue)
    Console.WriteLine(score.Value);

// 字符串插值与原始字符串（C# 11+）
var message = $"Hello, {name}! 价格是 {price:F2}";
var json = """
    {
        "name": "C#",
        "year": 2000
    }
    """;
```

### var 与类型推断

```csharp
var number = 42;            // 编译器推断为 int
var text = "类型推断";       // 推断为 string
// var 只能用于局部变量，类型在编译期仍然确定
```

---

## 维度二：控制流 {#control-flow}

控制流决定程序的执行顺序：分支、循环与跳转。

### 条件分支

```csharp
int score = 85;

// if-else
if (score >= 90)
    Console.WriteLine("优秀");
else if (score >= 60)
    Console.WriteLine("及格");
else
    Console.WriteLine("不及格");

// switch 表达式（C# 8+），比传统 switch 更简洁
var grade = score switch
{
    >= 90 => "A",
    >= 80 => "B",
    >= 60 => "C",
    _     => "D"   // 弃元，匹配其余所有情况
};
Console.WriteLine(grade);
```

### 循环

```csharp
// for：已知次数
for (int i = 0; i < 5; i++)
    Console.Write(i + " ");   // 0 1 2 3 4

// foreach：遍历集合（C# 中最常用）
var fruits = new[] { "苹果", "香蕉", "橙子" };
foreach (var fruit in fruits)
    Console.WriteLine(fruit);

// while / do-while：条件循环
int n = 0;
while (n < 3) n++;

// 循环控制：break 跳出、continue 跳过本次
foreach (var f in fruits)
{
    if (f == "香蕉") continue;
    if (f == "橙子") break;
    Console.WriteLine(f);     // 只输出 苹果
}
```

---

## 维度三：函数与方法 {#functions-and-methods}

函数是代码复用的基本单元。C# 中函数必须属于某个类型（静态方法属于类，实例方法属于对象）。

### 定义与调用

```csharp
// 方法定义：返回值、参数、默认参数
static double CalculateArea(double width, double height = 1.0)
{
    return width * height;
}

var area = CalculateArea(3.5, 2.0);
var square = CalculateArea(4.0);   // height 使用默认值 1.0
```

### 参数修饰符

```csharp
// ref：按引用传递，可读写
static void DoubleIt(ref int x) => x *= 2;

// out：用于返回多个值
static bool Divide(int a, int b, out int quotient, out int remainder)
{
    quotient = a / b;
    remainder = a % b;
    return b != 0;
}

// in：只读引用传递，避免大结构体拷贝
static decimal TotalPrice(in Order order) => order.Price * order.Count;

// params：可变参数
static int Sum(params int[] numbers)
{
    var sum = 0;
    foreach (var n in numbers) sum += n;
    return sum;
}
Sum(1, 2, 3, 4);   // 10
```

### 本地函数与 Lambda 表达式

```csharp
// Lambda：C# 函数式编程的核心
Func<int, int> square = x => x * x;
Func<int, int, int> add = (x, y) => x + y;
Console.WriteLine(square(5));   // 25

// 集合配合 Lambda（这是 C# 的日常）
var numbers = new[] { 1, 2, 3, 4, 5 };
var evens = numbers.Where(n => n % 2 == 0).ToList();  // [2, 4]
```

---

## 维度四：集合与数据结构 {#collections}

数据结构决定了数据如何组织和高效访问。

### 常用集合

```csharp
// List<T>：动态数组，最常用的集合
var list = new List<string> { "C#", "Java", "Go" };
list.Add("Rust");
list.Insert(1, "Python");
list.Remove("Java");

// Dictionary<TKey, TValue>：键值对哈希表
var scores = new Dictionary<string, int>
{
    ["小明"] = 90,
    ["小红"] = 85
};
scores["小刚"] = 78;
if (scores.TryGetValue("小明", out var s))
    Console.WriteLine(s);

// HashSet<T>：去重集合
var tags = new HashSet<string> { "后端", "桌面", "后端" };
Console.WriteLine(tags.Count);   // 2，自动去重

// Queue<T> / Stack<T>：队列与栈
var queue = new Queue<int>();
queue.Enqueue(1); queue.Enqueue(2);
queue.Dequeue();                 // 1（先进先出）

var stack = new Stack<int>();
stack.Push(1); stack.Push(2);
stack.Pop();                     // 2（后进先出）
```

### Span<T> 与高性能内存（C# 进阶）

```csharp
// Span<T>：栈上分配的连续内存视图，零拷贝切片
var array = new int[] { 1, 2, 3, 4, 5 };
Span<int> slice = array.AsSpan(1, 3);   // 视图：[2, 3, 4]，不复制数据
slice[0] = 99;
Console.WriteLine(array[1]);            // 99，修改的是同一块内存
```

---

## 维度五：面向对象与抽象 {#object-oriented}

C# 是典型的面向对象语言，同时吸收了接口、委托等抽象机制。

### 类与继承

```csharp
// 基类
public class Animal
{
    // 属性：比字段更安全
    public string Name { get; set; }
    public int Age { get; init; }   // init：只能在初始化时赋值（C# 9+）

    // virtual：允许子类重写
    public virtual void Speak() => Console.WriteLine($"{Name} 发出声音");
}

// 派生类
public class Dog : Animal
{
    public override void Speak() => Console.WriteLine($"{Name}: 汪汪！");
}

var dog = new Dog { Name = "旺财", Age = 3 };
dog.Speak();   // 旺财: 汪汪！
```

### 接口与多态

```csharp
// 接口：定义契约，类可实现多个接口
public interface IShape
{
    double Area();           // 接口成员默认 public
}

public class Circle : IShape
{
    public double Radius { get; set; }
    public double Area() => Math.PI * Radius * Radius;
}

public class Rectangle : IShape
{
    public double W { get; set; }
    public double H { get; set; }
    public double Area() => W * H;
}

// 多态：统一接口，不同实现
IShape[] shapes = { new Circle { Radius = 2 }, new Rectangle { W = 3, H = 4 } };
foreach (var shape in shapes)
    Console.WriteLine(shape.Area());   // 12.57...  12
```

### 抽象类与密封类

```csharp
// 抽象类：可有实现，只能被继承不能直接实例化
public abstract class ShapeBase
{
    public abstract double Area();           // 抽象方法，子类必须实现
    public virtual string Describe() => $"面积是 {Area():F2}";
}

// 密封类：禁止被继承（C# 的 final）
public sealed class Config { /* ... */ }
```

### record：为数据而生的类型（C# 9+）

```csharp
// record：值语义的引用类型，适合不可变数据模型
public record Person(string Name, int Age);

var p1 = new Person("小明", 20);
var p2 = new Person("小明", 20);
Console.WriteLine(p1 == p2);   // True，按值比较（普通 class 是引用比较）

// with 表达式：非破坏性拷贝并修改
var p3 = p1 with { Age = 21 };
```

---

## 维度六：错误处理与资源管理 {#errors-and-resources}

健壮的程序必须妥善处理异常与资源释放。

### try-catch-finally

```csharp
try
{
    var result = int.Parse("abc");   // 抛出 FormatException
}
catch (FormatException ex)
{
    Console.WriteLine($"格式错误: {ex.Message}");
}
catch (Exception ex)                 // 兜底捕获
{
    Console.WriteLine($"未知错误: {ex.Message}");
}
finally
{
    Console.WriteLine("无论是否异常都会执行");
}
```

### 自定义异常

```csharp
public class InsufficientBalanceException : Exception
{
    public decimal Balance { get; }
    public InsufficientBalanceException(decimal balance)
        : base($"余额不足，当前余额: {balance}")
        => Balance = balance;
}

public class Account
{
    public decimal Balance { get; private set; }

    public void Withdraw(decimal amount)
    {
        if (amount > Balance)
            throw new InsufficientBalanceException(Balance);
        Balance -= amount;
    }
}
```

### using 与 IDisposable：确定性资源释放

```csharp
// using 确保资源被及时释放（文件、连接、流等）
using (var reader = new StreamReader("data.txt"))
{
    var content = reader.ReadToEnd();
}   // 离开作用域自动调用 Dispose()

// using 声明（C# 8+）：更简洁的写法
using var writer = new StreamWriter("out.txt");
writer.WriteLine("自动释放");
```

---

## 维度七：并发与异步编程 {#concurrency-and-async}

现代程序几乎都要面对多任务场景，C# 的 `async/await` 是业界标杆级的异步模型。

### Task 与 async/await

```csharp
// 异步方法：遇到 await 时挂起，不阻塞线程
static async Task<string> DownloadAsync(string url)
{
    using var http = new HttpClient();
    var content = await http.GetStringAsync(url);   // 等待 I/O，线程去干别的
    return content;
}

static async Task Main()
{
    // 并发发起多个请求，最后一起等待
    var task1 = DownloadAsync("https://example.com");
    var task2 = DownloadAsync("https://example.org");
    await Task.WhenAll(task1, task2);

    Console.WriteLine(task1.Result.Length);
}
```

### Parallel 与多线程

```csharp
// Parallel.ForEach：数据并行的简单方式
var numbers = Enumerable.Range(1, 100).ToArray();
Parallel.ForEach(numbers, n =>
{
    Console.WriteLine($"{n} 的平方 = {n * n}（线程 {Environment.CurrentManagedThreadId}）");
});

// 传统线程（较少直接使用，多由 Task/线程池托管）
var thread = new Thread(() => Console.WriteLine("后台工作"));
thread.Start();
thread.Join();   // 等待线程结束
```

### 并发集合与锁

```csharp
// ConcurrentDictionary：线程安全的字典
var cache = new ConcurrentDictionary<string, string>();
cache.TryAdd("key", "value");
var val = cache.GetOrAdd("config", _ => LoadConfig());

// lock：互斥锁，保护临界区
private static readonly object _lock = new();
private static int _counter;

static void Increment()
{
    lock (_lock)
    {
        _counter++;
    }
}
```

---

## 总结 {#summary}

| 维度 | 核心问题 | C# 的关键词 |
|------|----------|-------------|
| 类型系统与变量 | 数据如何表示？ | 值/引用类型、可空、`var` |
| 控制流 | 代码按什么顺序执行？ | `if`/`switch` 表达式、`foreach` |
| 函数与方法 | 逻辑如何复用与组合？ | 方法、`ref/out/in`、`Lambda` |
| 集合与数据结构 | 数据如何组织？ | `List`、`Dictionary`、`Span` |
| 面向对象与抽象 | 复杂度如何管理？ | 类、接口、`record`、多态 |
| 错误处理与资源管理 | 出错和资源怎么办？ | `try-catch`、`using`、自定义异常 |
| 并发与异步 | 如何同时做多件事？ | `async/await`、`Task`、`Parallel` |

掌握这七个维度，你就掌握了从"语法细节"到"工程实践"的完整脉络——不只是学会 C#，更是获得一套可以迁移到任何编程语言的学习框架。
