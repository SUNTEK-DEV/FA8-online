# FA8 SDK API

Android SDK for SUNTEK FA8 / FA10 series devices. Provides system-level control over hardware peripherals, device configuration, display, network, and application lifecycle.

Confirm which peripherals a given board actually exposes against the hardware datasheet. Do not ship this JAR together with older packages (`FA8API-SUNTEK.jar`, `YNHAPI-*.jar`, `FA8API-*.jar`).

## Contents

| File | Description |
|------|-------------|
| `SUNAPI.jar` | Java SDK (`com.suntek.SUNAPI`) |
| `API-SUNTEK使用说明.docx` | API usage (Chinese Word) |
| `API-SUNTEK Instructions for Use.docx` | API usage (English Word) |

The Word guides may still mention `FA8API-SUNTEK.jar` / `com.suntek.SUNTEK`. The artifact in this folder is `SUNAPI.jar`; the entry class is `com.suntek.SUNAPI`. Replace `SUNTEK` with `SUNAPI` in those examples.

## Getting Started

Copy `SUNAPI.jar` into the Android project's `app/libs` folder:

```gradle
dependencies {
    implementation files('libs/SUNAPI.jar')
}
```

```java
import com.suntek.SUNAPI;
import com.suntek.StorageInfo;
import com.suntek.entity.IpConfig;

SUNAPI mAPI = SUNAPI.getInstance();
float version = mAPI.getApiVersion();
```

`SUNAPI.init(Context)` is available if the firmware requires an explicit init.

On-device checks: **Settings > User Settings**. The firmware menu may still show the old label; the test app is `FA8CommonApi` (`com.suntek.fa8`).

### Migration from `FA8API-SUNTEK.jar`

| Old | New |
|-----|-----|
| `FA8API-SUNTEK.jar` | `SUNAPI.jar` |
| `import com.suntek.SUNTEK` | `import com.suntek.SUNAPI` |
| `SUNTEK mAPI = SUNTEK.getInstance()` | `SUNAPI mAPI = SUNAPI.getInstance()` |
| `SUNTEK.GpioState` / other nested types | `SUNAPI.GpioState` / same nested names |
| `mAPI.feedWatchdog()` | `mAPI.feedWatchDog()` |
| `mAPI.wakeup()` | `mAPI.wakeUp()` |

## API Overview

### System Parameters

Most setters need `mAPI.reboot()` to take effect.

| Feature | Getter | Setter |
|---------|--------|--------|
| API version | `getApiVersion()` | — |
| Board model | `getBoardModel()` | — |
| Serial number | `getSerialNo()` | `setSerialNo(String)` |
| Ethernet MAC address | `getEthernetMAC()` | `setEthernetMAC(String)` |
| 4G module IMEI | `getIMEI()` | `setIMEI(String)` |
| Device model | `getProductModel()` | `setProductModel(String)` |
| Storage info | `getStorageInfos()` | — |
| Language | — | `updateLanguage(String language, String country)` |
| 4G module | `get4GModuleNames()` | `set4GModule(String)` |
| GPS module | `getGPSModuleNames()` | `setGPSModule(String, String, String)` |

Storage sizes are in KB. `StorageInfo` types: `TYPE_MEMORY`, `TYPE_LOCAL_STORAGE`, `TYPE_TFCARD`, `TYPE_USB_STORAGE`.

- **Boot logo** — `setBootLogo(String path)` (8-bit BMP)
- **Boot animation** — `setBootAnimation(String path)` (standard Android zip)
- **APK install policy** — `setInstallPackagePolicy(SUNAPI.InstallPackagePolicy)`
  - `ALLOW_INSTALL_TYPE_NORMAL` (`-1`) cancel policy
  - `ALLOW_INSTALL_TYPE_NOT_ALLOW` (`0`) whitelist only
  - `ALLOW_INSTALL_TYPE_PASSWORD` (`1`) password for non-whitelist
