---
title: "Ghidra 在 macOS 上的下载、配置、编译与打包为 .app 全流程"
date: 2026-03-26 23:01:29
permalink: java/ghidra-build-package-on-macos/
tags:
  - Java
  - macOS
  - Ghidra
  - Reverse Engineering
  - Gradle
  - jpackage
categories:
  - Java
  - 逆向工程
description: "记录 Ghidra 在 macOS 上从源码下载、依赖初始化、构建，到封装成可双击 .app 的完整过程。"
---

> 系列顺序
>
> 1. 环境准备：{% post_link macos-java-homebrew-sdkman-guide "现代 macOS Java 环境：Homebrew 加速 + SDKMAN 多版本管理" %}
> 2. 打包练手：{% post_link macos-build-itext-rups "macOS 下编译、运行、打包与精简 iText RUPS 实战记录" %}
> 3. 当前篇：Ghidra 下载、编译与 .app 打包

这篇文章把 Ghidra 在 macOS 上的本地构建与应用化流程完整梳理一遍。重点不是“把源码编过”，而是把最终结果做成一个能双击启动的 `Ghidra.app`。

下面按一篇可直接复用的教程来整理完整的 Ghidra 本地构建与打包流程，包含：

- 如何下载源码
- 如何准备依赖环境
- 如何初始化依赖
- 如何分步编译
- 如何找到构建产物
- 为什么 `buildGhidra` 默认不会直接生成 macOS `.app`
- 如何把已经编译好的 Ghidra 封装成可双击启动的 `Ghidra.app`
- 构建和打包时的排查、操作与验证过程
- 自动打包脚本

## 1. 示例环境

- 系统：macOS
- 架构：`arm64`
- 工程目录：`/Users/xxx/Projects/ghidra-Ghidra_12.0.4_build`
- 构建版本：`Ghidra 12.0.4 DEV`
- 构建输出目录：`build/dist/ghidra_12.0.4_DEV`
- 生成的压缩包：`build/dist/ghidra_12.0.4_DEV_20260326_mac_arm_64.zip`
- 最终生成的 macOS App：`build/dist/Ghidra.app`

从 `build/dist/ghidra_12.0.4_DEV/Ghidra/application.properties` 可以看到：

```properties
application.name=Ghidra
application.version=12.0.4
application.release.name=DEV
application.build.date=2026-Mar-26 2253 CST
application.java.min=21
application.python.supported=3.14, 3.13, 3.12, 3.11, 3.10, 3.9
```

## 2. 下载源码

有两种常见方式。

### 2.1 使用 Git 克隆

```bash
git clone https://github.com/NationalSecurityAgency/ghidra.git
cd ghidra
```

### 2.2 下载 GitHub 源码压缩包（推荐）

```bash
unzip ghidra-master.zip
cd ghidra-master
```

如果下载的是 GitHub 源码压缩包而不是 `git clone`，那么目录里可能没有 `.git` 信息。这不影响构建，但像 `git status` 这样的命令会报不是 Git 仓库，这是正常现象。

## 3. 前置依赖与配置

根据 Ghidra 仓库 README，构建开发版需要这些依赖：

- JDK 21 64-bit
- Gradle 8.5+，或者直接使用仓库自带的 `gradlew`（推荐）
- Python 3，建议 3.9 到 3.13，要求带 pip
- macOS 下还需要 C/C++ 编译工具链，例如 Clang 和 `make`

推荐先确认这些工具在 PATH 中可用：

```bash
java -version
python3 --version
clang --version
make --version
./gradlew --version
```

如果 `gradlew` 没有执行权限，先处理权限。

## 4. 具体步骤

可以按下面的顺序执行，这也是比较稳妥的一套流程。

### 4.1 赋予执行权限

```bash
chmod +x gradlew
```

### 4.2 初始化并拉取构建依赖

Ghidra 依赖一些不在 Maven Central 的二进制组件，所以要先跑初始化脚本：

```bash
./gradlew -I gradle/support/fetchDependencies.gradle init
```

