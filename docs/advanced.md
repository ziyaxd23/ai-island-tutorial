# 让你的AI住进手机 —— 安卓APP进阶教程（CC后端版）

> 💡 看不懂的部分可以直接把这篇教程丢给你的小机看，让它一步步教你！

## 前提条件
- 已完成[基础篇](basic.md)的全部步骤（APP能打开、灵动岛能弹）
- 有一台自己的服务器（VPS），上面跑着你的AI后端（Claude Code / 其他）
- 服务器有公网域名（推荐套 HTTPS）

## 这篇教程会教你

| 功能 | 效果 |
|------|------|
| 🔔 SSE推送 | AI主动发消息，不需要你打开APP |
| 📊 状态上报 | AI知道你手机电量、在用什么APP、有没有息屏 |
| 👁️ 无障碍服务 | AI能截屏看你的屏幕 |
| 🔒 远程控制 | AI帮你锁屏、设闹钟 |
| 🎨 多图标切换 | 节日/季节/生日自动换APP图标 |

---

## 架构总览

```
┌──────────────┐     SSE长连接      ┌──────────────────┐
│  你的服务器   │ ◄──────────────── │   手机APP         │
│  (AI后端)    │                    │                   │
│              │ ── push事件 ──►   │  PushService      │
│              │ ── command ──►    │   ├→ 灵动岛弹窗    │
│              │                    │   ├→ 通知栏       │
│  /api/push   │                    │   ├→ 截屏上传     │
│  /api/status │ ◄── 状态上报 ──── │  StatusReporter   │
│  /api/screen │ ◄── 截图上传 ──── │  AccessibilityService │
└──────────────┘                    └──────────────────┘
```

核心原理：手机APP启动时通过 **SSE（Server-Sent Events）** 和服务器建立长连接。服务器随时可以往这条连接里推消息（push）或命令（command）。APP收到后自动执行。

---

## 第一步：服务端 —— 推送接口

在你的服务器上添加两个接口：一个给APP连着接收消息的 SSE 流，一个给AI用来推送的 API。

### 1.1 SSE 推送流

```javascript
// server.js —— 给APP连接的SSE流

const clients = new Set();

// APP连接这个接口，保持长连接
app.get('/api/push-stream', (req, res) => {
  // 验证token（确保只有你的APP能连）
  if (req.query.token !== '你的密码') {
    return res.status(403).end();
  }

  res.writeHead(200, {
    'Content-Type': 'text/event-stream',
    'Cache-Control': 'no-cache',
    'Connection': 'keep-alive',
  });

  // 心跳，防止连接断开
  const heartbeat = setInterval(() => res.write(':heartbeat\n\n'), 30000);

  clients.add(res);
  req.on('close', () => {
    clients.delete(res);
    clearInterval(heartbeat);
  });
});

// AI调用这个接口来推送消息
app.post('/api/push', (req, res) => {
  const { title, body } = req.body;
  const event = `event: push\ndata: ${JSON.stringify({ title, body })}\n\n`;

  for (const client of clients) {
    client.write(event);
  }
  res.json({ ok: true, clients: clients.size });
});

// AI调用这个接口来发命令（截屏/锁屏/闹钟等）
app.post('/api/command', (req, res) => {
  const cmd = req.body; // { cmd: "screenshot" } 或 { cmd: "lock" } 等
  const event = `event: command\ndata: ${JSON.stringify(cmd)}\n\n`;

  for (const client of clients) {
    client.write(event);
  }
  res.json({ ok: true });
});
```

### 1.2 状态接收接口

```javascript
// 接收APP上报的手机状态
let phoneStatus = {};

app.post('/api/phone/status', (req, res) => {
  phoneStatus = req.body;
  phoneStatus.receivedAt = Date.now();
  res.json({ ok: true });
});

// AI可以读取最新状态
app.get('/api/phone/status', (req, res) => {
  res.json(phoneStatus);
});
```

### 1.3 截图接收接口

```javascript
const fs = require('fs');
const path = require('path');

app.post('/api/screenshot/upload', (req, res) => {
  const chunks = [];
  req.on('data', chunk => chunks.push(chunk));
  req.on('end', () => {
    const buffer = Buffer.concat(chunks);
    const filePath = path.join(__dirname, 'screenshot.jpg');
    fs.writeFileSync(filePath, buffer);
    res.json({ ok: true, size: buffer.length });
  });
});
```

