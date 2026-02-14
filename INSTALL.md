# Installation Guide - MS9132 Driver Patches

## Prerequisites

```bash
# Check kernel version
uname -r

# Install build dependencies
sudo apt install linux-headers-$(uname -r) build-essential
```

## Step 1: Get the official driver source

Download the official MS9132 driver source (v3.0.1.3) from [MacroSilicon](http://en.macrosilicon.com/info.asp?base_id=2&third_id=72) and extract it. The zip contains a folder with `drm/` and `usb_hal/` subdirectories.

## Step 2: Apply patches

Check the [README](README.md) to see which patches you need for your kernel version.

```bash
# From the driver source root (where drm/ and usb_hal/ are):
cd drm
for p in /path/to/patches/msdisp_*.patch; do patch -p0 < "$p"; done
cd ../usb_hal
for p in /path/to/patches/usb_hal_*.patch; do patch -p0 < "$p"; done
cd ..
```

Verify no errors. Each patch uses `#if LINUX_VERSION_CODE` guards, so they are safe for any kernel version. You can apply all patches regardless of your kernel.

## Step 3: Compile

```bash
make clean && make
```

Should complete without errors (warnings about `-Wmissing-prototypes` are harmless).
Two modules are generated:
- `drm/usbdisp_drm.ko`
- `drm/usbdisp_usb.ko`

## Step 4: Install modules

```bash
sudo mkdir -p /lib/modules/$(uname -r)/extra
sudo cp drm/usbdisp_drm.ko drm/usbdisp_usb.ko /lib/modules/$(uname -r)/extra/
sudo depmod -a
```

## Step 5: Autoload at boot

```bash
echo -e "usbdisp_drm\nusbdisp_usb" | sudo tee /etc/modules-load.d/usbdisp.conf
```

## Step 6: Sign modules (Secure Boot only)

Check if Secure Boot is enabled: `mokutil --sb-state`

If not enabled, skip to Step 7.

### 6a. Generate MOK key (first time only)

```bash
sudo mkdir -p /root/mok-keys
sudo openssl req -new -x509 -newkey rsa:2048 \
    -keyout /root/mok-keys/MOK.priv \
    -outform DER -out /root/mok-keys/MOK.der \
    -nodes -days 36500 \
    -subj "/CN=MS9132 Module Signing Key/"
```

### 6b. Sign the modules

```bash
KVER=$(uname -r)
SIGN=/usr/src/linux-headers-$KVER/scripts/sign-file
sudo $SIGN sha256 /root/mok-keys/MOK.priv /root/mok-keys/MOK.der \
    /lib/modules/$KVER/extra/usbdisp_drm.ko
sudo $SIGN sha256 /root/mok-keys/MOK.priv /root/mok-keys/MOK.der \
    /lib/modules/$KVER/extra/usbdisp_usb.ko
```

### 6c. Enroll MOK key (first time only)

```bash
sudo mokutil --import /root/mok-keys/MOK.der
# Enter a temporary password (e.g., 1234)
sudo reboot
```

At reboot, in the blue MOK Manager screen:
1. Enroll MOK -> Continue -> Yes
2. Enter the temporary password
3. Reboot

> The MOK key is enrolled once. For future recompilations, only repeat step 6b.

## Step 7: Reboot and verify

```bash
sudo reboot
```

**Do NOT create any X11 xorg.conf.d configuration for the USB display.** X.org auto-detects the second GPU. Explicit Device sections cause GDM to crash.

### Verification

```bash
# Modules loaded
lsmod | grep usbdisp
# Expected: usbdisp_usb + usbdisp_drm

# DRM device opens successfully
python3 -c 'import os; fd=os.open("/dev/dri/card1", os.O_RDWR); print("OK", fd); os.close(fd)'
# Expected: "OK 3" (or any number)

# No WARNING in dmesg
sudo dmesg | grep 'WARNING.*drm_file'
# Expected: no output

# DRM connectors
ls /sys/class/drm/card1-*
# Expected: card1-HDMI-A-2, card1-HDMI-A-3, card1-HDMI-A-4

# USB device detected (with adapter plugged in)
lsusb | grep 534d
# Expected: Bus xxx Device xxx: ID 534d:9132

# xrandr
xrandr --listproviders
# Expected: 2 providers (your GPU + msdisp)
```

## Recompile after kernel update

```bash
cd /path/to/patched-driver-source
make clean && make

KVER=$(uname -r)
sudo cp drm/usbdisp_drm.ko drm/usbdisp_usb.ko /lib/modules/$KVER/extra/

# If Secure Boot:
SIGN=/usr/src/linux-headers-$KVER/scripts/sign-file
sudo $SIGN sha256 /root/mok-keys/MOK.priv /root/mok-keys/MOK.der \
    /lib/modules/$KVER/extra/usbdisp_drm.ko
sudo $SIGN sha256 /root/mok-keys/MOK.priv /root/mok-keys/MOK.der \
    /lib/modules/$KVER/extra/usbdisp_usb.ko

sudo depmod -a
sudo reboot
```

> No need to re-enroll the MOK key.

## Troubleshooting

| Symptom | Likely Cause | Check |
|---------|-------------|-------|
| `modprobe: Key was rejected` | Module not signed or MOK not enrolled | `mokutil --test-key /root/mok-keys/MOK.der` |
| `open card1: EINVAL` | Missing Patch 4 (FOP_UNSIGNED_OFFSET) | `sudo dmesg \| grep WARNING.*drm_file` |
| `logind: failed to take device` | Consequence of EINVAL on open | Fix the open issue first |
| xrandr missing HDMI-A-2 | logind could not give fd to Xorg | Check Xorg log |
| Empty EDID with USB connected | Patch 3 incorrect or USB not detected | `lsusb \| grep 534d` |
| Compiles but fails to load | Kernel version changed | Recompile with `make clean && make` |
| GDM crashes | xorg.conf.d with explicit Device sections | Remove the config and restart GDM |
| `glamor initialization failed` | Normal for USB display | Not an error, uses software rendering |
| `usb hal is null` in dmesg | USB adapter unplugged | Normal behavior on disconnect |
| dmesg flooded with `fb id:` | Missing Patch 6 (log spam fix) | Recompile with Patch 6 applied |

For detailed investigation notes, see [TROUBLESHOOTING.md](TROUBLESHOOTING.md).
