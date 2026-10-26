---
layout: post
title: "Cocos Creator 开发知识地图：从 JavaScript 原理到小游戏性能优化"
date: 2026-10-26
categories: [游戏开发, Cocos Creator]
tags: [Cocos Creator, TypeScript, JavaScript, 性能优化, 内存管理, 小游戏, 面试]
excerpt: "围绕语言执行、节点生命周期、渲染与资源成本、小游戏运行边界，系统梳理 Cocos Creator 开发与面试中的核心概念、常见误区和排查方法。"
---

一个背包界面第一次打开时卡顿，关闭后内存没有下降，重复进入还会收到多次刷新事件。这几个现象看起来分别属于性能、内存和业务逻辑，背后却可能是同一个问题：对象的创建、使用、订阅和释放没有围绕生命周期组织起来。

学习 Cocos Creator 时，把知识拆成孤立的 API 很容易，真正困难的是理解它们如何共同影响运行结果。本文沿着四条主线建立知识地图：JavaScript 如何执行代码，Cocos 如何组织对象，渲染和资源如何消耗时间与内存，以及小游戏平台如何限定运行环境。

引擎部分以 **Cocos Creator 3.8、2D/UI 场景**为参考。2.4 的命名空间、资源接口和坐标 API 存在差异，不能直接混用。代码均为通用教学示例；标注为局部片段的内容省略了项目配置和外围业务，并未作为完整 Cocos 项目编译验证。平台规则核对日期为 2026 年 10 月 26 日，发布产品时仍需检查对应平台与接入方案的最新文档。

## 一、语言基础：理解引用、回调与执行时机

### 1. 闭包为什么能保留状态

闭包使函数能够继续访问定义时的外部词法环境，即使外层函数已经返回。

```ts
function createCounter() {
    let count = 0;

    return () => {
        count += 1;
        return count;
    };
}

const next = createCounter();
console.log(next()); // 1
console.log(next()); // 2
```

这里的状态没有放在全局变量里，而是由返回的函数继续访问。游戏中的按钮回调、资源加载回调、Promise 和定时器都经常使用闭包。

闭包本身不会天然造成泄漏。真正的问题是：一个已经不需要的对象，仍被长期存活的对象间接持有。例如下面的局部片段：

```ts
globalBus.on("refresh", () => {
    this.refreshPanel();
});
```

可能形成这样的引用链：

```text
全局事件中心 → 监听函数 → 面板组件 → 面板关联的数据
```

如果面板关闭后监听仍未注销，组件和相关数据可能继续存活。排查时应沿着引用链寻找持有者，并明确订阅应该在哪个生命周期结束。

一个常见错误是对注册和注销分别调用 `bind()`：

```ts
bus.on("refresh", this.refresh.bind(this));
bus.off("refresh", this.refresh.bind(this));
```

两次 `bind()` 返回不同函数，通常无法匹配。可以保存绑定后的函数，或者在 Cocos `EventTarget` 中使用同一组回调与目标参数：

```ts
bus.on("refresh", this.refresh, this);
bus.off("refresh", this.refresh, this);
```

### 2. 原型链与 this 分别解决什么问题

原型链决定对象找不到属性时去哪里继续查找。

```ts
class Enemy {
    hp = 100;

    attack() {
        console.log("attack");
    }
}

const enemy = new Enemy();
```

`hp` 是实例属性，普通类方法 `attack` 位于 `Enemy.prototype`，可以由实例共享。属性查找大致沿着下面的路径进行：

```text
enemy → Enemy.prototype → Object.prototype → null
```

构造函数的 `prototype` 属性与对象内部的 `[[Prototype]]` 不是同一个概念。前者通常参与创建实例的原型关系；后者是对象实际指向的原型，可以用 `Object.getPrototypeOf()` 查看。[MDN：继承与原型链](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Inheritance_and_the_prototype_chain)

`this` 则主要与调用方式有关：

| 写法 | 普通函数中的 this |
|---|---|
| `obj.fn()` | 通常为 obj |
| 严格模式下直接 `fn()` | undefined |
| `fn.call(obj)`、`fn.apply(obj, args)` | 显式指定的 obj |
| `fn.bind(obj)` | 返回一个绑定 this 的新函数 |
| `new Fn()` | 新创建的实例，另有构造返回值规则 |