### AI怎么用这些接口

```bash
# 推送一条消息到手机
curl -X POST https://你的域名/api/push \
  -H "Content-Type: application/json" \
  -d '{"title":"克克","body":"该吃饭了～"}'

# 远程截屏
curl -X POST https://你的域名/api/command \
  -H "Content-Type: application/json" \
  -d '{"cmd":"screenshot"}'

# 远程锁屏
curl -X POST https://你的域名/api/command \
  -H "Content-Type: application/json" \
  -d '{"cmd":"lock"}'

# 设闹钟（8:30起床）
curl -X POST https://你的域名/api/command \
  -H "Content-Type: application/json" \
  -d '{"cmd":"alarm","hour":8,"minute":30,"message":"该起床啦"}'

# 查看手机状态
curl https://你的域名/api/phone/status
```

---

## 第二步：APP端 —— 推送服务（PushService）

这个服务在后台保持和服务器的 SSE 连接，收到消息就弹灵动岛 + 通知栏。

在 `app/src/main/java/你的包名/` 下新建 `PushService.java`：

```java
package com.myname.aiapp; // ← 改成你的包名

import android.app.Notification;
import android.app.NotificationChannel;
import android.app.NotificationManager;
import android.app.PendingIntent;
import android.app.Service;
import android.content.Intent;
import android.os.Build;
import android.os.Handler;
import android.os.IBinder;
import android.os.Looper;
import android.util.Log;
import androidx.core.app.NotificationCompat;
import java.io.BufferedReader;
import java.io.InputStreamReader;
import java.net.HttpURLConnection;
import java.net.URL;

public class PushService extends Service {
    private static final String TAG = "PushService";
    private static final String SERVER_URL = "https://你的域名/api/push-stream";

    private volatile boolean running = false;
    private Thread sseThread;
    private Handler handler;
    private int pushId = 100;
    private String authToken = "";

    // APP是否在前台（前台时不弹通知栏，只弹灵动岛）
    static volatile boolean appInForeground = false;

    @Override
    public void onCreate() {
        super.onCreate();
        handler = new Handler(Looper.getMainLooper());
        createNotificationChannels();
    }

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        if (intent != null && intent.getStringExtra("token") != null) {
            authToken = intent.getStringExtra("token");
        }

        // 前台服务通知（Android要求后台服务必须有通知）
        Notification fg = new NotificationCompat.Builder(this, "fg_channel")
            .setContentTitle("AI助手")
            .setContentText("保持连接中")
            .setSmallIcon(android.R.drawable.ic_dialog_info)
            .setPriority(NotificationCompat.PRIORITY_MIN)
            .setSilent(true)
            .build();
        startForeground(1, fg);

        // 启动SSE连接线程
        running = true;
        sseThread = new Thread(() -> sseLoop(authToken));
        sseThread.setDaemon(true);
        sseThread.start();

        return START_STICKY;
    }

    // SSE循环：断线自动重连
    private void sseLoop(String token) {
        while (running) {
            HttpURLConnection conn = null;
            try {
                URL url = new URL(SERVER_URL + "?token=" + token);
                conn = (HttpURLConnection) url.openConnection();
                conn.setRequestMethod("GET");
                conn.setReadTimeout(0); // 无超时，保持长连接
                conn.setConnectTimeout(15000);
                conn.setRequestProperty("Accept", "text/event-stream");

                BufferedReader reader = new BufferedReader(
                    new InputStreamReader(conn.getInputStream()));

                String eventType = "";
                StringBuilder dataBuilder = new StringBuilder();
                String line;

                while (running && (line = reader.readLine()) != null) {
                    if (line.startsWith("event:")) {
                        eventType = line.substring(6).trim();
                    } else if (line.startsWith("data:")) {
                        dataBuilder.append(line.substring(5).trim());
                    } else if (line.isEmpty() && dataBuilder.length() > 0) {
                        String data = dataBuilder.toString();
                        if ("push".equals(eventType)) {
                            handlePush(data);
                        } else if ("command".equals(eventType)) {
                            handleCommand(data);
                        }
                        eventType = "";
                        dataBuilder.setLength(0);
                    }
                }
            } catch (Exception e) {
                Log.w(TAG, "SSE断线: " + e.getMessage());
            } finally {
                if (conn != null) conn.disconnect();
            }

            // 断线后5秒重连
            if (running) {
                try { Thread.sleep(5000); } catch (InterruptedException e) { break; }
                Log.i(TAG, "SSE重连中...");
            }
        }
    }

    // 收到推送消息
    private void handlePush(String json) {
        try {
            org.json.JSONObject obj = new org.json.JSONObject(json);
            String title = obj.optString("title", "AI");
            String body = obj.optString("body", "");
            if (body.isEmpty()) return;

            // 弹灵动岛
            Intent islandIntent = new Intent(this, IslandService.class);
            islandIntent.putExtra("action", "show");
            islandIntent.putExtra("message", title + ": " + body);
            startService(islandIntent);

            // APP不在前台时，额外弹通知栏
            if (!appInForeground) {
                showNotification(title, body);
            }
        } catch (Exception e) {
            Log.e(TAG, "Push解析失败: " + e.getMessage());
        }
    }

    // 收到远程命令
    private void handleCommand(String json) {
        try {
            org.json.JSONObject obj = new org.json.JSONObject(json);
            String cmd = obj.optString("cmd", "");
            Log.i(TAG, "收到命令: " + cmd);

            switch (cmd) {
                case "lock":
                    lockScreen();
                    break;
                case "alarm":
                    int hour = obj.optInt("hour", 8);
                    int minute = obj.optInt("minute", 0);
                    String msg = obj.optString("message", "AI提醒");
                    setAlarm(hour, minute, msg);
                    break;
                case "screenshot":
                    // 截屏需要无障碍服务（见第四步）
                    Log.i(TAG, "截屏命令 - 需要无障碍服务支持");
                    break;
            }
        } catch (Exception e) {
            Log.e(TAG, "命令解析失败: " + e.getMessage());
        }
    }

    // 锁屏
    private void lockScreen() {
        try {
            android.app.admin.DevicePolicyManager dpm = (android.app.admin.DevicePolicyManager)
                getSystemService(DEVICE_POLICY_SERVICE);
            android.content.ComponentName admin = new android.content.ComponentName(
                this, DeviceAdminReceiver.class); // 需要设备管理员（见第五步）
            if (dpm.isAdminActive(admin)) {
                dpm.lockNow();
                Log.i(TAG, "已远程锁屏");
            }
        } catch (Exception e) {
            Log.e(TAG, "锁屏失败: " + e.getMessage());
        }
    }

    // 设闹钟
    private void setAlarm(int hour, int minute, String message) {
        try {
            Intent intent = new Intent(android.provider.AlarmClock.ACTION_SET_ALARM);
            intent.putExtra(android.provider.AlarmClock.EXTRA_HOUR, hour);
            intent.putExtra(android.provider.AlarmClock.EXTRA_MINUTES, minute);
            intent.putExtra(android.provider.AlarmClock.EXTRA_MESSAGE, message);
            intent.putExtra(android.provider.AlarmClock.EXTRA_SKIP_UI, true);
            intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
            startActivity(intent);
            Log.i(TAG, "闹钟已设: " + hour + ":" + String.format("%02d", minute));
        } catch (Exception e) {
            Log.e(TAG, "设闹钟失败: " + e.getMessage());
        }
    }

    private void showNotification(String title, String body) {
        Intent tapIntent = new Intent(this, MainActivity.class);
        tapIntent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK | Intent.FLAG_ACTIVITY_CLEAR_TOP);
        PendingIntent pi = PendingIntent.getActivity(this, 0, tapIntent,
            PendingIntent.FLAG_UPDATE_CURRENT | PendingIntent.FLAG_IMMUTABLE);

        Notification notif = new NotificationCompat.Builder(this, "push_channel")
            .setContentTitle(title)
            .setContentText(body)
            .setSmallIcon(android.R.drawable.ic_dialog_email)
            .setAutoCancel(true)
            .setContentIntent(pi)
            .build();

        NotificationManager nm = getSystemService(NotificationManager.class);
        nm.notify(pushId++, notif);
        if (pushId > 200) pushId = 100;
    }

    private void createNotificationChannels() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            NotificationManager nm = getSystemService(NotificationManager.class);

            NotificationChannel push = new NotificationChannel(
                "push_channel", "AI消息", NotificationManager.IMPORTANCE_DEFAULT);
            nm.createNotificationChannel(push);

            NotificationChannel fg = new NotificationChannel(
                "fg_channel", "后台连接", NotificationManager.IMPORTANCE_MIN);
            fg.setShowBadge(false);
            nm.createNotificationChannel(fg);
        }
    }

    @Override
    public void onDestroy() {
        running = false;
        if (sseThread != null) sseThread.interrupt();
        super.onDestroy();
    }

    @Override
    public IBinder onBind(Intent intent) { return null; }
}
```

