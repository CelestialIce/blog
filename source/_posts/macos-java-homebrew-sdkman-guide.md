---
title: "现代 macOS Java 环境：Homebrew 加速 + SDKMAN 多版本管理"
date: 2026-03-23 14:43:46
permalink: java/macos-homebrew-sdkman/
tags:
  - Java
  - macOS
  - Homebrew
  - SDKMAN
  - Maven
categories:
  - Java
  - 开发环境
description: "在 macOS 上用 Homebrew 和 SDKMAN 搭建稳定、可切换、适合国内网络环境的 Java 开发环境。"
---

> 系列顺序
>
> 1. 当前篇：Homebrew + SDKMAN 环境准备
> 2. 下一篇：{% post_link macos-build-itext-rups "macOS 下编译、运行、打包与精简 iText RUPS 实战记录" %}
> 3. 然后看：{% post_link macos-build-ghidra-package-app "Ghidra 在 macOS 上的下载、配置、编译与打包为 .app 全流程" %}

这篇笔记解决的是两个常见问题：

- 在大陆网络环境下，如何更稳定地装 Java
- 如何避免手工改 `JAVA_HOME`、`PATH` 带来的环境混乱

我的推荐组合是：

- **Homebrew** 负责下载和更新 JDK
- **SDKMAN** 负责切换和管理当前使用的 Java 版本

这样做的优点是：

- 下载快
- 切版本方便
- 不需要手动往系统目录里乱塞链接
- 对多项目、多版本 JDK 更友好

---

## 1. Homebrew 4.x 的镜像配置思路

Homebrew 4.x 默认是 **API 驱动模式**，很多情况下不再需要完整克隆 `homebrew-core`。

在 `~/.zshrc` 里加入：

```bash
export HOMEBREW_API_DOMAIN="https://mirrors.ustc.edu.cn/homebrew-bottles/api"
export HOMEBREW_BOTTLE_DOMAIN="https://mirrors.ustc.edu.cn/homebrew-bottles"
export HOMEBREW_BREW_GIT_REMOTE="https://mirrors.ustc.edu.cn/brew.git"
```

生效：

```bash
source ~/.zshrc
```

把 brew 程序本体仓库也切到镜像：

```bash
git -C "$(brew --repo)" remote set-url origin https://mirrors.ustc.edu.cn/brew.git
brew update
```

如果你尝试去改 `homebrew-core` 时提示目录不存在，通常说明你当前就在 API 模式下，这不是故障。

---

## 2. 镜像站怎么选

在 Homebrew 的使用场景里，**延迟和稳定性** 往往比峰值下载速度更重要。

可以用下面的方式简单测一下：

```bash
curl -o /dev/null -s -w "USTC: %{speed_download} B/s, Time: %{time_total}s\n" \
  https://mirrors.ustc.edu.cn/homebrew-bottles/api/formula/openjdk.json

curl -o /dev/null -s -w "Aliyun: %{speed_download} B/s, Time: %{time_total}s\n" \
  https://mirrors.aliyun.com/homebrew/homebrew-bottles/api/formula/openjdk.json

curl -o /dev/null -s -w "TUNA: %{speed_download} B/s, Time: %{time_total}s\n" \
  https://mirrors.tuna.tsinghua.edu.cn/homebrew-bottles/api/formula/openjdk.json
```

一般经验：

- `USTC`：首选，延迟低，整体稳定
- `Aliyun`：可作为备选
- `TUNA`：可兜底，但未必总是最快

---

## 3. 用 Homebrew 安装 Java 和 Maven

安装最新 OpenJDK：

```bash
brew install openjdk
```

如需 Maven：

```bash
brew install maven
```

验证：

```bash
brew info openjdk
java -version
mvn -version
```

需要注意的是，Homebrew 的 `openjdk` 通常是 **keg-only**，不会自动替换系统默认 `/usr/bin/java`。这正是 SDKMAN 发挥作用的地方。

---

## 4. 用 SDKMAN 接管 Brew 安装的 JDK

