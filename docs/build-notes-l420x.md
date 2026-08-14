# Kelivo UOS ARM64 (l420x) 构建笔记

本文记录 Kelivo 在 UOS ARM64（HUAWEI Kirin 9000C / Maleoon 910 GPU）上从上游代码构建桌面应用的全流程、踩过的坑与解决方案。适用于 `l420x` 分支。

- 上游：https://github.com/Chevey339/kelivo
- Fork：https://github.com/zyqfork/kelivo.git
- 分支：`l420x`（基于上游 master 的 UOS ARM64 适配）

---

## 1. 环境准备

| 项目 | 值 |
|---|---|
| 系统 | UOS Desktop 20 Professional（aarch64） |
| 内核 | 5.10.97-arm64-desktop-full |
| 处理器 | HUAWEI Kirin 9000C（Maleoon 910 GPU） |
| Flutter | `/home/zyq/bin/flutter-arm64`，3.44.9 stable（Linux arm64 toolchain ✓） |
| 屏幕 | 2880×1920（3:2，~244 DPI 高分屏） |

> 注意：Flutter 的 Linux 桌面构建必须用 **arm64 版 Flutter SDK**（`flutter build linux --target-platform linux-arm64`），x86_64 SDK 无法产出 ARM64 二进制。

## 2. 构建流程

```bash
cd /home/zyq/kelivo
git checkout l420x

# 清理增量缓存（踩坑：不清理有时会用旧产物）
rm -rf .dart_tool/flutter_build build/linux

# 构建（arm64 release）
/home/zyq/bin/flutter-arm64 build linux --release --target-platform linux-arm64

# 产物：build/linux/arm64/release/bundle/
#   kelivo           主程序
#   lib/libflutter_linux_gtk.so 等插件库
```

## 3. 打包与安装（deb）

```bash
# 1) 补充缺失的 libsqlite3（构建产物里没有，必须从系统拷贝）
cp /lib/aarch64-linux-gnu/libsqlite3.so.0.8.6 \
   build/linux/arm64/release/bundle/lib/libsqlite3.so

# 2) 组装 deb 目录
rm -rf /tmp/kelivo-deb && mkdir -p /tmp/kelivo-deb/{DEBIAN,opt/kelivo,usr/bin,usr/share/applications}
cp -r build/linux/arm64/release/bundle/* /tmp/kelivo-deb/opt/kelivo/
chmod 755 /tmp/kelivo-deb/opt/kelivo/kelivo
```

`DEBIAN/control`（注意：文件必须以换行结尾，否则 dpkg-deb 报错）：

```
Package: kelivo
Version: 1.2.0
Section: utils
Priority: optional
Architecture: arm64
Maintainer: zyq <zyq@zyq-PC.local>
Description: Kelivo LLM Chat Client (UOS ARM64 l420x build)
```

`usr/bin/kelivo`（启动包装脚本，关键：**硬件渲染守卫** + GLES + 缩放）：

```bash
#!/bin/bash
# Kelivo launcher for UOS ARM64 (HUAWEI Kirin 9000C / Maleoon 910)
#
# Hardware rendering only: the Maleoon 910 GPU is reachable via EGL
# (libGFX_hisi) on the Wayland backend. X11 GLX falls back to llvmpipe
# (software rendering), so refuse to start without Wayland instead of
# silently degrading.
export GDK_GL=gles
export KELIVO_UI_SCALE=1.5

if [ ! -e /run/user/$(id -u)/wayland-0 ]; then
  echo "Kelivo 需要 Wayland 会话（硬件 GLES 渲染）。请切换 Wayland 后重试。" >&2
  exit 1
fi
export GDK_BACKEND=wayland

if [ ! -e /usr/lib/aarch64-linux-gnu/libGFX_hisi.so.0.8.0 ]; then
  echo "警告: 未检测到 Maleoon GPU 驱动 libGFX_hisi，渲染可能为软件模式" >&2
fi

exec /opt/kelivo/kelivo "$@"
```

```bash
# 3) 构建并安装
dpkg-deb --build --root-owner-group /tmp/kelivo-deb /tmp/kelivo-1.2.0-arm64.deb
sudo dpkg -i /tmp/kelivo-1.2.0-arm64.deb
```

