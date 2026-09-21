# FA8 SDK API

Android SDK for SUNTEK FA8 / FA10 series devices. Provides system-level control over hardware peripherals, device configuration, display, network, and application lifecycle.

## Contents

- `FA8API-SUNTEK.jar` — Java SDK (`com.suntek.SUNTEK`)
- `API-SUNTEK使用说明.docx` — API usage (Word, new JAR)

## Getting Started

Copy `FA8API-SUNTEK.jar` into the Android project's `app/libs` folder:

```gradle
dependencies {
    implementation files('libs/FA8API-SUNTEK.jar')
}
```

```java
import com.suntek.SUNTEK;
import com.suntek.StorageInfo;
import com.suntek.entity.IpConfig;

SUNTEK mAPI = SUNTEK.getInstance();
```

On-device checks: **Settings > User Settings**. The firmware menu may still show the old label; the test app is `FA8CommonApi` (`com.suntek.fa8`).



## API Overview

### System Parameters

Most setters need `mAPI.reboot()` to take effect.

| Feature | Getter | Setter |
|---------|--------|--------|
| Board model | `getBoardModel()` | — |
| Serial number | `getSerialNo()` | `setSerialNo(String)` |
| Ethernet MAC address | `getEthernetMAC()` | `setEthernetMAC(String)` |
| 4G module IMEI | `getIMEI()` | `setIMEI(String)` |
| Device model | `getProductModel()` | `setProductModel(String)` |
| Storage info | `getStorageInfos()` | — |
| Language | — | `updateLanguage(String language, String country)` |

- **Boot logo** — `setBootLogo(String path)` (8-bit BMP)
- **Boot animation** — `setBootAnimation(String path)` (standard Android zip)
- **APK install policy** — `setInstallPackagePolicy(SUNTEK.InstallPackagePolicy)` (normal / deny-all / password)

### OTA Upgrade

```java
mAPI.otaUpdate("/sdcard/update.zip");
```

The package must be named `update.zip` and placed on sdcard.

### APK Management

- **Auto-launch on boot** — `setBootLaunchApk(String packageName, boolean launch)`
- **Silent install** — `installApkSilently(String apkPath, String packageName, String className)` — pass `null` for the last two parameters to skip auto-launch
- **Silent uninstall** — `uninstallApkSilently(String packageName)`
- **App keep-alive** — `setAppKeepLive(String packageName, int keepAliveTimeSec)`
- **Foreground keep-alive** — `setForegroundAppKeepLive(String packageName, int keepAliveTimeSec)`

Keep-alive does not persist across reboots.

### Display

| Feature | Method |
|---------|--------|
| Screen rotation | `getScreenRotation(SUNTEK.ScreenType)` / `setScreenRotation(SUNTEK.ScreenType, SUNTEK.RotationDegree)` |
| Screen density | `getLcdDensity()` / `setLcdDensity(SUNTEK.LcdDensity)` |
| Screen on/off | `isScreenOn()` / `setScreenOnOff(boolean)` |

`ScreenType.MAIN` is the main screen, `ScreenType.AUX` is the secondary screen.

### Watchdog

```java
mAPI.enableWatchdog(true);
mAPI.setWatchdogTimeout(15);
mAPI.feedWatchdog();
```

Feed about every 10 seconds. If not fed before timeout, the device resets.

### Power Management

- **Shutdown** — `mAPI.shutdown()`
- **Reboot** — `mAPI.reboot()`
- **Sleep** — `mAPI.sleep()`
- **Wake up** — `mAPI.wakeup()`

### Navigation Bar & Status Bar

- **Navigation bar** — `setNavigationBarVisibility(SUNTEK.NavigationBarVisibility)`: `VISIBLE`, `INVISIBLE` (swipeable), `ALWAYS_INVISIBLE`
- **Status bar** — `setExtendStatusBarVisibility(SUNTEK.ExtendStatusBarVisibility)` — reboot required

### Root Privileges

```java
if (!mAPI.isRoot()) {
    mAPI.enableRoot(true);
}
```

### Network (Ethernet)

- **IP configuration** — `getIpConfig()` returns `IpConfig` (ip, mask, gateway, dnsList)
- **Static IP** — `setStaticIp(IpConfig)`
- **DHCP** — `setDhcpIp()`
- **IP mode** — `getIpMode()` returns `SUNTEK.IpMode.STATIC` or `SUNTEK.IpMode.DHCP`
- **Ethernet switch** — `isEthernetOpen()` / `setEthernetState(boolean)`

### System Time

- **Set time** — `setSystemTime(long timeInMills)`
- **Scheduled power on/off** — `setPowerOnOffAlarmCycle(int type, int[] weekdays, int[] poweroff, int[] poweron)`
  - `type=1`: one-time
  - `type=3`: weekly
- **Cancel schedule** — `cancelPowerOnOffAlarm()`
- **Network time sync** — `isEnableNetworkProvidedTime()` / `setEnableNetworkProvidedTime(boolean)`

### Hardware — GPIO

General IO, relays, fill lights (red/green/blue/white/infrared), USB power, cash box, doorbell, onboard LEDs. Actual pins depend on the board.

| Operation | Method |
|-----------|--------|
| Get GPIO state | `getGpioState(int index)` |
| Set GPIO state | `setGpioState(int index, SUNTEK.GpioState)` |
| Get GPIO mode | `getGpioMode(int index)` |
| Set GPIO mode | `setGpioMode(int index, SUNTEK.GpioMode)` |
| Listen | `listenGpio(int index, SUNTEK.GpioListenerCallback)` |
| Unlisten | `unlistenGpio(int index, SUNTEK.GpioListenerCallback)` |

Listener polls about once per second — not for sub-second button presses.

### Hardware — LED Brightness

```java
mAPI.setLightBrightness(SUNTEK.LIGHT_RED, 204); // 80%, range 0-255
```

### Hardware — Wiegand

- **Mode** — `readWiegandMode()` / `writeWiegandMode(SUNTEK.WiegandMode)`
- **Synchronous read** — `readWiegand()` (blocking, use a background thread)
- **Asynchronous read** — `readWiegandAsyn(SUNTEK.WiegandCallback)` — call again in `onSuccess` / `onFailure` for continuous reading
- **Write** — `writeWiegand(SUNTEK.WiegandFormat, long code)`

## Reference Code

Cash box — pull low for 500ms, then high:

```java
CompletableFuture.runAsync(() -> {
    mAPI.setGpioState(SUNTEK.CASHBOX_0, SUNTEK.GpioState.LOW);
    try { TimeUnit.MILLISECONDS.sleep(500); } catch (InterruptedException e) { }
}).thenAccept(u -> {
    mAPI.setGpioState(SUNTEK.CASHBOX_0, SUNTEK.GpioState.HIGH);
});
```

Root shell:

```java
Process p = Runtime.getRuntime().exec("su");
PrintWriter pw = new PrintWriter(p.getOutputStream(), true);
pw.println("your_command_here");
pw.println("exit");
boolean success = p.waitFor() == 0;
```
