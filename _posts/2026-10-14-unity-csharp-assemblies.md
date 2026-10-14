---
layout: post
title: "理解 C# 与 Unity 程序集：从 asmdef、引用关系到代码提示失效"
date: 2026-10-14
categories: [游戏开发, Unity]
tags: ["C#", Unity, 程序集, asmdef, 模块化]
excerpt: "解释 public 类型为何仍然不可见，用一个最小示例理清程序集、命名空间、using、asmdef 与编辑器项目文件的关系。"
---

在 Unity 中创建了一个 `public` 类，另一个脚本却找不到它；添加 `using` 没用，换个目录似乎又能识别；修改配置后，直到回到 Unity 刷新，代码提示才恢复。这些现象通常需要从程序集边界和工具同步两个层面理解。

本文面向刚接触 C# 和 Unity 模块化的读者，以 Unity 6 的程序集机制为背景。示例是独立编写的教学代码，用于说明类型引用，不涉及业务框架实现。

## 1. 程序集是源代码编译后的组织单位

`.cs` 是源文件，程序集（Assembly）是编译后组织代码和类型信息的单位。在 Unity 的脚本编译阶段，一组 C# 文件通常会产生一个托管 `.dll`：

```text
ScoreRules.cs + ScoreRecord.cs
              ↓ C# 编译
        Tutorial.Domain.dll
```