箭头函数没有自己的 `this`，而是使用定义处外层的 `this`，不能再用 `call()` 重新绑定。[MDN：this](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/this)

```ts
const enemy = {
    hp: 100,
    printHp() {
        console.log(this.hp);
    },
};

enemy.printHp();

const print = enemy.printHp;
// 严格模式下，直接 print() 会失去 enemy 这个调用者。
```

在 Cocos 中，监听方法常通过第三个参数指定回调目标：

```ts
this.node.on(Node.EventType.TOUCH_END, this.onClick, this);
```

还要区分普通类方法和箭头函数字段。`attack() {}` 通常在原型上共享，而 `onHit = () => {}` 会为每个实例创建函数。后者绑定方便，但大量实例中也会产生额外分配。

### 3. 参数按值传递，值可以是对象引用

JavaScript 的基本类型包括 `number`、`string`、`boolean`、`undefined`、`null`、`bigint` 和 `symbol`。对象、数组和函数属于对象类型。

```ts
const a = { hp: 100 };
const b = a;

b.hp = 50;
console.log(a.hp); // 50
```

赋值复制了访问同一个对象的引用值，没有复制对象本身。但这不意味着函数参数是按引用传递：

```ts
function change(data: { hp: number }) {
    data.hp = 50;
    data = { hp: 0 };
}

const enemy = { hp: 100 };
change(enemy);
console.log(enemy.hp); // 50
```

修改属性影响共享对象；给参数重新赋值，只改变函数内部的参数变量。游戏中应特别注意配置与运行时状态的区别：`runtimeData = configData` 并不会创建独立状态，随后修改属性可能污染公共配置。

### 4. 浅拷贝、深拷贝与引擎对象复制

浅拷贝只复制第一层：

```ts
const original = {
    hp: 100,
    position: { x: 10, y: 20 },
};

const copy = { ...original };
copy.hp = 50;          // 不影响 original.hp
copy.position.x = 99; // 会影响 original.position.x
```

对象展开、`Object.assign()`、数组展开和 `slice()` 都是常见浅拷贝方式。

