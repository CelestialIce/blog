---
title: "macOS 下编译、运行、打包与精简 iText RUPS 实战记录"
date: 2026-03-23 14:42:59
permalink: java/itext-rups-on-macos/
tags:
  - Java
  - macOS
  - Maven
  - iText
  - RUPS
  - jpackage
  - jlink
categories:
  - Java
  - 实战
description: "从编译到运行，再到 jpackage 和 jlink 精简，完整记录 iText RUPS 在 macOS 上的构建与应用化流程。"
---

> 系列顺序
>
> 1. 前置环境：{% post_link macos-java-homebrew-sdkman-guide "现代 macOS Java 环境：Homebrew 加速 + SDKMAN 多版本管理" %}
> 2. 当前篇：iText RUPS 构建与打包
> 3. 下一篇：{% post_link macos-build-ghidra-package-app "Ghidra 在 macOS 上的下载、配置、编译与打包为 .app 全流程" %}

这篇笔记基于我在 macOS 上实际完成 `itext/rups` 编译、运行、打包和瘦身的全过程整理而成，适合已经按前一篇 {% post_link macos-java-homebrew-sdkman-guide "Homebrew + SDKMAN 环境文档" %} 配好 Java 环境的场景。

目标不是“把 JAR 跑起来”就结束，而是把它做成一个能放进 `/Applications`、可双击启动、体积也相对合理的 macOS 应用。

---

## 1. 项目与环境前提

项目仓库：

```bash
git clone https://github.com/itext/rups.git
cd rups
```

切到你通过 SDKMAN 注册的 Brew JDK：

```bash
sdk use java latest-brew
java -version
```

如果系统里还没有 Maven：

```bash
brew install maven
mvn -version
```

---

## 2. 先说结论：这个项目的产物是什么

`itext/rups` 是 Maven 项目，但它的 `pom.xml` 不是最普通的那种薄 JAR 配置。

这个工程里：

- `maven-jar-plugin` 的主类是 `com.itextpdf.rups.RupsLauncher`
- `maven-assembly-plugin` 会在 `package` 阶段执行
- `appendAssemblyId` 被设成了 `false`

这意味着最终主产物不是 `-with-dependencies.jar`，而是直接覆盖成：

```bash
target/itext-rups-26.02-SNAPSHOT.jar
```

也就是说，这个 `jar` 本身就是最终可运行的主产物，不要再按很多通用教程去找 `*-with-dependencies.jar`。

---

## 3. 编译项目

最直接的做法：

```bash
mvn package
```

如果你只是想快速出产物，不想跑测试，可以尝试：

```bash
mvn -Dmaven.test.skip=true package
```

我这次实际跑的是完整 `package`，结果如下：

- `BUILD SUCCESS`
- 总测试数：`965`
- 最终产物：`target/itext-rups-26.02-SNAPSHOT.jar`
- JAR 体积：约 `30M`

检查产物：

```bash
ls -lh target/itext-rups-26.02-SNAPSHOT.jar
```

---

## 4. 直接运行 JAR

最稳妥的启动方式：

```bash
java \
  -Dsun.java2d.uiScale=2 \
  --add-opens java.base/java.lang=ALL-UNNAMED \
  -jar target/itext-rups-26.02-SNAPSHOT.jar
```

说明：

- `-Dsun.java2d.uiScale=2`：改善 Retina 屏的 UI 缩放表现
- `--add-opens java.base/java.lang=ALL-UNNAMED`：规避较新 JDK 下的反射封装问题

这个项目实际入口类虽然是 `com.itextpdf.rups.RupsLauncher`，但直接 `java -jar` 就够了，因为主类已经写进 manifest。

---

## 5. 做成命令行快捷入口

如果你不想每次都 `cd` 到项目目录，可以在 `~/.zshrc` 里加一个函数：

