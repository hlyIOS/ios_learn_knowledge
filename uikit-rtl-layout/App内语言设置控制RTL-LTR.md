# App 内语言设置控制整体 RTL / LTR

> 场景：App 内有语言设置；**有设置**时按 App 语言决定 RTL/LTR，**未设置**时跟随系统语言。

---

## 一句话结论

**改环境层，不是语义层，也不是结果层。**

| 要控制什么 | 在哪一步改 |
|-----------|-----------|
| 整个 App 的 RTL / LTR | **① 环境层** — Override `UIApplication.userInterfaceLayoutDirection` |
| 文案语言（Localized String） | **① 环境层** — `UserDefaults` 的 `AppleLanguages` + 自定义 `Bundle` |
| 个别控件例外（如播放键） | ② 语义层 — `semanticContentAttribute` |

---

## 决策流程

```
用户打开 App
    ↓
读取 App 内语言设置
    ↓
┌─ 有设置 ──→ 用 App 语言判断 RTL / LTR
│
└─ 未设置 ──→ 用系统语言判断 RTL / LTR（默认行为）
    ↓
写入环境层（UIApplication + Trait）
    ↓
整个 App 的基准布局方向更新
    ↓
各控件结合 semanticContentAttribute 算出最终方向
```

---

## 你要改的两件事

App 内切语言通常涉及**两个独立但都要改的环境配置**：

| 配置项 | 作用 | 不改会怎样 |
|--------|------|-----------|
| **布局方向** | 控制 RTL / LTR | 文案变了，但界面还是从左往右排 |
| **文案语言** | 控制 `NSLocalizedString` 取哪份翻译 | 布局变了，但文案还是旧语言 |

两者都在**环境层**处理，但 API 不同。

---

## 第一步：判断当前该用哪种方向

```swift
enum AppLanguageDirection {
    case ltr
    case rtl
}

/// App 内语言 code，nil 表示跟随系统
var appLanguageCode: String?

func resolvedLayoutDirection() -> UIUserInterfaceLayoutDirection {
    let code = appLanguageCode ?? Locale.preferredLanguages.first ?? "en"
    return isRTLLanguage(code) ? .rightToLeft : .leftToRight
}

func isRTLLanguage(_ code: String) -> Bool {
    let rtlLanguages = ["ar", "he", "fa", "ur"] // 阿拉伯语、希伯来语等
    return rtlLanguages.contains { code.hasPrefix($0) }
}
```

| App 语言设置 | 实际使用的语言 | 方向 |
|-------------|---------------|------|
| `nil`（未设置） | 系统语言 | 系统语言是 RTL → RTL，否则 LTR |
| `"ar"` | 阿拉伯语 | RTL |
| `"en"` | 英语 | LTR |
| `"zh-Hans"` | 简体中文 | LTR |

---

## 第二步：Override 环境层 — 控制整体 RTL / LTR

这是**整个 App 布局方向的核心开关**。

### 1. 自定义 UIApplication

```swift
class AppApplication: UIApplication {
    override var userInterfaceLayoutDirection: UIUserInterfaceLayoutDirection {
        LanguageManager.shared.resolvedLayoutDirection()
    }
}
```

### 2. 用 main.swift 启动（去掉 @main / @UIApplicationMain）

```swift
// main.swift
import UIKit

UIApplicationMain(
    CommandLine.argc,
    CommandLine.unsafeArgv,
    NSStringFromClass(AppApplication.self),
    NSStringFromClass(AppDelegate.self)
)
```

> `userInterfaceLayoutDirection` 在视图创建/更新时会被多次读取，**内部逻辑要保持轻量**（读 UserDefaults / 内存变量即可）。

### 3. 为什么选这一层？

```
UIApplication.userInterfaceLayoutDirection  ← 你要改这里（App 全局）
        ↓
UITraitCollection.layoutDirection 传播
        ↓
基准布局方向
        ↓
effectiveUserInterfaceLayoutDirection（各控件最终结果）
```

改这里 = 改整个 App 的「大背景规则」，所有 `.unspecified` 控件都会跟随。

### 4. `userInterfaceLayoutDirection` 的作用范围

