# Nothing Phone (3a) & (3a) Pro (Asteroids) — SukiSU Root & Seamless OTA Guide

A verified, step-by-step guide for rooting the Nothing Phone (3a) / (3a) Pro (Codename: `Asteroids` / `AsteroidsPro`, Model: `A059P`) with SukiSU / SukiSU Ultra (KernelSU-based LKM), surviving official Nothing OS OTA updates, and avoiding partition verification bricks.

---

## 1. Device Architecture & Critical Rules

### Target Partitions
* **Target Partition:** `init_boot` (8 MB).
* Modern Android (13+) uses GKI 2.0. The bootloader ramdisk containing `init` and root hooks is strictly inside `init_boot`, NOT `boot`.
* **DO NOT** flash or patch `boot.img` for root. Flashing patched images to `boot` will produce an unbootable state or bootloop.
* **DO NOT** use `--disable-verity` or `--disable-verification` when flashing. Nothing OS employs AVB 2.0 with hardware key validation; touching `vbmeta` with disable flags triggers an immediate red-screen DM-Verity verification failure.

### Slot System (A/B)
* The device utilizes seamless A/B updates.
* When flashing via fastboot, running `fastboot flash init_boot <file.img>` automatically targets the currently active slot (`init_boot_a` or `init_boot_b`). Manually specifying suffixes is generally unnecessary unless recovering an inactive slot.

---

## 2. Essential Links & Downloads

* **Firmware Dumps & Boot Images:**
  * [Nothing Archive (GitHub)](https://github.com/spike0en/nothing_archive) — Official stock firmware payload extractions.
  * [Nothing Archive Web](https://nothingarchive.tech) — Direct image-boot archives (`boot.img`, `init_boot.img`, `dtbo.img`, `vbmeta.img`).
* **Root Solutions (KernelSU / SukiSU):**
  * [SukiSU Ultra (GitHub)](https://github.com/DarkEmpire78/SukiSU-Ultra) — KernelSU fork with extended feature set and LKM support.
  * [KernelSU-Next (GitHub)](https://github.com/rifsxd/KernelSU-Next) — Standalone KernelSU Next ecosystem.
* **Host PC Tools:**
  * Google Platform-Tools (Fastboot v34.0.0 or newer).
  * Universal ADB Driver (ClockworkMod / Google WinUSB) for Fastboot ID `VID_18D1&PID_D00D`.

---

## 3. Initial Rooting Process

### Step 1: Identify Build & Download Matching Stock Boot
Check your currently installed build string via ADB or Settings:
```bash
adb shell getprop ro.build.display.id
```
*(Example: `Asteroids_B4.1-260810-1153` or `Asteroids_C5.0-261023-2146`).*

From [Nothing Archive](https://github.com/spike0en/nothing_archive/releases), download the corresponding `Asteroids_<BUILD>-image-boot.7z` package. Extract `init_boot.img` from the archive.

### Step 2: Patch init_boot via SukiSU
1. Push the stock `init_boot.img` to your phone:
   ```bash
   adb push init_boot.img /sdcard/Download/init_boot_stock.img
   ```
2. Open the **SukiSU Ultra** app on the phone.
3. Tap **Install** -> **Select and patch a file**.
4. Select `/sdcard/Download/init_boot_stock.img`.
5. Once processing finishes, pull the patched output back to the PC:
   ```bash
   adb pull /sdcard/Download/kernelsu_patched_*.img ./kernelsu_patched.img
   ```

### Step 3: Flash Patched Image
1. Reboot into Fastboot mode:
   ```bash
   adb reboot bootloader
   ```
2. Verify communication and slot:
   ```bash
   fastboot devices
   fastboot getvar current-slot
   ```
3. Flash to `init_boot`:
   ```bash
   fastboot flash init_boot kernelsu_patched.img
   fastboot reboot
   ```
4. Verify root access after boot:
   ```bash
   adb shell su -c id
   # Output must display uid=0(root) gid=0(root) context=u:r:ksu:s0
   ```

---

## 4. Surviving Official OTA Updates (Step-by-Step)

When an official Nothing OS system update arrives, you cannot install it while `init_boot` is modified. Follow this zero-data-loss cycle.

### Phase A: Revert to Stock init_boot
1. Take the stock `init_boot.img` corresponding to your **currently installed** firmware (the one you are running before updating).
2. Reboot into Fastboot:
   ```bash
   adb reboot bootloader
   ```
3. Flash the clean stock image:
   ```bash
   fastboot flash init_boot stock_init_boot.img
   fastboot reboot
   ```
4. The system will boot into pure stock unrooted state. SukiSU app will report uninstalled.

### Phase B: Install Official OTA
1. Navigate to **Settings -> System -> System Update**.
2. Download and install the update normally.
3. Allow the system to reboot when prompted. It will switch to the updated slot (e.g. from slot `a` to `b`).
4. Ensure the phone finishes boot initialization on the new Nothing OS version.

### Phase C: Pre-Patch New Version & Re-Flash Root
1. Identify the new build ID:
   ```bash
   adb shell getprop ro.build.display.id
   ```
2. Download the new version's `image-boot.7z` from [Nothing Archive](https://github.com/spike0en/nothing_archive/releases).
3. Extract `init_boot.img` for the **new** build.
4. Push `init_boot.img` to the phone's Download folder.
5. In SukiSU Ultra, select and patch the new `init_boot.img`.
6. Transfer the newly generated `kernelsu_patched_*.img` back to the PC.
7. Reboot to bootloader:
   ```bash
   adb reboot bootloader
   ```
8. Confirm active slot (`fastboot getvar current-slot`) and flash:
   ```bash
   fastboot flash init_boot kernelsu_patched_new.img
   fastboot reboot
   ```

---

## 5. Troubleshooting & Known Pitfalls

### Android 15 (Nothing OS 3.0 / C5.0) SukiSU App Crash
* **Symptom:** After updating from Android 14 (B4.x) to Android 15 (C5.0), `su` works via ADB shell (`uid=0`), but the SukiSU UI crashes on launch or reports unsupported.
* **Cause:** SukiSU Manager versions prior to v4.3.0 crash on Android 15 due to `libksud.so` API incompatibility (tombstone segmentation fault).
* **Fix:** Update SukiSU Ultra Manager to **v4.3.0 (versionCode 41100)** or newer via `pm install -r`.

### Fastboot Not Detecting Phone (USB Hub Power Suspend)
* **Symptom:** Phone is in Fastboot mode, but `fastboot devices` returns empty while connected via a USB-C hub.
* **Cause:** Windows selective suspend / power-saving can place downstream USB hub controllers in a low-power D2/D3 state where Fastboot endpoints (`VID_18D1&PID_D00D`) fail enumeration.
* **Fix:** Connect directly to motherboard rear USB ports, or reset the USB hub stack via PowerShell:
  ```powershell
  pnputil /restart-device "USB\VID_1A40&PID_0101\<INSTANCE_ID>"
  pnputil /scan-devices
  ```

### Missing Fastboot Driver on Windows
* Hardware ID: `USB\VID_18D1&PID_D00D`.
* If marked with a yellow exclamation mark in Device Manager, install the [ClockworkMod Universal ADB Driver](https://github.com/koush/UniversalAdbDriver) via winget:
  ```powershell
  winget install ClockworkMod.UniversalADBDriver --silent
  ```

---

## License
MIT. Firmware binaries belong to Nothing Technology Limited. Patching and recovery documentation provided for maintenance and enthusiast use.