```bash
rups() {
    sdk use java latest-brew > /dev/null

    local RUPS_JAR="$HOME/Projects/rups/target/itext-rups-26.02-SNAPSHOT.jar"

    if [[ -f "$RUPS_JAR" ]]; then
        java \
          -Dsun.java2d.uiScale=2 \
          --add-opens java.base/java.lang=ALL-UNNAMED \
          -jar "$RUPS_JAR" "$@"
    else
        echo "RUPS JAR not found: $RUPS_JAR"
        echo "Run: mvn package"
    fi
}
```

重载配置：

```bash
source ~/.zshrc
```

之后可直接：

```bash
rups
```

如果 RUPS 支持命令行传文件，也可以试：

```bash
rups ~/Downloads/example.pdf
```

---

## 6. 为什么只把 JAR 扔进 `/Applications` 不够

macOS 想要像原生应用一样：

- 双击启动
- 在 Dock 里显示为独立 App
- 有标准图标
- 能放进 `/Applications`

就不能只放一个 `.jar`。

正确做法是把它封装成一个符合 macOS 规范的 **App Bundle**，也就是 `.app` 目录结构。

这一步最适合用 JDK 自带的 `jpackage`。

---

## 7. 用 jpackage 生成 macOS App

先确认工具存在：

```bash
jpackage --version
```

这个仓库里已经自带图标文件：

```bash
config/logo.icns
```

最基础的打包命令：

```bash
mkdir -p dist

jpackage \
  --input target \
  --name "iText-RUPS" \
  --main-jar itext-rups-26.02-SNAPSHOT.jar \
  --main-class com.itextpdf.rups.RupsLauncher \
  --type app-image \
  --dest dist \
  --icon config/logo.icns \
  --app-version 26.02 \
  --java-options "--add-opens=java.base/java.lang=ALL-UNNAMED" \
  --java-options "-Dsun.java2d.uiScale=2"
```

生成结果：

```bash
dist/iText-RUPS.app
```

可直接启动：

```bash
open dist/iText-RUPS.app
```

---

## 8. 为什么第一次打出来会很大

我第一次这样打包后，`.app` 体积大约是 `166M`。后来检查发现，体积偏大主要有两个原因。

### 原因 1：把整个 `target/` 都喂给了 `jpackage`

如果 `--input target`，而 `target/` 里除了 JAR 还有这些东西：

- `classes/`
- `test-classes/`
- `surefire-reports/`
- `maven-status/`
- `jacoco-integration.exec`

那么它们也会被一并塞进 `.app` 里。

### 原因 2：默认 runtime 偏胖

第一次打包时，`jpackage` 带进去的 Java runtime 单独就有大约 `129M`。

所以第一次的体积结构大致是：

- App 总体积：`166M`
- 其中 runtime：`129M`
- 其中 app 目录：`36M`

这不是不能用，只是不够干净。

---

## 9. 正确的瘦身思路

要把体积压下来，需要做两件事：

### 9.1 只给 jpackage 一个干净输入目录

不要直接把整个 `target/` 当输入目录。

单独做一个 staging 目录，只放最终 JAR：

```bash
mkdir -p build/jpackage-input
cp target/itext-rups-26.02-SNAPSHOT.jar build/jpackage-input/
```

### 9.2 用 jlink 生成裁剪后的自定义 runtime

先用 `jdeps` 分析依赖模块：

```bash
mvn -q dependency:build-classpath -Dmdep.outputFile=/tmp/rups.classpath

jdeps \
  --multi-release 11 \
  --ignore-missing-deps \
  --print-module-deps \
  --class-path "$(cat /tmp/rups.classpath)" \
  target/classes
```

我这次实际整理出来可用的一组模块如下：

```text
java.base
java.desktop
java.logging
java.naming
java.prefs
java.sql
java.xml
jdk.javadoc
jdk.unsupported
```

然后用 `jlink` 生成瘦身 runtime：