文档里出现了多个名字相似的 API，**作用范围不同**，不要混用：

| API | 作用范围 | 角色 |
|-----|----------|------|
| `UIApplication.shared.userInterfaceLayoutDirection` | **整个 App** | 全局基准方向（你要 override 的） |
| `UIView.userInterfaceLayoutDirection(for:)` | **计算用** | 根据 semantic attribute 推算方向 |
| `view.effectiveUserInterfaceLayoutDirection` | **单个 View** | 该 View 的最终布局方向（只读） |

#### `UIApplication.userInterfaceLayoutDirection` 管什么？

它是 **App 级别的默认/基准方向**，影响的是「没有特别声明例外」的部分：

| ✅ 会被它影响 | ❌ 不会被它直接决定 |
|-------------|-------------------|
| 所有 `.unspecified` 的 View | 设了 `.forceLeftToRight` / `.forceRightToLeft` 的 View |
| Auto Layout 的 leading/trailing 默认解析 | 设了 `.playback` / `.spatial` 的 View |
| `userInterfaceLayoutDirection(for: .unspecified)` 的返回值 | 子 VC 被 `setOverrideTraitCollection` 覆盖后的子树 |
| Trait 向下传播时的基础输入 | 手动 `transform` 翻转的 View |

#### 和语义层的关系：不是替代，是分层

```
UIApplication.userInterfaceLayoutDirection
    = App 全局基准（例如 RTL）
              ↓
    ┌─────────┼─────────┐
    ↓         ↓         ↓
.unspecified .playback .forceLeftToRight
    ↓         ↓         ↓
  跟随 RTL   保持 LTR   强制 LTR
（受它管）  （语义例外） （语义例外）
```

**局部用语义层，不是不用 `userInterfaceLayoutDirection`。**  
而是：全局靠它定基调，个别 View 用 `semanticContentAttribute` 在基调之上声明例外。

#### 三个层级的分工（App 内语言场景）

| 层级 | API | 范围 | 用途 |
|------|-----|------|------|
| App 全局 | `UIApplication.userInterfaceLayoutDirection` | 整个 App | App 内语言 → 整体 RTL/LTR |
| 页面级 | `setOverrideTraitCollection` | 某个子 VC 子树 | 整页固定方向 |
| 控件级 | `semanticContentAttribute` | 单个 View | 播放键、代码块等例外 |

#### 什么时候读、什么时候写？

```swift
// ✍️ 写（控制）— 只在 App 全局 override 一处
override var userInterfaceLayoutDirection: UIUserInterfaceLayoutDirection { ... }

// 👀 读（使用）— 自定义布局时
let direction = myView.effectiveUserInterfaceLayoutDirection  // 单个 View 最终结果
let isAppRTL = UIApplication.shared.userInterfaceLayoutDirection == .rightToLeft  // App 基准
```

> **一句话：`UIApplication.userInterfaceLayoutDirection` 定的是 App 全局「默认方向」；语义层是在这个默认之上做局部例外，两者配合而非互斥。**

---

## 第三步：控制文案语言（可选但通常需要）

```swift
// 保存 App 内语言设置
func setAppLanguage(_ code: String?) {
    if let code {
        UserDefaults.standard.set([code], forKey: "AppleLanguages")
    } else {
        UserDefaults.standard.removeObject(forKey: "AppleLanguages")
    }
    UserDefaults.standard.synchronize()
}
```

| 设置值 | 效果 |
|--------|------|
| 写入 `["ar"]` | 文案读阿拉伯语 `.lproj` |
| 删除 / `nil` | 回退到系统默认语言 |

运行时切换通常还需**自定义 Bundle** 读取对应 `.lproj`，或切换后**重建根界面**。

---

## 第四步：用户切换语言后刷新 UI

环境层改了之后，**已在屏幕上的 View 不会自动全部刷新**，需要主动触发：

```swift
func applyLanguageChange() {
    // 1. 保存语言设置（环境层 · 文案）
    LanguageManager.shared.setAppLanguage(selectedCode)

    // 2. 布局方向已通过 UIApplication override 生效（环境层 · 方向）

    // 3. 重建根界面
    guard let window = UIApplication.shared.connectedScenes
        .compactMap({ ($0 as? UIWindowScene)?.keyWindow }).first else { return }

    let root = MainTabBarController() // 重新创建
    window.rootViewController = root
    window.makeKeyAndVisible()

    // 可选：过渡动画
    UIView.transition(with: window, duration: 0.3, options: .transitionCrossDissolve, animations: nil)
}
```