### 4.3 分步构建

先编 Native 组件，再编完整分发包：

```bash
./gradlew buildNatives
./gradlew buildGhidra
```

这个流程的好处是：

- `buildNatives` 先把 Decompiler 等本地 C/C++ 组件编出来
- `buildGhidra` 再把 Java 部分和整套分发目录组装完成

## 5. 编译完成后的产物在哪里

构建完成后，产物通常位于 `build/dist/` 下：

```text
build/dist/
build/dist/ghidra_12.0.4_DEV
build/dist/ghidra_12.0.4_DEV_20260326_mac_arm_64.zip
```

其中：

- `build/dist/ghidra_12.0.4_DEV` 是已经解开的完整发行目录
- `build/dist/ghidra_12.0.4_DEV_20260326_mac_arm_64.zip` 是对应压缩包

Ghidra 官方默认的运行方式不是 `.app`，而是直接运行分发目录里的启动脚本：

```bash
./build/dist/ghidra_12.0.4_DEV/ghidraRun
```

辅助入口还包括：

- `build/dist/ghidra_12.0.4_DEV/support/analyzeHeadless`
- `build/dist/ghidra_12.0.4_DEV/support/pyghidraRun`

## 6. 为什么 `buildGhidra` 不会直接生成 `.app`

这是整个流程里最容易误解的一点。结合仓库 README、Gradle 分发逻辑和实际产物，可以得到下面的结论：

- `buildGhidra` 负责生成 Ghidra 的 distribution 目录和 zip 包
- 默认产物是可运行目录加压缩包
- 它不会自动生成 macOS 原生 `.app`
- 当前产物里也没有现成的 `*.app` 或 `*.dmg`

换句话说，Ghidra 官方构建更接近“跨平台分发目录”，而不是“平台原生安装包”。

## 7. 实际排查过程

为了确认默认构建产物的形式，可以按下面的思路检查。

### 7.1 查看仓库说明和构建说明

仓库根目录 `README.md` 中给出的官方构建说明是：

```bash
gradle buildGhidra
```

并且 README 明确说明：

- 输出压缩包位于 `build/dist/`
- 官方运行方式是 `./ghidraRun`

### 7.2 查看实际构建产物

检查 `build/dist` 后可以看到：

```text
build/dist/ghidra_12.0.4_DEV
build/dist/ghidra_12.0.4_DEV_20260326_mac_arm_64.zip
```

同时分发目录中包含：

- `ghidraRun`
- `support/launch.sh`
- `Ghidra/`
- `Extensions/`
- `docs/`
- `server/`

### 7.3 确认没有现成的 `.app`

可以直接搜索：

```bash
find build/dist/ghidra_12.0.4_DEV -name '*.app' -o -name '*.dmg'
```

结果为空，说明默认构建产物里没有 `.app`。

### 7.4 检查启动入口

可以重点查看：

- `build/dist/ghidra_12.0.4_DEV/ghidraRun`
- `build/dist/ghidra_12.0.4_DEV/support/launch.sh`

确认启动模型是：

- `ghidraRun` 只是一个入口脚本
- 真正会调用 `support/launch.sh`
- `launch.sh` 再去找 JDK、拼接 VM 参数、启动 `ghidra.GhidraRun`

### 7.5 检查可用图标与 macOS 打包工具

还可以继续检查：

- 图标源文件，例如 `Ghidra/Features/Base/src/main/resources/images/GHIDRA_1.png`
- 系统工具是否存在：`iconutil`、`sips`、`osascript`

本机具备这些工具，因此可以直接在 macOS 本地拼一个标准 `.app` bundle。

## 8. `.app` 打包方案

由于 Ghidra 默认不会生成 `.app`，因此需要补一个额外的打包脚本：

```text
tools/macos/make-ghidra-app.sh
```

脚本的思路如下：

1. 读取已经编好的 `build/dist/ghidra_12.0.4_DEV`
2. 在 `build/dist/` 下生成一个标准 `Ghidra.app`
3. 把整套 Ghidra distribution 嵌入到：

