# Android Exporter

A Prometheus-compatible metrics exporter for Android devices. Exposes device metrics (battery, memory, storage, CPU, screen state) via HTTP on port 9100.

## Features

- **Battery Metrics**: Battery percentage, charging state, power source type (USB/AC/Wireless)
- **Screen State**: Screen on/off detection
- **CPU Usage**: Real-time CPU usage percentage
- **Memory**: Available and total RAM
- **Storage**: Total, free, and available storage
- **Device Labels**: Android model and version

## Prerequisites

- Android Studio (Arctic Fox or newer)
- Android device or emulator (API level 26+)
- Kotlin 1.8+

## Setup Instructions

### 1. Create New Android Project

1. Open Android Studio
2. Click **File → New → New Project**
3. Select **Empty Activity** (Compose)
4. Configure:
   - Name: `android_exporter`
   - Package name: `com.example.android_exporter`
   - Language: **Kotlin**
   - Minimum SDK: **API 26 (Android 8.0)**
5. Click **Finish**

### 2. Add Dependencies

Open `app/build.gradle.kts` and add NanoHTTPD dependency:

```kotlin
dependencies {
    implementation("org.nanohttpd:nanohttpd:2.3.1")
    
    // Existing dependencies
    implementation("androidx.core:core-ktx:1.12.0")
    implementation("androidx.activity:activity-compose:1.8.2")
    // ... other dependencies
}
```

Sync the project (**File → Sync Project with Gradle Files**).

### 3. Add Source Files

Create the following Kotlin files in `app/src/main/java/com/example/android_exporter/`:

#### MetricsServer.kt

```kotlin
package com.example.android_exporter

import android.app.ActivityManager
import android.content.*
import android.os.*
import fi.iki.elonen.NanoHTTPD
import java.io.RandomAccessFile

class MetricsServer(private val context: Context) : NanoHTTPD(9100) {

    override fun serve(session: IHTTPSession): Response {
        return when (session.uri) {
            "/metrics" -> newFixedLengthResponse(getMetrics())
            "/health" -> newFixedLengthResponse("ok")
            else -> newFixedLengthResponse(Response.Status.NOT_FOUND, MIME_PLAINTEXT, "404")
        }
    }

    private fun getMetrics(): String {
        val sb = StringBuilder()

        // Device labels
        sb.appendLine(
            """android_up{model="${Build.MODEL}",android_version="${Build.VERSION.RELEASE}"} 1"""
        )

        // Battery metrics
        val bm = context.getSystemService(Context.BATTERY_SERVICE) as BatteryManager
        val batteryPct = bm.getIntProperty(BatteryManager.BATTERY_PROPERTY_CAPACITY)

        val batteryIntent = context.registerReceiver(
            null, IntentFilter(Intent.ACTION_BATTERY_CHANGED)
        )

        val status = batteryIntent?.getIntExtra(BatteryManager.EXTRA_STATUS, -1)
        val isCharging =
            status == BatteryManager.BATTERY_STATUS_CHARGING ||
            status == BatteryManager.BATTERY_STATUS_FULL

        val chargePlug = batteryIntent?.getIntExtra(BatteryManager.EXTRA_PLUGGED, -1)

        val plugType = when (chargePlug) {
            BatteryManager.BATTERY_PLUGGED_USB -> "usb"
            BatteryManager.BATTERY_PLUGGED_AC -> "ac"
            BatteryManager.BATTERY_PLUGGED_WIRELESS -> "wireless"
            else -> "none"
        }

        sb.appendLine("android_battery_percent $batteryPct")
        sb.appendLine("android_charging ${if (isCharging) 1 else 0}")
        sb.appendLine("""android_power_source{type="$plugType"} 1""")

        // Screen state
        val pm = context.getSystemService(Context.POWER_SERVICE) as PowerManager
        sb.appendLine("android_screen_on ${if (pm.isInteractive) 1 else 0}")

        // Memory metrics
        val am = context.getSystemService(Context.ACTIVITY_SERVICE) as ActivityManager
        val mem = ActivityManager.MemoryInfo()
        am.getMemoryInfo(mem)

        sb.appendLine("android_memory_available_bytes ${mem.availMem}")
        sb.appendLine("android_memory_total_bytes ${mem.totalMem}")

        // Storage metrics
        val stat = StatFs(Environment.getDataDirectory().path)
        sb.appendLine("android_storage_total_bytes ${stat.totalBytes}")
        sb.appendLine("android_storage_free_bytes ${stat.freeBytes}")
        sb.appendLine("android_storage_available_bytes ${stat.availableBytes}")

        // CPU usage
        sb.appendLine("android_cpu_usage_percent ${readCpuUsage()}")

        return sb.toString()
    }

    private fun readCpuUsage(): Float {
        return try {
            val reader = RandomAccessFile("/proc/stat", "r")
            val load = reader.readLine()
            reader.close()

            val toks = load.split("\\s+".toRegex())
            val idle = toks[4].toLong()
            val total = toks.drop(1).sumOf { it.toLong() }

            100f * (total - idle) / total
        } catch (e: Exception) {
            0f
        }
    }
}
```

