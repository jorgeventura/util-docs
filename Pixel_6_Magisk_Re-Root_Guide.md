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



On A/B slot devices like the Pixel 6, flashing to the wrong slot or leaving the backup slot unpatched can cause bootloops or unexpected loss of root after an OTA or fallback reboot.

---

### Understanding the A/B Slot Behavior on Pixel 6

* **Active vs. Inactive Slot:** Android always runs from one slot (`a` or `b`) while system OTAs write to the inactive slot in the background.
* **The `fastboot flash boot` Trap:** Running `fastboot flash boot <image>` targets whichever slot is currently flagged **active** in the bootloader. If a recent OTA just finished installing and swapped the active flag, flashing `boot` without checking can accidentally overwrite the old slot instead of the newly updated one.
* **Kernel & Vendor Boot:** On the Pixel 6 (Google Tensor), standard Magisk patching modifies the standard `boot.img` (which contains the generic kernel), not `vendor_boot.img`.

---

### Recommended Best Practices

#### 1. Always Query the Active Slot First

Before flashing, check which slot the bootloader is currently set to boot:

```bash
fastboot getvar current-slot

```

If it returns `current-slot: a`, the device will boot from slot `a`. If it returns `b`, it boots from `b`.

#### 2. Explicitly Target the Active Slot

Rather than relying on `fastboot flash boot magisk_patched.img` (which defaults to active, but can be error-prone if state changes mid-session), specify the slot directly:

```bash
# If current-slot was 'a':
fastboot flash boot_a magisk_patched.img

# If current-slot was 'b':
fastboot flash boot_b magisk_patched.img

```

#### 3. Test with `fastboot boot` Before Flashing

To ensure the patched kernel doesn't bootloop (especially if kernel/build versions mismatch):

```bash
fastboot boot magisk_patched.img

```

This performs a one-time RAM boot without altering either `boot_a` or `boot_b` on flash storage. If the device boots cleanly with Magisk functional, reboot into bootloader and flash permanently.

#### 4. Never Flash Both Slots with the Same Image After an OTA

During a monthly OTA, only the inactive slot receives the new build's kernel and vendor partitions. Flashing a new build's patched `boot.img` onto the *old* slot will mismatch that slot's vendor/system partitions and break rollback protection. Only flash the slot matching the OS build the `boot.img` came from.

---

### Quick Markdown Snippet for Your Docs

```markdown
### A/B Slot Verification & Flashing

1. Check active slot:
   ```bash
   fastboot getvar current-slot

```

2. One-time test boot (recommended):
```bash
fastboot boot magisk_patched.img

```


3. Flash specifically to the verified active slot:
```bash
# Replace `<slot>` with either 'a' or 'b'
fastboot flash boot_<slot> magisk_patched.img

```


4. Reboot into system:
```bash
fastboot reboot

```

