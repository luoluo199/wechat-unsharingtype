# wechat-unsharingtype

解除 macOS 微信的防截图/防录屏保护，让系统截图、屏幕录制、以及 AI Agent 都能正常看到微信窗口。

## 问题

微信 macOS 版对窗口设置了 `NSWindow.sharingType = .none`，导致：

- 系统截图（⌘⇧3/4）截到微信窗口时显示**透明/穿透**
- 第三方截图工具截到**白屏**
- 屏幕录制看不到微信内容
- AI Agent（如 Claude Code）无法通过 ScreenCaptureKit 获取微信窗口画面

即使在 macOS 15+ 上，ScreenCaptureKit 仍然尊重 `sharingType = .none`，无法绕过。

## 原理

通过 DYLD 注入一个动态库到微信进程，使用 Objective-C 运行时 swizzle `NSWindow.setSharingType:` 方法，强制所有窗口的 `sharingType` 为 `readWrite`（值为 2），同时启动定时器每 2 秒修正已有窗口。

**不依赖特定微信版本号**，任何版本通用。

## 环境要求

- macOS（已在 macOS 27 / Darwin 27.0.0 上测试）
- Apple Silicon（arm64）
- Xcode Command Line Tools（需要 `clang`、`codesign`）
- [insert_dylib](https://github.com/tyilo/insert_dylib)（脚本会自动安装）

## 使用方法

### 安装（注入补丁）

```bash
sudo ./inject_v2.sh
```

脚本会自动：
1. 关闭微信
2. 备份原始二进制（`WeChat.backup`）
3. 编译 `unsharingtype.dylib`
4. 注入到微信主二进制
5. 仅重新签名主二进制（不影响其他组件）

### 启动微信

由于修改了签名，`open` 命令和 Dock 双击可能被 Gatekeeper 阻止。提供以下启动方式：

```bash
# 方式1：使用微信启动器（推荐）
# 将「微信启动器.app」复制到 /Applications/ 后拖到 Dock，点击即可启动
cp -R 微信启动器.app /Applications/

# 方式2：命令行
/Applications/WeChat.app/Contents/MacOS/WeChat &

# 方式3：双击启动脚本
./start_wechat.command
```

### 微信启动器

仓库内附带 `微信启动器.app`，这是一个轻量 .app 包装器，使用微信原始图标，功能等同于命令行启动。复制到 `/Applications/` 后可拖入 Dock，点击体验与原版一致。

### 验证效果

注入成功后，微信日志中会显示：

```
[UnSharingType] 正在加载...
[UnSharingType] ✅ hook 成功
[UnSharingType] ✅ 定时修正已启动
```

微信窗口的 `sharingType` 会从 `0`（none）变为 `2`（readWrite）。

### 还原

```bash
sudo ./inject_v2.sh --restore
```

## 文件说明

| 文件 | 说明 |
|------|------|
| `unsharingtype.m` | 核心 dylib 源码，swizzle NSWindow.setSharingType |
| `inject_v2.sh` | 注入/还原脚本 |
| `start_wechat.command` | 微信启动脚本（绕过 Gatekeeper） |
| `微信启动器.app/` | 微信启动器，复制到 /Applications/ 拖入 Dock 使用 |
| `capture_v2.swift` | 诊断工具：检查微信窗口的 sharingType 状态 |
| `capture_v4.swift` | 截图工具：用 ScreenCaptureKit 截取微信窗口 |

## 注意事项

- 修改后微信签名会变化，可能影响 iCloud 同步等功能（可随时还原）
- 此工具仅供个人学习研究使用
- 每次微信更新后需要重新注入

## 致谢

- [fzlzjerry/wechat-antirecall](https://github.com/fzlzjerry/wechat-antirecall) — 思路参考
- [privateai0/macos-window-privacy-research](https://github.com/privateai0/macos-window-privacy-research) — macOS 窗口隐私 API 研究
- [tyilo/insert_dylib](https://github.com/tyilo/insert_dylib) — dylib 注入工具