```text
Ghidra.app/Contents/Resources/ghidra
```

4. 自动生成：

- `Contents/Info.plist`
- `Contents/MacOS/ghidra-wrapper`
- `Contents/Resources/Ghidra.icns`

5. 最终让 Finder 可以把它识别为一个可双击启动的 macOS App

## 9. 打包脚本的使用方法

### 9.1 默认用法

```bash
tools/macos/make-ghidra-app.sh
```

默认行为：

- 输入目录：`build/dist/ghidra_12.0.4_DEV`
- 输出 App：`build/dist/Ghidra.app`

### 9.2 指定输入和输出路径

```bash
tools/macos/make-ghidra-app.sh build/dist/ghidra_12.0.4_DEV /Applications/Ghidra.app
```

### 9.3 打包完成后如何启动

可以直接双击：

```text
build/dist/Ghidra.app
```

也可以命令行启动：

```bash
open build/dist/Ghidra.app
```

更推荐的安装方式是把它移到 `/Applications`：

```bash
mv build/dist/Ghidra.app /Applications/
open /Applications/Ghidra.app
```

这样后续就可以直接从启动台或“应用程序”目录启动。

## 10. 实际执行的打包与验证命令

### 10.1 给脚本执行权限并打包

```bash
chmod 755 tools/macos/make-ghidra-app.sh
tools/macos/make-ghidra-app.sh
```

预期输出类似于：

```text
created: /Users/xxx/Projects/ghidra-Ghidra_12.0.4_build/build/dist/Ghidra.app
embedded dist: /Users/xxx/Projects/ghidra-Ghidra_12.0.4_build/build/dist/ghidra_12.0.4_DEV
```

### 10.2 检查生成的 App 结构

核心结构如下：

```text
build/dist/Ghidra.app
build/dist/Ghidra.app/Contents
build/dist/Ghidra.app/Contents/MacOS/ghidra-wrapper
build/dist/Ghidra.app/Contents/Resources/Ghidra.icns
build/dist/Ghidra.app/Contents/Resources/ghidra
build/dist/Ghidra.app/Contents/Info.plist
```

### 10.3 校验 `Info.plist`

生成后的 `Info.plist` 核心字段包括：

- `CFBundleDisplayName = Ghidra`
- `CFBundleExecutable = ghidra-wrapper`
- `CFBundleIconFile = Ghidra.icns`
- `CFBundleShortVersionString = 12.0.4-DEV`

### 10.4 启动验证

可以执行：

```bash
open build/dist/Ghidra.app
```

如果随后能观察到 Java 进程已经从这个 `.app` 启动，说明这个 bundle 可以正常使用。

### 10.5 移动到 `/Applications`

为了后续直接使用，可以把 `Ghidra.app` 移动到 `/Applications`：

```bash
mv /Users/xxx/Projects/ghidra-Ghidra_12.0.4_build/build/dist/Ghidra.app /Applications/Ghidra.app
```

移动后再次执行：

```bash
open /Applications/Ghidra.app
```

如果此时 Java 进程已经从 `/Applications/Ghidra.app` 启动，就说明安装位置切换没有问题。

## 11. 首次启动时的注意事项

Ghidra 首次启动时经常会涉及 JDK 路径确认。`Ghidra.app` 内部最终还是调用 Ghidra 自己的 `ghidraRun` 与 `support/launch.sh`。

这里有一个关键坑点需要注意：

- 在终端里运行时，通常可以继承你 shell 里的 `JAVA_HOME`、SDKMAN 环境和 PATH
- 但从 Finder 或 `/Applications` 双击启动时，环境变量会干净很多
- 如果 Java 只存在于终端环境里，例如 `~/.sdkman/candidates/java/current/bin/java`，那么 `.app` 双击时就可能表现为“点击没反应”

这一类问题的典型表现是：

