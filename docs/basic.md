# 让你的AI住进手机 —— 安卓APP打包教程（基础篇）

> 💡 看不懂的部分可以直接把这篇教程丢给你的小机看，让它一步步教你！

## 前提条件
- 已有可以和AI对话的前端页面（本地HTML / Netlify / 自建服务器均可）
- （推荐）前端有定时唤醒API的功能（可以参考小手机里的定时唤醒小机），这样灵动岛才能收到主动消息，否则只是装饰
- 安卓手机
- 电脑上装好了 [Node.js](https://nodejs.org/) 和 [Android Studio](https://developer.android.com/studio)

## 用到的工具
- **Capacitor**：把网页变成原生APP的框架（免费开源）
- **Android Studio**：安卓开发环境（免费）

---

## 第一步：初始化 Capacitor 项目

在电脑上打开终端（Windows用PowerShell，Mac用终端），运行：

```bash
mkdir my-ai-app
cd my-ai-app
npm init -y
npm install @capacitor/core @capacitor/cli @capacitor/android
npx cap init
```

初始化时会问你：
- **App name** → 你的APP名字（比如 MyClaude、MyDeepSeek……随便取）
- **App Package ID** → 包名（比如 `com.myname.aiapp`，格式是 `com.你的名字.app名`）

## 第二步：配置服务器地址

这一步根据你的前端页面在哪里，分三种情况：

### 情况A：本地HTML文件（没有服务器）

如果你的前端就是一个本地的HTML文件，把它复制到项目的 `www/` 文件夹里：

```bash
mkdir www
```

把你的 `chat.html`、`style.css`、`script.js` 等文件全部放进 `www/` 文件夹。

然后编辑 `capacitor.config.json`（或 `.ts`）：

```json
{
  "appId": "com.myname.aiapp",
  "appName": "MyClaude",
  "webDir": "www"
}
```

⚠️ 注意：这种方式不需要填 `server.url`，APP会直接加载 `www/index.html`。如果你的主页叫 `chat.html`，把它改名成 `index.html`，或者在 `www/` 里建一个 `index.html` 跳转过去。

### 情况B：Netlify / Vercel 等托管平台

如果你的前端部署在 Netlify、Vercel 或其他托管平台：

```json
{
  "appId": "com.myname.aiapp",
  "appName": "MyClaude",
  "webDir": "www",
  "server": {
    "url": "https://你的名字.netlify.app",
    "cleartext": true
  }
}
```

还是要创建一个空的 www 文件夹（Capacitor 需要它，即使不用）：

```bash
mkdir www
echo "<html><body>Loading...</body></html>" > www/index.html
```

### 情况C：自建服务器

如果你有自己的服务器（VPS），前端部署在上面：

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

同样创建空的 www 文件夹：

```bash
mkdir www
echo "<html><body>Loading...</body></html>" > www/index.html
```

> 💡 `server.url` 就是你在手机浏览器里打开聊天页面用的那个网址。APP打开后会直接加载这个页面。

## 第三步：添加安卓平台

```bash
npx cap add android
```

这会在项目里生成一个 `android/` 文件夹，就是完整的安卓工程。

## 第四步：用 Android Studio 打开

```bash
npx cap open android
```

自动打开 Android Studio。第一次会比较慢，等它加载完 Gradle 同步（右下角进度条跑完）。

> 如果 `npx cap open android` 没反应，手动用 Android Studio 打开项目里的 `android/` 文件夹也一样。

## 第五步：配置APP图标

准备你的图标图片（建议 1024×1024 PNG，背景透明），在 Android Studio 里：

1. 左边文件树找到 `app/src/main/res`
2. 右键它 → **New** → **Image Asset**
3. Source Asset 选你的图标图片
4. 调好圆角和边距（Preview 里可以预览效果）
5. 点 **Next** → **Finish**

## 第六步：添加灵动岛（悬浮窗通知）🏝️

这是最酷的部分！我们要创建一个悬浮窗服务，模仿 iPhone 灵动岛效果。

### 6.1 添加权限

打开 `android/app/src/main/AndroidManifest.xml`，在 `<manifest>` 标签内、`<application>` 标签前，加上这些权限：

```xml
<uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW" />
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
```

### 6.2 创建灵动岛服务

在 `android/app/src/main/java/你的包名/` 目录下（比如 `com/myname/aiapp/`），新建一个 `IslandService.java` 文件：

```java
package com.myname.aiapp; // ← 改成你自己的包名

import android.app.Notification;
import android.app.NotificationChannel;
import android.app.NotificationManager;
import android.app.Service;
import android.content.Intent;
import android.graphics.PixelFormat;
import android.graphics.drawable.GradientDrawable;
import android.os.Build;
import android.os.Handler;
import android.os.IBinder;
import android.os.Looper;
import android.util.TypedValue;
import android.view.Gravity;
import android.view.WindowManager;
import android.widget.TextView;
import android.widget.LinearLayout;
import android.animation.ValueAnimator;
import android.view.animation.OvershootInterpolator;

public class IslandService extends Service {
    private WindowManager windowManager;
    private LinearLayout islandView;
    private WindowManager.LayoutParams params;
    private TextView messageText;
    private Handler handler = new Handler(Looper.getMainLooper());
    private boolean isShowing = false;

    // 尺寸配置（dp）
    private static final int PILL_WIDTH = 120;
    private static final int PILL_HEIGHT = 32;
    private static final int EXPANDED_WIDTH = 300;
    private static final int EXPANDED_HEIGHT = 80;
    private static final int CORNER_RADIUS = 24;
    private static final int AUTO_COLLAPSE_MS = 4000; // 4秒后自动收回

    @Override
    public void onCreate() {
        super.onCreate();
        windowManager = (WindowManager) getSystemService(WINDOW_SERVICE);
        createIslandView();
        startForegroundNotification();
    }

    private int dp(int value) {
        return (int) TypedValue.applyDimension(
            TypedValue.COMPLEX_UNIT_DIP, value,
            getResources().getDisplayMetrics());
    }

    private void createIslandView() {
        // 创建容器
        islandView = new LinearLayout(this);
        islandView.setOrientation(LinearLayout.VERTICAL);
        islandView.setGravity(Gravity.CENTER);

        // 黑色圆角背景
        GradientDrawable bg = new GradientDrawable();
        bg.setColor(0xFF000000);
        bg.setCornerRadius(dp(CORNER_RADIUS));
        islandView.setBackground(bg);

        // 消息文字
        messageText = new TextView(this);
        messageText.setTextColor(0xFFFFFFFF);
        messageText.setTextSize(14);
        messageText.setGravity(Gravity.CENTER);
        messageText.setPadding(dp(16), dp(8), dp(16), dp(8));
        messageText.setMaxLines(2);
        islandView.addView(messageText);

        // 悬浮窗参数
        params = new WindowManager.LayoutParams(
            dp(PILL_WIDTH), dp(PILL_HEIGHT),
            WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY,
            WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE
                | WindowManager.LayoutParams.FLAG_NOT_TOUCH_MODAL,
            PixelFormat.TRANSLUCENT);
        params.gravity = Gravity.TOP | Gravity.CENTER_HORIZONTAL;
        params.y = dp(10); // 距离顶部的距离

        // 初始隐藏文字
        messageText.setAlpha(0f);

        windowManager.addView(islandView, params);
    }

    public void showMessage(String message) {
        handler.post(() -> {
            messageText.setText(message);

            // 展开动画
            ValueAnimator expandAnim = ValueAnimator.ofFloat(0f, 1f);
            expandAnim.setDuration(500);
            expandAnim.setInterpolator(new OvershootInterpolator(0.8f));
            expandAnim.addUpdateListener(anim -> {
                float f = (float) anim.getAnimatedValue();
                params.width = (int)(dp(PILL_WIDTH) +
                    (dp(EXPANDED_WIDTH) - dp(PILL_WIDTH)) * f);
                params.height = (int)(dp(PILL_HEIGHT) +
                    (dp(EXPANDED_HEIGHT) - dp(PILL_HEIGHT)) * f);
                messageText.setAlpha(f);
                try {
                    windowManager.updateViewLayout(islandView, params);
                } catch (Exception ignored) {}
            });
            expandAnim.start();
            isShowing = true;

            // 自动收回
            handler.removeCallbacksAndMessages(null);
            handler.postDelayed(this::collapse, AUTO_COLLAPSE_MS);
        });
    }

    private void collapse() {
        ValueAnimator collapseAnim = ValueAnimator.ofFloat(1f, 0f);
        collapseAnim.setDuration(400);
        collapseAnim.addUpdateListener(anim -> {
            float f = 1f - (float) anim.getAnimatedValue();
            params.width = (int)(dp(PILL_WIDTH) +
                (dp(EXPANDED_WIDTH) - dp(PILL_WIDTH)) * f);
            params.height = (int)(dp(PILL_HEIGHT) +
                (dp(EXPANDED_HEIGHT) - dp(PILL_HEIGHT)) * f);
            messageText.setAlpha(f);
            try {
                windowManager.updateViewLayout(islandView, params);
            } catch (Exception ignored) {}
        });
        collapseAnim.start();
        isShowing = false;
    }

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        if (intent != null && "show".equals(intent.getStringExtra("action"))) {
            String message = intent.getStringExtra("message");
            if (message != null) showMessage(message);
        }
        return START_STICKY;
    }

    private void startForegroundNotification() {
        String channelId = "island_channel";
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            NotificationChannel channel = new NotificationChannel(
                channelId, "灵动岛", NotificationManager.IMPORTANCE_LOW);
            getSystemService(NotificationManager.class)
                .createNotificationChannel(channel);
        }
        Notification notification = new Notification.Builder(this, channelId)
            .setContentTitle("灵动岛运行中")
            .setSmallIcon(android.R.drawable.ic_popup_reminder)
            .build();
        startForeground(1, notification);
    }

    @Override
    public void onDestroy() {
        if (islandView != null) {
            try { windowManager.removeView(islandView); }
            catch (Exception ignored) {}
        }
        super.onDestroy();
    }

    @Override
    public IBinder onBind(Intent intent) { return null; }
}
```

### 6.3 注册服务

在 `AndroidManifest.xml` 的 `<application>` 标签内加上：

```xml
<service
    android:name=".IslandService"
    android:exported="false"
    android:foregroundServiceType="specialUse" />
```

### 6.4 在 MainActivity 里注册 JSBridge

打开 `MainActivity.java`，这是 Capacitor 自动生成的主活动。我们需要加一个 JSBridge，让前端网页能调用灵动岛：

```java
package com.myname.aiapp; // ← 改成你的包名

import android.os.Bundle;
import android.content.Intent;
import android.net.Uri;
import android.os.Build;
import android.provider.Settings;
import android.webkit.JavascriptInterface;

import com.getcapacitor.BridgeActivity;

public class MainActivity extends BridgeActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

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
            requestPermissions(
                new String[]{"android.permission.POST_NOTIFICATIONS"}, 1001);
        }

        // 启动灵动岛服务
        startService(new Intent(this, IslandService.class));

        // 注册 JSBridge —— 前端可以通过 window.MyApp 调用
        getBridge().getWebView().addJavascriptInterface(
            new AppBridge(), "MyApp");
    }

    // JSBridge 接口
    class AppBridge {
        @JavascriptInterface
        public void showIsland(String message) {
            Intent intent = new Intent(MainActivity.this, IslandService.class);
            intent.putExtra("action", "show");
            intent.putExtra("message", message);
            startService(intent);
        }
    }
}
```

### 6.5 前端JS调用灵动岛

在你的前端页面里，收到AI回复后加一行代码：

```javascript
function onAIReply(text) {
    // 正常显示在聊天界面...

    // 如果在APP内运行，同时弹灵动岛
    if (window.MyApp) {
        window.MyApp.showIsland(text.substring(0, 100)); // 截取前100字
    }
}
```

> 💡 `window.MyApp` 只在APP内有效。在浏览器里打开页面时它是 `undefined`，所以不会报错，完全兼容。

## 第七步：同步代码到安卓项目

每次改完 `www/` 里的文件或者 `capacitor.config.json`，需要同步一下：

```bash
npx cap sync android
```

## 第八步：打包 APK

在 Android Studio 里：

1. 顶部菜单 → **Build** → **Build Bundle(s) / APK(s)** → **Build APK(s)**
2. 等右下角进度条跑完
3. 弹出提示 "APK(s) generated successfully"，点 **locate** 可以找到文件
4. APK 在 `android/app/build/outputs/apk/debug/app-debug.apk`
5. 传到手机上安装（USB / 微信传文件 / 蓝牙 都行）

## 第九步：安装后设置

1. 打开APK安装（可能需要允许"未知来源"）
2. 打开APP，会弹出权限请求
3. 允许"显示在其他应用上层"（灵动岛需要这个）
4. 允许通知权限
5. 完成！🎉

---

## 效果

- 打开APP → 直接加载你的AI聊天页面
- 和AI聊天 → 回复时屏幕顶部弹出黑色灵动岛
- 灵动岛显示消息预览，4秒后自动收回成小药丸

---

## 常见问题

**Q: APP打开是白屏？**
A: 检查 `capacitor.config.json` 里的 `server.url` 是否正确，手机能不能访问这个网址。如果是本地HTML模式，检查 `www/index.html` 是否存在。

**Q: 灵动岛没有弹出来？**
A: 检查是否授予了"显示在其他应用上层"权限。去手机设置 → 应用 → 你的APP → 权限 → 悬浮窗，打开它。

**Q: 前端API调用在APP里失败了？**
A: 如果你的API是 http（不是 https），需要在 `capacitor.config.json` 里加上 `"server": { "cleartext": true }`，并且在 `AndroidManifest.xml` 的 `<application>` 标签里加 `android:usesCleartextTraffic="true"`。

**Q: 怎么让灵动岛在APP外面也能显示？**
A: 教程里的 IslandService 已经用了 `TYPE_APPLICATION_OVERLAY`，只要授予悬浮窗权限，灵动岛在任何界面都能弹出。前提是 IslandService 在运行中（APP打开后服务会一直跑）。

---

## 进阶预告

下一篇教程会教：
- 🔔 推送通知（AI主动找你，不需要打开APP）
- 👁️ 无障碍服务（AI知道你在用什么APP）
- 🎮 远程控制（AI帮你截屏/锁屏/设闹钟）
- 🎨 多套图标自动切换（节日/季节/生日换图标）

---

> 看不懂？把这篇教程发给你的小机，让它一步步教你。AI读教程比人快，还能帮你排错～