```bash
jlink \
  --add-modules java.base,java.desktop,java.logging,java.naming,java.prefs,java.sql,java.xml,jdk.javadoc,jdk.unsupported \
  --output build/runtime-slim \
  --compress=2 \
  --strip-debug \
  --no-header-files \
  --no-man-pages
```

我这次生成出来的 `build/runtime-slim` 体积约 `65M`。

---

## 10. 生成 95M 的精简版 `.app`

有了干净输入目录和裁剪 runtime 之后，再打一次：

```bash
mkdir -p dist-final

jpackage \
  --input build/jpackage-input \
  --name "iText-RUPS" \
  --main-jar itext-rups-26.02-SNAPSHOT.jar \
  --main-class com.itextpdf.rups.RupsLauncher \
  --type app-image \
  --dest dist-final \
  --icon config/logo.icns \
  --app-version 26.02 \
  --runtime-image build/runtime-slim \
  --java-options "--add-opens=java.base/java.lang=ALL-UNNAMED" \
  --java-options "-Dsun.java2d.uiScale=2"
```

最终结果：

```bash
dist-final/iText-RUPS.app
```

这次的实际体积：

- 总体积：`95M`
- JAR：`30M`
- runtime：`64M`

这就是一个更适合长期保留的版本。

---

## 11. 什么是“自定义 runtime”，要不要它

这里的“自定义 runtime”不是完整 JDK，而是一份跟应用一起分发的、按需裁剪后的 Java 运行环境。

保留它的好处：

- 不依赖用户系统里另外装 Java
- 双击即可运行
- 放进 `/Applications` 后更像真正的 macOS 应用
- 发给别人用时更稳定

如果你只是自己本机临时使用，而且确定机器上永远有合适版本的 Java，那么可以不要 runtime，只保留 JAR。

但如果目标是“无感应用化”，那就应该保留 runtime。

---

## 12. 安装到 `/Applications`

把精简版安装到系统应用目录：

```bash
ditto dist-final/iText-RUPS.app /Applications/iText-RUPS.app
open -a /Applications/iText-RUPS.app
```

我这次最终安装完成后的结果是：

```text
/Applications/iText-RUPS.app
95M
```

这样它就可以像普通 macOS App 一样从 Finder 或 Launchpad 启动。

---

## 13. 清理本地打包过程中的临时目录

如果你不想保留中间产物，可以清掉这些目录：

```bash
rm -rf dist dist-final build/jpackage-input build/runtime-slim
```

如果还保留过旧版本大包，也可以一起删掉。

---

## 14. 常见问题

### 14.1 找不到 `-with-dependencies.jar`

这个项目默认不会产出那个名字。正确产物就是：

```bash
target/itext-rups-26.02-SNAPSHOT.jar
```

### 14.2 双击 `.jar` 不能像原生应用一样显示在 Dock

这是正常现象。`.jar` 不是 macOS 原生 App Bundle，应该用 `jpackage` 封装成 `.app`。

### 14.3 App 太大

先检查两件事：

- 你是不是把整个 `target/` 原样喂给了 `jpackage`
- 你是不是让 `jpackage` 自动带了一个偏大的默认 runtime

如果是，就按本文的 `jpackage-input + jlink runtime-slim` 重打。

### 14.4 新 JDK 下反射报错

先尝试：

```bash
--add-opens java.base/java.lang=ALL-UNNAMED
```

### 14.5 Retina 屏显示不舒服

先尝试：

```bash
-Dsun.java2d.uiScale=2
```

---

## 15. 一条推荐路线

如果你只是想把事情一次做对，建议按这条顺序：

1. `sdk use java latest-brew`
2. `mvn package`
3. 先用 `java -jar` 验证功能正常
4. 用 `jpackage` 生成 `.app`
5. 再用 `jlink + jpackage` 做精简版
6. 最后把精简版复制到 `/Applications`

这样你得到的是：

- 一个能直接运行的 JAR
- 一个能双击启动的 macOS App
- 一个体积相对合理的精简版应用

这比停留在“编译成功”更完整，也更接近真正的软件交付。
