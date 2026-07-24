# FA8 SDK API

Android SDK for SUNTEK FA8 series devices, providing system-level control over hardware peripherals, device configuration, display settings, network management, and application lifecycle.

## Contents

- `API-en_20251224.pdf` — API reference documentation (English)
- `API-ZH_20251224.pdf` — API reference documentation (Chinese)
- `FA8API-20251224.jar` — Java SDK library (`SUNTEK.jar`)

## Getting Started

Add `FA8API-20251224.jar` to your Android project's `libs` folder and include it as a dependency. Then obtain the singleton API instance:

```java
SUNTEK mAPI = SUNTEK.getInstance();
```

You can verify API functionality on-device by navigating to **Settings > User Settings** (or **Settings > User Settings > YNHCommonAPI**) and selecting the corresponding feature.

## API Overview

### System Parameters

Get and set core device properties. Most setters require a reboot (`mAPI.reboot()`) to take effect.

| Feature | Getter | Setter |
|---------|--------|--------|
| Board model | `getBoardModel()` | — |
| Serial number | `getSerialNo()` | `setSerialNo(String)` |
| Ethernet MAC address | `getEthernetMAC()` | `setEthernetMAC(String)` |
| 4G module IMEI | `getIMEI()` | `setIMEI(String)` |
| Device model | `getProductModel()` | `setProductModel(String)` |
| Storage info | `getStorageInfos()` | — |
| Language | — | `updateLanguage(String language, String country)` |

Additional system configuration:

- **Boot logo** — `setBootLogo(String path)` (8-bit BMP format)
- **Boot animation** — `setBootAnimation(String path)` (standard Android zip)
- **APK install whitelist** — `setInstallPackagePolicy(InstallPackagePolicy)` with support for normal, deny-all, and password-gated installation modes

### OTA Upgrade

```java
mAPI.otaUpdate("/sdcard/update.zip");
```

The OTA package must be placed in the sdcard directory and named `update.zip`.

### APK Management

- **Auto-launch on boot** — `setBootLaunchApk(String packageName, boolean launch)`
- **Silent install** — `installApkSilently(String apkPath, String packageName, String className)` — pass `null` for the last two parameters to skip auto-launch after installation
- **Silent uninstall** — `uninstallApkSilently(String packageName)`
- **App keep-alive (background guardian)** — `setAppKeepLive(String packageName, int keepAliveTimeSec)`
- **Foreground app keep-alive** — `setForegroundAppKeepLive(String packageName, int keepAliveTimeSec)`

Keep-alive settings do not persist across reboots.

### Display

| Feature | Method |
|---------|--------|
| Screen rotation (get/set) | `getScreenRotation(ScreenType)` / `setScreenRotation(ScreenType, RotationDegree)` |
| Screen density (get/set) | `getLcdDensity()` / `setLcdDensity(LcdDensity)` |
| Screen on/off status | `isScreenOn()` / `setScreenOnOff(boolean)` |

Supports both main screen (`ScreenType.MAIN`) and secondary screen (`ScreenType.AUX`).

### Watchdog

```java
mAPI.enableWatchdog(true);       // Enable
mAPI.setWatchdogTimeout(15);     // Timeout in seconds
mAPI.feedWatchdog();             // Feed (recommended every ~10s)
```

If the watchdog is not fed within the timeout period, the device will automatically reset.

### Power Management

- **Shutdown** — `mAPI.shutdown()`
- **Reboot** — `mAPI.reboot()`
- **Sleep** — `mAPI.sleep()`
- **Wake up** — `mAPI.wakeup()`

### Navigation Bar & Status Bar

- **Navigation bar visibility** — `setNavigationBarVisibility(NavigationBarVisibility)` with modes: `VISIBLE`, `INVISIBLE` (swipeable), `ALWAYS_INVISIBLE` (non-swipeable)
- **Status bar visibility** — `setExtendStatusBarVisibility(ExtendStatusBarVisibility)` — requires a reboot to take effect

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
- **IP mode** — `getIpMode()` returns `IpMode.STATIC` or `IpMode.DHCP`
- **Ethernet switch** — `isEthernetOpen()` / `setEthernetState(boolean)`

### System Time

- **Set time** — `setSystemTime(long timeInMills)`
- **Scheduled power on/off** — `setPowerOnOffAlarmCycle(int type, int[] timeOff, int[] timeOn)`
  - `type=1`: one-time schedule
  - `type=3`: weekly recurring schedule (with weekday array)
- **Cancel schedule** — `cancelPowerOnOffAlarm()`
- **Network time sync** — `isEnableNetworkProvidedTime()` / `setEnableNetworkProvidedTime(boolean)`

### Hardware — GPIO

Supports up to 30 GPIO channels including general-purpose IO pins, relays, LED fill lights (red/green/blue/white/infrared), USB power switches, cash box triggers, doorbells, and onboard indicator LEDs.

| Operation | Method |
|-----------|--------|
| Get GPIO state | `getGpioState(int index)` |
| Set GPIO state | `setGpioState(int index, GpioState)` |
| Get GPIO mode | `getGpioMode(int index)` |
| Set GPIO mode | `setGpioMode(int index, GpioMode)` |
| Listen for changes | `listenGpio(int index, GpioListenerCallback)` |
| Cancel listener | `unlistenGpio(int index, GpioListenerCallback)` |

GPIO listener polls every 1 second — not suitable for detecting sub-second button presses.

### Hardware — LED Brightness

```java
mAPI.setLightBrightness(SUNTEK.LIGHT_RED, 204); // 80% brightness (range: 0-255)
```

### Hardware — Wiegand

- **Mode** — `readWiegandMode()` / `writeWiegandMode(WiegandMode)`
- **Synchronous read** — `readWiegand()` (blocking, use in a background thread)
- **Asynchronous read** — `readWiegandAsyn(WiegandCallback)` — re-register callback in `onSuccess`/`onFailure` for continuous reading
- **Write** — `writeWiegand(WiegandFormat, long code)`

## Reference Code Snippets

**Cash box trigger** — pull GPIO low for 500ms, then high:

```java
CompletableFuture.runAsync(() -> {
    mAPI.setGpioState(SUNTEK.CASHBOX_0, SUNTEK.GpioState.LOW);
    try { TimeUnit.MILLISECONDS.sleep(500); } catch (InterruptedException e) { }
}).thenAccept(u -> {
    mAPI.setGpioState(SUNTEK.CASHBOX_0, SUNTEK.GpioState.HIGH);
});
```

**Execute root shell command:**

```java
Process p = Runtime.getRuntime().exec("su");
PrintWriter pw = new PrintWriter(p.getOutputStream(), true);
pw.println("your_command_here");
pw.println("exit");
boolean success = p.waitFor() == 0;
```

## Version History

| Date | Changes |
|------|---------|
| 2025-12-24 | Added watchdog timeout configuration |
| 2025-03-10 | Added language setting interface |
| 2024-05-21 | Added APK installation whitelist policy |
| 2024-03-21 | Updated Wiegand async read interface |
| 2023-04-13 | Added separate screen display on/off control |
| 2022-11-07 | Added foreground app keep-alive interface |
| 2022-09-22 | Updated scheduled power on/off with multiple modes |
| 2022-08-30 | Added screen on/off status query |
| 2022-07-13 | Added boot logo, boot animation, and status bar settings |
| 2022-07-01 | Added APP guardian interface |
| 2022-06-30 | Added Ethernet switch and status query |
| 2022-06-25 | Added serial number, device model, MAC address, and IMEI interfaces |
| 2022-05-31 | Deprecated single-param `installApkSilently`; added auto-launch variant |
