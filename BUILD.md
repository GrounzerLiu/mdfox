# MDFox Build Guide

> Custom Firefox for Android with Material Design UI

---

## Prerequisites

### Linux (Arch Linux)

```bash
sudo pacman -S --needed \
  base-devel \
  python python-pip \
  java-17-openjdk \
  rust clang llvm \
  nodejs npm \
  nasm unzip zip \
  android-studio \
  android-platform

# git-cinnabar
paru -S git-cinnabar-git
```

### Linux (Ubuntu/Debian)

```bash
sudo apt install \
  build-essential \
  python3 python3-pip \
  openjdk-17-jdk \
  rustc cargo \
  clang llvm \
  nodejs npm \
  nasm unzip zip

# Install Android Studio manually from https://developer.android.com/studio
```

### Windows

1. Install **MozillaBuild** from https://firefox-source-docs.mozilla.org/setup/windows_build.html
2. Install **Java 17** (Adoptium/Temurin): https://adoptium.net/
3. Install **Android Studio** with SDK 34 + NDK 26+
4. Install **git-cinnabar**:
   - Download from https://github.com/glandium/git-cinnabar/releases
   - Extract and add to PATH

---

## Setup

### 1. Clone mozilla-central

```bash
git clone --depth=1 hg::https://hg.mozilla.org/mozilla-central/ mdfox
cd mdfox
```

### 2. Apply MDFox custom changes

```bash
# Add the MDFox GitHub repo
git remote add github https://github.com/GrounzerLiu/mdfox.git
git fetch github md-ui-clean

# Create a branch from your current commit (so we know where we are)
git checkout -b md-ui-clean
git reset --hard github/md-ui-clean
```

Now the custom files are applied on top of the full mozilla-central codebase.

### 3. Create .mozconfig

Create `.mozconfig` in the project root:

```
ac_add_options --enable-application=mobile/android
ac_add_options --target=aarch64-linux-android
ac_add_options --enable-artifact-builds
mk_add_options MOZ_OBJDIR=./objdir-android
export JAVA_HOME=/path/to/java-17
```

- **Linux**: `export JAVA_HOME=/usr/lib/jvm/java-17-openjdk`
- **Windows**: `export JAVA_HOME=C:/Program Files/Eclipse Adoptium/jdk-17.0.x`
- **macOS**: `export JAVA_HOME=/Library/Java/JavaVirtualMachines/jdk-17.jdk/Contents/Home`

Set `ANDROID_HOME` environment variable to your Android SDK path.

### 4. Configure

```bash
./mach configure
```

Select **Android** → **Fenix** when prompted.

---

## Build

### Build GeckoView + Fenix (first time or after engine changes)

```bash
./mach build
```

### Build Fenix APK only (after UI changes)

```bash
./mach gradle :fenix:assembleDebug
```

### Output APK

```
objdir-android/gradle/build/mobile/android/fenix/app/outputs/apk/debug/
```

Architecture variants: `arm64-v8a`, `armeabi-v7a`, `x86_64`.

---

## Development workflow

Because `md-ui-clean` is an orphan branch (only contains your changes), daily development requires switching between branches:

```bash
# 1. Switch to the full codebase to make changes and build
git checkout branches/default/tip

# ... edit code, run ./mach build / ./mach gradle :fenix:assembleDebug ...

# 2. After testing, switch back to your custom branch
git checkout md-ui-clean

# 3. Copy your changed files from the full tree
git checkout branches/default/tip -- mobile/android/fenix/app/build.gradle
git checkout branches/default/tip -- mobile/android/fenix/app/src/main/res/values/static_strings.xml
# Add any other files you modified

# 4. Commit and push
git commit -m "你的修改描述"
git push github md-ui-clean --force

# 5. Switch back to branches/default/tip to continue working
git checkout branches/default/tip
```

> **Tip**: Use `git status` after switching to see what files you changed.

## Install