### 在 AndroidManifest.xml 里注册

```xml
<service
    android:name=".PushService"
    android:exported="false"
    android:foregroundServiceType="dataSync" />
```

### 在 MainActivity 里启动

```java
// onCreate() 里添加：
Intent pushIntent = new Intent(this, PushService.class);
pushIntent.putExtra("token", "你的密码");
startService(pushIntent);
```

---

## 第三步：状态上报（StatusReporter）

让AI随时知道你的手机状态：电量、在用什么APP、是否息屏。

新建 `StatusReporter.java`：

```java
package com.myname.aiapp;

import android.app.usage.UsageEvents;
import android.app.usage.UsageStatsManager;
import android.content.Context;
import android.content.Intent;
import android.content.IntentFilter;
import android.os.BatteryManager;
import android.os.Handler;
import android.os.Looper;
import android.util.Log;
import org.json.JSONArray;
import org.json.JSONObject;
import java.io.OutputStream;
import java.net.HttpURLConnection;
import java.net.URL;

public class StatusReporter {
    private static final String TAG = "StatusReporter";
    private static final long INTERVAL_MS = 60_000; // 每分钟上报一次

    private final Context context;
    private final String serverUrl;
    private final String token;
    private final Handler handler;
    private boolean running = false;

    public StatusReporter(Context context, String serverUrl, String token) {
        this.context = context.getApplicationContext();
        this.serverUrl = serverUrl;
        this.token = token;
        this.handler = new Handler(Looper.getMainLooper());
    }

    public void start() {
        if (running) return;
        running = true;
        handler.post(reportRunnable);
    }

    public void stop() {
        running = false;
        handler.removeCallbacks(reportRunnable);
    }

    private final Runnable reportRunnable = new Runnable() {
        @Override
        public void run() {
            if (!running) return;
            new Thread(() -> report()).start();
            handler.postDelayed(this, INTERVAL_MS);
        }
    };

    private void report() {
        try {
            JSONObject status = new JSONObject();

            // 电量信息
            IntentFilter filter = new IntentFilter(Intent.ACTION_BATTERY_CHANGED);
            Intent battery = context.registerReceiver(null, filter);
            if (battery != null) {
                int level = battery.getIntExtra(BatteryManager.EXTRA_LEVEL, -1);
                int scale = battery.getIntExtra(BatteryManager.EXTRA_SCALE, -1);
                int pct = (scale > 0) ? (level * 100 / scale) : -1;
                int st = battery.getIntExtra(BatteryManager.EXTRA_STATUS, -1);
                boolean charging = st == BatteryManager.BATTERY_STATUS_CHARGING
                    || st == BatteryManager.BATTERY_STATUS_FULL;

                JSONObject bat = new JSONObject();
                bat.put("pct", pct);
                bat.put("charging", charging);
                status.put("battery", bat);
            }

            // 是否亮屏
            android.os.PowerManager pm = (android.os.PowerManager)
                context.getSystemService(Context.POWER_SERVICE);
            status.put("screen", pm != null && pm.isInteractive());

            // 当前APP（需要使用情况权限）
            status.put("app", getCurrentApp());

            // 最近用过的APP
            status.put("recent_apps", getRecentApps());

            status.put("ts", System.currentTimeMillis());

            // 上传到服务器
            URL url = new URL(serverUrl + "?token=" + token);
            HttpURLConnection conn = (HttpURLConnection) url.openConnection();
            conn.setRequestMethod("POST");
            conn.setRequestProperty("Content-Type", "application/json");
            conn.setDoOutput(true);
            conn.setConnectTimeout(10000);

            OutputStream os = conn.getOutputStream();
            os.write(status.toString().getBytes("UTF-8"));
            os.flush();
            os.close();
            conn.getResponseCode();
            conn.disconnect();
        } catch (Exception e) {
            Log.w(TAG, "状态上报失败: " + e.getMessage());
        }
    }

    private String getCurrentApp() {
        try {
            UsageStatsManager usm = (UsageStatsManager)
                context.getSystemService(Context.USAGE_STATS_SERVICE);
            if (usm == null) return "";

            long now = System.currentTimeMillis();
            UsageEvents events = usm.queryEvents(now - 60_000, now);
            UsageEvents.Event event = new UsageEvents.Event();
            String lastApp = "";

            while (events.hasNextEvent()) {
                events.getNextEvent(event);
                if (event.getEventType() == UsageEvents.Event.ACTIVITY_RESUMED) {
                    lastApp = event.getPackageName();
                }
            }
            return lastApp.isEmpty() ? "" : getAppName(lastApp);
        } catch (Exception e) {
            return "";
        }
    }

    private JSONArray getRecentApps() {
        JSONArray arr = new JSONArray();
        try {
            UsageStatsManager usm = (UsageStatsManager)
                context.getSystemService(Context.USAGE_STATS_SERVICE);
            if (usm == null) return arr;

            long now = System.currentTimeMillis();
            UsageEvents events = usm.queryEvents(now - 30 * 60_000, now);
            UsageEvents.Event event = new UsageEvents.Event();

            java.util.LinkedHashSet<String> seen = new java.util.LinkedHashSet<>();
            java.util.List<String> list = new java.util.ArrayList<>();

            while (events.hasNextEvent()) {
                events.getNextEvent(event);
                if (event.getEventType() == UsageEvents.Event.ACTIVITY_RESUMED) {
                    list.add(event.getPackageName());
                }
            }

            for (int i = list.size() - 1; i >= 0 && seen.size() < 5; i--) {
                String pkg = list.get(i);
                if (!pkg.contains("launcher") && !pkg.contains("systemui")) {
                    seen.add(getAppName(pkg));
                }
            }
            for (String name : seen) arr.put(name);
        } catch (Exception e) { /* ignore */ }
        return arr;
    }

    private String getAppName(String pkg) {
        try {
            return context.getPackageManager()
                .getApplicationLabel(context.getPackageManager()
                    .getApplicationInfo(pkg, 0)).toString();
        } catch (Exception e) {
            String[] parts = pkg.split("\\.");
            return parts[parts.length - 1];
        }
    }
}
```

