# UIAppearance 与 appearance proxy

> 整理自 Apple Developer Documentation  
> - [UIAppearance](https://developer.apple.com/documentation/uikit/uiappearance)  
> - [UIAppearanceContainer](https://developer.apple.com/documentation/uikit/uiappearancecontainer)

---

## 一句话

**`UIAppearance` 用来拿到某个类的「外观代理」（appearance proxy）。**  
对代理发设置消息，就能统一改这类控件的外观——不必一个个改实例。

```swift
@MainActor protocol UIAppearance : NSObjectProtocol
```

| 名称 | 是什么 |
|------|--------|
| `UIAppearance` | 协议：提供取 proxy 的方法。被改外观的类遵循它 |
| appearance proxy | 对该类发「改外观」消息的入口 |
| `UIAppearanceContainer` | **空标记协议**：能作为「容器」出现在层级匹配里的类遵循它 |

官方定义：*A collection of methods that gives you access to the appearance proxy for a class.*

---

## 和 `UIAppearanceContainer` 的关系

两个协议分工不同，不是「要配外观就必须遵循 Container」。

```objc
@protocol UIAppearanceContainer <NSObject> @end   // 空，纯标记

@protocol UIAppearance <NSObject>
+ (instancetype)appearance;
+ (instancetype)appearanceWhenContainedInInstancesOfClasses:(NSArray<Class<UIAppearanceContainer>> *)containerTypes;
// ...
@end
```

| | `UIAppearance` | `UIAppearanceContainer` |
|--|----------------|-------------------------|
| 角色 | **被配**的那一类：能调 `appearance()` | **装着别人**的那一类：能写进 `whenContainedInInstancesOf:` |
| 方法 | `appearance()` 等 | 无（空协议） |
| 典型 | `UIView`、`UIBarItem` | `UIView`、`UIViewController`、`UIPresentationController` |

- `UIView` **两者都遵循**：既能被配，也能当容器。
- `UIViewController` **只有 Container**：可以写 `whenContainedInInstancesOf: [MyVC.self]`，但不能 `UIViewController.appearance()`。
- `UIBarItem`（`UIBarButtonItem` 的父类）**只有 Appearance**：能配按钮外观，但不能当容器。

[是什么 × 在哪里（韦恩图）](./diagrams/protocols-venn.html)

![UIAppearance 与 UIAppearanceContainer：UIView 在交集](./diagrams/protocols-venn.svg)

```swift
// 被配的是 UIBarButtonItem（UIAppearance）
// 容器是 UIToolbar（UIAppearanceContainer，经 UIView 继承）
UIBarButtonItem.appearance(whenContainedInInstancesOf: [UIToolbar.self])
    .tintColor = .systemOrange
```

---

## 设计思想

iOS 5 之前，统一换肤只能逐个实例设，或再发明一套 theme 文件、`useThemeForXxx` 那种**平行 API**。WWDC 2011（Session 114）说得很直：那种 API **难写也难用**。于是走代理。

1. **同一套 setter。** `appearance()` 只是替身，把消息记下来，以后回放到真实控件上（类似 AppKit 的 animator proxy）。你会配一个 Slider，就会配所有 Slider。*If you know how to customize one instance, you know how to customize every single one.*
2. **「是什么」和「在哪里」拆开。** `UIAppearance` 管被配的类；`UIAppearanceContainer` 是空标记，只声明「我是树上的一个位置」（View / ViewController）。容器不需要多方法，只要能被认出来。VC 是位置但不是控件（不能 `appearance()`）；BarItem 是控件但不在 View/VC 树上（不能当容器）——所以两个协议不能并成一个。
3. **等层级齐了再算。** 进 window 时才知道外面套了谁（`layoutSubviews` 之前回放）。按 **类继承**（更具体的赢）+ **物理容器**（外层优先，链更深破平局）级联；实例上已经赋过的值不动，相当于内联样式压过规则。

不要对太泛的类（如 `UILabel`）做全局 appearance——内部也会有你想不到的子类。用 Container 把范围收窄。

---

## 这是什么思维？代理怎么记账？

不是 CSS 这门语言，是同一套 **「先写规则，后匹配到实例」** 的想法：

| CSS | UIAppearance |
|-----|----------------|
| `button { color: red }` | `UIButton.appearance().setTitleColor(.red, for: .normal)` |
| `nav button { color: blue }` | `UIButton.appearance(whenContainedInInstancesOf: [UINavigationBar.self])…` |
| `<button style="color:green">` | 这个实例自己 `setTitleColor(.green, …)`，规则不再覆盖 |

「套在现有 setter 上」= 不另写 `useThemeForTitleColor:`，规则里调用的就是实例那套方法。  
「空协议标容器」= `whenContainedIn` 里的类型只要能被认成树上的位置，不必再实现方法。

### 代理不是 HTTP 代理

`appearance()` 返回的是一个**替身对象**（系统内部类 `_UIAppearance`）。它本身几乎没有 `setTitleColor` 这类方法。你对它发 setter 时，走的是 Objective-C **消息转发**：

1. `methodSignatureForSelector:` — 问真实类「这个 setter 签名是什么」
2. `forwardInvocation:` — 把这次调用打成 `NSInvocation`（方法名 + 参数），放进数组里，**此刻并不改任何控件**
3. 视图进入 window 后，`_applyInvocationsTo:window:` 用 `invokeWithTarget:` 把记下的调用回放到真实实例上

WWDC 2011 原话：*recording all of those message sends and we play them back … just before layoutSubviews is called.*

公开入口只有 `+appearance` / `+appearanceWhenContainedInInstancesOfClasses:` 等，用来**拿到**这个替身。真正干活的是上面两个转发方法。

### 运行时还是编译时？

**干活是运行时。** 编译器不会把 `UIButton.appearance().setTitleColor` 展开成「给每个 Button 写死颜色」。

| 阶段 | 做什么 |
|------|--------|
| 编译 | 只打标：`UI_APPEARANCE_SELECTOR`（clang `annotate`）；Swift 还要 `@objc dynamic`，好走 ObjC 消息发送 |
| 运行 · 记账 | `didFinishLaunching` 里对替身发 setter → `forwardInvocation:` 把 `NSInvocation` 放进数组 |
| 运行 · 回放 | 某个实例 **进入 window** 时才匹配容器链、再 `invokeWithTarget:` |

所以时序图里才有「No views yet — the ledger just waits」：规则先记在内存里，控件还没出生。已在 window 里的视图不会自动刷新，也是因为回放绑在「进层级」这个运行时事件上，不是编译产物。

私有类 `_UIAppearance` 的完整接口、从头文件的推断、以及按接口自己接的一版实现，见 [_UIAppearance：接口与按接口实现](./_UIAppearance.md)。

![记账与回放：工厂把 setter 记进账本，进 window 再打到实例上](./diagrams/record-apply.svg)

![一次 setter，两次人生：启动时记账，进 window 时回放](./diagrams/record-apply-sequence.svg)

---

## 怎么用

不是改某一个已有的 `UINavigationBar`，而是改「所有（或某范围内）导航栏」的默认样子：

```swift
UINavigationBar.appearance().barTintColor = .systemBlue
```

官方说法：向该类的 appearance proxy 发送外观修改消息（*sending appearance-modification messages to the class’s appearance proxy*）。

---

## 和逐个实例比

WWDC 2011 先演示「原来的写法」：每个控件拿 outlet，在各自 VC 里设一遍；再换成 appearance，同一套 setter 只写一次。

**原来：对着实例设**

```swift
// 分散在 AppDelegate、各 VC、各 outlet 上
navigationBar.barTintColor = .systemBlue
navigationBar.titleTextAttributes = [.foregroundColor: UIColor.white]
scanSwitch.onTintColor = .systemRed
spinner.color = .systemRed
slider.minimumTrackTintColor = .systemRed
nextItem.setTitleTextAttributes([.foregroundColor: UIColor.white], for: .normal)
```

**现在：对着类的代理设**

```swift
UINavigationBar.appearance().barTintColor = .systemBlue
UINavigationBar.appearance().titleTextAttributes = [.foregroundColor: UIColor.white]
UISwitch.appearance().onTintColor = .systemRed
UIActivityIndicatorView.appearance().color = .systemRed
UISlider.appearance().minimumTrackTintColor = .systemRed
UIBarButtonItem.appearance().setTitleTextAttributes(
    [.foregroundColor: UIColor.white], for: .normal
)
```

| | 逐个实例 | appearance proxy |
|--|----------|------------------|
| 写在哪 | 每个 VC / 每个 outlet | 通常一处（如 `didFinishLaunching`） |
| 新加的同类控件 | 容易漏 | 自动吃到规则 |
| API | 实例 setter | **同一套** setter，只是发给替身 |
| 何时生效 | 立刻 | 进入 window 时回放；已在 window 里的**不会**刷新 |
| 作用面 | 只这个对象，精确 | 该类（含子类）所有实例，范围大 |
| 局部例外 | 本来就是局部 | 用 `whenContainedIn`；或这个实例自己再设一次（实例覆盖规则） |
| 能配的属性 | 公开属性都行 | 仅 `UI_APPEARANCE_SELECTOR`（`tintColor` 还被禁止） |

**代理的利：** 换肤代码收拢；以后新建的控件不用再配；「工具栏里的按钮」这种按位置区分，不必给每个按钮设 outlet。

**代理的弊：** 对 `UILabel` / `UIView` 这种太泛的类下手会误伤内部子视图；匹配规则（外层优先、子类继承）不好调试；已显示的界面不会跟着改；只能配系统点名允许的属性。Storyboard 里已经填过的值算「实例已设」，规则盖不上。

**实例的利：** 所见即所得，立刻生效，不会波及别的控件。

**实例的弊：** 同一套外观复制到很多文件；Marian 演示里人会跟丢「现在改的是栈里哪一层」；漏一个新页面就花掉。

实务上两者一起用：全局默认走 appearance，某一个必须不同的实例直接设属性。不要发明第三套 `useThemeForXxx`。

---

## 作用范围

| 范围 | API | 作用对象 |
|------|-----|----------|
| 全局 | `appearance()` | 该类**所有**实例 |
| 容器内 | `appearance(whenContainedInInstancesOf:)` | 只在指定容器层级里的实例 |
| 按环境 | `appearance(for:)` | 匹配指定 **trait collection**（如深色模式） |
| 环境 + 容器 | `appearance(for:whenContainedInInstancesOf:)` | 上面两者叠加 |

### 全局

```swift
UINavigationBar.appearance().barTintColor = navBarTintColor
```

### 只在某个容器里

例如：导航栏里的按钮一套图，工具栏里的按钮另一套图：

```swift
UIBarButtonItem.appearance(whenContainedInInstancesOf: [UINavigationBar.self])
    .setBackgroundImage(navImage, for: .normal, barMetrics: .default)

UIBarButtonItem.appearance(whenContainedInInstancesOf: [UIToolbar.self])
    .setBackgroundImage(toolbarImage, for: .normal, barMetrics: .default)
```

`containerTypes` 按**从内到外**的升序写。例如「在 Tab 里的 Navigation 中的导航栏」：

```swift
UINavigationBar.appearance(
    whenContainedInInstancesOf: [UINavigationController.self, UITabBarController.self]
)
```

列表要和真实界面层级一致，不要塞无关类型。

---

## 子类会不会生效？

**会。** 对父类 appearance 的设置，系统子类和用户自定义子类都会吃到。`UIView` 已遵循 `UIAppearance` / `UIAppearanceContainer`，子类不用再声明。

进入 window 时沿 **真实 class → 父类 → … → UIView** 收集规则；子类上的更具体，盖过父类。

```swift
UIView.appearance().backgroundColor = .systemRed   // Label、Button、MyView: UIView 都会被设上
MyCardView.appearance().backgroundColor = .systemBackground  // 只覆盖 MyCardView 这条链
```

`UIView.h` 对 `backgroundColor` 的注释：*Can be useful with the appearance proxy on custom UIView subclasses.* 应写在**你的子类**上，不要对 `UIView` 做全局 appearance——内部私有子视图也会被刷到。

只能配父类真正有、且标了 `UI_APPEARANCE_SELECTOR` 的属性（`UIView.appearance()` 设不了 `textColor`）。自己加的属性要在子类上标该宏（Swift 还需 `@objc dynamic`）。

---

## 多条规则谁生效？

同一视图层级里：

1. **更外层**的 proxy 优先（outermost wins）  
2. 外层相同时，**containment 链更深**的优先（更具体）  
3. UIKit 从 **window 往下**读真实层级，取第一个唯一匹配

可记作：**外层优先；更具体的规则破平局。**

---

## 何时生效？

| 时机 | 行为 |
|------|------|
| View **进入 window** | 应用当前 appearance 设置 |
| View **已经在 window 里** | **不会**自动刷新 |

要改已经显示中的控件：先移出视图层级，再加回去；或直接改该实例属性。

建议在 App 启动早期（例如 `didFinishLaunching`）统一配置 appearance。

---

## 谁能用？哪些属性能配？

- **被配外观**：类遵循 `UIAppearance`，且属性访问器标了 `UI_APPEARANCE_SELECTOR`。
- **当容器**：类遵循 `UIAppearanceContainer`（`UIView` / `UIViewController` 已遵循，自定义子类不用再写）。

只有带 `UI_APPEARANCE_SELECTOR` 的属性才能通过 proxy 设置；普通属性不行。  
`UIView`、`UILabel`、`UINavigationBar`、`UIBarButtonItem` 等系统控件大多已支持。

---

## API 速查

| 方法 | 含义 |
|------|------|
| `appearance()` | 该类全部实例 |
| `appearance(for:)` | 指定 trait |
| `appearance(whenContainedInInstancesOf:)` | 指定容器层级 |
| `appearance(for:whenContainedInInstancesOf:)` | trait + 容器 |

---

## 最小示例

```swift
func application(
    _ application: UIApplication,
    didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
) -> Bool {
    // 全局导航栏
    UINavigationBar.appearance().tintColor = .white

    // 仅工具栏里的按钮
    UIBarButtonItem.appearance(whenContainedInInstancesOf: [UIToolbar.self])
        .tintColor = .systemOrange

    return true
}
```

---

## 参考链接

- [UIAppearance](https://developer.apple.com/documentation/uikit/uiappearance)
- [UIAppearanceContainer](https://developer.apple.com/documentation/uikit/uiappearancecontainer)
- [WWDC 2011 Session 114 — Customizing the Appearance of UIKit Controls](https://developer.apple.com/videos/play/wwdc2011/114/)
- [forwardInvocation:](https://developer.apple.com/documentation/objectivec/nsobject-swift.class/forwardinvocation:)
- [_UIAppearance：接口与按接口实现](./_UIAppearance.md)
- [appearance()](https://developer.apple.com/documentation/uikit/uiappearance/appearance())
- [appearance(whenContainedInInstancesOf:)](https://developer.apple.com/documentation/uikit/uiappearance/appearance(whencontainedininstancesof:))
- [appearance(for:)](https://developer.apple.com/documentation/uikit/uiappearance/appearance(for:))
- [UI_APPEARANCE_SELECTOR](https://developer.apple.com/documentation/uikit/ui_appearance_selector)

---

*来源：Apple Developer Documentation © 2026 Apple Inc.*