- **Camera** — `setCameraInfo(int, SUNAPI.CameraInfo)` (`facing` / `mirror` / `hdr` / `rotation`)

### OTA Upgrade

```java
mAPI.otaUpdate("/sdcard/update.zip");
```

The package must be named `update.zip` and placed on sdcard.

### APK Management

- **Auto-launch on boot** — `setBootLaunchApk(String packageName, boolean launch)`
- **Silent install** — `installApkSilently(String apkPath)` or `installApkSilently(String apkPath, String packageName, String className)` — pass `null` for the last two parameters to skip auto-launch
- **Silent uninstall** — `uninstallApkSilently(String packageName)`
- **App keep-alive** — `setAppKeepLive(String packageName, int keepAliveTimeSec)`
- **Foreground keep-alive** — `setForegroundAppKeepLive(String packageName, int keepAliveTimeSec)`
- **Clear keep-alive** — `removeAppKeepLive()` or call the setters with `""` / `1`

Keep-alive does not persist across reboots.

### Display

| Feature | Method |
|---------|--------|
| Screen rotation | `getScreenRotation(SUNAPI.ScreenType)` / `setScreenRotation(SUNAPI.ScreenType, SUNAPI.RotationDegree)` |
| Input rotation | `getInputRotation(SUNAPI.ScreenType)` / `setInputRotation(SUNAPI.ScreenType, SUNAPI.RotationDegree)` |
| Screen density | `getLcdDensity()` / `setLcdDensity(SUNAPI.LcdDensity)` — `DENSITY_140` / `160` / `240` / `320` |
| Screen on/off | `isScreenOn()` / `setScreenOnOff(boolean)` |
| LCD brightness | `setLcdBrightness(int)` |

`ScreenType.MAIN` is the main screen, `ScreenType.AUX` is the secondary screen. `isScreenOn()` only reflects `setScreenOnOff`; after `sleep()` / `wakeUp()` use `PowerManager.isInteractive()`.

### Watchdog

```java
mAPI.enableWatchdog(true);
mAPI.setWatchdogTimeout(15);   // seconds; platform may drift 1–2s
mAPI.feedWatchDog();           // feed about every 10 seconds
int timeout = mAPI.getWatchdogTimeout();
mAPI.enableWatchdog(false);
```

If not fed before timeout, the device resets.

### Power Management

- **Shutdown** — `mAPI.shutdown()`
- **Reboot** — `mAPI.reboot()`
- **Sleep** — `mAPI.sleep()`
- **Wake up** — `mAPI.wakeUp()`

### Navigation Bar & Status Bar

Navigation bar — `setNavigationBarVisibility(SUNAPI.NavigationBarVisibility)`:

| Value | Meaning |
|-------|---------|
| `VISIBLE` | shown |
| `INVISIBLE` | hidden, swipeable |
| `ALWAYS_INVISIBLE` | hidden, not swipeable |
| `*_FOREVER` variants | persist across reboot |

Status bar — `setExtendStatusBarVisibility(SUNAPI.ExtendStatusBarVisibility)` — reboot required:

- `VISIBLE_NOT_EXPAND` / `VISABLE_EXPAND` / `INVISIBLE`
- matching `*_FOREVER` variants

(`VISABLE_EXPAND` is the spelling in the JAR.)

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
- **IP mode** — `getIpMode()` returns `SUNAPI.IpMode.STATIC` or `SUNAPI.IpMode.DHCP`
- **Ethernet switch** — `isEthernetOpen()` / `setEthernetState(boolean)`

### System Time

- **Set time** — `setSystemTime(long timeInMills)`
- **One-shot power on/off** — `setPowerOnOffAlarm(int[] poweroff, int[] poweron)`
- **Scheduled power on/off** — `setPowerOnOffAlarmCycle(int type, int[] weekdays, int[] poweroff, int[] poweron)`
  - `type=1`: one-time
  - `type=3`: weekly (re-apply after factory reset / flash)
