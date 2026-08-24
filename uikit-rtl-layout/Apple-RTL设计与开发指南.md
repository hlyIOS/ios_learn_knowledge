# Apple RTL 设计与开发指南

> 整理自 Apple 官方文档  
> - [Right to left · Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/right-to-left)  
> - [Supporting Right-to-Left Languages · Internationalization Guide (Archive)](https://developer.apple.com/library/archive/documentation/MacOSX/Conceptual/BPInternational/SupportingRight-To-LeftLanguages/SupportingRight-To-LeftLanguages.html)

与 API 详解见本目录：[概念总览](./概念总览.md) · [UISemanticContentAttribute](./UISemanticContentAttribute.md)

---

## 概述

两份文档互补：

| 文档 | 类型 | 回答的问题 |
|------|------|-----------|
| **HIG · Right to left** | 设计规范 | 界面**应该怎么设计**才符合 RTL 用户习惯 |
| **Supporting RTL Languages** | 开发指南 | 代码里**怎么实现**镜像、对齐、测试 |

**核心原则：** 显示 RTL 语言（如阿拉伯语、希伯来语）时，用户界面应**按需镜像**，使布局与文字阅读方向一致。使用 Base Internationalization + Auto Layout + 系统控件时，大部分 UI 会自动镜像。

---

# 一、HIG 设计规范

> 来源：[Right to left](https://developer.apple.com/design/human-interface-guidelines/right-to-left)

## 1.1 系统默认支持

系统提供的 UI 框架**默认支持 RTL**，系统控件可在 RTL 环境下自动翻转。若 App 主要使用系统元素和标准布局，可能无需额外改动。

## 1.2 文本对齐（Text Alignment）

| 场景 | 建议 |
|------|------|
| 一般 UI 文本 | 跟随界面方向对齐；LTR 左对齐的内容，在 RTL 下应右对齐 |
| **段落**（3 行及以上） | 按**段落本身的语言**对齐，而非按当前界面方向 |
| 一两行短文本 | 跟随当前界面的阅读方向 |

**原因：** 若 LTR 语言的段落被强制右对齐，每行开头难以辨认，影响阅读。

## 1.3 应翻转的控件

| 类型 | 说明 |
|------|------|
| **进度控件** | 滑块、进度条等；「前进」方向应与阅读方向一致， accompanying 图标/数值位置也需反转 |
| **导航控件** | RTL 下返回按钮应指向**右**；上一/下一按钮需翻转以匹配阅读顺序 |
| **有序列表导航** | 帮助用户按固定顺序访问条目的按钮需翻转 |

## 1.4 界面图标（Interface Icons）

| 应翻转 | 不应翻转 |
|--------|----------|
| 表示文字或阅读方向的图标（如 LTR 下左对齐文本条 → RTL 下右对齐） | Logo、品牌标识 |
| 表示「沿阅读方向移动」的图标 | 通用符号（如播放键） |
| | 表示真实世界物体的图标 |

## 1.5 设计层与 API 的对应

| HIG 设计意图 | UIKit API |
|-------------|-----------|
| 一般 UI 随语言翻转 | `.unspecified`（默认） |
| 播放/媒体控件不翻转 | `.playback` |
| 物理/绝对方向控件不翻转 | `.spatial` |
| 强制固定方向 | `.forceLeftToRight` / `.forceRightToLeft` |

详见 [UISemanticContentAttribute](./UISemanticContentAttribute.md)。

---

# 二、开发指南：创建 RTL 界面

> 来源：[Supporting Right-to-Left Languages](https://developer.apple.com/library/archive/documentation/MacOSX/Conceptual/BPInternational/SupportingRight-To-LeftLanguages/SupportingRight-To-LeftLanguages.html)

## 2.1 基本原则

- 使用 **Base Internationalization** 和 **Auto Layout**
- 开发语言为 LTR 时：从**左上角**开始对齐，约束向**右下**扩展
- 使用 **`leading` / `trailing`**，**不要**用 `left` / `right`
- iOS 9+ 链接的 UIKit 控件会自动翻转
- 文本方向变为 RTL；**电话号码和国家代码始终 LTR**
- 部分 View/控件不会自动变向，需按需编程修复

## 2.2 不应翻转的内容

| 类型 | 说明 |
|------|------|
| 视频控件和时间轴指示器 | 媒体语义，全球统一 |
| 图片 | 除非传达方向感（如箭头） |
| 时钟 | 物理方向固定 |
| 音符和五线谱 | 音乐符号固定 |
| 图表 | x/y 轴方向保持不变 |

必要时为 RTL 语言提供语言特定的图片和音频资源。

## 2.3 获取布局方向

### iOS

```objc
// Objective-C（Archive 原文）
if ([UIView userInterfaceLayoutDirectionForSemanticContentAttribute:view.semanticContentAttribute]
    == UIUserInterfaceLayoutDirectionRightToLeft) {
    // ...
}
```

```swift
// Swift 现代写法
if view.effectiveUserInterfaceLayoutDirection == .rightToLeft {
    // ...
}
```

详见 [UIUserInterfaceLayoutDirection](./UIUserInterfaceLayoutDirection.md)。

### macOS

```objc
if (view.userInterfaceLayoutDirection == NSUserInterfaceLayoutDirectionRightToLeft) {
    // ...
}
```

---

# 三、文本对齐与书写方向

## 3.1 Natural 对齐（推荐）

| 平台 | 默认 |
|------|------|
| iOS | Natural |
| macOS | Left（需手动设为 Natural） |

使用 **Natural** 对齐：LTR 语言左对齐，RTL 语言自动镜像为右对齐。

```objc
// Objective-C
[(NSMutableParagraphStyle *)paraStyle setAlignment:NSNaturalTextAlignment];
```

```swift
// Swift
paragraphStyle.alignment = .natural
label.textAlignment = .natural
```

## 3.2 反向对齐（特殊场景）

若控件在 LTR 下应**靠右**（RTL 下靠左），需读取布局方向后手动设置：

```objc
// macOS 示例
if ([NSApp userInterfaceLayoutDirection] == NSUserInterfaceLayoutDirectionRightToLeft) {
    [(NSMutableParagraphStyle *)paraStyle setAlignment:NSLeftTextAlignment];
} else {
    [(NSMutableParagraphStyle *)paraStyle setAlignment:NSRightTextAlignment];
}
```

## 3.3 Natural 书写方向

运行时根据文本前几个字符自动判断书写方向。

| 平台 | API |
|------|-----|
| iOS | `UITextInput` 的 `setBaseWritingDirection(_:for:)`，传 `UITextWritingDirectionNatural` |
| macOS | `setBaseWritingDirection:`，传 `NSWritingDirectionNatural` |

---

# 四、双向文本（Bidirectional Text）

RTL 语言中，数字和拉丁文本（如未本地化的产品名）仍从左到右书写。

## 4.1 一般原则

- 优先使用**标准控件**，自动处理 Bidi 文本
- 自定义文本输入控件需自行处理

## 4.2 Unicode 方向标记

默认行为不正确时，使用 Unicode 双向算法中的不可见字符：

| 字符 | Unicode | 用途 |
|------|---------|------|
| LRE（Left-to-Right Embedding） | U+202A | 强制后续内容为 LTR |
| RLE（Right-to-Left Embedding） | U+202B | 强制后续内容为 RTL |
| PDF（Pop Directional Formatting） | U+202C | 结束嵌入 |
| LRM（Left-to-Right Mark） | U+200E | 在变量前插入，用于 LTR 本地化 |
| RLM（Right-to-Left Mark） | U+200F | 在变量前插入，用于 RTL 语言 |

### 电话号码（始终 LTR）

```swift
let phoneNumber = "408-555-1212"
let localized = "\u{202A}\(phoneNumber)\u{202C}"
```

### 变量在字符串开头

若变量方向未知（如用户名可能是阿拉伯语或英语），在变量前插入方向标记，避免整串文本被错误定向。

更细的对比（钉子 vs 嵌入 vs 隔离、头尾 `200E` 能否只留开头）见 [Unicode 双向文本方向标记](./Unicode双向文本方向标记.md)。

---

# 五、编程方式控制翻转（iOS）

## 5.1 Semantic Content Attribute

iOS 9+ 使用 `UIView` 的 semantic content attributes 指定 RTL 下的显示方式。

```swift
// 视频 scrubber 滑块：不翻转
slider.semanticContentAttribute = .playback
```

详见 [UISemanticContentAttribute](./UISemanticContentAttribute.md)。

## 5.2 图片翻转

三种方式任选：

| 方式 | 说明 |
|------|------|
| 编程翻转 | `UIImage.imageFlippedForRightToLeftLayoutDirection()` |
| 本地化资源 | 为 RTL 语言提供单独图片 |
| Asset Catalog | 添加 RTL 备用图片，代码中按语言选用 |

Asset Catalog 中可将 Image 的 Direction 设为 **Both**，配合 layout direction 自动镜像。

---

# 六、编程方式控制翻转（macOS 摘要）

> Archive 文档含 Mac 专属内容，iOS 开发者可略读。

| 场景 | 处理方式 |
|------|----------|
| 图片 | 编程镜像 / 本地化资源 / `awakeFromNib` 中翻转 |
| 工具栏 | RTL 下反转 toolbar items 顺序 |
| 表格 | Cell-based 表格设 Natural 对齐；列顺序需 `reverseColumnOrder` |

---

# 七、测试

## 7.1 Xcode Scheme

- 无需改为阿拉伯语/希伯来语即可测试 RTL 区域格式
- 在 Scheme Editor 中更改语言和地区 → [Testing Specific Languages and Regions](https://developer.apple.com/library/archive/documentation/MacOSX/Conceptual/BPInternational/TestingYourInternationalApp/TestingYourInternationalApp.html)
- 仅改文字方向不改语言 → **Right-to-Left Pseudolanguage**

## 7.2 真机测试

| 平台 | 方法 |
|------|------|
| iOS | 设置 → 地区；阿拉伯语可从子菜单选具体国家；希伯来语选 Hebrew (Israel) |
| macOS | 系统偏好设置 → 语言与地区 → Region + Format language 分别设置 |

## 7.3 建议

- 在 RTL 语言下**充分测试** UI
- 仅在必要时修改代码
- 推荐观看 [Get it right (to left) · WWDC22](https://developer.apple.com/videos/play/wwdc2022/10107/)

---

# 八、快速对照

## 设计 vs 开发

| 设计规范（HIG） | 开发实现（Archive + API） |
|----------------|-------------------------|
| 界面按需镜像 | Base i18n + Auto Layout leading/trailing |
| 文本 Natural 对齐 | `textAlignment = .natural` |
| 进度/导航控件翻转 | 默认 `.unspecified` |
| 播放键/时钟不翻转 | `.playback` / 不翻转列表 |
| 对齐分段控件不翻转 | `.spatial` |
| 方向性图标翻转 | Asset Both + 系统镜像 |
| 电话号码始终 LTR | Unicode LRE/PDF 包裹 |

## 与本目录文档的关系

```
Apple HIG + Archive 指南（本文）
        ↓ 设计原则 & 实践清单
uikit-rtl-layout/ API 文档
        ↓ Trait / Semantic / LayoutDirection
App 内语言设置控制 RTL-LTR.md
        ↓ 实战落地
```

---

## 参考链接

- [Right to left · HIG](https://developer.apple.com/design/human-interface-guidelines/right-to-left)
- [Supporting Right-to-Left Languages (Archive)](https://developer.apple.com/library/archive/documentation/MacOSX/Conceptual/BPInternational/SupportingRight-To-LeftLanguages/SupportingRight-To-LeftLanguages.html)
- [Get it right (to left) · WWDC22](https://developer.apple.com/videos/play/wwdc2022/10107/)
- [UISemanticContentAttribute](https://developer.apple.com/documentation/uikit/uisemanticcontentattribute)
- [UIImage.imageFlippedForRightToLeftLayoutDirection()](https://developer.apple.com/documentation/uikit/uiimage/imageflippedforrighttoleftlayoutdirection())

---

*来源：Apple Human Interface Guidelines · Apple Developer Documentation (Archive) © Apple Inc.*
