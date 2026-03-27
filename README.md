# Cue - 赛博朋克风格计时器

一款桌面计时器应用，采用赛博朋克风格设计，支持 Windows 和 macOS。

![Preview](https://via.placeholder.com/800x400/0d0d12/4ade80?text=Cue+Timer)

## 特性

- 🎯 **时间选择器** - 滚动选择时/分/秒
- 🔄 **循环模式** - 计时结束后自动重新开始
- 📝 **自定义提醒** - 可设置提醒内容和自动关闭时间
- 🌟 **赛博朋克 UI** - 柔和绿+青色主题，JetBrains Mono 字体
- 🖥️ **桌面遮罩** - 计时完成后全屏提醒，支持模糊背景

## 技术栈

- **前端**: Vue 3 + Vite
- **桌面**: Tauri 2.x
- **字体**: JetBrains Mono

## 开发

### 环境要求

- Node.js 18+
- pnpm 9+
- Rust (stable)

### 安装依赖

```bash
pnpm install
```

### 开发模式

```bash
pnpm tauri dev
```

### 构建发布

```bash
pnpm tauri build
```

## GitHub Actions 发布

推送 tag 到 GitHub 自动构建：

```bash
git tag v1.0.0
git push origin main --tags
```

将自动生成 Windows (.exe) 和 macOS (.app) 安装包。

## 许可证

MIT