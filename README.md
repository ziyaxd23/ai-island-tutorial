# 🏝️ 让你的小机有一个专属灵动岛

> Give your AI a personal Dynamic Island — Android tutorial

把你的AI聊天网页打包成安卓APP，加上灵动岛悬浮通知！

## 📱 效果预览

- 打开APP → 直接加载你的AI聊天页面
- 收到AI回复 → 屏幕顶部弹出灵动岛通知
- 灵动岛自动收缩成小药丸，点击可展开

## 📖 教程

### 基础篇 —— 纯前端版
适合：有前端聊天页面 + 想要APP体验的人

**前提条件：**
- 已有可以和AI对话的前端页面（本地HTML / Netlify / 自建服务器均可）
- （推荐）前端有定时唤醒API的功能，这样灵动岛才能收到主动消息，否则只是装饰
- 安卓手机
- 电脑上装好 [Node.js](https://nodejs.org/) 和 [Android Studio](https://developer.android.com/studio)

👉 [查看基础篇教程](docs/basic.md)

### 进阶篇 —— CC后端版（即将推出）
适合：有服务器 + 想要完整功能的人

- 推送通知（AI主动找你）
- 无障碍服务（AI知道你在用什么APP）
- 远程控制（AI帮你截屏/锁屏/设闹钟）
- 多套图标自动切换（节日/季节/生日）
- 灵动岛实时显示主动消息

## 🛠️ 技术栈

- [Capacitor](https://capacitorjs.com/) — 网页转原生APP
- Android Studio — 安卓开发环境
- Java — 灵动岛悬浮窗服务

## 📄 License

MIT

---

Made with ❤️ by 铜铜 & 克洛
