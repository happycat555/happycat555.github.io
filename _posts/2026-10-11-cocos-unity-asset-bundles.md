---
layout: post
title: "从 Cocos 到 Unity：理解 Bundle 的分包、构建、加载与释放"
date: 2026-10-11
categories: [游戏开发, 资源管理]
tags: [Cocos Creator, Unity, AssetBundle, Addressables, "C#", TypeScript]
excerpt: "对照 Cocos Creator 与 Unity 的 Bundle 工作流程，从分包配置和构建产物出发，理解资源定位、批量加载、依赖管理与生命周期，并通过 cube 示例读懂 Unity 清单。"
---

学习资源管理时，最容易产生的误解，是把 Bundle 当成一个带名字的文件夹：设置好名字，运行时按名字读取即可。实际上，这个简单操作背后还存在构建产物、资源索引、依赖关系、缓存和生命周期管理。

Cocos Creator 和 Unity 都有 Bundle，但提供的抽象层次不同。迁移资源框架时，需要迁移职责，而不是逐个替换 API 名称。

本文以使用 `cc.assetManager` 的 **Cocos Creator 2.4 风格 API**，以及练习项目使用的 **Unity 6 原生 AssetBundle** 为基础。Cocos Creator 3.x 的导入方式和部分接口不同，不宜直接复制本文的 TS 示例。Addressables 则作为 Unity 的另一种管理方案单独讨论。

## 一、先分清资源、资源包和场景实例

以背包界面为例，项目里可能有一个背包预制体，以及它使用的图标、材质和音效。

- **资源**：预制体、纹理、音频等可加载的数据。
- **资源包**：用于组织、发布和按需获取资源的一组构建产物。
- **实例**：由预制体创建出来、真正出现在场景里的对象。

对应的流程是：

```text
配置分包 → 构建资源包 → 加载包 → 加载资源 → 创建实例
```

这几个阶段不能合并理解。加载了包，不代表包里的所有资源已经成为可使用的对象；加载了预制体，也不代表场景里已经出现了一个实例。

例如，Unity 中：

```csharp
AssetBundle bundle = AssetBundle.LoadFromFile(bundleFilePath);
GameObject prefab = bundle.LoadAsset<GameObject>(assetName);
GameObject instance = UnityEngine.Object.Instantiate(prefab);
```

三行分别完成包加载、资源加载和实例化。销毁 `instance` 也不等于立即释放预制体、贴图或整个 Bundle。

## 二、两种 Bundle，解决同一问题的不同层次

| 对比项 | Cocos Creator 2.4 Asset Bundle | Unity 原生 AssetBundle |
|---|---|---|
| 分包入口 | 文件夹上的 Bundle 配置 | 资源或文件夹的 AssetBundle 分配，也可通过构建脚本指定 |
| 构建方式 | 通常集成在 Creator 的项目构建流程中 | 显式调用 `BuildPipeline.BuildAssetBundles()`，也可接入自动化流程 |
| 输出形态 | 与平台、压缩设置有关，常包含配置、资源文件、脚本等 | 平台相关的二进制资源包，以及清单产物 |
| 查找已加载包 | `cc.assetManager.getBundle(name)` | 遍历 `GetAllLoadedAssetBundles()`，或维护自己的字典 |
| 首次加载包 | `loadBundle(name或URL, callback)` | 本地文件、内存、流，或通过网络下载加载 |
| 指定多个资源 | `bundle.load()` 支持路径数组 | 通常逐项调用加载 API，再汇总结果 |
| 资源引用计数 | `cc.Asset` 提供 `addRef()` / `decRef()` | `UnityEngine.Object` 没有同等接口 |
| 更完整的管理层 | Asset Manager 已提供不少管理能力 | 可自行封装，也可使用 Addressables |

这里尤其要避免把 Cocos 的 Bundle 理解为“一定对应一个二进制文件”。它的输出形式受构建配置和平台影响。Unity 原生 AssetBundle 则以序列化资源归档为核心；它不会替你分发和执行新的 C# 程序集，资源引用的脚本类型通常需要已经存在于 Player 中。