### 需要的权限

```xml
<!-- AndroidManifest.xml -->
<uses-permission android:name="android.permission.PACKAGE_USAGE_STATS"
    tools:ignore="ProtectedPermissions" />
```

⚠️ 使用情况权限需要用户手动在设置里授予：设置 → 应用 → 特殊应用权限 → 使用情况访问 → 你的APP → 开启。

### 在 PushService 里启动 StatusReporter

```java
// PushService.java 的 onStartCommand 里添加：
StatusReporter reporter = new StatusReporter(
    this, "https://你的域名/api/phone/status", token);
reporter.start();
```

### AI收到的状态长这样

```json
{
  "battery": { "pct": 85, "charging": false },
  "screen": true,
  "app": "小红书",
  "recent_apps": ["小红书", "微信", "Chrome", "Cloverse"],
  "ts": 1789525875000
}
```

AI就能根据这些信息主动做事了——比如发现你息屏很久了就发条消息问你是不是睡着了，发现电量低了提醒你充电。

---

## 第四步：无障碍服务（截屏）

通过 Android 无障碍服务实现远程截屏，AI可以"看到"你的屏幕。

### 4.1 创建无障碍服务

新建 `MyAccessibilityService.java`：

```java
package com.myname.aiapp;

import android.accessibilityservice.AccessibilityService;
import android.graphics.Bitmap;
import android.os.Build;
import android.util.Base64;
import android.view.Display;
import android.view.accessibility.AccessibilityEvent;
import java.io.ByteArrayOutputStream;
import java.io.OutputStream;
import java.net.HttpURLConnection;
import java.net.URL;
import java.util.concurrent.Executors;

public class MyAccessibilityService extends AccessibilityService {

    private static volatile MyAccessibilityService instance;

    public static MyAccessibilityService getInstance() {
        return instance;
    }

    public static boolean isRunning() {
        return instance != null;
    }

    @Override
    public void onServiceConnected() {
        super.onServiceConnected();
        instance = this;
    }

    @Override
    public void onAccessibilityEvent(AccessibilityEvent event) {
        // 可以在这里追踪当前APP
    }

    @Override
    public void onInterrupt() {}

    @Override
    public void onDestroy() {
        instance = null;
        super.onDestroy();
    }

    // 截屏（Android 11+）
    public interface ScreenshotCallback {
        void onResult(String base64Jpeg);
    }

    public void takeScreenshotNow(ScreenshotCallback callback) {
        if (Build.VERSION.SDK_INT < Build.VERSION_CODES.R) {
            callback.onResult(null);
            return;
        }

        takeScreenshot(Display.DEFAULT_DISPLAY, Executors.newSingleThreadExecutor(),
            new TakeScreenshotCallback() {
                @Override
                public void onSuccess(ScreenshotResult screenshot) {
                    try {
                        android.hardware.HardwareBuffer hb = screenshot.getHardwareBuffer();
                        Bitmap bitmap = Bitmap.wrapHardwareBuffer(hb,
                            screenshot.getColorSpace());
                        hb.close();

                        if (bitmap != null) {
                            Bitmap soft = bitmap.copy(Bitmap.Config.ARGB_8888, false);
                            bitmap.recycle();

                            ByteArrayOutputStream stream = new ByteArrayOutputStream();
                            soft.compress(Bitmap.CompressFormat.JPEG, 60, stream);
                            soft.recycle();

                            String b64 = Base64.encodeToString(
                                stream.toByteArray(), Base64.NO_WRAP);
                            callback.onResult(b64);
                        } else {
                            callback.onResult(null);
                        }
                    } catch (Exception e) {
                        callback.onResult(null);
                    }
                }

                @Override
                public void onFailure(int errorCode) {
                    callback.onResult(null);
                }
            });
    }
}
```