核心思路不是让 SDKMAN 去国外源下载 JDK，而是把 Brew 已经装好的 JDK 注册为一个本地版本。

在 Apple Silicon Mac 上，Homebrew `openjdk` 常见路径是：

```bash
/opt/homebrew/opt/openjdk/libexec/openjdk.jdk/Contents/Home
```

把它注册进 SDKMAN：

```bash
sdk install java latest-brew /opt/homebrew/opt/openjdk/libexec/openjdk.jdk/Contents/Home
```

使用它：

```bash
sdk use java latest-brew
```

设为默认：

```bash
sdk default java latest-brew
```

查看当前实际生效版本：

```bash
sdk current java
java -version
echo $JAVA_HOME
```

---

## 5. 为什么这样比手工改 PATH 更稳

如果你手工在 `~/.zshrc` 里硬写很多 `JAVA_HOME=/some/path`，时间一长会出现这些问题：

- 不同项目需要不同 JDK 时切换痛苦
- `brew upgrade` 后你忘了改路径
- IDE、终端、脚本三处环境不一致

而 `Homebrew + SDKMAN` 分工清晰：

- Brew 管安装和更新
- SDKMAN 管当前会话和默认版本

这比只用其中一个工具更顺手。

---

## 6. 一个实用同步函数

如果未来 Brew 更新了 JDK，而你又想重新确认 SDKMAN 的映射，可以在 `~/.zshrc` 里放一个函数：

```bash
sdk-brew() {
    local brew_path="/opt/homebrew/opt/openjdk/libexec/openjdk.jdk/Contents/Home"

    if [[ -d "$brew_path" ]]; then
        sdk uninstall java latest-brew >/dev/null 2>&1
        sdk install java latest-brew "$brew_path"
        echo "[OK] latest-brew has been refreshed."
    else
        echo "[Error] Brew OpenJDK path not found: $brew_path"
    fi
}
```

执行：

```bash
source ~/.zshrc
sdk-brew
```

---

## 7. 推荐的日常工作流

平时进入 Java 项目目录后，我更建议这样做：

```bash
sdk use java latest-brew
java -version
mvn -version
```

这三步能快速确认：

- 当前 shell 用的是不是你预期的 JDK
- Maven 会不会拿错 Java
- 后面构建失败是不是环境问题

---

## 8. 与 RUPS 和 Ghidra 实战如何衔接

这篇文档只负责把 Java 环境铺平。

在此基础上，你就可以继续做：

- 克隆 `itext/rups`
- 执行 `mvn package`
- 运行 `target/itext-rups-26.02-SNAPSHOT.jar`
- 用 `jpackage` 封成 `.app`
- 用 `jlink` 把体积从 `166M` 压到约 `95M`

这些实战步骤已经整理在下一篇：

{% post_link macos-build-itext-rups "macOS 下编译、运行、打包与精简 iText RUPS 实战记录" %}

建议阅读顺序：

1. 先看这篇，把 Java 环境配好
2. 再看 {% post_link macos-build-itext-rups "RUPS 实战篇" %}，按实战流程完成构建、打包和安装
3. 如果你还要处理更大的 Java 桌面工具，再继续看 {% post_link macos-build-ghidra-package-app "Ghidra 打包篇" %}

---

## 9. 清理建议

常用清理命令：

```bash
brew cleanup
sdk flush archives
sdk flush temp
```

如果你只是想清掉当前 Maven 项目的构建产物：

```bash
mvn clean
```

如果不是明确知道自己在做什么，不建议动不动就删整个 `~/.m2/repository`。

---

## 10. 一句话总结

在 macOS 上做 Java 开发，比较稳的一套组合是：

- 用 Homebrew 解决下载与更新
- 用 SDKMAN 解决版本切换与 `JAVA_HOME`
- 把具体项目的构建、运行、打包逻辑写进项目笔记，而不是把环境篇和项目篇混在一起

这样后面不管是编译 RUPS，还是换别的 Java 工程，路径都会清楚很多。
