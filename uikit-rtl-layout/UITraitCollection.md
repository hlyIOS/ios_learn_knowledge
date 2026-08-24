# UITraitCollection 是什么？

> 整理自 Apple Developer Documentation  
> 与 RTL 布局的关系见：[基准布局方向（环境层）](./基准布局方向（环境层）.md) · [语义属性与布局方向的关系](./SemanticAttribute与LayoutDirection的关系.md)

---

## 一句话解释

**UITraitCollection** 是 iOS 用来描述「**当前界面处于什么环境**」的一组属性集合——就像一张**环境快照**，告诉 App 现在是横屏还是竖屏、深色模式还是浅色模式、LTR 还是 RTL 等。

```swift
class UITraitCollection
```

---

## 用生活例子理解

把 App 界面想象成一家**餐厅**：

| 概念 | 类比 |
|------|------|
| **UITraitCollection** | 餐厅今天的「环境公告板」 |
| **layoutDirection** | 公告板上写着「从左读菜单」还是「从右读菜单」 |
| **userInterfaceStyle** | 公告板上写着「开灯（浅色）」还是「关灯（深色）」 |
| **horizontalSizeClass** | 公告板上写着「桌子宽（regular）」还是「桌子窄（compact）」 |

每个 View / ViewController 都能读到自己所在位置的「公告板」，并据此调整布局和行为。

---

## 它描述哪些环境信息？

`UITraitCollection` 包含多种 trait（特征），常用的大致分几类：

### 与 RTL 布局直接相关

| 属性 | 类型 | 说明 |
|------|------|------|
| `layoutDirection` | `UITraitEnvironmentLayoutDirection` | 当前环境的布局方向（LTR / RTL / unspecified） |

### 其他常见 trait（与 RTL 间接相关）

| 属性 | 说明 |
|------|------|
| `horizontalSizeClass` / `verticalSizeClass` | 横/纵尺寸类别（compact / regular） |
| `userInterfaceStyle` | 浅色 / 深色模式 |
| `userInterfaceIdiom` | 设备类型（iPhone / iPad 等） |
| `displayScale` | 屏幕缩放比例（@2x、@3x） |
| `preferredContentSizeCategory` | 用户字体大小偏好（无障碍） |

本文重点讲 `layoutDirection`；其余 trait 共同构成完整的「界面环境」。

---

## 谁拥有 UITraitCollection？

遵循 `UITraitEnvironment` 协议的对象都有 `traitCollection` 属性：

| 类型 | 说明 |
|------|------|
| `UIScreen` | 屏幕 |
| `UIWindow` / `UIWindowScene` | 窗口 / 场景 |
| `UIViewController` | 视图控制器 |
| `UIView` | 视图 |
| `UIPresentationController` |  presentation 控制器 |

```swift
@MainActor protocol UITraitEnvironment {
    var traitCollection: UITraitCollection { get }
}
```

**读取方式：**

```swift
// 任意 View / ViewController 上
let traits = view.traitCollection
let direction = traits.layoutDirection
```

---

## Trait 如何传递？（层级传播）

Trait 从视图层级**顶部向下传播**，类似继承：

```
UIWindowScene
    └── UIWindow
            └── UIViewController（根）
                    └── 子 ViewController
                            └── UIView
                                    └── 子 View
```

**规则：**

1. 子对象默认**继承**父对象的环境
2. 父对象可以 **override（覆盖）** trait，影响自己和所有后代
3. 覆盖是**向下传播**的——改 window 会影响整棵树，改某个 view 只影响它和子视图

Apple 文档原话：修改任意层级的 trait，会影响该对象及其**所有后代**。

---

## layoutDirection 详解

### 枚举值：UITraitEnvironmentLayoutDirection

Trait 里的布局方向用的是 `UITraitEnvironmentLayoutDirection`（注意与 `UIUserInterfaceLayoutDirection` 不同）：

| 值 | 含义 |
|----|------|
| `.unspecified` | 未知 / 未指定，由系统推断 |
| `.leftToRight` | 从左到右 |
| `.rightToLeft` | 从右到左 |

```swift
var layoutDirection: UITraitEnvironmentLayoutDirection { get }
```

**用途：** 判断当前底层环境是使用 LTR 还是 RTL 方向。

### 与 UIUserInterfaceLayoutDirection 的区别

| | UITraitEnvironmentLayoutDirection | UIUserInterfaceLayoutDirection |
|---|-----------------------------------|-------------------------------|
| **所在位置** | `UITraitCollection.layoutDirection` | `effectiveUserInterfaceLayoutDirection` 等 API 的返回值 |
| **含义** | 环境 trait 中的「布局方向特征」 | 综合语义 + 环境后，视图**实际应使用的布局方向** |
| **枚举值** | 含 `.unspecified` | 仅 `.leftToRight` / `.rightToLeft` |
| **关系** | 环境的输入之一 | 最终计算结果 |

可以简单记：**Trait 里的 layoutDirection 是「环境信号」，UIUserInterfaceLayoutDirection 是「落地结果」。**

---

## 系统语言如何变成 layoutDirection？

大致流程：

