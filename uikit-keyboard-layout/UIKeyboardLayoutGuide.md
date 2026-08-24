# UIKeyboardLayoutGuide 与 keyboardLayoutGuide

> 整理自 Apple Developer Documentation  
> - [UIKeyboardLayoutGuide](https://developer.apple.com/documentation/uikit/uikeyboardlayoutguide)  
> - [UIView.keyboardLayoutGuide](https://developer.apple.com/documentation/uikit/uiview/keyboardlayoutguide)  
> - [Adjusting your layout with keyboard layout guide](https://developer.apple.com/documentation/uikit/adjusting-your-layout-with-keyboard-layout-guide)

---

## 一句话

**`keyboardLayoutGuide` 是 View 上的一块「假区域」，位置跟着键盘走。**  
把输入框、工具栏约束到它上面，键盘弹出/收起时布局会自动让开，不必再监听 `keyboardWillShow` 自己改 constant。

| 名称 | 是什么 |
|------|--------|
| `UIView.keyboardLayoutGuide` | View 上的属性，拿到这块区域 |
| `UIKeyboardLayoutGuide` | 这块区域的类型（继承自可跟踪边缘的 Layout Guide） |

```swift
var keyboardLayoutGuide: UIKeyboardLayoutGuide { get }
```

```swift
@MainActor class UIKeyboardLayoutGuide
```

---

## 最常用的写法

把内容底边贴在键盘**顶边**上方（系统间距）：

```swift
view.keyboardLayoutGuide.topAnchor.constraint(
    equalToSystemSpacingBelow: textView.bottomAnchor,
    multiplier: 1.0
).isActive = true
```

| 键盘状态 | Guide 在哪（默认） |
|----------|-------------------|
| 已弹出、贴在屏幕底部（docked） | 跟着键盘，顶边 ≈ 键盘顶边 |
| 收起 / 不在屏幕上 | 贴在窗口底部，高度等于底部 safe area |
| 悬浮、拆分、未停靠 | **默认当「没键盘」处理**（见下文 `followsUndockedKeyboard`） |

可以当普通 `UILayoutGuide` 用：`topAnchor`、`bottomAnchor`、`leadingAnchor` 等。

---

## 三个配置项

### followsUndockedKeyboard

悬浮/未停靠键盘要不要跟着走。

```swift
var followsUndockedKeyboard: Bool { get set }
```

| 值 | 行为 |
|----|------|
| `false`（默认） | 只跟踪**贴在底部**的键盘 |
| `true` | 悬浮、拆分、未停靠也跟踪 |

默认 `false` 时：键盘离开底部或关掉，guide 的 `topAnchor` 对齐 `safeAreaLayoutGuide.bottomAnchor`。

iPad 上要跟着手动拖动的键盘，需要设为 `true`，并用下面的「靠近某条边时切换约束」。

### keyboardDismissPadding

加大键盘**上方**可滑动收起的触摸区域。

```swift
var keyboardDismissPadding: CGFloat { get set }
```

默认 `0`；负数按 `0` 处理。交互键盘（interactive dismiss）时，输入区紧贴键盘，手势不好划，可把这个值调大一点。

### usesBottomSafeArea

guide 是否把 View 的 **safe area** 算进去（底部 Home 条等）。

```swift
var usesBottomSafeArea: Bool { get set }
```

键盘收起时，guide 高度通常等于底部安全区；多数情况保持默认即可。

---

## 悬浮键盘：靠近边缘时切换约束

`followsUndockedKeyboard = true` 之后，直接写死 `keyboardLayoutGuide.topAnchor` **不会**随「靠近哪条边」自动开/关。

要用 `UITrackingLayoutGuide` 的方法：

```swift
setConstraints(_: activeWhenNearEdge:)   // 靠近某条边时启用
setConstraints(_: activeWhenAwayFrom:)   // 远离某条边时启用
```

**例子：** 键盘不靠顶部时，编辑区贴键盘；靠顶部时改贴 safe area，避免飞出屏幕。

```swift
view.keyboardLayoutGuide.followsUndockedKeyboard = true

let onKeyboard = view.keyboardLayoutGuide.topAnchor.constraint(equalTo: editView.bottomAnchor)
view.keyboardLayoutGuide.setConstraints([onKeyboard], activeWhenAwayFrom: .top)

let onSafeBottom = view.safeAreaLayoutGuide.bottomAnchor.constraint(equalTo: editView.bottomAnchor)
view.keyboardLayoutGuide.setConstraints([onSafeBottom], activeWhenNearEdge: .top)
```

约束**不必**连到 keyboard guide 本身。例如图片没有约束到键盘，但仍可在键盘靠近左边时把图片推到右边。

多条边要**同时**满足才会切换，例如 `activeWhenAwayFrom: [.leading, .trailing, .bottom]` 只在「左右底都不靠」时启用（悬浮键盘在屏幕中部等）。

---

## 不同键盘「靠近哪些边」

| 键盘类型 | near / awayFrom（系统规则） |
|----------|----------------------------|
| 底部停靠（Docked） | 总是靠近 **bottom**；远离 leading / trailing / top |
| 拆分、未停靠 | 总是远离 leading / trailing / bottom；可能靠近 **top** |
| 悬浮（Floating） | 可远离所有边，可靠近任意一边或**相邻两边** |
| 外接键盘快捷栏 | 靠近 bottom，远离 top；收起时可靠近 leading 或 trailing |

悬浮键盘在 App 上方但没盖住当前 App 时，guide 按**已收起**处理。  
1/3 分屏时：远离左右边；停靠则靠近底；悬浮/未停靠可靠近顶。

---

## 和自己监听键盘通知的差别

| | `keyboardLayoutGuide` | `keyboardWillShow` 改约束 |
|---|------------------------|---------------------------|
| 弹出/收起 | Auto Layout 自动跟 | 自己算高度、动画 |
| 旋转、分屏 | 系统更新 guide | 容易漏 |
| 悬浮键盘 | `followsUndockedKeyboard` + tracking | 更麻烦 |

新代码优先用 layout guide。

---

## 最小示例

```swift
override func viewDidLoad() {
    super.viewDidLoad()

    textView.translatesAutoresizingMaskIntoConstraints = false
    view.addSubview(textView)

    NSLayoutConstraint.activate([
        textView.leadingAnchor.constraint(equalTo: view.safeAreaLayoutGuide.leadingAnchor),
        textView.trailingAnchor.constraint(equalTo: view.safeAreaLayoutGuide.trailingAnchor),
        textView.topAnchor.constraint(equalTo: view.safeAreaLayoutGuide.topAnchor),
        view.keyboardLayoutGuide.topAnchor.constraint(
            equalToSystemSpacingBelow: textView.bottomAnchor,
            multiplier: 1.0
        )
    ])
}
```

---

## 参考链接

- [UIKeyboardLayoutGuide](https://developer.apple.com/documentation/uikit/uikeyboardlayoutguide)
- [UIView.keyboardLayoutGuide](https://developer.apple.com/documentation/uikit/uiview/keyboardlayoutguide)
- [Adjusting your layout with keyboard layout guide](https://developer.apple.com/documentation/uikit/adjusting-your-layout-with-keyboard-layout-guide)
- [UILayoutGuide](https://developer.apple.com/documentation/uikit/uilayoutguide)

---

*来源：Apple Developer Documentation © 2026 Apple Inc.*