深拷贝需要继续处理嵌套结构，但必须先明确数据类型。`JSON.parse(JSON.stringify(data))` 无法处理循环引用，会丢失部分数据，不能保留自定义原型，也不能正确保留 `Map`、`Set` 等结构。运行环境支持时，`structuredClone()` 支持更多数据类型和循环结构，但仍不能克隆函数，也不应用来复制引擎节点。[MDN：structuredClone](https://developer.mozilla.org/en-US/docs/Web/API/Window/structuredClone)

在 Cocos 中，按对象语义选择方式更可靠：

| 对象 | 常用处理方式 |
|---|---|
| 节点、Prefab | instantiate |
| Vec3 等数学对象 | clone 或专用复制方法 |
| 普通业务数据 | 按结构选择浅拷贝或深拷贝 |
| 纹理、音频等资源 | 通常共享并管理生命周期 |

### 5. 事件循环：异步为什么仍然可能卡住画面

以下示例放在通常的浏览器脚本环境中理解：

```ts
console.log("A");

setTimeout(() => console.log("B"), 0);
Promise.resolve().then(() => console.log("C"));

async function run() {
    console.log("D");
    await Promise.resolve();
    console.log("E");
}

run();
console.log("F");
```

输出顺序为 `A → D → F → C → E → B`。当前同步代码先执行，Promise 后续回调与 `await` 后续通过微任务继续，定时器回调在后续符合条件的任务中执行。不同宿主的具体调度还需要分别讨论。[MDN：JavaScript 执行模型](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Execution_model)

理解异步时有两个关键边界。第一，`async` 不会把函数自动放到另一线程；第一个 `await` 之前依然同步执行，Promise 构造器的执行函数也是同步调用的。第二，`await Promise.resolve()` 不保证进入下一渲染帧。微任务持续产生时，同样可能延迟渲染。

因此，给耗时计算套一层 `async` 并不能解决卡顿。应减少计算、分帧执行，或者在平台支持时使用 Worker。需要分帧的工作应明确接入引擎帧调度，不能把微任务当成帧边界。

### 6. GC 与资源引用计数要分开理解

简单引用计数根据持有关系增减计数，计数归零后释放。它独立处理循环引用时存在困难。现代 JavaScript 垃圾回收主要依据可达性：即使两个对象互相引用，只要从 GC 根无法再访问它们，通常仍能被回收。[MDN：内存管理](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Memory_management)

所以不应把“避免循环引用”当成 JS 内存管理的全部答案。更值得关注的是全局缓存、事件中心、定时器和未清理的集合是否仍持有业务对象。

另一方面，Cocos 还有纹理、音频和渲染缓冲等资源。下面三件事不能画等号：

```text
JS 对象不再可达 ≠ 节点被销毁 ≠ 底层资源已释放
```

销毁一个使用共享纹理的节点，并不意味着纹理应立即释放。

### 7. EventEmitter 的价值与实现边界

发布订阅让发送方不必直接依赖所有接收方。例如战斗模块发布“敌人死亡”，任务、掉落和 UI 模块分别订阅。

下面是最小教学实现，约定监听不重复注册，本轮派发使用监听快照，监听抛出的异常继续向外传播：

```ts
type Listener = (...args: unknown[]) => void;

class EventBus {
    private events = new Map<string, Set<Listener>>();

    on(type: string, listener: Listener) {
        let listeners = this.events.get(type);
        if (!listeners) {
            listeners = new Set();
            this.events.set(type, listeners);
        }
        listeners.add(listener);
        return () => this.off(type, listener);
    }

    off(type: string, listener: Listener) {
        const listeners = this.events.get(type);
        if (!listeners) return;
        listeners.delete(listener);
        if (listeners.size === 0) this.events.delete(type);
    }

    emit(type: string, ...args: unknown[]) {
        const listeners = this.events.get(type);
        if (!listeners) return;
        for (const listener of [...listeners]) {
            listener(...args);
        }
    }
}
```

快照意味着：本轮开始后注销的另一个监听，仍可能在本轮执行。这是此示例的约定，并非所有事件系统的规则。实际项目还需要考虑 `once()`、参数类型、目标对象和异常策略；Cocos 已提供 `EventTarget`，通常可以直接使用。[Cocos：EventTarget API](https://docs.cocos.com/creator/3.8/api/en/class/EventTarget)

## 二、引擎基础：围绕生命周期和坐标空间组织行为

### 1. 节点组织结构，组件提供功能

一个敌人节点可以挂载 Sprite、碰撞组件、生命值组件和行为组件。节点承载层级与变换，组件通过组合提供能力。

拆分组件时应看职责、复用和生命周期，而不是把每个小函数都变成组件。适合单独拆分的功能通常具有清楚的状态边界，或需要编辑器配置。

父子节点的变换满足如下概念关系：

```text
世界矩阵 = 父节点世界矩阵 × 自身本地矩阵
```

只有在没有相关旋转和缩放等条件下，世界位置才可以简单理解为位置相加。角色旋转影响子弹、换父节点后 UI 跳动，往往都与坐标空间混用有关。换父节点时，先明确需要保留本地变换还是世界变换。[Cocos：坐标系和节点变换](https://docs.cocos.com/creator/3.8/manual/zh/concepts/scene/coord.html)

### 2. 生命周期比一条顺序链更完整

常见回调可以这样理解：

| 阶段 | 回调 | 常见职责 |
|---|---|---|
| 首次加载 | onLoad | 获取自身组件，初始化内部状态 |
| 每次启用 | onEnable | 订阅启用期间需要的事件 |
| 首次更新前 | start | 启动依赖初始化结果的逻辑 |
| 每帧 | update | 推进每帧逻辑 |
| 帧内后续阶段 | lateUpdate | 跟随、最终位置修正 |
| 禁用 | onDisable | 结束启用期间的订阅和行为 |
| 销毁 | onDestroy | 清理对象持有的长期内容 |

`onLoad` 和 `start` 只执行一次，启用与禁用可以反复发生。不要仅凭节点排列假设跨组件初始化顺序；有严格依赖时，应显式组织初始化流程。[Cocos：生命周期回调](https://docs.cocos.com/creator/3.8/manual/zh/scripting/life-cycle-callbacks.html)

对象池尤其依赖这一区别：回收不等于销毁，再次取出不会重新执行 `start()`。每次出生都需要重置的血量、目标和动画，应放在明确的复用初始化方法中。

### 3. 事件和调度要有对称的结束动作

节点输入事件与业务事件承担不同职责。前者可以使用 `Node.EventType`，后者可以使用独立 `EventTarget`。普通 `emit()` 在当前事件目标触发监听，不能认为任意自定义事件都会沿父节点冒泡。

以下局部组件片段展示启用期订阅与调度的对称关系：

```ts
onEnable() {
    this.node.on(Node.EventType.TOUCH_END, this.onTouchEnd, this);
    this.schedule(this.tick, 1);
}

onDisable() {
    this.node.off(Node.EventType.TOUCH_END, this.onTouchEnd, this);
    this.unschedule(this.tick);
}
```

`schedule()` 按秒设置间隔，`scheduleOnce()` 设置一次性任务。调度依赖游戏帧推进，不能保证精确准点；带 `repeat` 的接口中，总调用次数通常为 `repeat + 1`。[Cocos：Component API](https://docs.cocos.com/creator/3.8/api/en/class/Component)

业务倒计时不要简单依赖“回调一次减一秒”。卡顿和切后台都会让回调次数偏离真实时间。更可靠的是保存结束时间，每次显示时重新计算：

```ts
const remainingMs = Math.max(0, endTime - Date.now());
```

奖励、活动有效期等关键状态仍需以服务端规则为准。

### 4. 坐标转换先确认来源与目标

本地坐标相对父节点，世界坐标属于场景统一空间，屏幕坐标经过相机投影，UI 坐标还涉及 Canvas、适配和锚点约定。

不要把触摸返回的位置直接赋给任意 UI 节点的本地位置。可靠路线是：

```text
来源空间 → 统一的世界或屏幕空间 → 目标空间
```

下面的局部片段适用于已经明确处于同一世界坐标语境下的 UI 节点，把源节点世界位置转换成目标父节点的锚点相对本地坐标：

```ts
const transform = targetParent.getComponent(UITransform);
if (!transform) return;

const localPosition = transform.convertToNodeSpaceAR(
    sourceNode.worldPosition,
);
targetNode.setPosition(localPosition);
```

它不能直接替代不同相机、不同 Canvas 或 3D 世界到屏幕 UI 的转换。后者需要按相机关系投影后再处理目标 UI 空间。[Cocos：UITransform API](https://docs.cocos.com/creator/3.8/api/en/class/UITransform)

### 5. 分辨率适配、Widget 与安全区各有职责

以设计分辨率 `750 × 1334` 为例，固定宽度可理解为根据屏幕宽度确定缩放比例，屏幕宽高比变化主要影响垂直可见范围；固定高度则反过来。实际效果还取决于所选适配策略，不能把 Fit Width 和 Fit Height 的组合笼统理解为拉伸铺满。[Cocos：多分辨率适配](https://docs.cocos.com/creator/3.8/manual/zh/ui-system/components/engine/multi-resolution.html)

| 工具 | 主要职责 |
|---|---|
| Widget | 相对父容器对齐和拉伸 |
| Layout | 排列子元素 |
| SafeArea | 让关键内容避开设备不安全区域 |

常见设计是让背景覆盖可显示区域，把按钮和文字放在安全内容区域。布局结果还可能受父节点缩放和计算时机影响，不是调对分辨率就结束了。

### 6. Prefab 与对象池解决不同阶段的问题

Prefab 是可复用的结构模板，`instantiate()` 生成实例；对象池保存暂时不用的实例，减少后续创建和销毁。

```ts
// 局部片段：pool、enemyPrefab 和 enemyRoot 已由调用方准备。
const node = pool.size() > 0
    ? pool.get()!
    : instantiate(enemyPrefab);

node.setParent(enemyRoot);
node.active = true;
// 随后执行项目自行定义的本轮出生初始化。
```

复用前需要恢复位置、缩放、血量、目标、动画和粒子状态；回收时需要处理 Tween、定时行为与监听。异步结果还可能在下一轮复用时返回，因此仅检查节点是否存在不够，必要时要检查本轮请求或使用代次。

NodePool 的回收动作不能被理解为自动完成全部业务清理。它也需要容量和退出策略，否则会持续保留节点及其关联资源。[Cocos：NodePool API](https://docs.cocos.com/creator/3.8/api/en/class/NodePool)

## 三、性能优化：把时间和内存成本分开测量

### 1. DrawCall 少不一定快

DrawCall 可以理解为向图形 API 提交一次绘制工作。大量小批次可能增加 CPU 提交与状态切换开销，合批则把兼容内容组织进较少的提交。

但 GPU 的工作量还取决于像素覆盖、透明叠加、Shader 复杂度和纹理访问。一个 DrawCall 的全屏特效，也可能比许多简单小图更昂贵。因此至少需要区分 CPU 逻辑、CPU 渲染提交、GPU 绘制，以及资源流程的成本。

同图集只是合批条件之一。还要看纹理与采样、材质与渲染状态、模板测试、实际绘制顺序和缓冲区限制。相同材质资源若被拆成不兼容的实例或参数，也可能影响合批。[Cocos：2D 合批规则](https://docs.cocos.com/creator/3.8/manual/en/ui-system/components/engine/ui-batch.html)

例如：

```text
图集 A → 字体纹理 B → 图集 A
```

通常不能简单跨过 B 合并前后两个 A，因为重新排序可能改变透明 UI 的遮挡结果。所谓“连续层级”，更准确地说是实际绘制顺序连续且状态兼容。

### 2. Label、Mask 与 UI 结构的成本

Label 经常带来纹理切换，但并非一定打断合批。结果取决于系统字体、位图字体、缓存模式、动态图集和具体引擎版本。文字方案应结合更新频率与字符范围选择；高频变化的数字和大量动态中文，通常不应采用完全相同的策略。

UI 优化还应检查无意义的重复赋值、频繁布局计算以及过多的不可见列表项。只有内容变化时才刷新文本，长列表只维护可见项，往往比单纯调整节点顺序更有价值。

Mask 的常见模板裁剪路径涉及额外绘制与模板状态，可能形成批次边界。嵌套遮罩、大面积透明叠加和大量子项值得重点观察。但关闭 Mask 会改变画面及可见内容，实验时必须结合绘制面积和对象数量判断原因，不能只看 FPS 变化。

### 3. 图集应按共同使用与共同释放组织

| 方案 | 发生阶段 | 主要用途 |
|---|---|---|
| 构建时自动图集 | 构建阶段 | 按配置合并图片 |
| SpriteAtlas | 资源制作与加载阶段 | 显式管理图集资源 |
| 动态图集 | 运行阶段 | 合并符合条件的纹理 |

动态图集不是没有成本的万能开关，支持范围与运行条件需要对应版本确认。[Cocos：动态图集](https://docs.cocos.com/creator/3.8/manual/zh/advanced-topics/dynamic-atlas.html)

把全部图片塞进一张大图集，也可能增加常驻内存。如果一个全局常驻小图标与某活动的全部图片共用纹理，小图标就可能延长整张纹理的生命周期。图集不仅影响绘制批次，也影响加载和释放粒度。

### 4. 实例化、销毁和对象池的取舍

复杂 Prefab 实例化可能涉及对象创建、属性恢复、节点树构建、组件初始化和渲染数据准备。销毁后还有清理和后续 GC 成本，不能把所有尖峰都简单归因于 GC。

对象池适合高频子弹、飘字、掉落物和列表项；极少出现的大型界面或长期不用的关卡对象，未必值得常驻。对象池本质上是在创建成本、管理复杂度和保留内存之间做交换。

预热也要安排预算。把所有对象都挪到启动时一次性创建，只是移动了卡顿发生的位置。

### 5. 下载完成到首次显示之间还有很多工作

资源可见前可能经历：

```text
下载 → 解压或解码 → 反序列化 → 建立依赖
     → 创建实例 → 上传 GPU → 首次绘制
```

因此，“已经下载，第一次打开仍然卡”并不矛盾。需要确认预加载究竟完成了哪一步。

Cocos 3.x 的 `assetManager` 管理资源与 Bundle；加载 Bundle 与加载其中具体资源是不同动作，`resources.load()` 则用于相应资源目录。代码示意如下，省略业务重试、进度和资源所有权管理：

```ts
import { assetManager, Prefab } from "cc";

assetManager.loadBundle("battle", (bundleError, bundle) => {
    if (bundleError) {
        console.error(bundleError);
        return;
    }

    bundle.load("prefabs/Enemy", Prefab, (assetError, prefab) => {
        if (assetError) {
            console.error(assetError);
            return;
        }
        // 检查接收者及本轮请求是否有效，再使用 prefab。
    });
});
```

加载期间界面可能关闭，甚至再次打开。接收结果时应同时考虑实例有效性与请求版本，避免旧请求把结果写进新一轮界面。[Cocos：Asset Bundle](https://docs.cocos.com/creator/3.8/manual/zh/asset/bundle.html)

### 6. 资源释放先确定所有权

公共 UI 资源、战斗资源和临时头像往往具有不同生命周期。应先明确谁负责持有、共享、缓存和释放，再选择相应 API。

`addRef()` 与 `decRef()` 用于表达资源持有关系，但需要与实际所有权配对。动态赋值的资源和共享依赖尤其要注意，不能对所有资源无条件强制释放。[Cocos：资源释放](https://docs.cocos.com/creator/3.8/manual/en/asset/release-manager.html)

节点销毁、Bundle 移出管理器、Bundle 内资源释放，以及本地下载缓存删除，是不同操作。管理其中一层，并不意味着其余层都已结束。

排查内存时，可以重复“进入、使用、退出”，比较多轮操作后稳定阶段的变化。同时观察 JS 堆、节点、资源缓存、纹理、监听和对象池容量。第一次上升可能来自字体缓存或运行时扩容，不能只凭内存没有立即下降就认定泄漏。更有说服力的证据是：对象生命周期已经结束，重复操作仍持续增长，而且能找到不应存在的持有关系。

### 7. 用对比实验定位卡顿

60 FPS 的单帧预算约为 `16.67 ms`，30 FPS 约为 `33.33 ms`。平均 FPS 正常也可能有显著尖峰，应该观察具体帧耗时和发生条件。

| 怀疑方向 | 对比实验 | 下一步证据 |
|---|---|---|
| CPU 逻辑 | 分别暂停 AI、寻路、碰撞 | 热点函数与调用次数 |
| CPU 渲染提交 | 简化对象与状态切换 | 批次数与提交耗时 |
| GPU | 降低分辨率、减少透明覆盖 | 像素成本与 Shader |
| 资源流程 | 对比首次与再次进入 | 解码、实例化、纹理上传 |
| GC | 对齐分配与回收时间 | 临时对象和回收记录 |

每次尽量只改一个变量。隐藏节点可能同时减少逻辑、布局和绘制成本，所以单个实验通常只是线索，需要交叉验证。

以“首次打开背包卡顿”为假设案例，可以依次测量下载、加载、实例化、绑定数据、布局和首次绘制阶段；如果证据指向大量格子一次性实例化，再比较可见项复用、分帧初始化等方案。这个流程是方法示例，不代表任何项目已经取得了特定优化结果。

面试讲真实案例时，应给出设备与复现动作、测量数据、排除过程、根因、修改及同条件复测结果。也要说明代价，例如预热增加了多少常驻内容，而不只说“使用对象池优化成功”。

### 8. Shader 置灰的基础与边界

Shader 定义顶点与像素如何处理，材质提供所用 Shader、纹理、参数和渲染状态。常见置灰思路是计算颜色亮度，再把 RGB 设为同一值。

下面是示意性的 GLSL 片元逻辑，并非可直接导入的完整 Cocos Effect：

```glsl
vec4 color = texture(mainTexture, uv);
float gray = dot(color.rgb, vec3(0.299, 0.587, 0.114));
color.rgb = mix(color.rgb, vec3(gray), grayAmount);
// 保留 color.a，随后按实际管线输出。
```

这是一种常用近似，实际色彩计算还涉及工作色彩空间。Cocos UI 实现还需要处理内置宏、图集 UV、顶点颜色和 Alpha 约定。简单 Sprite 置灰应先检查引擎内置能力；为每个按钮创建独立材质实例前，也要观察合批和内存影响。[Cocos：Effect 语法](https://docs.cocos.com/creator/3.8/manual/zh/shader/effect-syntax.html)

## 四、小游戏平台：把能力边界纳入设计

### 1. 主包、分包和远程资源各有职责

主包保留启动所需的最小内容，分包在进入对应功能时获取，远程资源则按部署和加载策略从服务器下载。Cocos Bundle 是引擎资源组织方式，平台分包是交付机制；只有结合构建配置，二者才建立具体映射。

Cocos 3.8 的微信分包说明列出了主包 4M、整体包 30M 等约束，可用作引擎配置参考；最终应以[微信小游戏分包文档](https://developers.weixin.qq.com/minigame/dev/guide/base-ability/subPackage/useSubPackage.html)和项目后台校验为准。[Cocos：小游戏分包](https://docs.cocos.com/creator/3.8/manual/en/editor/publish/subpackage.html)

此次检索到的抖音普通小游戏代码包说明列出：未配置分包时为 20MB；配置分包后主包为 4MB，整体包与单个分包默认上限为 20MB，开通虚拟支付后的相应上限为 30M。这些数字不能套用到 Unity 等其他接入方案，也不应当成永久规则。[抖音开放平台：代码包](https://developer.open-douyin.com/docs/resource/zh-CN/mini-game/develop/framework/subpackages)

实际工程更值得回答的是：首屏保留什么、公共依赖放在哪里、何时加载分包，以及弱网失败怎样重试。分了包却在启动时把全部内容加载回来，并不能改善启动等待。

### 2. 登录、支付、广告与分享需要完整流程

登录通常由客户端获得临时凭证，交给业务服务端向平台校验或换取身份，再建立业务会话。客户端不应持有平台服务端密钥；还要处理重复登录、凭证过期和切后台恢复。

支付应以服务端确认形成闭环：创建订单、调起支付、服务端校验或接收通知、发货、同步最终状态。同一订单必须具备幂等性，重复通知和重试只能导致一次有效发货。支付能力本身还取决于平台、系统、类目和开通权限。

激励广告适合按状态机管理：

```text
空闲 → 加载中 → 可展示 → 展示中 → 结束或失败
```

需要按平台规定的有效完成结果发奖，防止重复注册监听、重复展示与重复发奖；失败后应恢复可操作状态，并处理游戏暂停和音频恢复。

分享接口调用成功与满足业务奖励条件不是同一个概念，不能默认一次成功回调就证明内容已经分享给真实好友。各平台具体能力与限制应查阅各自 API 文档，不能从通用流程推导支持情况。[微信小游戏 API](https://developers.weixin.qq.com/minigame/dev/api/)、[抖音小游戏 API](https://developer.open-douyin.com/docs/resource/zh-CN/mini-game/develop/api/overview)

跨平台 SDK 可以统一公共入口，但应保留能力检测和平台特有结果，避免把所有状态压成一个成功或失败的布尔值。

### 3. iOS 内存治理重点是同时存活的峰值

iOS 可用内存并不是适用于所有小游戏的固定数字，设备、系统、宿主和运行模式都会影响边界。Apple 也指出进程内存上限与设备有关，超过上限可能被终止。[Apple：减少 App 的内存使用](https://developer.apple.com/cn/documentation/xcode/reducing-your-app-s-memory-use)

游戏内存不能只看 JS 堆，还包括引擎对象、纹理、音频、渲染缓冲、解码临时数据与宿主开销。图片的下载体积也不等于纹理内存。一张未压缩的 `2048 × 2048` RGBA8 纹理，基础像素数据约为：

```text
2048 × 2048 × 4 字节 = 16 MiB
```

完整 Mipmap 链通常再增加约三分之一，实际占用还取决于压缩格式、对齐和实现。场景切换时，旧资源未释放、新资源已加载，再叠加解码和上传，容易出现峰值。

治理时应控制纹理尺寸、同时存活数量、预加载并发和对象池容量，合理安排切换期间的加载与释放，并在低内存真机测试。平台支持的内存告警可以作为降级信号，但不能假定每次异常退出前都会留下可处理的告警，也不能把告警当成唯一的内存管理机制。

## 五、把知识地图变成实践能力

可以用一个小型背包练习把这些内容连接起来：界面打开时异步加载资源，通过事件刷新数据，使用可见项复用显示长列表，并把关键按钮放进安全内容区域。

随后连续验证几种操作：加载中关闭、关闭后重开、多次进入、快速滚动、切后台，以及低内存设备运行。观察旧回调是否污染新界面、监听是否重复、对象池是否增长，以及首次打开的耗时究竟落在哪个阶段。

这项练习的产出不只是一个能显示的界面，还应包括清楚的生命周期约定、资源持有关系，以及一组能复现和解释的测量结果。面试中的语言题、引擎题和优化题，最终都可以落回这些具体证据。
