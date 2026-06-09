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
git remote add mdfox https://github.com/GrounzerLiu/mdfox.git
git fetch mdfox md-ui-clean

# Create a branch from your current commit (so we know where we are)
git checkout -b md-ui-clean
git reset --hard mdfox/md-ui-clean
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

## Install

```bash
adb devices
adb -s <device-id> install -r path/to/fenix-arm64-v8a-debug.apk
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
git push mdfox md-ui-clean --force
```

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