- 在原工程目录里运行 `build/dist/ghidra_12.0.4_DEV/ghidraRun` 正常
- 但双击 `/Applications/Ghidra.app` 无效

根因在于 Finder 环境找不到 Java，导致 Ghidra 的启动脚本在最前面就失败了。

### 11.1 临时排查办法

如果遇到双击 `.app` 没反应，可以先在终端里直接运行原始启动脚本：

```bash
build/dist/ghidra_12.0.4_DEV/ghidraRun
```

如果这样能正常启动，而 Finder 双击不行，那么基本可以确认是 `.app` 启动环境找不到 Java。

### 11.2 推荐修复办法

更稳妥的方式不是单纯依赖“先在终端启动一次”，而是直接修改 `ghidra-wrapper`。

修复思路是：

- 如果 Finder 环境没有 `JAVA_HOME`
- 就让 wrapper 主动去几个常见位置找 Java
- 找到后先 `export JAVA_HOME`
- 再调用 Ghidra 自己的 `ghidraRun`

优先查找的位置包括：

- `${HOME}/.sdkman/candidates/java/current`
- `/Library/Java/JavaVirtualMachines/*/Contents/Home`
- `/opt/homebrew/opt/openjdk/libexec/openjdk.jdk/Contents/Home`
- `/usr/local/opt/openjdk/libexec/openjdk.jdk/Contents/Home`

这样一来，Finder 双击 `/Applications/Ghidra.app` 时就不再依赖终端环境。

### 11.3 修复后的验证结果

修复后可以从两个角度验证：

- 用接近 Finder 的极简环境执行 `ghidra-wrapper`
- 再次执行 `open /Applications/Ghidra.app`

如果两项都通过，那么 `/Applications/Ghidra.app` 就可以直接双击启动。

## 12. 这个 `.app` 的特点

这个方案的 `.app` 有几个特点：

- 它是一个真正的 macOS `.app` bundle
- 它不是“轻壳”，而是把完整 Ghidra distribution 嵌进去
- 因此体积接近原始 `build/dist/ghidra_12.0.4_DEV`
- 当前实测 `Ghidra.app` 和 `ghidra_12.0.4_DEV` 都在约 `832M`

优点：

- 可直接双击
- 适合自己本机用
- 不依赖额外安装流程

缺点：

- 体积大
- 每次重新编译 Ghidra 后，最好重新执行一次打包脚本

## 13. 重新编译后的更新流程

如果后续重新更新源码并重新编译，建议流程如下：

```bash
./gradlew buildNatives
./gradlew buildGhidra
tools/macos/make-ghidra-app.sh
```

如果想直接覆盖到 `/Applications`：

```bash
tools/macos/make-ghidra-app.sh build/dist/ghidra_12.0.4_DEV /Applications/Ghidra.app
```

## 14. 打包脚本源码

脚本路径：

```text
tools/macos/make-ghidra-app.sh
```

完整内容如下：