---

## 完整链路图

```
┌─────────────────────────────────────────────────────────┐
│  App 内语言设置（UserDefaults / 内存）                    │
│  nil → 跟随系统 │ "ar" → RTL │ "en" → LTR               │
└──────────────────────────┬──────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────┐
│  ① 环境层 · 布局方向                                      │
│  AppApplication.userInterfaceLayoutDirection  ← 改这里    │
└──────────────────────────┬──────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────┐
│  ① 环境层 · 文案语言                                      │
│  AppleLanguages + Bundle  ← 改这里                      │
└──────────────────────────┬──────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────┐
│  UITraitCollection 向下传播 → 基准布局方向                 │
└──────────────────────────┬──────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────┐
│  ② 语义层 · semanticContentAttribute（个别控件例外）       │
└──────────────────────────┬──────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────┐
│  ③ 结果层 · effectiveUserInterfaceLayoutDirection        │
│  （只读，用来布局，不用来切换）                             │
└─────────────────────────────────────────────────────────┘
```

---

## 局部固定方向：环境层 + 语义层配合

**可以处理。** 这正是 Apple 分层设计的用意：

- **环境层** → 定整个 App 的默认方向
- **语义层** → 对个别 View 声明「我不跟随环境，我要固定 / 例外」

两者不冲突，是**组合使用**的关系。

### 分工

```
App 环境层（整体 RTL）
    │
    ├── 普通按钮 (.unspecified)     → 跟随 RTL ✅
    ├── 播放控件 (.playback)      → 固定不翻转 ✅
    ├── 方向键   (.spatial)        → 固定物理方向 ✅
    ├── 英文区块 (.forceLeftToRight) → 强制 LTR ✅
    └── 阿语预览 (.forceRightToLeft) → 强制 RTL ✅
```

### 四种例外方式

| semanticContentAttribute | 效果 | 典型场景 |
|--------------------------|------|----------|
| `.playback` | 不随环境翻转 | 播放/暂停、快进/快退、进度条 |
| `.spatial` | 不随环境翻转 | 游戏方向键、地图控件、物理「左/右/上/下」 |
| `.forceLeftToRight` | **永远 LTR**，无视 App 语言设置 | 代码编辑器、图表横轴、必须从左读的 UI |
| `.forceRightToLeft` | **永远 RTL**，无视 App 语言设置 | 强制展示阿语排版的内容块 |

### 代码示例

```swift
// App 整体已是 RTL（环境层已设置），但以下控件保持固定：

// 1. 媒体控件：全球统一方向，不镜像
playButton.semanticContentAttribute = .playback
progressView.semanticContentAttribute = .playback

// 2. 物理方向控件
dpadView.semanticContentAttribute = .spatial

// 3. 某块 UI 永远 LTR（即使 App 切到阿拉伯语）
codeEditorView.semanticContentAttribute = .forceLeftToRight

// 4. 某块 UI 永远 RTL（即使 App 是英文）
arabicPreviewPanel.semanticContentAttribute = .forceRightToLeft
```

### 容器级固定：整块区域统一方向

如果一个 **容器及其所有子视图** 都要固定方向，设容器即可：

```swift
// 整个卡片区域强制 LTR
cardContainer.semanticContentAttribute = .forceLeftToRight
// 子视图默认 .unspecified 时会跟随容器的有效方向
```

自定义布局容器内部排列子视图时：

```swift
let direction = UIView.userInterfaceLayoutDirection(
    for: container.semanticContentAttribute
)
// 按 direction 排列子视图
```

### 局部 override 环境（整块页面例外）

如果是一整个 **ViewController 子树** 需要与环境相反的方向，除了 semantic attribute，还可在 VC 层 override trait：

```swift
// 整个 App 是 RTL，但这个子页面强制 LTR
let ltrTraits = UITraitCollection(layoutDirection: .leftToRight)
setOverrideTraitCollection(ltrTraits, forChild: englishOnlyVC)
```