安装到：`/opt/kelivo/`（主程序）+ `/usr/bin/kelivo`（包装脚本）。

## 4. 运行验证

```bash
/usr/bin/kelivo
```

- 图形后端：**强制 Wayland**（`GDK_BACKEND=wayland`），无 Wayland 拒绝启动（见下方"硬件渲染守卫"）
- 渲染：`GDK_GL=gles`（HiGFX GLES 加速，Maleoon 910 无桌面版 GLX 的 GL）
- 缩放：`KELIVO_UI_SCALE=1.5`

### 硬件渲染守卫（2026-08 修订）

Maleoon 910 的硬件 EGL 实现是 `libGFX_hisi.so.0.8.0`（导出 713 个 EGL/GLES 符号）+ `libbishenggpucompiler_v200.so.15`（GPU 着色器编译器），通过 `/dev/dri/card0`（hisi-dpu-drm）驱动。该硬件路径只在 **Wayland + GDK_GL=gles** 下可用：

- X11 GLX 会回退到 **llvmpipe 软件渲染**（glxinfo 实证：`llvmpipe (LLVM 13.0.1)`）
- 因此启动脚本**强制 Wayland**，无 Wayland 会话直接报错退出，绝不让 llvmpipe 兜底
- 附加检查 libGFX_hisi 存在性，缺失时警告

验证进程确为硬件渲染：

```bash
# 进程应持有 /dev/dri/card0 fd，且加载 libGFX_hisi + libbishenggpucompiler
lsof -p $(pgrep -x kelivo) | grep -iE "GFX|bisheng|dri"
# Wayland EGL 下 eglinfo 应显示 EGL_HUAWEI_partial_update（华为 GPU 特有扩展）
eglinfo | grep HUAWEI
```

验证窗口尺寸（Wayland 会话下 X 工具不可见，用 KWin 截图）：

```bash
export WAYLAND_DISPLAY=wayland-0 XDG_RUNTIME_DIR=/run/user/1000
qdbus org.kde.KWin /Screenshot org.kde.kwin.Screenshot.screenshotArea 0 0 2880 1920
```

日志中出现 `Timed out waiting for software frame of size 2880x1920` 属 resize 瞬间的正常告警，窗口稳定后无新告警即为正常。

---

## 5. 缩放问题的解决（核心）

### 现象

`KELIVO_UI_SCALE=1.5` 加上后 UI 完全没有放大。

### 根因排查（关键：不猜，用日志实测）

1. **第一层怀疑**：DPR / MediaQuery 缩放。加 `debugPrint` 验证 —— **release 模式下 debugPrint 被禁用**，无输出。
2. 改用 `stderr.writeln` —— **AOT release 下也被吞**，仍无输出。
3. 改用**写文件日志**（绝对可靠）—— 发现 `_scaleApp`（缩放函数）**从未被调用**。
4. 检查 `lib/main.dart`，发现应用里有 **3 个 MaterialApp**：

   | 位置 | 用途 | 是否挂 `_scaleApp` |
   |---|---|---|
   | `_RestoreFailureApp`（恢复失败诊断屏） | 数据恢复失败才出现 | 有（**正常启动不经过**） |
   | `MigrationApp`（数据迁移屏） | 升级迁移才出现 | 有（**正常启动不经过**） |
   | `MyApp`（**正常启动路径**） | 主界面 | ❌ **没有** |

5. **真相**：正常启动 → `MyApp` 的 MaterialApp（lib/main.dart:990），其 `builder` 是自定义的（系统状态栏样式 + 全局浮层），之前没有缩放包装 → 缩放代码从未执行，UI 自然不放大。

### 最终方案

**布局层整体缩放**（`lib/main.dart` 的 `_scaleApp`）：

- GTK embedder 只暴露**整数 DPR**，无法用 devicePixelRatio 实现 1.5×，所以放弃 DPR 方案
- **子组件按 `窗口尺寸/scale` 布局**，用 `Transform.scale(scale)` 整体放大 —— 缩放结果精确铺满窗口，任何屏幕比例（16:9、3:2）都无黑边
- `KELIVO_UI_SCALE=1.5` 时：2880×1920 屏上逻辑布局 1920×1280，放大 1.5× = 2880×1920 铺满
- 配套改动：`_scaleApp` 挂到**正常路径** `MyApp` 的 builder 上