Cocos 的配置、输出与加载方式参见[官方 Asset Bundle 文档](https://docs.cocos.com/creator/2.4/manual/zh/scripting/asset-bundle.html)。Unity 原生构建流程参见[AssetBundle workflow](https://docs.unity3d.com/2023.2/Documentation/Manual/AssetBundles-Workflow.html)。

## 三、设置包名只是分配规则，构建才生成实际内容

### 3.1 Cocos：在文件夹上配置，再随项目构建输出

在 Creator 中选中文件夹，启用 Bundle 配置，可以设置包名、优先级以及平台相关选项。构建器读取这些设置，生成对应的资源索引和资源内容。

运行时能够写出：

```ts
cc.assetManager.loadBundle('ui', (err, bundle) => {
    if (err) {
        cc.error(err);
        return;
    }

    bundle.load('prefabs/BagPanel', cc.Prefab, (error, prefab) => {
        if (error) {
            cc.error(error);
            return;
        }
        const node = cc.instantiate(prefab);
        cc.director.getScene().addChild(node);
    });
});
```

并不是引擎凭空知道 `ui` 在哪里，而是构建配置及运行时加载系统提供了定位信息。远程包还需要相应的服务器地址、URL 或版本配置。

`loadBundle()` 主要建立包的入口、读取配置及相关脚本，不意味着一次把包里的所有图片、音频、预制体都加载完毕。随后还需要 `bundle.load()` 等接口获取具体资源。[Cocos 配置与加载说明](https://docs.cocos.com/creator/2.4/manual/en/scripting/asset-bundle.html)

### 3.2 Unity：分包规则、构建包、构建游戏是三个步骤

假设给资源设置：

```text
BagPanel.prefab  → ui
ShopPanel.prefab → ui
Click.wav       → audio
```

设置只记录资源归属。调用构建 API 后，才按照这些规则生成 `ui`、`audio` 等文件。

```csharp
// 编辑器代码，需要放在 Editor 文件夹或编辑器专用程序集内。
BuildPipeline.BuildAssetBundles(
    outputPath,
    BuildAssetBundleOptions.ChunkBasedCompression,
    EditorUserBuildSettings.activeBuildTarget
);
```

然后才是构建游戏程序。如果将已生成的文件放进 `StreamingAssets`，它们会随 Player 一起发布；如果放在远程服务器，游戏则需要下载。

```text
设置包名：规定资源归属
    ↓
构建 AssetBundle：处理、分类、序列化并压缩资源
    ↓
构建 Player：生成游戏，并携带配置为随包发布的文件
    ↓
运行游戏：由加载代码读取包和资源
```

因此，**不是先生成清单，再由 Player 构建根据清单决定如何分包**。生成 Bundle 时，资源分类和实际打包就已经完成了。普通 Player Build 也不会自动运行我们自己编写的 `BuildBundles` 菜单方法。

练习项目的工具位于 `Assets/Editor/BuildBundles.cs`。它按当前目标平台构建，并输出到 `Assets/StreamingAssets`，与现有加载器的路径相匹配。构建目标应与实际运行平台一致；切换平台后应重新构建相应资源包，避免混用产物。

## 四、通过 cube 示例读懂 Unity 的输出文件

本次练习生成了以下文件：

```text
Assets/StreamingAssets/
├── cube
├── cube.manifest
├── StreamingAssets
├── StreamingAssets.manifest
└── 上述文件对应的 .meta
```

### 4.1 cube：真正的资源包

`cube` 是约 267 KiB 的二进制文件。它的清单列出的显式资源为：

```text
Assets/Bundle/Cube/cube.prefab
```

真正需要打开并读取预制体内容的是 `cube`，不是 `cube.manifest`。

### 4.2 cube.manifest：资源包的文本记录

该文件记录构建版本、CRC、哈希、涉及的对象类型、显式资源列表和外部包依赖等信息。当前内容包括：

```yaml
Assets:
- Assets/Bundle/Cube/cube.prefab
Dependencies: []
```

`Dependencies: []` 表示没有依赖其他 AssetBundle，不表示预制体没有网格、材质等资源依赖。相关内容可以包含在当前包中，也可以属于引擎内置资源。

`Assets` 列表也不应被理解为包内所有底层序列化对象的完整清单。

### 4.3 StreamingAssets：总清单包

目录里面这个没有扩展名的 `StreamingAssets` 是文件，不是外层文件夹。它采用输出目录的名称，内部保存 `AssetBundleManifest` 对象。

运行时可以读取它，查询包名、哈希和依赖：

```csharp
AssetBundle manifestBundle = AssetBundle.LoadFromFile(
    System.IO.Path.Combine(Application.streamingAssetsPath, "StreamingAssets")
);

if (manifestBundle == null)
    throw new System.InvalidOperationException("总清单包加载失败");

AssetBundleManifest manifest =
    manifestBundle.LoadAsset<AssetBundleManifest>("AssetBundleManifest");

if (manifest == null)
    throw new System.InvalidOperationException("总清单对象不存在");

string[] dependencies = manifest.GetAllDependencies("cube");
```

这是 Windows 本地读取的示意代码，实际项目还要把清单包纳入统一生命周期管理。当前 `cube` 的依赖列表为空。

### 4.4 StreamingAssets.manifest：总清单的文本形式

当前文件记录了一项包信息：`cube`，没有其他包依赖。它适合阅读和构建工具处理，不能把它直接当作二进制 AssetBundle 加载。

### 4.5 .meta：编辑器的身份与导入记录

这些文件是 Unity 导入 `Assets` 下文件时生成的，主要记录 GUID、导入器等元数据。它们既不是包内资源数据，也不是运行时依赖清单。

### 4.6 没有清单是否还能加载？

需要区分具体缺少什么：

| 缺少内容 | 影响 |
|---|---|
| 没有实际的 `cube` 文件 | 无法从该路径加载资源包 |
| 发布时没有文本 `.manifest` | 原生运行时加载 API 通常不需要它 |
| 删除构建输出中的文本清单 | 影响增量构建，后续可能需要重新构建 |
| 没有总清单包 | 仍能按路径加载包，但需要其他方式掌握依赖 |

文本总清单还可以通过 `BuildPlayerOptions.assetBundleManifestPath` 提供给 Player 构建，帮助保留只在 Bundle 中用到的类型，避免相关剥离问题。不能因为运行时不直接读取文本清单，就认定它在整个构建流程中没有价值。[Unity 构建 API](https://docs.unity3d.com/cn/2023.2/ScriptReference/BuildPipeline.BuildAssetBundles.html)、[清单文件说明](https://docs.unity3d.com/Manual/assetbundles-file-format.html)

## 五、包名、包文件路径、包内资源名，必须分开

这三个名称分别回答不同问题：

| 名称 | 示例 | 回答的问题 |
|---|---|---|
| 包名 | `cube` | 查找、管理哪个包？ |
| 包文件路径 | `StreamingAssets/cube` 的完整磁盘路径 | 从哪里打开文件？ |
| 包内资源名 | `assets/bundle/cube/cube.prefab` | 从包中获取哪个资源？ |

### 5.1 为什么 Cocos 能直接按名字查找？

```ts
const bundle = cc.assetManager.getBundle('ui');
```

这里只查询已加载包的缓存，不会发起文件加载。如果没有找到，才通过 `loadBundle()` 发起加载。

### 5.2 Unity 也能查询已加载的包

```csharp
AssetBundle loadedBundle = null;
foreach (AssetBundle bundle in AssetBundle.GetAllLoadedAssetBundles())
{
    if (bundle.name == bundleName)
    {
        loadedBundle = bundle;
        break;
    }
}
```

初学时可以这样写。后续需要频繁查询时，可以维护 `Dictionary<string, AssetBundle>`，在成功加载时登记，在卸载时移除。

仅记录加载成功的包还不够：多个请求可能同时发现包不存在，然后重复发起加载。完整管理器还应记录“正在加载”的请求，让相同包的调用者共享结果。[Unity 已加载包查询 API](https://docs.unity3d.com/cn/6000.0/ScriptReference/AssetBundle.GetAllLoadedAssetBundles.html)

### 5.3 Path.Combine 只负责文件路径拼接

```csharp
string bundlePath = System.IO.Path.Combine(
    Application.streamingAssetsPath,
    bundleName
);
```

它不会构建、查找或加载 Bundle，只是把目录和文件名拼成路径。不要把文件系统路径拼接规则直接用于 HTTP URL，也不要把包内资源名当成本地文件路径处理。

当前 Windows 练习可以从 `StreamingAssets` 使用本地文件加载。跨平台时必须重新检查访问方式，尤其 Android、Web 平台的 StreamingAssets 不能统一按普通目录处理，通常要考虑 `UnityWebRequest` 等平台适配。[StreamingAssets 官方说明](https://docs.unity3d.com/cn/2023.2/Manual/StreamingAssets.html)

### 5.4 Cocos 的相对资源路径不能直接照搬

Cocos 通常传相对于 Bundle 根目录、不带扩展名的路径。Unity 原生 AssetBundle 默认可使用构建时的资源路径，也可以通过 `AssetBundleBuild.addressableNames` 指定自定义名称。

最可靠的确认方式是打印实际资源名：

```csharp
foreach (string assetName in bundle.GetAllAssetNames())
    Debug.Log(assetName);
```

练习加载器使用“第一段作为包名，其余部分作为包内名称”的约定。那么 cube 示例可表示为：

```text
cube/assets/bundle/cube/cube.prefab
│    └─ 包内资源名
└─ 包名
```

这属于项目协议，不是引擎强制格式。该协议还默认包名本身不包含 `/`；若以后采用 `environment/forest` 这样的包名，按第一个斜杠拆分就会出错。长期接口可以把 `bundleName` 和 `assetName` 作为两个参数传递。

关于资源名称与自定义名称，参见[Unity LoadAsset 文档](https://docs.unity3d.com/2023.2/Documentation/ScriptReference/AssetBundle.LoadAsset.html)。

## 六、异步加载包，不代表加载资源也是异步

项目中使用的结构是：

```csharp
AssetBundleCreateRequest request = AssetBundle.LoadFromFileAsync(bundlePath);
request.completed += operation =>
{
    AssetBundle bundle = request.assetBundle;
    if (bundle == null)
    {
        // 调用者应通过失败回调收到通知。
        return;
    }

    // 从这里继续加载包内资源。
};
```

`completed` 表示操作结束，不代表成功。必须检查结果。

如果回调中接着调用 `bundle.LoadAsset()`，包内资源仍然同步加载。大型资源可能造成主线程停顿；需要异步时，应进一步使用 `LoadAssetAsync<T>()`，并处理完成结果。

原生 Bundle 不支持像普通预制体那样用 `LoadAsset()` 加载场景。场景包需要配合 `SceneManager.LoadSceneAsync()` 使用。[Unity AssetBundle API](https://docs.unity3d.com/cn/6000.0/ScriptReference/AssetBundle.html)

## 七、批量加载：统一接口，明确回调契约

Cocos 的 `bundle.load()` 支持路径数组，而 Unity 原生 `LoadAsset()` 一次加载一个指定资源。Unity 有加载全部资源的接口，但“加载包里全部资源”与“加载调用者指定的一组路径”并不相同。

迁移时可以在自己的加载器中遍历路径，并统一返回数组：

```csharp
public delegate void CompletedCallback(
    System.Exception error,
    UnityEngine.Object[] resources,
    object parameters
);
```

下面是同步批量处理的示意函数。它假设资源包及依赖已加载、参数已验证，并让所有资源使用同一种类型：

```csharp
void LoadAssets(
    AssetBundle bundle,
    string[] assetNames,
    System.Type assetType,
    CompletedCallback onComplete)
{
    var resources = new UnityEngine.Object[assetNames.Length];
    try
    {
        for (int i = 0; i < assetNames.Length; i++)
        {
            resources[i] = bundle.LoadAsset(assetNames[i], assetType);
            if (resources[i] == null)
                throw new System.InvalidOperationException(
                    "资源加载失败：" + assetNames[i]);
        }
    }
    catch (System.Exception error)
    {
        onComplete?.Invoke(error, null, null);
        return;
    }

    onComplete?.Invoke(null, resources, null);
}
```

完成回调应位于循环外。成功时，返回数组与输入顺序一致；失败时，通知调用者并终止该批次。单资源接口可以包装成长度为 1 的数组，复用批量逻辑。

这个失败策略不等于自动回滚：失败之前已经加载的资源可能仍然留在内存或包管理结构中。是否缓存、释放或继续返回部分成功结果，是加载器需要明确的策略。

还有一个值得从练习代码中修正的设计：`UnityEngine.Object[] type` 实际表示资源对象数组，不是类型数组。通过对象调用 `GetType()` 虽然可以取得类型，但要求调用者先提供一个对象，不适合作为常规加载接口。

更自然的选择是：

- 一批同类型资源：传 `System.Type`，例如 `typeof(Sprite)`。
- 每条路径类型不同：传与路径一一对应的 `System.Type[]`。
- 希望返回明确类型：使用泛型 `LoadRes<T>() where T : UnityEngine.Object`。

C# 的委托、`Action`、可选参数和重载可以用来定义清晰的加载接口。

## 八、依赖管理决定分包是否真正有效

假设背包和商店都使用同一张图集。

```text
背包包 ─┐
        ├─ 公共图集
商店包 ─┘
```

在 Unity 原生 AssetBundle 中，如果公共资源没有单独分配，构建时可能被复制进多个包。这样会增加包体，还可能产生多份独立的运行时资源对象。

把公共资源放入 `shared` 包，可以建立明确依赖：

```text
ui_bag  → shared
ui_shop → shared
```

代价是加载器必须处理它。通过总清单查询依赖后，需要确保所有必要依赖包在调用 `LoadAsset` 前都已加载。可以先加载依赖包再加载主包；关键条件是资源解析前依赖可用，不是只有一种固定的包文件打开顺序。

`LoadFromFileAsync()` 不会仅因为总清单在同一个目录，就自动递归替你加载依赖。[Unity 依赖与重复资源说明](https://docs.unity3d.com/cn/2023.1/Manual/AssetBundles-Troubleshooting.html)

Cocos 也不能概括为“依赖全部自动处理”。Creator 2.4 的 Bundle 优先级会影响共享资源归属与重复：低优先级包可能依赖高优先级包；使用相关共享资源前，要遵守依赖包的加载要求。同优先级共享资源也可能产生复制。应区分引擎对资源内部依赖的处理，以及应用对包加载顺序的安排。[Cocos Asset Bundle 依赖与优先级](https://docs.cocos.com/creator/2.4/manual/en/asset/bundle.html)

## 九、资源引用计数不能直接按类型迁移

### 9.1 Cocos 的资源引用计数

Cocos Creator 2.4 的 `cc.Asset` 提供 `addRef()` 和 `decRef()`。应用需要持续使用资源时，应按持有约定增加引用；结束使用时，配对减少引用。

普通变量赋值与引擎引用计数是两回事。将资源保存到变量或集合中，不代表引擎会自动为这一业务用途增加引用。多个使用者共享资源时，每个使用者都需要遵循一致的持有与释放约定。[Cocos Asset Manager](https://docs.cocos.com/creator/2.4/manual/en/asset-manager/)

### 9.2 Unity Object 列表不等于引用计数管理器

常见类型可以做概念对应：

| Cocos | Unity |
|---|---|
| `cc.Asset` | 常见资源可统一接收为 `UnityEngine.Object`，但后者也包含场景对象 |
| `cc.Prefab` | 预制体通常加载为 `GameObject` |
| `cc.SpriteFrame` | `Sprite` |
| `cc.Sprite` | UI 中通常使用 `Image`，场景中使用 `SpriteRenderer` |
| `cc.AudioClip` | `AudioClip` |

但不能因为列表改为 `List<UnityEngine.Object>`，就认为 Unity 自动提供了 Cocos 的 `addRef()` / `decRef()`。

原生 AssetBundle 的卸载需要明确策略：

- `Unload(false)`：卸载包的相关数据结构，保留已经加载的资源对象；不能再通过该包对象继续加载。
- `Unload(true)`：同时卸载从该包加载的对象，可能使仍在使用它们的实例出现缺失资源。

共享依赖仍被使用时，不应随某一个界面的关闭卸载。C# GC 也不能替代这些 Unity 原生资源的生命周期安排。[Unity Unload 文档](https://docs.unity3d.com/ScriptReference/AssetBundle.Unload.html)

Cocos 的 `removeBundle()` 同样不等于释放包内所有已加载资源；包入口与资源生命周期需要分别理解。[Cocos Bundle 移除说明](https://docs.cocos.com/creator/2.4/manual/en/scripting/asset-bundle.html)

## 十、Addressables 在这张图中的位置

Unity 原生 AssetBundle 是较底层的资源容器与访问 API。Addressables 在它之上增加了资源地址、分组、依赖加载以及加载句柄的引用计数等机制。

Addressables 的 Group 可以配置一起打包、分开打包或按标签组合打包。一个 Group 不一定只生成一个 Bundle，场景与普通资源等情况也会影响拆分。它也不使用原生 BuildPipeline 输出的同一套 `.manifest` 与总清单包流程，而是依赖自己的目录和定位数据。[Addressables 分包规则](https://docs.unity3d.com/Packages/com.unity.addressables@1.21/manual/PackingGroupsAsBundles.html)、[Unity 构建输出](https://docs.unity3d.com/cn/current/Manual/AssetBundles-Building.html)

如果使用 Addressables，资源使用者应记录由自己负责释放的加载句柄，并将加载与 `Addressables.Release()` 配对。释放引用不保证内存当场回收，具体还受包及其他使用者影响。[Addressables 内存管理](https://docs.unity3d.com/Packages/com.unity.addressables@1.21/manual/MemoryManagement.html)

学习原生 AssetBundle 有助于理解底层机制；选择 Addressables 后，则应围绕它的所有权协议设计封装，避免对它管理的包直接调用原生卸载方法。

## 十一、用当前项目完成一次最小验证

可以按下面顺序验证每一层，而不是一开始就实现完整热更新系统：

1. 给 `Assets/Bundle/Cube/cube.prefab` 设置包名 `cube`。
2. 执行 `Tools → AssetBundles → Build for Current Platform`。
3. 在 `StreamingAssets` 中确认 `cube` 和清单已生成。
4. 查看 `cube.manifest`，确认显式资源路径与包依赖。
5. 使用本地包路径加载 `cube`。
6. 打印 `GetAllAssetNames()`，用实际名称加载预制体。
7. 调用 `Instantiate()`，确认场景实例出现。
8. 再引入第二个资源，验证结果数组顺序和一次完成回调。
9. 最后把公共资源分到另一个包，验证依赖查询和加载。

当前项目已经具备构建入口、已加载包查找、异步打开包、同步加载包内资源和批量结果回调。它仍是学习中的加载器，需要逐步补上空路径检查、类型参数设计、失败回调、并发请求复用、依赖加载和释放策略。

实践中还要关注两个问题：第一，改名或删除分包设置后，构建输出目录中可能残留旧文件；第二，资源同时被 Player 场景直接引用并放入独立 Bundle，可能产生重复内容。正式发布前应检查产物和依赖，而不只看构建是否成功。

理解完整流程后，两个引擎的差别就比较清楚了：Cocos 的 Bundle API 已经结合了较多构建配置与资源管理能力；Unity 原生 AssetBundle 把更多调度工作留给上层。迁移时真正需要保持一致的，是资源定位规则、加载完成的定义，以及“谁持有、谁释放”的约定。
