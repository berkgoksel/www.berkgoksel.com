---
layout: post
title: "Setting Up Pixel 8/9 for Android App Reversing and Mobile Security Testing"
date: 2026-01-22
categories: mobile-security
---

This post aims to serve as a guide on how to set a Pixel device up for Android security testing. I've tested this on both the Pixel 8 and the Pixel 9. The steps are identical for both.

This covers flashing stock Pixel OS, rooting with Magisk, and getting Burp Suite working for traffic interception.

<!-- more -->

## Prerequisites

- Pixel 8 (codename: shiba) or Pixel 9 (codename: tokay)
- Ubuntu 22.04 (or similar Linux distro)
- USB cable
- adb and fastboot installed: `sudo apt install android-tools-adb android-tools-fastboot`

## Step 1: Download Factory Image

Get the latest factory image from https://developers.google.com/android/images

- Pixel 8: look for **shiba**
- Pixel 9: look for **tokay**

Google factory images are named like `shiba-bp4a.251205.006-factory-*.zip`. Don't confuse with GrapheneOS images which use format `shiba-install-*.zip`.

## Step 2: Enable Developer Options and OEM Unlocking

On the phone:

1. **Settings → About phone → tap "Build number" 7 times** to enable Developer Options
2. **Settings → System → Developer options → USB debugging → ON**
3. **Settings → System → Developer options → OEM unlocking → ON**

Note: If OEM unlocking is greyed out, connect to WiFi/mobile data and wait ~24 hours (Google's anti-theft measure).

## Step 3: Extract Factory Image

```bash
cd ~/Downloads
unzip shiba-*.zip  # or tokay-*.zip for Pixel 9
cd shiba-*         # or tokay-* for Pixel 9
```

For Pixel 8, extract the inner image zip:

```bash
unzip image-shiba-*.zip
```

For Pixel 9, the images come extracted already.

Make the flash script executable:

```bash
chmod +x flash-all.sh
```

## Step 4: Unlock Bootloader and Flash

Reboot into fastboot mode using the hardware buttons (Power + Volume Down), then:

```bash
fastboot flashing unlock
```

Confirm on device with volume keys + power. This wipes everything.

Once back at fastboot:

```bash
fastboot devices
./flash-all.sh
```

**Do not re-lock the bootloader** since we need it unlocked for Magisk.

## Step 5: Post-Flash Setup

After the phone boots:

1. Skip through initial setup
2. Re-enable Developer Options (tap Build number 7 times)
3. Re-enable USB Debugging

## Step 6: Install Magisk

Magisk is a systemless root solution that modifies the boot image to provide root access while passing SafetyNet checks. It allows installing modules that modify the system without actually touching the system partition.

Download the latest Magisk APK from https://github.com/topjohnwu/Magisk/releases

```bash
adb install Magisk-*.apk
```

Allow notifications when prompted. Useful for superuser prompts.

## Step 7: Patch init_boot Image

**Important:** Pixel 8 on Android 13+ uses `init_boot.img` for the ramdisk, not `boot.img`.

Push the init_boot image to the phone:

```bash
adb push init_boot.img /sdcard/Download/
```

On the phone:

1. Open Magisk app
2. Tap **Install**
3. Select **Select and Patch a File**
4. Navigate to Downloads and select `init_boot.img`

Note: You can also extract boot.img from a rooted device using dd, but for fresh installs use the init_boot.img from the factory zip. Just make sure it matches your flashed version.

## Step 8: Flash Patched Image

List the patched file:

```bash
ls /sdcard/Download/
```

Pull it:

```bash
adb pull /sdcard/Download/magisk_patched-*.img .
```

You can test by booting the image directly without flashing:

```bash
adb reboot bootloader
fastboot boot magisk_patched-*.img
```

If Magisk shows as installed, flash it permanently:

```bash
adb reboot bootloader
fastboot flash init_boot magisk_patched-*.img
fastboot reboot
```

## Step 9: Verify Root

```bash
adb shell
su
```

You should see a superuser prompt on the phone. Grant access and you'll have a root shell.

## Step 10: Configure Magisk

In Magisk app, go to **Settings** and configure:

1. **Zygisk → ON** (enables code injection into apps, needed for DenyList)
2. **Enable Zygisk DenyList** (hide root from specific apps)
3. **Enable DNS over HTTPS**
4. **Hide the Magisk app** → enter a custom name → add shortcut icon when prompted

Reboot the phone. After reboot, verify Zygisk shows **Yes** in Magisk home screen.

## Step 11: Install Burp CA Certificate

Android 14+ has read-only system partitions protected by dm-verity. We'll use a Magisk module to install the CA cert as a system certificate.

### Export Burp Certificate

In Burp Suite: **Proxy → Proxy settings → Import/Export CA Certificate → Export → Certificate in DER format** → save as `cacert.der`

### Convert and Hash the Certificate

```bash
openssl x509 -inform DER -in cacert.der -out cacert.pem
hash=$(openssl x509 -inform PEM -subject_hash_old -in cacert.pem | head -1)
mv cacert.pem $hash.0
```

### Install MagiskTrustUserCerts Module

Download from https://github.com/NVISOsecurity/MagiskTrustUserCerts/releases

```bash
adb push MagiskTrustUserCerts.zip /sdcard/Download/
```

In Magisk app: **Modules → Install from storage → select the zip → reboot**

### Install Certificate as User Cert

```bash
adb push $hash.0 /sdcard/Download/
```

On phone: **Settings → Security & Privacy → More security settings → Encryption & credentials → Install a certificate → CA certificate → Install anyway** → select your .0 file from Downloads

Reboot the phone. The MagiskTrustUserCerts module will promote the user certificate to system trust.

## Step 12: Configure Proxy

### Option A: WiFi Proxy

On phone: **Settings → Network & internet → Internet → tap your WiFi network → Edit → Advanced options → Proxy → Manual**

Set proxy hostname to your Burp machine's IP and port 8080.

### Option B: ADB Reverse (for VM setups)

If Burp is running in a VM, redirect USB to the VM first:

- **QEMU/virt-manager:** Virtual Machine → Redirect USB device → select Pixel
- **VirtualBox/VMware:** Configure USB passthrough in VM settings

Stop adb on host first if running:

```bash
adb kill-server
```

In the VM:

```bash
adb devices
adb reverse tcp:8080 tcp:8080
```

On phone, set proxy to `127.0.0.1:8080`.

**Important:** Disable any VPN on the phone. VPN overrides proxy settings.

## Step 13: Test

Open browser on phone. Traffic should appear in Burp's HTTP history.

## Troubleshooting

- **Magisk shows "N/A" after flashing:** Make sure you're flashing `init_boot` not `boot` on Pixel 8
- **Can't remount system:** Android 14+ uses dm-verity, use Magisk modules instead
- **No traffic in Burp:** Check VPN is disabled, proxy settings are correct, and Burp is bound to all interfaces (0.0.0.0)
- **Certificate not trusted:** Ensure MagiskTrustUserCerts module is installed and phone was rebooted