#### ExporterService.kt

```kotlin
package com.example.android_exporter

import android.app.*
import android.content.Intent
import android.os.*

class ExporterService : Service() {

    private lateinit var server: MetricsServer

    override fun onCreate() {
        super.onCreate()
        createNotificationChannel()

        val notification = Notification.Builder(this, "exporter")
            .setContentTitle("Android Exporter")
            .setContentText("Serving metrics on port 9100")
            .setSmallIcon(android.R.drawable.stat_notify_sync)
            .build()

        startForeground(1, notification)

        server = MetricsServer(this)
        server.start()
    }

    override fun onDestroy() {
        server.stop()
        super.onDestroy()
    }

    override fun onBind(intent: Intent?): IBinder? = null

    private fun createNotificationChannel() {
        if (Build.VERSION.SDK_INT >= 26) {
            val channel = NotificationChannel(
                "exporter",
                "Exporter",
                NotificationManager.IMPORTANCE_LOW
            )
            getSystemService(NotificationManager::class.java)
                .createNotificationChannel(channel)
        }
    }
}
```

#### MainActivity.kt

```kotlin
package com.example.android_exporter

import android.content.Intent
import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.compose.material3.*

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        startForegroundService(Intent(this, ExporterService::class.java))

        setContent {
            MaterialTheme {
                Text("Android Exporter running")
            }
        }
    }
}
```

### 4. Update AndroidManifest.xml

Open `app/src/main/AndroidManifest.xml` and add the required permissions and service:

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android">

    <uses-permission android:name="android.permission.INTERNET"/>
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE"/>
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE_DATA_SYNC"/>

    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="android_exporter"
        android:theme="@style/Theme.AndroidExporter">

        <activity
            android:name=".MainActivity"
            android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN"/>
                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>
        </activity>

        <service
            android:name=".ExporterService"
            android:foregroundServiceType="dataSync"
            android:exported="false"/>
    </application>
</manifest>
```

### 5. Build and Run

1. Connect your Android device via USB (with USB debugging enabled) or start an emulator
2. Click **Run** (green play button) in Android Studio
3. The app will install and start automatically
4. You should see "Android Exporter running" on the screen
5. A notification will appear showing the exporter is active

## Usage

### Access Metrics

Once the app is running, metrics are available at:

```
http://<device-ip>:9100/metrics
```

To find your device IP:
- Go to **Settings → About Phone → Status → IP Address**
- Or use `adb shell ip addr show wlan0`

### Test Locally (via ADB)

Forward the port to your development machine:

```bash
adb forward tcp:9100 tcp:9100
```

Then access metrics:

```bash
curl http://localhost:9100/metrics
```

### Health Check Endpoint

```bash
curl http://<device-ip>:9100/health
```

## Sample Metrics Output

```
android_up{model="Pixel 6",android_version="14"} 1
android_battery_percent 85
android_charging 1
android_power_source{type="usb"} 1
android_screen_on 1
android_memory_available_bytes 4294967296
android_memory_total_bytes 8589934592
android_storage_total_bytes 128849018880
android_storage_free_bytes 45678901234
android_storage_available_bytes 43210987654
android_cpu_usage_percent 23.5
```

## Prometheus Configuration

Add to your `prometheus.yml`:

```yaml
scrape_configs:
  - job_name: 'android'
    static_configs:
      - targets: ['<device-ip>:9100']
        labels:
          device: 'my-android-phone'
```

## Troubleshooting

### App crashes on startup
- Ensure minimum SDK is API 26+
- Check all permissions are added to AndroidManifest.xml

### Can't access metrics from network
- Verify device and Prometheus server are on same network
- Check firewall settings on the device
- Ensure the app is running (notification should be visible)

### Port already in use
- Change port in `MetricsServer.kt` (line with `NanoHTTPD(9100)`)
- Update Prometheus configuration accordingly

### CPU usage shows 0
- This is normal on some devices due to `/proc/stat` access restrictions
- The metric will still be exported, just with a 0 value

## License

MIT License - feel free to use and modify as needed.

## Contributing

Pull requests welcome! Areas for improvement:
- Network traffic metrics
- Temperature sensors
- App-specific metrics
- TLS/authentication support