```bash
#!/usr/bin/env bash

set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
REPO_ROOT="$(cd "${SCRIPT_DIR}/../.." && pwd)"

DEFAULT_DIST_DIR="${REPO_ROOT}/build/dist/ghidra_12.0.4_DEV"
DEFAULT_APP_PATH="${REPO_ROOT}/build/dist/Ghidra.app"
DEFAULT_ICON_SOURCE="${REPO_ROOT}/Ghidra/Features/Base/src/main/resources/images/GHIDRA_1.png"

usage() {
	cat <<'EOF'
Usage:
  make-ghidra-app.sh [dist_dir] [app_path]

Examples:
  tools/macos/make-ghidra-app.sh
  tools/macos/make-ghidra-app.sh build/dist/ghidra_12.0.4_DEV /Applications/Ghidra.app

Notes:
  - The app bundle embeds the selected Ghidra distribution under
    Ghidra.app/Contents/Resources/ghidra
  - First launch is safest from Terminal once, so LaunchSupport can persist the JDK path
EOF
}

if [[ "${1:-}" == "-h" || "${1:-}" == "--help" ]]; then
	usage
	exit 0
fi

DIST_DIR="${1:-${DEFAULT_DIST_DIR}}"
APP_PATH="${2:-${DEFAULT_APP_PATH}}"

if [[ ! -d "${DIST_DIR}" ]]; then
	echo "error: dist directory does not exist: ${DIST_DIR}" >&2
	exit 1
fi

if [[ ! -x "${DIST_DIR}/ghidraRun" || ! -f "${DIST_DIR}/support/launch.sh" ]]; then
	echo "error: dist directory is not a Ghidra installation: ${DIST_DIR}" >&2
	exit 1
fi

if [[ ! -f "${DEFAULT_ICON_SOURCE}" ]]; then
	echo "error: icon source is missing: ${DEFAULT_ICON_SOURCE}" >&2
	exit 1
fi

for tool in ditto iconutil sips plutil; do
	if ! command -v "${tool}" >/dev/null 2>&1; then
		echo "error: required tool not found: ${tool}" >&2
		exit 1
	fi
done

APP_CONTENTS_DIR="${APP_PATH}/Contents"
APP_MACOS_DIR="${APP_CONTENTS_DIR}/MacOS"
APP_RESOURCES_DIR="${APP_CONTENTS_DIR}/Resources"
EMBEDDED_GHIDRA_DIR="${APP_RESOURCES_DIR}/ghidra"
PLIST_PATH="${APP_CONTENTS_DIR}/Info.plist"
ICON_PATH="${APP_RESOURCES_DIR}/Ghidra.icns"
LAUNCHER_PATH="${APP_MACOS_DIR}/ghidra-wrapper"

TMP_DIR="$(mktemp -d)"
ICONSET_DIR="${TMP_DIR}/Ghidra.iconset"

cleanup() {
	rm -rf "${TMP_DIR}"
}
trap cleanup EXIT

mkdir -p "${ICONSET_DIR}"

make_square_png() {
	local source_png="$1"
	local target_png="$2"
	local size="$3"
	sips \
		--resampleWidth "${size}" \
		--resampleHeight "${size}" \
		--padToHeightWidth "${size}" "${size}" \
		"${source_png}" \
		--out "${target_png}" >/dev/null
}

make_square_png "${DEFAULT_ICON_SOURCE}" "${ICONSET_DIR}/icon_16x16.png" 16
make_square_png "${DEFAULT_ICON_SOURCE}" "${ICONSET_DIR}/icon_16x16@2x.png" 32
make_square_png "${DEFAULT_ICON_SOURCE}" "${ICONSET_DIR}/icon_32x32.png" 32
make_square_png "${DEFAULT_ICON_SOURCE}" "${ICONSET_DIR}/icon_32x32@2x.png" 64
make_square_png "${DEFAULT_ICON_SOURCE}" "${ICONSET_DIR}/icon_128x128.png" 128
make_square_png "${DEFAULT_ICON_SOURCE}" "${ICONSET_DIR}/icon_128x128@2x.png" 256
make_square_png "${DEFAULT_ICON_SOURCE}" "${ICONSET_DIR}/icon_256x256.png" 256
make_square_png "${DEFAULT_ICON_SOURCE}" "${ICONSET_DIR}/icon_256x256@2x.png" 512
make_square_png "${DEFAULT_ICON_SOURCE}" "${ICONSET_DIR}/icon_512x512.png" 512
make_square_png "${DEFAULT_ICON_SOURCE}" "${ICONSET_DIR}/icon_512x512@2x.png" 1024

rm -rf "${APP_PATH}"
mkdir -p "${APP_MACOS_DIR}" "${APP_RESOURCES_DIR}"

iconutil -c icns "${ICONSET_DIR}" -o "${ICON_PATH}"
ditto "${DIST_DIR}" "${EMBEDDED_GHIDRA_DIR}"

APP_NAME="$(basename "${APP_PATH}" .app)"
APP_VERSION="$(
	awk -F= '
		$1 == "application.version" { version = $2 }
		$1 == "application.release.name" { release = $2 }
		END {
			if (version == "") {
				version = "unknown"
			}
			if (release != "") {
				printf "%s-%s", version, release
			}
			else {
				printf "%s", version
			}
		}
	' "${DIST_DIR}/Ghidra/application.properties"
)"

cat > "${LAUNCHER_PATH}" <<'EOF'
#!/usr/bin/env bash

set -euo pipefail

APP_CONTENTS_DIR="$(cd "$(dirname "$0")/.." && pwd)"
GHIDRA_DIR="${APP_CONTENTS_DIR}/Resources/ghidra"

if [[ -z "${JAVA_HOME:-}" ]]; then
	for candidate in \
		"${HOME}/.sdkman/candidates/java/current" \
		/Library/Java/JavaVirtualMachines/*/Contents/Home \
		/opt/homebrew/opt/openjdk/libexec/openjdk.jdk/Contents/Home \
		/usr/local/opt/openjdk/libexec/openjdk.jdk/Contents/Home
	do
		if [[ -x "${candidate}/bin/java" ]]; then
			export JAVA_HOME="${candidate}"
			break
		fi
	done
fi

exec "${GHIDRA_DIR}/ghidraRun" "$@"
EOF
chmod 755 "${LAUNCHER_PATH}"

cat > "${PLIST_PATH}" <<EOF
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "https://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>CFBundleDevelopmentRegion</key>
	<string>en</string>
	<key>CFBundleDisplayName</key>
	<string>${APP_NAME}</string>
	<key>CFBundleExecutable</key>
	<string>ghidra-wrapper</string>
	<key>CFBundleIconFile</key>
	<string>Ghidra.icns</string>
	<key>CFBundleIdentifier</key>
	<string>local.ghidra.bundle</string>
	<key>CFBundleInfoDictionaryVersion</key>
	<string>6.0</string>
	<key>CFBundleName</key>
	<string>${APP_NAME}</string>
	<key>CFBundlePackageType</key>
	<string>APPL</string>
	<key>CFBundleShortVersionString</key>
	<string>${APP_VERSION}</string>
	<key>CFBundleVersion</key>
	<string>${APP_VERSION}</string>
	<key>NSHighResolutionCapable</key>
	<true/>
</dict>
</plist>
EOF

plutil -lint "${PLIST_PATH}" >/dev/null

echo "created: ${APP_PATH}"
echo "embedded dist: ${DIST_DIR}"
```