### 黑块/letterbox 修复（2026-08 修订）

早期版本把布局固定为 1280×720 再 `min()` 缩放（`s = scale × min(w/(1280·scale), h/(720·scale))`）：

- 窗口 1920×1080（16:9，与设计同比例）时恰好铺满 ✓
- 窗口跟随主屏到 2880×1920（3:2）后：`s = 1.5 × min(2880/1920, 1920/1080) = 2.25`，UI 高只有 `720×2.25 = 1620`，窗口高 1920 → **上下各 150px 纯黑带**（截图实证 y=604-716 处黑条）

修订为**布局 = 窗口/scale**（见上），黑带消失（截图实证：四边 0 黑像素）。

### 窗口跟随主屏（2026-08 修订）

早期版本把窗口硬编码为 1920×1080（= 1280×720×1.5），在 2880×1920 屏上只占 2/3，四周留黑边，不跟随系统分辨率。修订：

- `DesktopWindowController.primaryDisplaySize()`（`lib/desktop/desktop_window_controller.dart`）用 `screen_retriever` 获取主屏物理尺寸；失败时 fallback 到 `1280×uiScale × 720×uiScale`
- `uiScale > 1` 时窗口初始尺寸 = 主屏尺寸，`minSize` 保持 `1280×uiScale × 720×uiScale`（保证 UI 最小可读）
- `linux/runner/my_application.cc` 的 `gtk_window_set_default_size` 改回 1280×720（真实尺寸由 Dart 层决定）
- `_scaleApp` 改为布局 = 窗口/scale（见上"黑块修复"）

现在窗口铺满 2880×1920，UI 放大 1.5× 精确铺满窗口，`KELIVO_UI_SCALE` 语义为**最小 UI 缩放下限**。

---

## 6. 踩坑清单

### 6.1 缩放代码从未执行（最隐蔽）
- **现象**：KELIVO_UI_SCALE 无效
- **根因**：应用有 3 个 MaterialApp，缩放代码只加在诊断/迁移分支上，正常路径的 `MyApp` builder 是自定义的、没有缩放包装
- **解决**：挂到正常路径 builder；用写文件日志实证执行

### 6.2 release 调试输出被吞
- **现象**：debugPrint、stderr 都不输出
- **根因**：release/AOT 下调试通道被禁用
- **解决**：写临时文件日志（`/tmp/kelivo_debug.txt`），验证后移除

### 6.3 GTK DPR 只有整数
- **现象**：无法用 DPR 实现 1.5×
- **根因**：GTK embedder 的 devicePixelRatio 只支持整数
- **解决**：`Transform.scale` 布局层缩放

### 6.4 window_manager 持久化尺寸覆盖
- **现象**：设置窗口尺寸后重启又变小
- **根因**：window_manager 从 SharedPreferences 恢复上次尺寸
- **解决**：`waitUntilReadyToShow` 回调里 `setSize` 覆盖

### 6.5 中文字体渲染（豆腐块/缺字）
- **现象**：中文显示为方块，emoji 缺失
- **根因**：代码里引用 macOS 风格字体名（PingFang SC / Heiti SC / Hiragino Sans GB），Linux fontconfig 解析不到
- **解决**：
  - 捆绑合并字体 `assets/fonts/DroidSansFallbackFull.ttf`（Droid Sans Fallback + Symbola，38215 glyphs，含全部 emoji U+1F600-1F64F），pubspec 注册为 `DroidSansFallback` 族
  - 主题、聊天、markdown、代码块全部设为 `DroidSansFallback` 默认/回退
  - `/usr/share/fonts/kelivo-aliases/` 放 15 个字体别名文件（同一合并字体的多份拷贝，family 名分别声明为 Roboto、PingFang SC、monospace、emoji 等），让系统字体名解析到中文字体

