# Google Pixel 6 (`oriole`) — Magisk Re-Root Guide

This guide details how to restore Magisk root on a Google Pixel 6 after an update, system flash, or accidental root loss.

---

## Prerequisites

- **Unlocked Bootloader**: Ensure your OEM bootloader is already unlocked.  
- **Computer with Android Platform-Tools**:  
  - Download official [SDK Platform-Tools](https://developer.android.com/tools/releases/platform-tools).  
  - Verify terminal access to `adb` and `fastboot`.  
- **USB Cable**: Reliable USB-C cable for data transfer.  
- **Magisk App**: Installed on your Pixel 6\.

> \[\!NOTE\] **Partition Target for Pixel 6**: The Pixel 6 family (Tensor G1) patches the **`boot`** partition (`boot.img`), unlike the Pixel 7 and newer which use `init_boot`.

---

## Method 1: Re-Rooting via Patched `boot.img` (Standard / Lost Root)

Use this method if your phone has already rebooted and lost root access after an update or clean flash.

### Step 1: Identify Your Exact Build Number

1. On your Pixel 6, navigate to **Settings \> About phone**.  
2. Scroll to the bottom and note your exact **Build number** (e.g., `AP2A.240805.005` or similar).

### Step 2: Download the Matching Factory Image

1. Visit the official [Google Pixel Factory Images page](https://developers.google.com/android/images).  
2. Find the entry for **Pixel 6 ("oriole")**.  
3. Download the factory image `.zip` matching your exact build number.

### Step 3: Extract `boot.img`

1. Unzip the downloaded factory image archive.  
2. Inside the unzipped folder, locate the second inner zip file named `image-oriole-[build_number].zip`.  
3. Open this inner archive and extract **`boot.img`** to your computer.

### Step 4: Patch the Boot Image on Your Pixel 6

1. Transfer the extracted `boot.img` to your phone's internal storage (e.g., `Downloads` folder):

```sh
adb push boot.img /sdcard/Download/
```

2. Open the **Magisk** app on your phone.  
3. Tap **Install** on the Magisk card.  
4. Choose **Select and Patch a File**.  
5. Browse to your `Downloads` folder and select `boot.img`.  
6. Tap **LET'S GO**. Magisk will generate a patched file named:

```
magisk_patched-[random_hash].img
```

   (saved in your device's `Download/` folder).

### Step 5: Transfer the Patched Image to Your PC

Pull the patched image back to your computer terminal folder:

```sh
adb pull /sdcard/Download/magisk_patched-*.img .
```

### Step 6: Flash the Patched Boot Image

1. Reboot the phone into Bootloader (Fastboot) mode:

```sh
adb reboot bootloader
```

   *(Alternatively, power off the phone and hold **Power \+ Volume Down**).*

2. Verify communication with fastboot:

```sh
fastboot devices
```

3. Flash the patched boot image to the active boot partition:

```sh
fastboot flash boot magisk_patched-[random_hash].img
```

4. Reboot the phone:

```sh
fastboot reboot
```

### Step 7: Verify Root

Open the **Magisk** app. You should see Magisk reported as **Installed** along with the version number.

---

## Method 2: Seamless Re-Rooting During Monthly OTA Updates

Use this method before taking an official OTA update so you do not need to download the full factory image.

### Step 1: Restore Images in Magisk

1. Open **Magisk**.  
2. Tap **Uninstall Magisk** \> **Restore Images**.  
3. **DO NOT REBOOT** your phone.

### Step 2: Install the System OTA Update

1. Navigate to **Settings \> System \> System update**.  
2. Download and install the update.  
3. When the update finishes installing and displays a prompt to **Restart now**, **DO NOT RESTART**.

### Step 3: Install Magisk to Inactive Slot

1. Return to the **Magisk** app.  
2. Tap **Install** on the Magisk card.  
3. Select **Install to Inactive Slot (After OTA)**.  
4. Tap **LET'S GO**. Magisk will patch the inactive slot containing the newly installed OTA.

### Step 4: Reboot

Once complete, tap the **Reboot** button inside the Magisk app. Your phone will restart into the updated slot with root intact.

---

## Troubleshooting & Recovery

- **Bootloop or Soft Brick**:  
  - Reflash the stock `boot.img` extracted in Step 3:

```sh
fastboot flash boot boot.img
fastboot reboot
```

- **Modules Causing Bootloop**:  
  - Boot into safe mode (hold Volume Down during Google logo boot) to disable all Magisk modules, or use:

```sh
adb wait-for-device
adb shell magisk --remove-modules
```

&nbsp;
