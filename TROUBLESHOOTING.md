# Troubleshooting - MS9132 Driver on Kernel 6.10+

## The Critical Bug: FOP_UNSIGNED_OFFSET

Starting with kernel 6.10, `drm_open_helper()` (in `drm_file.c:312`) checks:

```c
if (WARN_ON_ONCE(!(filp->f_op->fop_flags & FOP_UNSIGNED_OFFSET)))
    return -EINVAL;
```

All in-tree DRM drivers were updated automatically. Out-of-tree drivers (like MS9132) were not. Without this flag, **any** `open()` call on the DRM device node returns `EINVAL`.

### The Chain of Failures

```
Driver missing FOP_UNSIGNED_OFFSET
    -> drm_open_helper() returns -EINVAL
        -> open("/dev/dri/card1") fails
            -> systemd-logind TakeDevice fails
                -> Xorg does not receive fd for card1
                    -> modeset(G0) never initializes for card1
                        -> HDMI-A-2/3/4 connectors don't appear in xrandr
                            -> EDID is never read (no display server using card1)
```

This is why all symptoms point away from the real cause. The logind error, the missing xrandr outputs, and the empty EDID are all consequences of one missing flag.

## Diagnostic Commands

```bash
# Basic status
uname -r && lsmod | grep usbdisp && lsusb | grep 534d

# Test if the DRM device can be opened (THE key test)
python3 -c 'import os; fd=os.open("/dev/dri/card1", os.O_RDWR); print("OK", fd); os.close(fd)'

# Check for the WARNING in dmesg
sudo dmesg | grep -i 'WARNING.*drm_file\|drm_open_helper'

# Xorg log (display number may vary)
cat ~/.local/share/xorg/Xorg.1.log | grep -E 'card1|USBDisplay|failed|EE.*logind|modeset.G'

# Connector status
for f in /sys/class/drm/card1-*/status; do echo "$f: $(cat $f)"; done

# xrandr (adjust DISPLAY and XAUTHORITY for your setup)
xrandr --listproviders
xrandr

# udev info
udevadm info /dev/dri/card1
```

## Common Issues

### Cursor disappears when moving between displays

The MS9132 driver does not implement hardware cursor planes. When using
Wayland (GNOME/Mutter) with multiple GPUs, the cursor can disappear when
moving from the USB display to a secondary output (e.g., HDMI) on the
primary GPU.

This happens because Mutter switches from software cursor (on the USB
display) to hardware cursor (on the primary GPU), but fails to activate
the cursor plane on the correct CRTC for secondary outputs.

**Symptoms:**
- Cursor disappears when moving from USB display to HDMI output
- Cursor works fine moving from USB display to the laptop screen (eDP)
- Cursor works fine moving between eDP and HDMI directly

**Fix:** Force software cursor rendering for all displays:

```bash
mkdir -p ~/.config/environment.d/
echo 'MUTTER_DEBUG_FORCE_SOFTWARE_CURSOR=1' > ~/.config/environment.d/10-mutter-software-cursor.conf
```

Log out and back in for the change to take effect. The performance cost is
negligible.

**Verify:**
```bash
# Check planes on USB display (no cursor plane = expected):
sudo cat /sys/kernel/debug/dri/1/state | grep -E 'plane|cursor'
# Only "plane-0", "plane-1", "plane-2" -- no "cursor" entries

# Check planes on primary GPU (has cursor planes):
sudo cat /sys/kernel/debug/dri/0/state | grep -E 'plane|cursor'
# Shows "cursor A", "cursor B", "cursor C"
```

### Module loads but device doesn't work

1. Check `open()` works: `python3 -c 'import os; os.open("/dev/dri/card1", os.O_RDWR)'`
2. If EINVAL: check dmesg for WARNING, verify Patch 4 was applied
3. If permission denied: check udev rules and seat assignment

### GDM crashes when USB adapter is connected

Do NOT create `/etc/X11/xorg.conf.d/20-usb-display.conf` or similar. X.org auto-detects the USB display through DRM. Explicit multi-GPU Device sections cause GDM session registration to fail with "Session never registered, failing" and "maximum number of X display failures reached".

Fix:
```bash
sudo rm /etc/X11/xorg.conf.d/20-usb-display.conf  # or rename to .disabled
sudo systemctl restart gdm3
```

### EDID empty even with USB connected

- Verify USB device is detected: `lsusb | grep 534d`
- Verify connector shows "connected": `cat /sys/class/drm/card1-HDMI-A-2/status`
- If connected but EDID empty: the display server is not using card1 (check open/logind chain above)
- If disconnected: check USB cable, try different port

### Module signing fails

```bash
# Verify MOK key is enrolled
mokutil --test-key /root/mok-keys/MOK.der
# Should say "is already enrolled"

# Re-sign modules
KVER=$(uname -r)
SIGN=/usr/src/linux-headers-$KVER/scripts/sign-file
sudo $SIGN sha256 /root/mok-keys/MOK.priv /root/mok-keys/MOK.der /lib/modules/$KVER/extra/usbdisp_drm.ko
sudo $SIGN sha256 /root/mok-keys/MOK.priv /root/mok-keys/MOK.der /lib/modules/$KVER/extra/usbdisp_usb.ko
```

## Kernel API Changes Reference

| Kernel | Change | Impact |
|--------|--------|--------|
| 6.2+ | `vmalloc.h` no longer included transitively | Compilation error in `msdisp_drm_gem.c` |
| 6.2+ | `scatterlist.h` no longer included transitively | Compilation error in `usb_hal_thread.c` |
| 6.7+ | EDID API replaced (`drm_do_get_edid` -> `drm_edid_read_custom`, etc.) | Compilation error in `msdisp_drm_connector.c` |
| 6.10+ | `FOP_UNSIGNED_OFFSET` required in DRM `file_operations` | `open()` returns EINVAL at runtime |
| 6.11+ | `platform_driver.remove` returns `void` instead of `int` | Compilation error in `msdisp_plat_dev.c` |