## 15. 一套可直接复用的完整命令清单

从零到可双击 `.app`，可以直接按下面这套执行：

```bash
git clone https://github.com/NationalSecurityAgency/ghidra.git
cd ghidra

chmod +x gradlew
./gradlew -I gradle/support/fetchDependencies.gradle init
./gradlew buildNatives
./gradlew buildGhidra

tools/macos/make-ghidra-app.sh
mv build/dist/Ghidra.app /Applications/
open /Applications/Ghidra.app
```

如果遇到 Finder 双击无反应，优先检查 Java 是否只存在于终端环境；修复后的 `make-ghidra-app.sh` 已经会把 Java 自动探测逻辑写进 `ghidra-wrapper`。

如果仍然要做最基础的排查，推荐先在终端运行一次：

```bash
build/dist/ghidra_12.0.4_DEV/ghidraRun
```

## 16. 结论

结论非常明确：

- Ghidra 官方构建默认只产出 distribution 目录和 zip，不直接产出 macOS `.app`
- `fetchDependencies -> buildNatives -> buildGhidra` 这一套流程是正确的
- 想要 `.app`，需要额外做一层 macOS bundle 封装
- `tools/macos/make-ghidra-app.sh` 已经可以直接把现有构建结果打成可双击启动的 `Ghidra.app`
- `/Applications/Ghidra.app` 已经完成验证，可以直接双击启动
- Finder 双击无效的核心原因通常不是 Ghidra 本体，而是 `.app` 启动时缺少终端里的 Java 环境