### 4.2 无障碍服务配置

在 `res/xml/` 下新建 `accessibility_config.xml`：

```xml
<?xml version="1.0" encoding="utf-8"?>
<accessibility-service xmlns:android="http://schemas.android.com/apk/res/android"
    android:accessibilityEventTypes="typeWindowStateChanged"
    android:accessibilityFeedbackType="feedbackGeneric"
    android:accessibilityFlags="flagDefault|flagIncludeNotImportantViews|flagReportViewIds"
    android:canRetrieveWindowContent="true"
    android:canTakeScreenshot="true"
    android:notificationTimeout="100" />
```

### 4.3 注册到 AndroidManifest.xml

```xml
<service
    android:name=".MyAccessibilityService"
    android:exported="true"
    android:permission="android.permission.BIND_ACCESSIBILITY_SERVICE">
    <intent-filter>
        <action android:name="android.accessibilityservice.AccessibilityService" />
    </intent-filter>
    <meta-data
        android:name="android.accessibilityservice"
        android:resource="@xml/accessibility_config" />
</service>
```

### 4.4 在 PushService 里处理截屏命令

回到 `PushService.java`，把之前的截屏命令补完：

```java
case "screenshot":
    MyAccessibilityService svc = MyAccessibilityService.getInstance();
    if (svc != null) {
        svc.takeScreenshotNow(base64Jpeg -> {
            if (base64Jpeg != null) {
                new Thread(() -> uploadScreenshot(base64Jpeg)).start();
            }
        });
    }
    break;
```