程序集不只是方法的机器码集合。典型托管程序集包含中间语言代码、类型元数据和程序集清单，记录类型、成员及依赖等信息。Unity 最终如何把代码转换成目标平台上的可执行程序，还取决于脚本后端，不能把上述 DLL 示意理解成所有平台最终发布物的完整形态。[Microsoft：程序集内容](https://learn.microsoft.com/en-us/dotnet/standard/assembly/contents)

拆分程序集意味着给代码划定编译和访问边界。文件夹主要用于组织文件；只有配合 Unity 的程序集配置，它才会影响脚本的编译归属。

## 2. asmdef 决定哪些脚本属于同一个程序集

Unity 使用 Assembly Definition 文件，也就是 `.asmdef`，定义自建程序集。它通常管理所在目录及子目录中的脚本；子目录有自己的程序集定义时，会形成另一个边界。Assembly Definition Reference（`.asmref`）还可以让其他目录加入已有程序集，初学时可以先掌握常见的 `.asmdef` 布局。[Unity：程序集简介](https://docs.unity.com/en-us/engine/6000.0/manual/programming-environment/script-compilation/assembly-definition-files/assembly-definitions-intro)

例如：

```text
Assets/Tutorial/
├── Domain/
│   ├── Tutorial.Domain.asmdef
│   └── ScoreRules.cs
└── Presentation/
    ├── Tutorial.Presentation.asmdef
    └── ScoreDisplay.cs
```

`ScoreRules.cs` 属于 `Tutorial.Domain`，`ScoreDisplay.cs` 属于 `Tutorial.Presentation`。这两个名字来自配置文件的 `name` 字段。

没有被自定义程序集配置覆盖的普通运行时脚本，通常进入默认的 `Assembly-CSharp`。编辑器脚本还有专门规则，不能简单认为所有 `.cs` 都属于这个默认程序集。

## 3. 一个完整的跨程序集调用示例

先在 `Domain` 目录建立 `Tutorial.Domain.asmdef`：

```json
{
  "name": "Tutorial.Domain",
  "autoReferenced": true
}
```

同目录的 `ScoreRules.cs` 定义一个公开的规则类：

```csharp
namespace Tutorial.Domain
{
    public static class ScoreRules
    {
        public static bool HasReachedGoal(int score)
        {
            return score >= 100;
        }
    }
}
```

接着在 `Presentation` 目录建立 `Tutorial.Presentation.asmdef`：

```json
{
  "name": "Tutorial.Presentation",
  "references": ["Tutorial.Domain"],
  "autoReferenced": true
}
```

同目录的 `ScoreDisplay.cs`：

```csharp
using Tutorial.Domain;
using UnityEngine;

public sealed class ScoreDisplay : MonoBehaviour
{
    private void Start()
    {
        bool reached = ScoreRules.HasReachedGoal(120);
        Debug.Log($"达到目标：{reached}");
    }
}
```

把 `ScoreDisplay` 挂到活动对象上，在组件启用的情况下进入 Play，预期会在 `Start` 中输出判断结果。这里给出的是教学示例，本文未在 Unity 中执行验证。

关键配置是：**调用方 `Tutorial.Presentation` 引用提供方 `Tutorial.Domain`。** 在 Inspector 中选中调用方的 `.asmdef`，也可以通过 Assembly Definition References 添加这条引用，不一定要手写 JSON。[Unity：配置程序集引用](https://docs.unity.com/en-us/engine/6000.5/manual/scripting/compilation-and-code-reload/script-compilation/assembly-definition-files/assembly-definitions-referencing)

## 4. public、using、namespace 各自解决什么问题

跨程序集调用至少涉及两层条件：调用方拥有正确引用；被调用的类型和成员对调用方可访问。

`public` 只解决后者。把一个类声明为公开，不会自动让所有程序集引用它。相反，即使引用已经正确，类型或成员不公开，也可能无法使用。

| 概念 | 解决的问题 | 不会自动完成的事 |
|---|---|---|
| 程序集引用 | 编译时可以使用哪个代码库中的类型 | 不绕过访问权限 |
| `namespace` | 组织类型名称，减少重名 | 不创建程序集依赖 |
| `using` | 简化名称查找与书写 | 不导入缺失的程序集 |
| `public` | 允许外部访问类型或成员 | 不替调用方添加引用 |
| `internal` | 通常将访问限定在当前程序集 | 不表示当前文件夹可见 |

例如，`using Tutorial.Domain;` 允许把 `Tutorial.Domain.ScoreRules` 写成 `ScoreRules`。删掉 `using` 后，写全限定名仍然可以调用；但删掉程序集引用，写全限定名也无法补救。

`internal` 与程序集尤其相关：两个文件即使在不同文件夹，只要属于同一程序集，通常就可以互相使用其中的 `internal` 类型。反过来，命名空间相同也不能消除程序集边界。C# 另有友元程序集机制，但一般模块划分不必依赖它。[Microsoft：internal](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/internal)

命名空间和程序集不是一一对应：一个程序集可以包含多个命名空间，一个命名空间也可以分布在多个程序集。将两者命名得相似是一种便于维护的约定，不是语言要求。

## 5. Unity 默认程序集的特殊限制

Unity 不允许自定义 `.asmdef` 程序集引用预定义程序集。因此，把共享类型留在 `Assembly-CSharp` 中，再尝试让自定义模块引用它，会遇到结构性障碍。

```text
Tutorial.Presentation ──不能这样引用──→ Assembly-CSharp
                                           └── 共享类型
```

处理方式是把共享类型放入合适的自定义程序集，再由使用方引用。不要尝试通过修改自动生成的 `.csproj`，为自定义程序集强行添加 `Assembly-CSharp`；那不会改变 Unity 的真实编译规则。

另一个常见误区是 `autoReferenced`。在 `.asmdef` 上设置为 `true`，影响的是默认程序集对该自定义程序集的自动引用，**不是让所有自定义程序集彼此可见**。前面的 `Tutorial.Presentation` 仍然要显式引用 `Tutorial.Domain`。[Unity：自动引用与限制](https://docs.unity.com/en-us/engine/6000.5/manual/scripting/compilation-and-code-reload/script-compilation/assembly-definition-files/assembly-definitions-referencing)

## 6. 引用方向也是模块设计的一部分

引用不是双向关系：

```text
Tutorial.Presentation → Tutorial.Domain
```

表示界面代码依赖规则代码，并不允许规则代码反过来调用界面类型。

Unity 不允许自定义程序集形成循环引用，包括通过多个节点形成的循环。遇到环时，可以把必须共同维护的代码放回一个程序集，或者提取共享契约，再让两边依赖它。[Unity：引用限制](https://docs.unity.com/en-us/engine/6000.5/manual/scripting/compilation-and-code-reload/script-compilation/assembly-definition-files/assembly-definitions-referencing)

例如下面是一种设计建议：

```text
Presentation → Contracts
Gameplay     → Contracts
```

`Contracts` 可以保存双方共同理解的数据和接口，但不应该再反向引用这两个上层模块。否则只是把循环搬到了新的位置。

“添加引用后能编译”也不等于依赖设计合理。通用存储模块为了发送“打开商店界面”的通知而引用具体 UI 模块，虽然可能编译成功，却会使底层模块难以复用。更常见的做法是由业务层组合底层能力，并决定何时更新界面。

## 7. asmdef 和编辑器项目文件不是同一层配置

| 文件 | 主要用途 | 在 Unity 项目中的定位 |
|---|---|---|
| `.asmdef` | 划分程序集，声明引用和平台条件 | Unity 编译配置 |
| `.csproj` | 描述源文件、引用等 C# 项目信息 | 通常由 Unity 的编辑器集成生成 |
| `.sln` / `.slnx` | 将多个项目组织为解决方案 | 供开发工具加载项目集合 |
| `.meta` | 保存 Unity 资源标识等元数据 | 应与对应资源一同管理 |

Unity 的导入和编译状态，与代码编辑器的语言服务状态并不总是同时更新。新增 `.asmdef` 后，Unity 可能已经知道新的程序集边界，而编辑器仍然加载旧项目；也可能尚未刷新 Unity，双方都还在使用旧关系。

这会表现为没有悬停信息、跳转失败、自动补全缺失，甚至编辑器与 Unity Console 的判断不一致。项目文件由 Unity 生成和维护，外部编辑器集成设置位于 Preferences 的 External Tools 中。[Unity：IDE 支持](https://docs.unity.com/en-us/engine/6000.6/manual/programming-environment/environment-and-tools/scripting-ide-support)

遇到问题时可以依次检查：

1. 确认代码本身完整。方法体里只写一个类名不是合法调用；使用静态方法时要写出成员、参数和完整语句。
2. 确认目标类型存在，拼写、命名空间、类型及成员访问权限正确。
3. 查明调用方和目标类型分别属于哪个程序集，并检查依赖方向。
4. 回到 Unity，确认资产已刷新，等待导入和编译结束，先处理 Console 中的真实编译错误。
5. 如果 Unity 编译正常而编辑器仍异常，再重新生成项目文件、重载解决方案或检查 C# 语言服务。

重新生成不是每次 Git 提交前的必需步骤。提交只记录文件版本；拉取或切换分支若改变了程序集结构，才需要让 Unity 和编辑器同步新的结构。

在通常的 Unity 工作流中，`.asmdef` 及其 `.meta` 应随源代码提交；由 Unity 生成的 `.csproj`、`.sln`、`.slnx` 通常可以忽略，具体以团队约定为准。

## 8. 按模块边界拆分，不必按每个文件夹拆分

程序集适合表达稳定的模块关系，例如规则层、表现层和编辑器工具。合理拆分可以减少部分修改引起的重新编译范围，但效果取决于依赖图；被很多模块依赖的公共程序集修改后，仍可能影响大量调用方。[Unity：程序集组织](https://docs.unity.com/en-us/engine/6000.0/manual/programming-environment/script-compilation/assembly-definition-files/assembly-definitions-intro)

编辑器工具应使用只包含 Editor 平台的程序集，并引用运行时代码，而不应让运行时代码反向依赖编辑器工具。创建上级 `.asmdef` 后，尤其要重新检查其中 `Editor` 子目录的归属，不要只凭目录名称认定隔离仍然正确。

另外，`partial` 只能把同一个类型拆成多个源代码部分，这些部分必须参与同一次编译并属于同一个程序集。它不能把两个程序集中的同名类合并成一个类型。[Microsoft：分部类与方法](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/partial-classes-and-methods)

开始拆分前，可以先画出“谁调用谁”的箭头。只在职责边界明确时建立程序集，把引用限制当作检查模块关系的工具。出现“明明是 public，却找不到”的问题时，就能分别判断：是名称、访问权限、依赖配置，还是工具尚未同步。
