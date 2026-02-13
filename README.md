# MS9132/MS9133 Linux Driver - Kernel 6.10+ Patches

Patches to make the official [MacroSilicon MS9132](http://en.macrosilicon.com/info.asp?base_id=2&third_id=72) USB-to-HDMI driver work on **Linux kernel 6.10 and newer** (tested on Debian 13 Trixie, kernel 6.12).

The official driver (v3.0.1.3) was written for kernel 6.1 and fails silently on newer kernels. The most critical issue is that `open("/dev/dri/card1")` returns `EINVAL` due to a missing `FOP_UNSIGNED_OFFSET` flag, making the device completely unusable.

## Disclaimer

> **This software is provided "as is", without warranty of any kind, express or implied. Use at your own risk. The authors are not responsible for any damage to your hardware, software, or system. This is an unofficial community patch -- not supported by MacroSilicon.**

## Symptoms on kernel 6.10+

If you have a MS9132/MS9133 USB-HDMI adapter and see any of these on a modern kernel:

- `systemd-logind: failed to take device /dev/dri/card1: Invalid argument`
- `xrandr` does not show the USB display outputs
- `xrandr --listproviders` shows only 1 provider (your main GPU)
- Empty EDID when reading `/sys/class/drm/card1-*/edid`
- `WARNING: CPU: ... at drivers/gpu/drm/drm_file.c:312 drm_open_helper` in dmesg

All of these are caused by the same root issue: the driver is missing the `FOP_UNSIGNED_OFFSET` flag required since kernel 6.10.

## Patches

5 patches are needed to compile and run on kernel 6.10+:

| Patch | File(s) | Kernel | Description |
|-------|---------|--------|-------------|
| 1 | `msdisp_drm_gem.c` | 6.2+ | Add missing `#include <linux/vmalloc.h>` |
| 2 | `msdisp_plat_dev.c`, `.h`, `msdisp_drm_drv.h` | 6.11+ | `platform_driver.remove` return type changed from `int` to `void` |
| 3 | `msdisp_drm_connector.c`, `.h` | 6.7+ | EDID API replaced (`drm_do_get_edid` -> `drm_edid_read_custom`, etc.) |
| 4 | `msdisp_drm_drv.c` | 6.10+ | **Critical**: Add `FOP_UNSIGNED_OFFSET` to `file_operations` |
| 5 | `usb_hal/usb_hal_thread.c` | 6.2+ | Add missing `#include <linux/scatterlist.h>` (if needed) |

All patches use `#if LINUX_VERSION_CODE` guards, so the patched driver remains compatible with older kernels.

## Quick Start

1. Download the official driver source from [MacroSilicon](http://www.macrosilicon.com:9080/download/USBDisplay/Linux/SourceCode/MS91xx_Linux_Drm_SourceCode_V3.0.1.3.zip)
2. Extract and apply the patches from `patches/`:
   ```bash
   cd ms9132-official/drm
   for p in /path/to/patches/*.patch; do patch -p0 < "$p"; done
   ```
3. Compile:
   ```bash
   cd ms9132-official
   make clean && make
   ```
4. Install:
   ```bash
   sudo mkdir -p /lib/modules/$(uname -r)/extra
   sudo cp drm/usbdisp_drm.ko drm/usbdisp_usb.ko /lib/modules/$(uname -r)/extra/
   sudo depmod -a
   ```
5. Load:
   ```bash
   sudo modprobe usbdisp_drm
   sudo modprobe usbdisp_usb
   ```

For the full installation guide (including Secure Boot/MOK signing and autoloading), see [INSTALL.md](INSTALL.md).

## Important Notes

- **Do NOT create an X11 xorg.conf.d file** for multi-GPU configuration. X.org auto-detects the USB display via DRM. Explicit Device sections cause GDM to crash.
- `glamor initialization failed` on the USB display is normal -- it falls back to software rendering (ShadowFB).
- `usb hal is null` in dmesg when unplugging the adapter is expected behavior.

## Tested Environment

- Debian 13 (Trixie), kernel 6.12.69+deb13-amd64
- systemd 257, GDM, GNOME (X11 and Wayland)
- Secure Boot enabled (MOK signed modules)
- USB adapter: BLEWE generic USB 3.0 to HDMI (MS9132 chip, USB ID `534d:9132`)

## Related Projects

- [Official driver source](http://www.macrosilicon.com:9080/download/USBDisplay/Linux/SourceCode/MS91xx_Linux_Drm_SourceCode_V3.0.1.3.zip) (MacroSilicon, kernel 6.1)
- [ms912x](https://github.com/rhgndf/ms912x) - Reverse-engineered open source driver for MS912x chips

## License

GPL v2, same as the original driver. See [LICENSE](LICENSE).