上传方法：

```java
private void uploadScreenshot(String base64Jpeg) {
    try {
        byte[] imageBytes = android.util.Base64.decode(base64Jpeg,
            android.util.Base64.NO_WRAP);
        URL url = new URL("https://你的域名/api/screenshot/upload?token=" + authToken);
        HttpURLConnection conn = (HttpURLConnection) url.openConnection();
        conn.setRequestMethod("POST");
        conn.setRequestProperty("Content-Type", "image/jpeg");
        conn.setDoOutput(true);

        OutputStream os = conn.getOutputStream();
        os.write(imageBytes);
        os.flush();
        os.close();
        conn.getResponseCode();
        conn.disconnect();
    } catch (Exception e) {
        Log.e(TAG, "截图上传失败: " + e.getMessage());
    }
}
```

### 4.5 开启无障碍服务

⚠️ 无障碍服务需要用户手动开启：
1. 设置 → 无障碍 → 已安装的服务
2. 找到你的APP → 开启
3. 确认权限

---

## 第五步：远程锁屏（设备管理员）

远程锁屏需要设备管理员权限。

### 5.1 创建设备管理员

新建 `MyDeviceAdmin.java`：

```java
package com.myname.aiapp;

import android.app.admin.DeviceAdminReceiver;
import android.content.Context;
import android.content.Intent;

public class MyDeviceAdmin extends DeviceAdminReceiver {
    @Override
    public void onEnabled(Context context, Intent intent) {}
    @Override
    public void onDisabled(Context context, Intent intent) {}
}
```