### 6.6 GLES 渲染
- **现象**：默认渲染后端黑屏/卡死
- **根因**：Maleoon 910 无桌面 GLX，只有 GLES
- **解决**：`GDK_GL=gles` 环境变量

### 6.7 X11 回退 llvmpipe 软件渲染
- **现象**：X11 会话下渲染缓慢/劣化（glxinfo 显示 `llvmpipe (LLVM 13.0.1)`）
- **根因**：Maleoon 910 的硬件 EGL（libGFX_hisi）只在 Wayland 路径可用，X11 GLX 没有硬件驱动
- **解决**：启动脚本强制 `GDK_BACKEND=wayland`，无 Wayland 会话直接拒绝启动（硬件渲染守卫）；验证：`lsof -p $(pgrep -x kelivo) | grep -E "GFX|bisheng|dri"` 应命中

### 6.8 deb 打包坑
- **现象**：`dpkg-deb: 解析 control 第 6 行…缺失结尾的换行符`
- **根因**：DEBIAN/control 末尾没有换行
- **解决**：文件末尾补 `\n`
- **另一个坑**：重装 deb 会覆盖 `/usr/bin/kelivo` 启动脚本——打包 recipe 必须始终包含 `usr/bin/kelivo`（不要只打包 opt/kelivo）

### 6.7 Linux 关窗不退出
- **现象**：关闭窗口进程残留（托盘）
- **根因**：桌面平台托盘默认开启
- **解决**：Linux 下 `_desktopShowTray` 默认改为 false（`lib/core/providers/settings_provider.dart`），关窗即退出；可在设置里重新开启

### 6.8 deb 打包坑
- **现象**：`dpkg-deb: 解析 control 第 6 行…缺失结尾的换行符`
- **根因**：DEBIAN/control 末尾没有换行
- **解决**：文件末尾补 `\n`

### 6.9 libsqlite3 缺失
- **现象**：启动报 libsqlite3.so 找不到
- **根因**：Flutter 构建产物不含 sqlite 动态库
- **解决**：从 `/lib/aarch64-linux-gnu/libsqlite3.so.0.8.6` 拷贝进 bundle/lib/

### 6.10 UI 缩放 letterbox 黑带
- **现象**：窗口铺满主屏后，上下出现纯黑条带（截图实证 y=604-716 黑带）
- **根因**：`_scaleApp` 布局固定 1280×720 再 min() 缩放；3:2 屏（2880×1920）与 16:9 设计比例不同，UI 高度 1620 < 窗口 1920，留出黑边
- **解决**：布局改为 `窗口尺寸/scale`，Transform.scale(scale) 后精确铺满任意比例屏幕

### 6.11 Trellis 脚本需要 Python 3.8+（开发环境）
- **现象**：`task.py` 报 `cannot import name 'TypedDict'`
- **根因**：UOS 系统 Python 3.7，TypedDict 是 3.8 才进 typing
- **解决**：用 nix 商店的 Python 3.13 运行（`/nix/store/…/python3.13`）

---

## 7. 相关文件清单（l420x 改动）

| 文件 | 改动 |
|---|---|
| `lib/main.dart` | `_scaleApp` 整体缩放（布局=窗口/scale，无黑边）+ 正常路径 builder + 恢复失败窗口跟随主屏 |
| `lib/desktop/desktop_window_controller.dart` | 窗口初始尺寸跟随主屏（screen_retriever） |
| `lib/desktop/window_size_manager.dart` | `uiScale` getter（读 KELIVO_UI_SCALE） |
| `lib/theme/theme_factory.dart` | Linux 字体回退 + DroidSansFallback 默认族 |
| `lib/features/chat/widgets/chat_message_widget.dart` | 聊天文本 CJK 字体 |
| `lib/shared/widgets/markdown_with_highlight.dart` | markdown/代码块 CJK 字体 |
| `lib/core/providers/settings_provider.dart` | Linux 托盘默认关 |
| `linux/runner/my_application.cc` | GTK 默认窗口尺寸回退 |
| `pubspec.yaml` | 字体资源注册 |
| `assets/fonts/DroidSansFallbackFull.ttf` | 合并中文字体 |
| `docs/build-notes-l420x.md` | 本文档 |