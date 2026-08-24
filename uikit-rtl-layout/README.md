# UIKit RTL 布局

本目录整理 iOS UIKit 中与 RTL（从右到左）布局相关的概念与 API。

| 文档 | 说明 |
|------|------|
| [Apple RTL 设计与开发指南](./Apple-RTL设计与开发指南.md) | Apple HIG + 国际化 Archive 文档整理（设计规范与开发实践） |
| [Unicode 双向文本方向标记](./Unicode双向文本方向标记.md) | `\u{200E}` / `\u{202A}` / `\u{2066}` 等不可见控制符的区别与用法 |
| [概念总览](./概念总览.md) | **最快入门**。所有概念的关系，一句话版 |
| [App 内语言设置控制 RTL/LTR](./App内语言设置控制RTL-LTR.md) | **实战**。App 内切语言时在哪一步控制整体 RTL/LTR |
| [语义属性与布局方向的关系](./SemanticAttribute与LayoutDirection的关系.md) | 通俗对比 UISemanticContentAttribute 与 UIUserInterfaceLayoutDirection |
| [基准布局方向（环境层）](./基准布局方向（环境层）.md) | 系统语言、App 设置、Trait Collection 如何决定基准布局方向 |
| [UITraitCollection](./UITraitCollection.md) | Trait 环境模型、`layoutDirection` 及 override 机制详解 |
| [UISemanticContentAttribute](./UISemanticContentAttribute.md) | 语义属性枚举与 `semanticContentAttribute` API 详解 |
| [UIUserInterfaceLayoutDirection](./UIUserInterfaceLayoutDirection.md) | 布局方向枚举与相关 API 详解 |