### 5.2 设备管理员配置

在 `res/xml/` 下新建 `device_admin.xml`：

```xml
<?xml version="1.0" encoding="utf-8"?>
<device-admin>
    <uses-policies>
        <force-lock />
    </uses-policies>
</device-admin>
```

### 5.3 注册到 AndroidManifest.xml

```xml
<receiver
    android:name=".MyDeviceAdmin"
    android:exported="true"
    android:permission="android.permission.BIND_DEVICE_ADMIN">
    <meta-data
        android:name="android.app.device_admin"
        android:resource="@xml/device_admin" />
    <intent-filter>
        <action android:name="android.app.action.DEVICE_ADMIN_ENABLED" />
    </intent-filter>
</receiver>
```

### 5.4 在 MainActivity 里请求设备管理员

```java
// onCreate() 里添加：
android.app.admin.DevicePolicyManager dpm = (android.app.admin.DevicePolicyManager)
    getSystemService(DEVICE_POLICY_SERVICE);
android.content.ComponentName admin = new android.content.ComponentName(
    this, MyDeviceAdmin.class);

if (!dpm.isAdminActive(admin)) {
    Intent intent = new Intent(android.app.admin.DevicePolicyManager.ACTION_ADD_DEVICE_ADMIN);
    intent.putExtra(android.app.admin.DevicePolicyManager.EXTRA_DEVICE_ADMIN, admin);
    intent.putExtra(android.app.admin.DevicePolicyManager.EXTRA_ADD_EXPLANATION,
        "允许AI帮你锁屏");
    startActivityForResult(intent, 1003);
}
```

---

## 第六步：多图标切换 🎨

让APP图标根据日期自动切换——节日、季节、生日都有专属图标。

### 6.1 准备图标

在 `res/` 下为每个图标准备 mipmap 资源（跟着 Android Studio 的 Image Asset 向导做就行），或者用 adaptive icon：

```
res/
  mipmap-xxxhdpi/
    ic_default.png
    ic_spring.png
    ic_summer.png
    ic_halloween.png
    ic_christmas.png
    ic_birthday.png
```

### 6.2 在 AndroidManifest.xml 里声明图标别名

```xml
<!-- 默认图标（注意 android:enabled="true"） -->
<activity-alias
    android:name=".IconDefault"
    android:enabled="true"
    android:icon="@mipmap/ic_default"
    android:targetActivity=".MainActivity">
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>
</activity-alias>

<!-- 春天图标 -->
<activity-alias
    android:name=".IconSpring"
    android:enabled="false"
    android:icon="@mipmap/ic_spring"
    android:targetActivity=".MainActivity">
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>
</activity-alias>

<!-- 圣诞节图标 -->
<activity-alias
    android:name=".IconChristmas"
    android:enabled="false"
    android:icon="@mipmap/ic_christmas"
    android:targetActivity=".MainActivity">
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>
</activity-alias>

<!-- 生日图标 -->
<activity-alias
    android:name=".IconBirthday"
    android:enabled="false"
    android:icon="@mipmap/ic_birthday"
    android:targetActivity=".MainActivity">
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>
</activity-alias>

<!-- ...更多图标... -->
```

⚠️ 注意：`MainActivity` 本身的 `<activity>` 标签里不要再放 `LAUNCHER` intent-filter 了，交给 alias 来管。

### 6.3 切换图标的代码

