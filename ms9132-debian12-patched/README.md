# MS9132 Driver - Patched for Debian 12

Pre-patched source ready to compile on **Debian 12** (kernel 6.1).

Applied patch: Patch 6 (log spam fix: `dev_info` -> `dev_dbg`).

## Build and install

```bash
sudo apt install linux-headers-$(uname -r) build-essential
make clean && make
sudo mkdir -p /lib/modules/$(uname -r)/extra
sudo cp drm/usbdisp_drm.ko drm/usbdisp_usb.ko /lib/modules/$(uname -r)/extra/
sudo depmod -a
sudo modprobe usbdisp_drm
sudo modprobe usbdisp_usb
```

For the full guide (Secure Boot, autoloading, troubleshooting), see the [main README](../README.md).