```
用户系统语言（如阿拉伯语）
        ↓
iOS 推断该语言为 RTL
        ↓
写入 UITraitCollection.layoutDirection = .rightToLeft
        ↓
向下传播到 ViewController / View
        ↓
参与计算 effectiveUserInterfaceLayoutDirection
```

所以 [基准布局方向（环境层）](./基准布局方向（环境层）.md) 里说的「Trait Collection」——**系统语言最终就是通过 trait 传递到界面每一层的**。

---

## 开发者如何修改 Trait？

### 1. 为子 ViewController 覆盖 trait（针对 ViewController）

> 文档：[setOverrideTraitCollection(_:forChild:)](https://developer.apple.com/documentation/uikit/uiviewcontroller/setoverridetraitcollection(_:forchild:))

**是的，这个方法就是针对 ViewController 来设置的。**

```swift
// UIViewController 上的方法
func setOverrideTraitCollection(
    _ collection: UITraitCollection?,
    forChild childViewController: UIViewController  // ← 参数是子 VC
)
```

| 角色 | 说明 |
|------|------|
| **调用者** | 父 ViewController（或容器 VC） |
| **作用对象** | 子 ViewController |
| **影响范围** | 该子 VC + 它的 `view` 及所有 subviews |

#### 「整个页面」是什么意思？

在 UIKit 里，**「整个页面」= 一个 `UIViewController` 及其下面整棵视图树**，不是单个 `UIView`。

```
UINavigationController
    └── SettingsViewController   ← 「这一整个页面」
            ├── view
            ├── tableView
            ├── headerView
            └── footerView
            （以上全部属于这个 VC 的子树）
```

对 `SettingsViewController` 调用 `setOverrideTraitCollection`，**这个 VC 本身 + 它 view 里的所有子视图**都会受影响。

#### 影响范围

| 会被影响 | 不会被影响 |
|----------|-----------|
| `childViewController` 本身 | 父 VC 的其他兄弟页面 |
| 它的 `view` 及所有 subviews | App 里其他无关 VC |
| 它作为 child 时的后代子树 | 已独立存在的其他 Tab 页面 |

Override 向下传播——改了 child VC 的环境，它整棵子树都跟着变。

#### 三层粒度对照

| 粒度 | API | 设置对象 |
|------|-----|----------|
| 整个 App | `UIApplication.userInterfaceLayoutDirection` | Application |
| **整个页面** | `setOverrideTraitCollection` | **ViewController** |
| 单个控件 | `semanticContentAttribute` | View |

#### 代码示例

```swift
class ContainerViewController: UIViewController {

    override func viewDidLoad() {
        super.viewDidLoad()

        let child = EnglishManualViewController()
        addChild(child)
        view.addSubview(child.view)
        child.didMove(toParent: self)

        // 整个 EnglishManualViewController 及其内部所有 UI 强制 LTR
        let ltrTraits = UITraitCollection(layoutDirection: .leftToRight)
        setOverrideTraitCollection(ltrTraits, forChild: child)
    }
}
```

Tab 中某一 Tab 固定方向：

```swift
// App 整体 RTL，但「帮助」Tab 永远是 LTR
setOverrideTraitCollection(
    UITraitCollection(layoutDirection: .leftToRight),
    forChild: helpNavigationController
)
```

#### 不是什么

| ❌ 不是 | 说明 |
|--------|------|
| 整个 App | 那是 `UIApplication.userInterfaceLayoutDirection` |
| 某一个按钮 | 那是 `semanticContentAttribute` |
| Storyboard 里一个 Scene 的视觉区域 | 除非它对应一个独立的 ViewController |

#### 典型场景

- 在 LTR 系统下，预览 RTL 界面
- 某段界面（整页）需要固定 RTL/LTR，不受 App 语言设置影响
- Split View 中强制 iPhone 横屏时使用 regular 尺寸类
- App 整体 RTL，但某个 Tab / 子页面保持 LTR（见 [App 内语言设置控制 RTL/LTR](./App内语言设置控制RTL-LTR.md)）

**说明：**

- 通常 trait 从父 VC **原样传递**给子 VC
- 自定义容器 VC 可用此方法为嵌入的子 VC 设置更合适的 trait
- 修改后会影响该子 VC 及其所有后代的布局行为

### 2. 合并多个 trait 创建独立 collection

```swift
let traits = UITraitCollection(traitsFrom: [
    UITraitCollection(layoutDirection: .rightToLeft),
    UITraitCollection(userInterfaceStyle: .dark)
])
```

用于：图片资源匹配、Appearance 定制、条件布局等。

### 3. 监听 trait 变化

当环境变化（旋转屏幕、切换深色模式、语言切换等），系统会通知：

```swift
override func traitCollectionDidChange(_ previousTraitCollection: UITraitCollection?) {
    super.traitCollectionDidChange(previousTraitCollection)

    if traitCollection.layoutDirection != previousTraitCollection?.layoutDirection {
        // 布局方向变了，刷新 UI
    }
}
```

iOS 17+ 也支持更精细的 trait 变更追踪（`registerForTraitChanges` 等）。

---

## 在 RTL 布局链路中的位置

```
┌──────────────────────────────────────────┐
│  系统语言 / App 设置                       │
└──────────────────┬───────────────────────┘
                   ↓
┌──────────────────────────────────────────┐
│  UITraitCollection.layoutDirection       │  ← 本文重点
│  （环境 trait，向下传播，可被 override）    │
└──────────────────┬───────────────────────┘
                   ↓
┌──────────────────────────────────────────┐
│  基准布局方向                               │
│  userInterfaceLayoutDirection(for: .unspecified)
└──────────────────┬───────────────────────┘
                   ↓
┌──────────────────────────────────────────┐
│  semanticContentAttribute（语义层）        │
└──────────────────┬───────────────────────┘
                   ↓
┌──────────────────────────────────────────┐
│  effectiveUserInterfaceLayoutDirection   │
│  （视图最终布局方向）                       │
└──────────────────────────────────────────┘
```

---

## 使用示例

### 读取当前环境的布局方向

```swift
switch view.traitCollection.layoutDirection {
case .rightToLeft:
    print("当前 trait 环境为 RTL")
case .leftToRight:
    print("当前 trait 环境为 LTR")
case .unspecified:
    print("未指定，由系统推断")
@unknown default:
    break
}
```

### 强制子页面 RTL（用于测试或特定 UI）

```swift
class ContainerViewController: UIViewController {

    override func viewDidLoad() {
        super.viewDidLoad()

        let child = DetailViewController()
        addChild(child)
        view.addSubview(child.view)
        child.didMove(toParent: self)

        // 仅对这个 child 及其后代生效
        let rtlTraits = UITraitCollection(layoutDirection: .rightToLeft)
        setOverrideTraitCollection(rtlTraits, forChild: child)
    }
}
```

### trait 变化时更新自定义布局

```swift
override func layoutSubviews() {
    super.layoutSubviews()

    let isRTL = effectiveUserInterfaceLayoutDirection == .rightToLeft
    // 根据最终布局方向排列子视图
}
```

> 自定义布局时读 `effectiveUserInterfaceLayoutDirection`（结果层），而不是直接读 trait 后自己换算。

---

## 常见误区

### 误区 1：把 traitCollection.layoutDirection 当成最终结果

```swift
// ❌ 忽略了 semanticContentAttribute 的影响
if view.traitCollection.layoutDirection == .rightToLeft { ... }

// ✅ 布局/绘制时用 effectiveUserInterfaceLayoutDirection
if view.effectiveUserInterfaceLayoutDirection == .rightToLeft { ... }
```

`.playback` 等语义控件在 RTL trait 环境下，最终方向可能与 trait 相反。

### 误区 2：以为 trait 会自动传给所有子视图

Trait 是**向下传播**的，但每个 view 的 `effectiveUserInterfaceLayoutDirection` 还需结合自身的 `semanticContentAttribute` 单独计算，不能假设子 view 和父 view 结果相同。

### 误区 3：混淆两个 LayoutDirection 枚举

- `UITraitEnvironmentLayoutDirection` → trait 环境属性，有 `.unspecified`
- `UIUserInterfaceLayoutDirection` → 布局 API 返回值，只有 LTR / RTL

---

## 快速对照

| 问题 | 用什么 |
|------|--------|
| 当前环境的 trait 方向是什么？ | `view.traitCollection.layoutDirection` |
| 当前 App 全局方向？ | `UIApplication.shared.userInterfaceLayoutDirection` |
| 某视图最终该怎么布局？ | `view.effectiveUserInterfaceLayoutDirection` |
| 给子 VC（整页）强制 RTL/LTR 环境？ | `setOverrideTraitCollection(..., forChild:)` — **针对 ViewController** |
| 环境变化了怎么响应？ | `traitCollectionDidChange(_:)` |

---

## 记忆口诀

> **TraitCollection 是环境快照，layoutDirection 是其中的方向信号。**  
> **从顶层向下传，override 针对 VC 改子树；最终布局方向还要看 semantic。**

---

## 参考链接

- [UITraitCollection](https://developer.apple.com/documentation/uikit/uitraitcollection)
- [UITraitEnvironment](https://developer.apple.com/documentation/uikit/uitraitenvironment)
- [UITraitCollection.layoutDirection](https://developer.apple.com/documentation/uikit/uitraitcollection/layoutdirection)
- [UITraitEnvironmentLayoutDirection](https://developer.apple.com/documentation/uikit/uitraitenvironmentlayoutdirection)
- [setOverrideTraitCollection(_:forChild:)](https://developer.apple.com/documentation/uikit/uiviewcontroller/setoverridetraitcollection(_:forchild:))
- [Adapting your app when traits change](https://developer.apple.com/documentation/uikit/adapting-your-app-when-traits-change)
- [App 内语言设置控制 RTL/LTR](./App内语言设置控制RTL-LTR.md)
- [基准布局方向（环境层）](./基准布局方向（环境层）.md)
- [UIUserInterfaceLayoutDirection](./UIUserInterfaceLayoutDirection.md)

---

*来源：Apple Developer Documentation © 2026 Apple Inc.*