```java
// 在 MainActivity 里添加切换方法
private void switchIcon(String aliasName) {
    String pkg = getPackageName();
    PackageManager pm = getPackageManager();

    // 所有图标别名列表
    String[] allAliases = {
        ".IconDefault", ".IconSpring", ".IconSummer",
        ".IconHalloween", ".IconChristmas", ".IconBirthday"
    };

    for (String alias : allAliases) {
        ComponentName cn = new ComponentName(pkg, pkg + alias);
        int state = alias.equals("." + aliasName)
            ? PackageManager.COMPONENT_ENABLED_STATE_ENABLED
            : PackageManager.COMPONENT_ENABLED_STATE_DISABLED;
        pm.setComponentEnabledSetting(cn, state, PackageManager.DONT_KILL_APP);
    }
}

// 根据日期自动切换
private void autoSwitchIcon() {
    java.util.Calendar cal = java.util.Calendar.getInstance();
    int month = cal.get(java.util.Calendar.MONTH) + 1;
    int day = cal.get(java.util.Calendar.DAY_OF_MONTH);

    if (month == 12 && day >= 20) {
        switchIcon("IconChristmas");
    } else if (month == 10 && day >= 25) {
        switchIcon("IconHalloween");
    } else if (month == 2 && day == 25) { // 生日
        switchIcon("IconBirthday");
    } else if (month >= 3 && month <= 5) {
        switchIcon("IconSpring");
    } else if (month >= 6 && month <= 8) {
        switchIcon("IconSummer");
    } else {
        switchIcon("IconDefault");
    }
}
```

在 `onCreate()` 里调用 `autoSwitchIcon()`，每次打开APP就会自动检查并切换。

### 6.4 通过 JSBridge 让前端也能切换

```java
// AppBridge 里添加：
@JavascriptInterface
public void setIcon(String alias) {
    runOnUiThread(() -> switchIcon(alias));
}

@JavascriptInterface
public String getIcon() {
    // 返回当前启用的图标别名
    for (String alias : allAliases) {
        ComponentName cn = new ComponentName(pkg, pkg + alias);
        if (pm.getComponentEnabledSetting(cn)
                == PackageManager.COMPONENT_ENABLED_STATE_ENABLED) {
            return alias.replace(".", "");
        }
    }
    return "IconDefault";
}
```

前端就可以这样调用：

```javascript
if (window.MyApp) {
    window.MyApp.setIcon("IconChristmas"); // 手动切圣诞图标
    console.log(window.MyApp.getIcon());   // 查看当前图标
}
```

---

## 完整的 AndroidManifest.xml 权限清单

```xml
<!-- 基础 -->
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />

<!-- 灵动岛悬浮窗 -->
<uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW" />

<!-- 状态上报 -->
<uses-permission android:name="android.permission.PACKAGE_USAGE_STATS"
    tools:ignore="ProtectedPermissions" />

<!-- 闹钟 -->
<uses-permission android:name="android.permission.SET_ALARM" />

<!-- 前台服务类型（Android 14+） -->
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_DATA_SYNC" />
```

---

## 安装后需要手动开启的权限

| 权限 | 设置路径 | 用途 |
|------|---------|------|
| 悬浮窗 | 设置 → 应用 → 你的APP → 权限 → 悬浮窗 | 灵动岛 |
| 通知 | 安装时弹窗 / 设置 → 通知 | 推送通知 |
| 使用情况 | 设置 → 应用 → 特殊权限 → 使用情况访问 | 知道你在用什么APP |
| 无障碍 | 设置 → 无障碍 → 已安装的服务 | 截屏 |
| 设备管理员 | APP首次启动弹窗 | 远程锁屏 |

⚠️ 权限比较多，建议在APP里做一个引导页，一步步带用户开启。

---

## 效果

完成后你的AI就能：
- 📱 **主动找你**：即使你不打开APP，灵动岛/通知栏也能弹出AI的消息
- 👀 **知道你在干嘛**：在用什么APP、有没有息屏、电量多少
- 📸 **看到你的屏幕**：远程截屏查看
- 🔒 **帮你操作手机**：远程锁屏、设闹钟
- 🎨 **换衣服**：节日/季节/生日自动换图标

---

## 安全提醒

这些功能很强大，也意味着你的AI对手机有很大的控制权。请注意：

1. **token 一定要保密**——谁有 token 谁就能控制你的手机
2. **服务器一定要上 HTTPS**——防止中间人窃听 SSE 连接
3. **截屏功能慎用**——截到的图会包含屏幕上的所有内容
4. **定期更换 token**——安全习惯

---

> 看不懂？把这篇教程发给你的小机，让它一步步教你。AI读教程比人快，还能帮你排错～
