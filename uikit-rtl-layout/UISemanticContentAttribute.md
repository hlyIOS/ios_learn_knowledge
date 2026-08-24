# UISemanticContentAttribute 与 semanticContentAttribute

> 整理自 Apple Developer Documentation  
> - [UISemanticContentAttribute](https://developer.apple.com/documentation/uikit/uisemanticcontentattribute)  
> - [UIView.semanticContentAttribute](https://developer.apple.com/documentation/uikit/uiview/semanticcontentattribute)

---

## 概述

**UISemanticContentAttribute** 是对视图内容的语义描述，用于决定在 LTR（从左到右）与 RTL（从右到左）布局切换时，视图是否应被翻转（mirror）。

**UIView.semanticContentAttribute** 是 UIView 上对应的属性，类型为 `UISemanticContentAttribute`，用于为具体视图设置该语义。

```swift
enum UISemanticContentAttribute
```

```swift
var semanticContentAttribute: UISemanticContentAttribute { get set }
```

---

## UIView.semanticContentAttribute

### 说明

某些视图在 LTR / RTL 布局切换时不应被翻转。例如：

- 播放控件（Play、Rewind、Fast Forward、进度条等）
- 表示物理方向的控件（上、下、左、右），其方向含义不随语言环境改变

不要只考虑「要不要翻转」，而应选择**最能描述该视图语义**的 `UISemanticContentAttribute` 值。

### 布局子视图

当创建的视图包含子视图时，可使用 [`userInterfaceLayoutDirection(for:)`](https://developer.apple.com/documentation/uikit/uiview/userinterfacelayoutdirection(for:)) 类方法，判断子视图是否应被翻转，并按正确顺序布局。

```swift
class func userInterfaceLayoutDirection(for attribute: UISemanticContentAttribute) -> UIUserInterfaceLayoutDirection
```

**参数**

| 参数 | 说明 |
|------|------|
| `attribute` | 视图的 semantic content attribute |

**返回值**

用户界面布局方向（`leftToRight` 或 `rightToLeft`）。

**用法说明**

创建包含子视图的视图时，可用此方法判断子视图是否应翻转，并按合适顺序排列。

---

## UISemanticContentAttribute 枚举值

### `.unspecified`

视图的默认值。

```swift
case unspecified
```

**行为：** 在 LTR 与 RTL 布局切换时，**会**被翻转。

---

### `.playback`

表示播放控件的视图，例如 Play、Rewind、Fast Forward 按钮或播放进度条（playhead scrubber）。

```swift
case playback
```

**行为：** 在 LTR 与 RTL 布局切换时，**不会**被翻转。

---

### `.spatial`

表示方向性控件的视图，例如文本对齐的分段控件，或游戏中的 D-pad 方向键。

```swift
case spatial
```

**行为：** 在 LTR 与 RTL 布局切换时，**不会**被翻转。

---

### `.forceLeftToRight`

始终按 LTR（从左到右）布局显示。

```swift
case forceLeftToRight
```

---

### `.forceRightToLeft`

始终按 RTL（从右到左）布局显示。

```swift
case forceRightToLeft
```

---

## 快速对照

| 枚举值 | RTL 切换时是否翻转 | 典型场景 |
|--------|-------------------|----------|
| `.unspecified` | ✅ 会翻转 | 默认，随系统布局方向 |
| `.playback` | ❌ 不翻转 | 播放/媒体控制 |
| `.spatial` | ❌ 不翻转 | 物理方向、对齐、游戏方向键 |
| `.forceLeftToRight` | 强制 LTR | 固定从左到右 |
| `.forceRightToLeft` | 强制 RTL | 固定从右到左 |

---

## 使用示例

```swift
// 播放按钮：RTL 下不翻转
playButton.semanticContentAttribute = .playback

// 文本对齐分段控件：方向语义固定
alignmentControl.semanticContentAttribute = .spatial

// 根据 semantic attribute 决定子视图布局方向
let direction = UIView.userInterfaceLayoutDirection(for: container.semanticContentAttribute)
if direction == .rightToLeft {
    // 按 RTL 顺序排列子视图
}
```

---

## 参考链接

- [UISemanticContentAttribute](https://developer.apple.com/documentation/uikit/uisemanticcontentattribute)
- [UIView.semanticContentAttribute](https://developer.apple.com/documentation/uikit/uiview/semanticcontentattribute)
- [UIView.userInterfaceLayoutDirection(for:)](https://developer.apple.com/documentation/uikit/uiview/userinterfacelayoutdirection(for:))

---

*来源：Apple Developer Documentation © 2026 Apple Inc.*
