# UIUserInterfaceLayoutDirection

> 整理自 Apple Developer Documentation  
> - [UIUserInterfaceLayoutDirection](https://developer.apple.com/documentation/uikit/uiuserinterfacelayoutdirection)  
> - [UIApplication.userInterfaceLayoutDirection](https://developer.apple.com/documentation/uikit/uiapplication/userinterfacelayoutdirection)  
> - [UIView.effectiveUserInterfaceLayoutDirection](https://developer.apple.com/documentation/uikit/uiview/effectiveuserinterfacelayoutdirection)  
> - [UIView.userInterfaceLayoutDirection(for:)](https://developer.apple.com/documentation/uikit/uiview/userinterfacelayoutdirection(for:))  
> - [UIView.userInterfaceLayoutDirection(for:relativeTo:)](https://developer.apple.com/documentation/uikit/uiview/userinterfacelayoutdirection(for:relativeto:))

---

## 概述

**UIUserInterfaceLayoutDirection** 是 UIKit 中用于描述用户界面**布局流向**的枚举，表示界面内容是从左到右（LTR）还是从右到左（RTL）排列。

`UIApplication.userInterfaceLayoutDirection` 等 API 会返回该枚举值，用于反映 App 或视图当前的布局方向。

```swift
enum UIUserInterfaceLayoutDirection
```

---

## 枚举值

### `.leftToRight`

布局方向为从左到右（LTR）。

```swift
case leftToRight
```

**典型场景：** 英文、中文等 LTR 语言环境。

---

### `.rightToLeft`

布局方向为从右到左（RTL）。

```swift
case rightToLeft
```

**典型场景：** 阿拉伯语、希伯来语等 RTL 语言环境。

---

## 快速对照

| 枚举值 | 含义 | 典型场景 |
|--------|------|----------|
| `.leftToRight` | 从左到右 | 英文、中文等 LTR 语言 |
| `.rightToLeft` | 从右到左 | 阿拉伯语、希伯来语等 RTL 语言 |

---

## UIView.userInterfaceLayoutDirection(for:)

> 文档：[userInterfaceLayoutDirection(for:)](https://developer.apple.com/documentation/uikit/uiview/userinterfacelayoutdirection(for:))

根据视图的 `semanticContentAttribute`，返回该视图应采用的布局方向。

```swift
class func userInterfaceLayoutDirection(for attribute: UISemanticContentAttribute) -> UIUserInterfaceLayoutDirection
```

**可用版本：** iOS 9.0+、iPadOS 9.0+、Mac Catalyst 13.1+、tvOS 9.0+、visionOS 1.0+

### 参数

| 参数 | 说明 |
|------|------|
| `attribute` | 视图的 semantic content attribute |

### 返回值

用户界面布局方向（`leftToRight` 或 `rightToLeft`）。

### 说明

创建包含子视图的自定义视图时，可用此方法：

1. 判断子视图在 RTL 下是否应被翻转（mirror）
2. 按正确的顺序排列子视图

传入的 `attribute` 通常来自容器视图自身的 `semanticContentAttribute`。与 [`UISemanticContentAttribute`](UISemanticContentAttribute.md) 配合使用。

### 与 semanticContentAttribute 的对应关系

| semanticContentAttribute | 返回的布局方向 |
|--------------------------|----------------|
| `.forceLeftToRight` | 始终 `.leftToRight` |
| `.forceRightToLeft` | 始终 `.rightToLeft` |
| `.unspecified` | 跟随当前 App / 系统布局方向 |
| `.playback` / `.spatial` | 不随语言翻转，相对当前环境取反（见下方 `relativeTo`） |

> 注意：`.unspecified` 的结果取决于当前 App 布局环境，而非某个子视图自身的 attribute。若需判断「当前界面是否为 RTL」，应传入 `.unspecified`，而不是子视图自己的 `semanticContentAttribute`。

---

## UIView.userInterfaceLayoutDirection(for:relativeTo:)

> 文档：[userInterfaceLayoutDirection(for:relativeTo:)](https://developer.apple.com/documentation/uikit/uiview/userinterfacelayoutdirection(for:relativeto:)

在指定基准布局方向的前提下，计算 semantic content attribute 所隐含的布局方向。

```swift
class func userInterfaceLayoutDirection(
    for semanticContentAttribute: UISemanticContentAttribute,
    relativeTo layoutDirection: UIUserInterfaceLayoutDirection
) -> UIUserInterfaceLayoutDirection
```

### 参数

| 参数 | 说明 |
|------|------|
| `semanticContentAttribute` | 视图的 semantic content attribute |
| `layoutDirection` | 基准布局方向（`.leftToRight` 或 `.rightToLeft`） |

### 返回值

semantic content attribute 相对于基准方向所隐含的布局方向。

### 说明

例如：基准方向为 `.rightToLeft`，attribute 为 `.playback` 时，返回 `.leftToRight`——播放控件在 RTL 环境下不翻转，因此其有效方向与容器相反。

布局和绘制代码可用此方法推算元素排列方式；若已有容器视图，直接读取 `effectiveUserInterfaceLayoutDirection` 通常更简单。

---

## 三种获取布局方向的方式

| API | 适用场景 |
|-----|----------|
| `UIApplication.shared.userInterfaceLayoutDirection` | 获取 App 全局布局方向（iOS 5.0+，App Extension 不可用） |
| `view.effectiveUserInterfaceLayoutDirection` | **推荐**。布局/绘制已有视图时，综合 semantic attribute、trait 环境、App 方向 |
| `UIView.userInterfaceLayoutDirection(for:)` | 自定义容器视图、尚无实例时，根据 attribute 预先计算方向 |

---

## 相关 API

### UIApplication.userInterfaceLayoutDirection

App 级别的用户界面布局方向。

```swift
var userInterfaceLayoutDirection: UIUserInterfaceLayoutDirection { get }
```

**说明：** 表示整个 App 界面的一般布局流向。

---

### UIView.effectiveUserInterfaceLayoutDirection

视图当前应采用的布局方向。

```swift
var effectiveUserInterfaceLayoutDirection: UIUserInterfaceLayoutDirection { get }
```

**说明：** 在布局或绘制视图的直接内容时，应始终参考此属性。该值**不会**自动向下传递给子视图树，每个视图需单独查询。

---

## 使用示例

```swift
// 1. 获取 App 全局布局方向
let appDirection = UIApplication.shared.userInterfaceLayoutDirection

// 2. 获取视图的有效布局方向（布局/绘制时优先使用）
let viewDirection = myView.effectiveUserInterfaceLayoutDirection

// 3. 判断当前界面是否为 RTL（传入 .unspecified）
let isRTL = UIView.userInterfaceLayoutDirection(for: .unspecified) == .rightToLeft

// 4. 自定义容器：根据 semantic attribute 排列子视图
let direction = UIView.userInterfaceLayoutDirection(for: container.semanticContentAttribute)
let orderedSubviews = direction == .rightToLeft
    ? subviews.reversed()
    : subviews

// 5. 指定基准方向，计算 playback 控件的有效方向
let playbackDirection = UIView.userInterfaceLayoutDirection(
    for: .playback,
    relativeTo: .rightToLeft
) // 返回 .leftToRight
```

---

## 参考链接

- [UIUserInterfaceLayoutDirection](https://developer.apple.com/documentation/uikit/uiuserinterfacelayoutdirection)
- [UIApplication.userInterfaceLayoutDirection](https://developer.apple.com/documentation/uikit/uiapplication/userinterfacelayoutdirection)
- [UIView.effectiveUserInterfaceLayoutDirection](https://developer.apple.com/documentation/uikit/uiview/effectiveuserinterfacelayoutdirection)
- [UIView.userInterfaceLayoutDirection(for:)](https://developer.apple.com/documentation/uikit/uiview/userinterfacelayoutdirection(for:))
- [UIView.userInterfaceLayoutDirection(for:relativeTo:)](https://developer.apple.com/documentation/uikit/uiview/userinterfacelayoutdirection(for:relativeto:))
- [UISemanticContentAttribute 与 semanticContentAttribute](UISemanticContentAttribute.md)

---

*来源：Apple Developer Documentation © 2026 Apple Inc.*
