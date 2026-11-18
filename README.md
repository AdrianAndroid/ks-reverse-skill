# ks-reverse-skill

快手 APK 逆向与功能分析技能集，按「先出源码工程、再查功能链路」两阶段组织。

## 技能

| 技能 | 路径 | 触发 |
|---|---|---|
| android-apk-decompile | `skills/android-apk-decompile/SKILL.md` | 「反编译这个 apk」「jadx 反编译」「apktool 解包」 |
| android-feature-investigation | `skills/android-feature-investigation/SKILL.md` | 「这个功能为什么没弹」「定位开关/实验/缓存/弹窗问题」 |

## 流程

1. **阶段一 反编译**：jadx + apktool 产出可读的 `app/src/main` 源码工程（仅供阅读，不保证可编译）。
2. **阶段二 功能调查**：在源码工程中沿调用链追触发条件、接口、缓存、配置与展示，产出可核查的代码位置与证据分级。

阶段一产出即阶段二的输入；两技能职责独立，互不替代。
