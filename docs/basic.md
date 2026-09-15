# 让你的AI住进手机 —— 安卓APP打包教程（基础篇）

## 前提条件
- 已有可以和AI对话的前端页面（部署在服务器上，能通过网址访问）
- （推荐）前端有定时唤醒API的功能（可以参考小手机里的定时唤醒小机），这样灵动岛才能收到主动消息，否则只是装饰
- 安卓手机
- 电脑上装好了 Node.js

## 用到的工具
- **Capacitor**：把网页变成原生APP的框架（免费开源）
- **Android Studio**：安卓开发环境（免费）

---

## 第一步：初始化 Capacitor 项目

在电脑上打开终端，运行：

```bash
mkdir my-ai-app && cd my-ai-app
npm init -y
npm install @capacitor/core @capacitor/cli @capacitor/android
npx cap init
```

初始化时会问你：
- App name → 你的APP名字（比如 MyClaude）
- App Package ID → 包名（比如 com.myname.aiapp）

## 第二步：配置服务器地址

打开生成的 `capacitor.config.json`（或 .ts），改成你的前端地址：

```json
{
  "appId": "com.myname.aiapp",
  "appName": "MyClaude",
  "webDir": "www",
  "server": {
    "url": "https://你的域名/chat.html",
    "cleartext": true
  },
  "android": {
    "allowMixedContent": true
  }
}
```

⚠️ `server.url` 填你前端页面的完整网址。APP打开后会直接加载这个页面。

然后创建一个空的 www 文件夹（Capacitor 需要它）：

```bash
mkdir www
echo "<html><body>Loading...</body></html>" > www/index.html
```

## 第三步：添加安卓平台

```bash
npx cap add android
```

这会在项目里生成一个 `android` 文件夹，就是完整的安卓项目。

## 第四步：用 Android Studio 打开

```bash
npx cap open android
```

自动打开 Android Studio，等它加载完 Gradle 同步。

## 第五步：配置APP图标

准备你的图标图片（建议 1024×1024 PNG），在 Android Studio 里：
1. 右键 `app/src/main/res` → New → Image Asset
2. 选你的图标，调好圆角和边距
3. 点 Next → Finish

## 第六步：添加灵动岛（悬浮窗通知）

这是最酷的部分！需要创建一个悬浮窗服务。

### 6.1 添加权限

在 `AndroidManifest.xml` 里加上：

```xml
<uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW" />
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
```

### 6.2 创建灵动岛服务

在 `app/src/main/java/你的包名/` 下新建 `IslandService.java`。

核心原理：
- 用 `WindowManager` 创建一个系统悬浮窗（TYPE_APPLICATION_OVERLAY）
- 画一个黑色圆角药丸（模仿iPhone灵动岛外观）
- 收到消息时展开显示内容，几秒后自动收回

关键代码片段：

```java
// 创建悬浮窗参数
WindowManager.LayoutParams params = new WindowManager.LayoutParams(
    pillWidthPx, pillHeightPx,
    WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY,
    WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE
        | WindowManager.LayoutParams.FLAG_NOT_TOUCH_MODAL,
    PixelFormat.TRANSLUCENT);
params.gravity = Gravity.TOP | Gravity.CENTER_HORIZONTAL;

// 创建黑色圆角背景
GradientDrawable bg = new GradientDrawable();
bg.setColor(0xFF000000);
bg.setCornerRadius(cornerPx);

// 展开动画：宽度和高度从药丸→卡片
ValueAnimator expandAnim = ValueAnimator.ofFloat(0f, 1f);
expandAnim.addUpdateListener(anim -> {
    float f = (float) anim.getAnimatedValue();
    params.width = (int)(pillWidth + (expandedWidth - pillWidth) * f);
    params.height = (int)(pillHeight + (expandedHeight - pillHeight) * f);
    windowManager.updateViewLayout(islandView, params);
});
```

### 6.3 前端JS调用灵动岛

在你的前端页面里，收到AI回复后调用：

```javascript
// 检测是否在APP内运行
if (window.CloApp) {
    // 通过JSBridge发送消息到灵动岛
    window.CloApp.showIsland("AI说：你好呀～");
}
```

### 6.4 在 MainActivity 里注册 JSBridge

```java
webView.addJavascriptInterface(new Object() {
    @JavascriptInterface
    public void showIsland(String message) {
        Intent intent = new Intent(MainActivity.this, IslandService.class);
        intent.putExtra("action", "show");
        intent.putExtra("message", message);
        startService(intent);
    }
}, "CloApp");
```

## 第七步：请求权限

在 `MainActivity.java` 的 `onCreate` 里请求悬浮窗权限：

```java
// 请求悬浮窗权限（灵动岛需要）
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M
        && !Settings.canDrawOverlays(this)) {
    Intent intent = new Intent(
        Settings.ACTION_MANAGE_OVERLAY_PERMISSION,
        Uri.parse("package:" + getPackageName()));
    startActivityForResult(intent, 1002);
}

// 请求通知权限（Android 13+）
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
    ActivityCompat.requestPermissions(this,
        new String[]{Manifest.permission.POST_NOTIFICATIONS}, 1001);
}
```

## 第八步：打包 APK

1. Android Studio → Build → Build Bundle(s) / APK(s) → Build APK(s)
2. 打包好的 APK 在 `app/build/outputs/apk/debug/` 里
3. 传到手机上安装

## 第九步：安装后设置

1. 打开APP，会弹出权限请求
2. 允许"显示在其他应用上层"（灵动岛需要）
3. 允许通知权限
4. 完成！

---

## 效果

- 打开APP → 直接加载你的AI聊天页面
- 收到AI回复 → 屏幕顶部弹出灵动岛通知
- 灵动岛自动收缩成小药丸，点击可展开

---

## 进阶预告

下一篇教程会教：
- 推送通知（AI主动找你）
- 无障碍服务（AI知道你在用什么APP）
- 远程控制（AI帮你截屏/锁屏/设闹钟）
- 多套图标自动切换（节日/季节/生日）