```bash
adb devices
adb -s <device-id> install -r objdir-android/gradle/build/mobile/android/fenix/app/outputs/apk/debug/fenix-arm64-v8a-debug.apk
```

---

## Sync with upstream

```bash
# Make sure you're on the upstream branch first
git checkout branches/default/tip
git pull origin branches/default/tip

# Switch back to your custom branch and rebase
git checkout md-ui-clean
git rebase branches/default/tip
# Resolve conflicts if any, then:
git add .
git rebase --continue
git push github md-ui-clean --force
```

---

## 指定上游版本 (Pin upstream version)

MDFox 的自定义文件（`build.gradle`、`static_strings.xml`）与 mozilla-central 版本**强绑定**。上游更新可能导致插件体系、资源结构变化，需要重新适配后才能编译。

### 当前固定版本

| 项 | 值 |
| --- | --- |
| 仓库 | `mozilla-central`（nightly） |
| Commit | `092b4be38e4fa`（2026-08-24） |
| 版本 | Firefox 156.0a1 / GeckoView 156.0 |
| 状态 | 已验证可编译（MDFox APK 构建成功） |

日常开发**不要随意 `pull` 上游**，保持固定在已验证的 commit。需要升级时按下面流程操作并重新检查适配点。

### 可选的上游版本线

| 版本线 | 仓库 URL | 特点 |
| --- | --- | --- |
| Nightly（现状） | `hg::https://hg.mozilla.org/mozilla-central/` | 每天更新，跟随最新 |
| Beta | `hg::https://hg.mozilla.org/releases/mozilla-beta/` | 版本号固定，每周更新 |
| Release | `hg::https://hg.mozilla.org/releases/mozilla-release/` | 正式版，几周更新一次 |
| ESR | `hg::https://hg.mozilla.org/releases/mozilla-esr140/` | 长期支持，一年一版，最稳定 |

### 切换版本线

```bash
# 方式一：用目标版本线重新 clone（推荐，干净）
git clone --depth=1 hg::https://hg.mozilla.org/releases/mozilla-release/ mdfox
cd mdfox
git remote add github https://github.com/GrounzerLiu/mdfox.git
git fetch github md-ui-clean
git checkout github/md-ui-clean -- .gitignore BUILD.md CONTRIBUTING.md mobile/android/fenix/app/build.gradle mobile/android/fenix/app/src/main/res/values/static_strings.xml

# 方式二：在现有仓库添加版本线 remote 并拉取
git remote add release hg::https://hg.mozilla.org/releases/mozilla-release/
git fetch --depth=1 release
```

### 升级/切换后需要检查的适配点

1. **build.gradle 插件体系**：`libs.plugins.*` 别名（如 `kotlin.android`）、AGP/Kotlin 版本是否与当前树匹配
2. **static_strings.xml 资源重复**：上游会把字符串在 `strings.xml` 与 `static_strings.xml` 之间挪动，合并时注意 Duplicate resources 错误
3. **SDK/NDK 版本**：`.mozconfig` 的 `--target` 与 SDK/NDK 要求（可在 `python/mozboot/mozboot/android.py` 查看 `NDK_VERSION`）
4. **artifact 保质期**：预编译 GeckoView 只保留几个月。固定版本太久后 `./mach build` 会因 artifact 过期失败，届时升级到新 commit 或改做完整构建（去掉 `--enable-artifact-builds`）

---

## Project structure

```
mobile/android/
├── fenix/                 # UI layer (Kotlin/XML)
│   └── app/src/main/
│       ├── kotlin/org/mozilla/fenix/   # Kotlin source
│       └── res/                        # Resources
├── geckoview/             # Browser engine (C++/Rust)
└── android-components/    # Shared components
```

---

## Notes

- Artifact Build only supports changes in Kotlin/JS/CSS/XML. Engine changes (C++/Rust) require a full build.
- Use `./mach build faster` for quick incremental rebuilds.
- The `.mozconfig` and `objdir-android/` are git-ignored.
