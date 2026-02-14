# MS9132/MS9133 Linux Driver - Patches

Patches for the official [MacroSilicon MS9132](http://en.macrosilicon.com/info.asp?base_id=2&third_id=72) USB-to-HDMI driver (v3.0.1.3) on Linux.

The official driver was written for kernel 6.1. On newer kernels it fails to compile or the display does not work at all. On any kernel, the driver floods the system log with unnecessary messages.

> **This software is provided "as is", without warranty of any kind. Use at your own risk. This is an unofficial community patch -- not supported by MacroSilicon.**

## Which patches do I need?

| Your kernel | What to apply |
|-------------|---------------|
| **6.1** (Debian 12) | Patch 6 only (recommended) |
| **6.2 - 6.6** | Patches 1, 5, 6 |
| **6.7 - 6.9** | Patches 1, 3, 5, 6 |
| **6.10** | Patches 1, 3, 4, 5, 6 |
| **6.11+** (Debian 13) | All patches (1-6) |

## Patches

| # | What it fixes | Files | Kernel |
|---|---------------|-------|--------|
| 1 | **Build error**: missing `vmalloc.h` header | `msdisp_drm_gem.c` | 6.2+ |
| 2 | **Build error**: `platform_driver.remove` must return void | `msdisp_plat_dev.c`, `.h`, `msdisp_drm_drv.h` | 6.11+ |
| 3 | **Build error**: old EDID functions removed from kernel | `msdisp_drm_connector.c`, `.h` | 6.7+ |
| 4 | **Display does not work**: `open("/dev/dri/card1")` returns error | `msdisp_drm_drv.c` | 6.10+ |
| 5 | **Build error**: missing `scatterlist.h` header | `usb_hal/usb_hal_thread.c` | 6.2+ |
| 6 | **Log spam**: driver prints ~40 messages/sec to system log | `msdisp_drm_fb.c` | All |

All patches use `#if LINUX_VERSION_CODE` guards and are safe to apply on any kernel version.

### Patch details

**Patch 1 - vmalloc.h** (`msdisp_drm_gem.c`): Kernel 6.2 stopped including `vmalloc.h` indirectly. Without it, the driver fails to compile with undefined symbol errors.

**Patch 2 - platform_driver.remove** (`msdisp_plat_dev.c`, `.h`, `msdisp_drm_drv.h`): Kernel 6.11 changed the `remove` callback from returning `int` to returning `void`. Without this, the driver fails to compile.

**Patch 3 - EDID API** (`msdisp_drm_connector.c`, `.h`): Kernel 6.7 replaced the EDID functions (`drm_do_get_edid`, `drm_connector_update_edid_property`, etc.) with new ones. Without this, the driver fails to compile.

**Patch 4 - FOP_UNSIGNED_OFFSET** (`msdisp_drm_drv.c`): Kernel 6.10 requires the `FOP_UNSIGNED_OFFSET` flag in DRM file operations. Without it, the display device node cannot be opened and the display is completely unusable. All other symptoms (logind errors, missing xrandr outputs, empty EDID) are consequences of this.

**Patch 5 - scatterlist.h** (`usb_hal/usb_hal_thread.c`): Same as Patch 1 -- kernel 6.2 stopped including `scatterlist.h` indirectly.

**Patch 6 - Log spam** (`msdisp_drm_fb.c`): The driver logs a message every time a framebuffer is created (~40 times/sec). This floods `dmesg`, wastes CPU on printk spinlocks, and can worsen USB controller contention (see [XHCI_CONTENTION.md](XHCI_CONTENTION.md)). The fix changes `dev_info` to `dev_dbg` so the message is only visible when debug logging is enabled. **Recommended for all kernels.**

## Repository Structure

```
patches/                     Individual patch files (apply to official source)
ms9132-debian12-patched/     Ready to compile on Debian 12 (kernel 6.1)
ms9132-debian13-patched/     Ready to compile on Debian 13 (kernel 6.11+)
```

## Quick Start (pre-patched source)

If your system matches Debian 12 or 13, use the pre-patched source directly:

```bash
# Install build dependencies:
sudo apt install linux-headers-$(uname -r) build-essential

# Debian 12:
cd ms9132-debian12-patched
make clean && make

# Debian 13:
cd ms9132-debian13-patched
make clean && make
```

Then install:
```bash
sudo mkdir -p /lib/modules/$(uname -r)/extra
sudo cp drm/usbdisp_drm.ko drm/usbdisp_usb.ko /lib/modules/$(uname -r)/extra/
sudo depmod -a
sudo modprobe usbdisp_drm
sudo modprobe usbdisp_usb
```

## Quick Start (apply patches manually)

For other kernels, get the official driver source (v3.0.1.3) from MacroSilicon and apply the patches you need:

```bash
# From the driver source root (where drm/ and usb_hal/ are):
cd drm
for p in /path/to/patches/msdisp_*.patch; do patch -p0 < "$p"; done
cd ../usb_hal
for p in /path/to/patches/usb_hal_*.patch; do patch -p0 < "$p"; done
cd ..
make clean && make
```

For the full guide (Secure Boot signing, autoloading, recompilation after kernel updates), see [INSTALL.md](INSTALL.md).

## Important Notes

- **Do NOT create an X11 xorg.conf.d file** for the USB display. X.org auto-detects it. Explicit configuration causes GDM to crash.
- `glamor initialization failed` on the USB display is normal -- it uses software rendering (ShadowFB).
- `usb hal is null` in dmesg when unplugging the adapter is expected.
- **Cursor disappears** when moving from the USB display to another monitor (Wayland/Mutter): the driver has no hardware cursor planes. See [TROUBLESHOOTING.md](TROUBLESHOOTING.md) for the fix.
- **USB contention**: On systems with a single USB controller, the video adapter can starve other USB devices (Bluetooth audio, webcams, etc.). See [XHCI_CONTENTION.md](XHCI_CONTENTION.md).

## Tested Environment

- Debian 12 (Bookworm), kernel 6.1.0-42-amd64 (Patch 6 only)
- Debian 13 (Trixie), kernel 6.12.69+deb13-amd64 (All patches)
- systemd 257, GDM, GNOME (X11 and Wayland)
- Secure Boot enabled (MOK signed modules)
- USB adapter: BLEWE generic USB 3.0 to HDMI (MS9132 chip, USB ID `534d:9132`)

## Related Projects

- [ms912x](https://github.com/rhgndf/ms912x) - Reverse-engineered open source driver for MS912x chips

## License

GPL v2, same as the original driver.