| 方式 | 粒度 | 适用 |
|------|------|------|
| `semanticContentAttribute` | 单个 View | 某个按钮、某块面板 |
| `setOverrideTraitCollection` | 整个子 VC 树 | 一整页都是固定方向 |
| `UIApplication` override | 整个 App | App 内语言设置 |

### 决策：用哪种方式？

```
需要固定方向的粒度是？
    │
    ├─ 整个 App          → 环境层 · UIApplication override
    ├─ 整个页面 / 子树    → 环境层 · setOverrideTraitCollection
    ├─ 某类控件（播放键等）→ 语义层 · .playback / .spatial
    └─ 某个 View / 容器   → 语义层 · .forceLeftToRight / .forceRightToLeft
```

### 注意

- 固定方向是**声明语义**，不要手动 `transform` 翻转来「抵消」环境层
- `.forceLeftToRight` / `.forceRightToLeft` 是**强制覆盖**，优先级高于环境层
- 子 View 的 `effectiveUserInterfaceLayoutDirection` 需**单独查询**，不会自动继承父 View 的计算结果（但 Auto Layout 的 leading/trailing 会跟随各自的有效方向）

---

## 开发 Checklist

### 环境层（必做）

- [ ] 自定义 `UIApplication`，override `userInterfaceLayoutDirection`
- [ ] `main.swift` 启动自定义 Application
- [ ] App 语言 `nil` 时回退系统语言
- [ ] 切换语言后重建根 ViewController

### 布局规范（必做）

- [ ] Auto Layout 用 **leading / trailing**，不用 left / right
- [ ] 文本对齐用 **`.natural`**（大部分情况）

### 语义层（按需）

- [ ] 播放键等媒体控件设 `.playback`
- [ ] 方向键等设 `.spatial`

### 不要做的事

- [ ] ❌ 不要用 `effectiveUserInterfaceLayoutDirection` 当开关
- [ ] ❌ 不要手动 `transform` 翻转整个界面
- [ ] ❌ 不要给每个 view 设 `.forceRightToLeft` 来切整体方向

---

## 已知注意点

| 问题 | 说明 |
|------|------|
| `NSTextAlignment.natural` 不完全跟随 App override | 有时只认系统语言，可能需要手动设 `textAlignment` |
| 图片镜像 | Asset Catalog 设 Direction 为 Both，配合 layout direction 自动镜像 |
| 导航返回手势方向 | 整体 RTL 后系统导航栏一般自动适配；自定义手势需读 layout direction |
| 性能 | `userInterfaceLayoutDirection` 调用频繁，内部不要 heavy 操作 |

---

## 快速对照：该改哪一步？

| 需求 | 改哪里 |
|------|--------|
| 整个 App 随 App 内语言变 RTL/LTR | **环境层** · `UIApplication.userInterfaceLayoutDirection` |
| 未设置时跟随系统 | 语言为 `nil` 时用 `Locale.preferredLanguages` |
| 文案随 App 内语言变 | **环境层** · `AppleLanguages` + Bundle |
| 某个播放按钮不翻转 | **语义层** · `.playback` |
| 某块 UI 永远 LTR / RTL | **语义层** · `.forceLeftToRight` / `.forceRightToLeft` |
| 整页固定方向 | **环境层** · `setOverrideTraitCollection`（子树级） |
| 自定义布局时判断方向 | **结果层** · 读 `effectiveUserInterfaceLayoutDirection`（只读） |

---

## 记忆口诀

> **App 内切语言改环境层：Application 管方向，AppleLanguages 管文案。**  
> **未设置跟系统，设了听 App；局部固定走语义层，整页例外 override trait。**

---

## 延伸阅读

- [概念总览](./概念总览.md)
- [基准布局方向（环境层）](./基准布局方向（环境层）.md)
- [UITraitCollection](./UITraitCollection.md)
- [UIApplication.userInterfaceLayoutDirection](https://developer.apple.com/documentation/uikit/uiapplication/userinterfacelayoutdirection)

---

*来源：Apple Developer Documentation · 社区实践 © 2026*
