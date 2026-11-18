---
name: android-apk-decompile
description: 将 APK 反编译为 Java 源码并整理成标准安卓源码工程结构（jadx + apktool 全流程）。
allowed-tools:
  bash(*)
  read(*)
  write(*)
  edit(*)
  glob(*)
  grep(*)
  todoWrite
when_to_use: 仅在明确关键词出现时触发。触发词：反编译 APK、反编译这个 apk、把 dex 转回 Java、 jadx 反编译、apktool 解包、还原安卓源码工程。示例："反编译这个 apk"、"帮我 jadx 反编译"、"把 apk 还原成源码工程"。不要仅因为用户提到安卓/apk 就触发。
argument-hint: "$apk_path [$output_dir]"
arguments:
  apk_path
  output_dir
context: inline
---

# APK 反编译并还原为安卓源码工程

将任意 APK 反编译为按包名归集的 Java 源码，并组装成标准安卓工程结构（`app/src/main/` 风格）。产物定位是**阅读分析用**，不保证可编译。

## Inputs
- `$apk_path`:（必填）APK 文件的绝对路径。
- `$output_dir`:（可选）产出工程目录；缺省为 APK 同目录下的 `<apk文件名去除扩展名>-src/`。

## Goal
产出 `<output_dir>/` 标准结构工程，含全部类（同包名跨 dex 归一）、Manifest、资源、jniLibs，以及在产出目录落盘的 `decompile-report.md`。

```
<output_dir>/
├── settings.gradle / build.gradle / gradle.properties / apktool.yml
├── decompile-report.md          # 反编译报告（落盘）
└── app/
    ├── build.gradle
    └── src/main/
        ├── AndroidManifest.xml
        ├── java/                # 146k+ .java，按包名归集
        ├── res/ assets/ jniLibs/ unknown/
```

**成功标准**：java 文件数与 APK dex 规模匹配（百万级 class 用 `/usr/bin/find` 统计）；中间目录已删除；报告已落盘。

## Steps

### 1. 环境预检
确认 `jadx --version`、`apktool --version` 可用，检查内存（`sysctl -n hw.memsize`）与目标盘剩余空间（`df -h`）。经验值：370MB / 73 dex 的 APK 需 24GB 堆 + 约 5GB 磁盘峰值 + 14 线程。

**Success criteria**: 两个工具均可用；磁盘剩余 ≥ 10GB；Apk 体积与资源已估算。

### 2. APK 摸底
`unzip -l $apk_path | grep -c "classes.*\.dex"` 统计 dex 数量并查看体积分布；用 todoWrite 建立全流程任务清单。

**Success criteria**: 已知 dex 数量、总体积；todo 清单已建立。

### 3. jadx 后台反编译 Java
后台任务执行（**用 `run_in_background`，勿前台阻塞**）：

```bash
JAVA_OPTS="-Xmx24g -Xss16m" jadx -d <work>/jadx-out --no-res --threads-count <ncpu-1> $apk_path
```

**Rules**（每条都是踩过的坑）：
- 线程数只传 `--threads-count`，不可再写 `-j`（重复指定义会报 "Can only specify option -j once" 且退出码 1，不能凭退出码 0 假象误判成功）
- 用 `-e` 前先确认想输出错误摘要；`--no-res` 跳过资源（资源由 apktool 负责，避免重复劳动）
- 退出码 3 = 反编译完成但有报错（属正常损耗，通常 <1%，**不要重跑**）
- 资源类桩码暂不处理，见第 5 步

**Success criteria**: 后台任务完成；`<work>/jadx-out/sources/` 存在且 `du -sh` 与 APK 规模相符。

### 4. apktool 并行解资源（与第 3 步同时启动）
后台任务执行：

```bash
apktool d -f -s -o <work>/apktool-out $apk_path
```

**Rules**: 必须带 `-s`（no-src），跳过 smali；apktool 3.x **没有** `--only-main-classes` 选项，传错参数只会打印 usage 且产 Nothing。解完立即 `rm -f apktool-out/classes*.dex && rm -rf apktool-out/original`（dex 副本约几 GB，无需保留）。