- **Cancel schedule** — `cancelPowerOnOffAlarm()`
- **Network time sync** — `isEnableNetworkProvidedTime()` / `setEnableNetworkProvidedTime(boolean)`

### Hardware — GPIO

General IO, relays, fill lights (red/green/blue/white/infrared), USB power, cash box, doorbell, onboard LEDs, PCIe net power. Actual pins depend on the board.

| Constant | Value | Meaning |
|----------|-------|---------|
| `GPIO_1` … `GPIO_15` | 0–14 | General I/O |
| `LIGHT_RED` / `GREEN` / `BLUE` / `WHITE` | 15–18 | Fill light |
| `LIGHT_INFRARED` | 19 | USB camera IR fill |
| `RELAY` | 20 | Relay |
| `BELL` | 21 | Doorbell |
| `CASHBOX_0` / `CASHBOX_1` | 22 / 23 | Cash box |
| `LIGHT_BOARD_BLUE` / `RED` / `YELLOW` | 24 / 28 / 30 | Onboard LEDs |
| `USB_1` / `USB_2` / `USB_3` | 25 / 26 / 27 | USB power |
| `LIGHT_CAMERA_EXTERNAL_INFRARED` | 29 | MIPI camera IR fill |
| `PCIE_NET_POWER` | 31 | PCIe net power |
| `USB_4` … `USB_10` | 32–38 | Extra USB power |

| Operation | Method |
|-----------|--------|
| Get GPIO state | `getGpioState(int index)` |
| Set GPIO state | `setGpioState(int index, SUNAPI.GpioState)` |
| Get GPIO mode | `getGpioMode(int index)` |
| Set GPIO mode | `setGpioMode(int index, SUNAPI.GpioMode)` |
| Listen | `listenGpio(int index, SUNAPI.GpioListenerCallback)` |
| Unlisten | `unlistenGpio(int index, SUNAPI.GpioListenerCallback)` |

Overloads that take `(char, int)` exist for bank/pin addressing. Listener polls about once per second — not for sub-second button presses.

Typed helpers (static):

- Lights — `SUNAPI.setLightState(SUNAPI.Light, boolean)` / `getLightState` / `setLightBrightness(SUNAPI.Light, int)`
- USB — `SUNAPI.setUsbState(SUNAPI.Usb, boolean)` / `getUsbState` (`Usb_1` … `Usb_3`)
- Device power — `SUNAPI.setDeviceState(SUNAPI.Device, boolean)` / `getDeviceState` (`Device_5V`, `Device_12V`, `Device_Relay`)

### Hardware — LED Brightness

```java
mAPI.setLightBrightness(SUNAPI.LIGHT_RED, 204); // 80%, range 0-255
```

### Hardware — Wiegand

Instance API (same pattern as the Word guide):

- **Mode** — `readWiegandMode()` / `writeWiegandMode(SUNAPI.WiegandMode)`
- **Synchronous read** — `readWiegand()` (blocking, use a background thread)
- **Asynchronous read** — `readWiegandAsyn(SUNAPI.WiegandCallback)` — call again in `onSuccess` / `onFailure` for continuous reading
- **Write** — `writeWiegand(SUNAPI.WiegandFormat, long code)` (`FORMAT_26` / `FORMAT_34`)

Static helpers also exist: `openWiegand` / `closeWiegand` / `setWiegandMode` / `readWiegand` / `readWiegandAsync` / `writeWiegand` with `SUNAPI.Wiegand` (`Wiegand_Input`, `Wiegand_Output26`, `Wiegand_Output34`).

## Reference Code

Cash box — pull low for 500ms, then high:

```java
CompletableFuture.runAsync(() -> {
    mAPI.setGpioState(SUNAPI.CASHBOX_0, SUNAPI.GpioState.LOW);
    try { TimeUnit.MILLISECONDS.sleep(500); } catch (InterruptedException e) { }
}).thenAccept(u -> {
    mAPI.setGpioState(SUNAPI.CASHBOX_0, SUNAPI.GpioState.HIGH);
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
