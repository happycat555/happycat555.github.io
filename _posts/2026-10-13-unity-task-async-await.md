---
layout: post
title: "Unity 中的 Task、async 与 await"
date: 2026-10-13
categories: [游戏开发, Unity开发]
tags: [Unity, "C#", Task, "async/await", Addressables, 异步编程]
excerpt: "梳理 Unity 中 Task、async 与 await 的关系，对比协程，并结合资源加载说明异常处理、对象生命周期和资源释放的边界。"
---

本文回答两个问题：

1. Task 在 Unity 游戏开发中用得多吗？
2. Task 和 async 相比，有什么优点和缺点？

<details markdown="1">
<summary>文章目录 · Unity 异步编程</summary>

- [1. Task 在 Unity 游戏开发中常见吗？](#section-1)
- [2. Task 和 async 不是替代关系](#section-2)
- [3. Task + async/await 的优点](#section-3)
- [4. Task + async/await 的局限](#section-4)
- [5. 如果想比较 Task 和 Unity 协程](#section-5)
- [6. 当前项目的选择](#section-6)

</details>

## 1. Task 在 Unity 游戏开发中常见吗？ {#section-1}

Task 是 Unity 项目中常见的异步方案之一，尤其适合这些场景：

- 网络请求、登录、存档读写。
- 资源加载和游戏初始化流程。
- 多个异步操作的组合与等待。
- 接入返回 Task 的 .NET SDK。

Unity 项目也会使用协程、Unity 原生 `Awaitable` 或第三方 `UniTask`。具体选择取决于 Unity 版本、已有代码和业务需求，不能认为所有项目都以 Task 为默认方案。

每帧移动、战斗数值更新等持续运行的逻辑，通常放在 `Update` 或项目自己的更新循环中，不需要为了“异步”而改用 Task。

## 2. Task 和 async 不是替代关系 {#section-2}

`Task` 是表示异步操作的类型，`async` 是方法修饰符，`await` 是等待异步操作的运算符。它们通常配合使用，不能直接比较谁比谁更好。

```csharp
using System.Threading.Tasks;

public async Task<int> GetValueAsync()
{
    await Task.Delay(100);
    return 42;
}
```

| 写法 | 含义 |
| --- | --- |
| `Task<int>` | 返回一个任务，任务成功完成后可以取得一个整数 |
| `async` | 允许方法内部使用 `await` |
| `await` | 如果操作尚未完成，暂时让出执行权，完成后继续执行后面的代码 |

调用方可以继续等待结果：

```csharp
int value = await GetValueAsync();
```

这里的 `Task.Delay` 仅用于演示异步等待，不遵循 Unity 的 `Time.timeScale`，不应直接当作游戏时间等待接口使用。

### 返回 Task 不一定要写 async

当方法只是转交另一个异步操作时，可以直接返回 Task：

```csharp
public Task<int> GetValueForwardedAsync()
{
    return GetValueAsync();
}
```

调用方仍然可以 `await GetValueForwardedAsync()`。外层方法不需要再写一次 `async` 和 `await`。

如果需要在操作完成后继续处理结果，或在外层捕获异步异常、安排跨越等待过程的清理，就可以使用 `async/await`。

### async 也可以搭配其他返回形式

`async` 不只适用于 Task。Unity 的 `Awaitable`、第三方 `UniTask` 也能与 `async/await` 配合使用。

还有一种写法是 `async void`：

```csharp
private async void Start()
{
    try
    {
        int value = await GetValueAsync();
        UnityEngine.Debug.Log(value);
    }
    catch (System.Exception error)
    {
        UnityEngine.Debug.LogException(error);
    }
}
```

`async void` 没有可返回给调用方的任务，因此调用方无法等待它完成，也无法通过任务接收异常。业务接口通常优先返回 `Task` 或 `Task<T>`；事件处理、必要的 Unity 生命周期入口可以使用 `async void`，并在内部处理异常。

## 3. Task + async/await 的优点 {#section-3}

### 执行顺序容易阅读

异步代码可以按“先做什么、再做什么”的顺序书写，减少多层回调嵌套。

### 方便返回结果和组合操作

`Task<T>` 可以返回结果；`Task.WhenAll` 等方法可以组合多个任务，等待它们全部完成。WhenAll 本身不会自动创建线程，也不会因某个任务失败就取消其他任务。

### 异常可以通过等待链传递

对于以 Task 故障状态报告错误的 API，调用方可以在 `await` 周围使用 `try/catch`。

具体 API 仍有自己的约定。例如本项目的 Addressables 4.1.1 加载使用 `await handle.Task`，加载失败需要检查 `handle.Status` 或 `handle.OperationException`，不能只依赖 `catch`。

### 与 .NET 生态衔接方便

许多网络、文件和第三方 SDK 接口使用 Task，可以直接组合到同一套异步流程中。

## 4. Task + async/await 的局限 {#section-4}

### 不会自动管理 Unity 对象的生命周期

对象销毁不会自动取消一个普通 Task。等待恢复后，目标对象可能已经不存在，需要业务处理结果失效、取消和资源回收。

取消也需要底层操作支持，或由调用方停止交付结果；它不一定能中断已经开始的下载或加载。

### 不会自动释放资源

Task 完成只表示操作结束，不代表资源不再被使用。Addressables 引用仍需要在正确的时机释放。

### 高频使用需要关注分配开销

Task、异步状态机及相关委托可能产生分配。每帧、大量对象反复创建异步操作时，应结合 Profiler 判断是否需要调整。

偶发的资源加载不能与每帧大量异步操作混为一谈，也不能简单认为换成协程就没有分配。

### Unity 的帧和游戏时间需要专用等待机制

标准 Task 不直接表达“下一帧”“固定更新之后”或“按游戏时间等待”。这些需求通常使用协程、Unity Awaitable 或 UniTask 的对应接口。

### 异步不等于后台线程

`async/await` 不会自动创建线程。方法在遇到尚未完成的等待之前，代码仍同步执行；重计算如果仍在主线程执行，一样会卡顿。

`Task.Run` 才是把工作安排到线程池的常见方式之一，但多数 Unity API 必须在主线程调用，不能随意放进 `Task.Run`。

## 5. 如果想比较 Task 和 Unity 协程 {#section-5}

| 方面 | Task + async/await | Unity 协程 |
| --- | --- | --- |
| 返回结果 | `Task<T>` 可直接交付结果 | 通常通过字段或回调交付 |
| 多操作组合 | `WhenAll` 等方式较方便 | 通常需要自行组织 |
| 错误传递 | 故障 Task 可沿等待链传播 | 跨协程传递错误通常需要额外约定 |
| 逐帧等待 | 需要相应的 Unity 等待机制 | `yield return null` 很直接 |
| 游戏时间等待 | 标准 `Task.Delay` 不遵循 `timeScale` | `WaitForSeconds` 可按缩放后的游戏时间等待 |
| 对象生命周期 | 普通 Task 不随对象销毁自动取消 | 对象销毁或 GameObject 失活会停止其协程；仅禁用 MonoBehaviour 不会自动停止 |
| 资源释放 | 需要显式安排 | 停止协程同样不等于释放底层资源 |
| 性能 | 高频使用可能产生分配 | 迭代器和部分等待对象也可能产生分配 |

两者都不会因为“正在等待”就持续阻塞 Unity 主线程，但等待前后执行的同步代码仍可能占用主线程。

## 6. 当前项目的选择 {#section-6}

当前资源框架使用 Task 是合理的，因为接口需要表达“内部等待加载结束，再把原生句柄返回给调用方”：

```csharp
var handle = await ResLoader.LoadAsync<UnityEngine.Sprite>(
    ResPath.MonsterIcon("slime_green"));

if (handle.Status != UnityEngine.ResourceManagement.AsyncOperations.AsyncOperationStatus.Succeeded)
{
    UnityEngine.Debug.LogException(handle.OperationException);
    ResLoader.Release(handle);
    return;
}

UnityEngine.Sprite sprite = handle.Result;
// 使用 sprite，自行保存 handle。
// 所有使用者都不再使用资源后，再调用 ResLoader.Release(handle)。
```

本项目的职责划分为：

- `ResLoader`：加载资源、等待操作结束、返回原生句柄，以及提供释放入口。
- `ResKeeper`：管理通过它设置的图片引用，处理换图、迟到结果和销毁后的清理。
- Task：让调用方等待方法结束，取得结果或接收方法抛出的异常。

当前阶段应优先保证资源释放和对象生命周期处理正确。如果以后出现大量逐帧异步逻辑，再根据实际开销和项目习惯评估 Awaitable 或 UniTask。