**Success criteria**: `AndroidManifest.xml`、`res/`、`assets/`、`lib/`、`unknown/` 均存在且非空。

### 5. 清理框架桩与无用文件
在 `jadx-out/sources/` 里 `rm -rf` 以下目录（jadx 生成的框架桩，非业务代码）：`java`、`javax`、`dalvik`、`junit`。

**Rules**: `android/`、`androidx/`、`kotlin/`、`kotlinx/` 各色第三方库源码**一律保留**；`defpackage/`、根级单字母混淆包（`a/`、`c/`、`K/` 等很可能是业务码——快手业务码恰在这里）**全量保留**，仅进入报告说明。

**Success criteria**: 桩目录已删；业务码与第三方库码完整保留。

### 6. 组装工程结构（用 mv 不用 cp）
```bash
mkdir -p $output_dir/app/src/main/java
mv jadx-out/sources/* $output_dir/app/src/main/java/
mv apktool-out/res       $output_dir/app/src/main/res
mv apktool-out/assets    $output_dir/app/src/main/assets
mv apktool-out/lib       $output_dir/app/src/main/jniLibs
mv apktool-out/unknown   $output_dir/app/src/main/unknown
mv apktool-out/AndroidManifest.xml $output_dir/app/src/main/
cp apktool-out/apktool.yml $output_dir/
```
采用 `mv` 而非 `cp`，避免 GB 级产出翻倍占盘。

**Success criteria**: 上述 7 项均已就位；`<output_dir>` 体积约等于 jadx-out + apktool-out 之和。

### 7. 写 Gradle 脚手架
从 `AndroidManifest.xml` 读出 `package` 与 `compileSdkVersion`，写入：
- `<output_dir>/settings.gradle`（pluginManagement + include ':app'）
- `<output_dir>/build.gradle`（根构建脚本）
- `<output_dir>/gradle.properties`
- `<output_dir>/app/build.gradle`（namespace/applicationId/compileSdk 按清单值，注明"仅供阅读，不保证编译"）

**Success criteria**: 4 个构建文件存在，值与 Manifest 一致。

### 8a. 校验产出
`/usr/bin/find`（**勿用裸 find**，本机可能被劫持，其包装器可能因缺 libonig.5.dylib 静默输出 0）统计 `.java` 数量，与 jadx 日志最终 `progress: N of N` 中的 N 对照。

**Success criteria**: java 文件数 > 0 且数量级与类总数相符；顶层包列表与 Manifest 包名匹配。

### 8b. 生成反编译报告并落盘 → decompile-report.md
写入 `<output_dir>/decompile-report.md`，内容包括：
- 工具版本（jadx/apktool）、APK 体积、dex 数量、类总数、java 文件总数、产出体积
- 反编译失败类数量与若干示例（从 jadx stderr 中找 `ERROR - failed to decompile`）
- 混淆单字母包清单（`a/`、`b/`、`c/` 等顶层目录列表），声明"可能含业务码，未过滤"
- 已删除内容清单（框架桩、dex 副本、签名残留）
- 顶部声明：产物仅供阅读分析，不保证可编译

**Artifacts**: `decompile-report.md`（落盘，供后续追溯）。

**Success criteria**: 报告存在且各数字真实（来自命令输出，非估算）。

### 9. 清理中间目录并汇报
`rm -rf <work>/jadx-out <work>/apktool-out`（**保留原始 APK 不动**），向用户汇报最终目录树与报告结论。

**Success criteria**: 中间目录已删除；APK 原件完好；最终结构已展示给用户。

## 后续
本技能只负责产出可读源码工程。若用户接着要查某个功能的调用链、开关、接口、缓存或弹窗展示，转入同仓库的 `android-feature-investigation` 技能（`skills/android-feature-investigation/SKILL.md`），不要在本技能内继续分析业务逻辑。
