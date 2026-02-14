# xHCI Controller Contention with USB Video Adapters

## The Problem

USB-to-HDMI adapters like the MS9132/MS9133 use **USB 3.0 bulk transfers** to
send compressed video frames to the display. At 1920x1080@60Hz, this generates
a sustained high volume of USB traffic (~250 interrupts/sec with large
payloads).

On systems with a **single xHCI controller** (common on laptops and budget
desktops), all USB devices -- including Bluetooth, webcams, audio interfaces,
USB storage, and network adapters -- share the same controller hardware and
the same IRQ line.

When the video adapter sends a burst of data, **other USB devices experience
increased latency** because the xHCI controller can only process one transfer
ring at a time. Devices with real-time requirements (isochronous transfers)
are the most affected.

## Who Is Affected

Any system where:
1. The USB video adapter and a latency-sensitive USB device share the same
   xHCI controller
2. There is only one xHCI controller (one IRQ for all USB traffic)

You can check with:
```bash
# How many USB controllers?
lspci | grep -i 'usb\|xhci'

# Which controller handles each device?
lsusb -t

# Single IRQ = single controller:
cat /proc/interrupts | grep xhci
```

If `lspci` shows only one USB controller and `lsusb -t` shows both the
video adapter and your latency-sensitive device under the same `xhci_hcd`,
you are affected.

## Symptoms

The specific symptoms depend on which USB device is being starved:

| Affected Device | Symptoms |
|----------------|----------|
| **Bluetooth audio (A2DP)** | Micro-stutters, dropouts, `hci0: command tx timeout` in dmesg |
| **USB audio interface** | Xruns, buffer underruns, clicks/pops |
| **USB webcam** | Frame drops, stuttering video, reduced framerate |
| **USB network adapter** | Latency spikes, packet loss under load |
| **USB HID (keyboard/mouse)** | Input lag (rare, low-bandwidth devices are less affected) |
| **USB storage** | Slower transfer speeds (usually tolerable) |

dmesg may show:
```
usb X-Y: wait urb failed! ret=-110
Bluetooth: hci0: command 0x0408 tx timeout
```

The `-110` error is `ETIMEDOUT` -- the USB transfer did not complete in time
because the xHCI controller was busy processing video bulk transfers.

## Why This Happens

The xHCI (eXtensible Host Controller Interface) uses **transfer rings** to
schedule USB transactions. Each endpoint gets a ring, but the controller
hardware processes them through a single command/event ring pair per
controller instance.

USB video uses **bulk transfers** on USB 3.0 SuperSpeed. Bulk transfers have
no guaranteed bandwidth -- they use whatever is available. However, because
video requires continuous high throughput, the bulk transfers effectively
monopolize the controller during frame transmission.

Meanwhile, Bluetooth A2DP and USB audio use **isochronous transfers** which
have timing guarantees in the USB spec. But the xHCI controller's
implementation of these guarantees varies by silicon vendor, and under heavy
bulk load, the controller may not service isochronous transfers in time.

```
Timeline (simplified):

  |===BULK(video)===|..iso..|===BULK(video)===|..iso..|===BULK===|
                         ^                        ^
                         |                        |
                   BT audio gets              BT audio gets
                   brief window               brief window

  When bulk bursts are large, iso windows shrink -> deadline misses
```

## Mitigations

These mitigations reduce the impact but **cannot fully eliminate** the
contention on single-controller systems. They are listed from most to least
effective.

### 1. Use a separate USB controller (hardware fix)

The only complete solution is to put the video adapter and the
latency-sensitive device on **different xHCI controllers**.

- **Thunderbolt/USB4 docks**: These typically have their own xHCI controller
  (separate from the chipset's). Plugging the video adapter into a
  Thunderbolt dock moves its traffic to a different controller.
- **PCIe USB cards**: Desktop systems can add a PCIe USB 3.0 card (e.g.,
  Renesas/NEC uPD720201) to create a second controller.
- **Modern laptops** (Intel 10th gen+, AMD Ryzen): Often have 2+ USB
  controllers (PCH xHCI + CPU-integrated USB4/Thunderbolt). Check `lspci`
  to verify.

**Note**: USB hubs do NOT help. A hub expands ports but all traffic still
flows through the same host controller. You need a physically separate
controller (different PCI device, different IRQ).

### 2. Realtime scheduling for audio (software)

If the affected device is audio (BT or USB), ensure the audio server uses
SCHED_FIFO:

```bash
# PipeWire: verify RT threads exist
ps -eLo pid,lwp,cls,rtprio,nice,comm | grep pipewire
# Look for threads with FF (SCHED_FIFO), not just TS (SCHED_OTHER)
```

On **Debian 12** (PipeWire 0.3.65), `rt.prio` is commented out by default.
See the [PipeWire RT section](#pipewire-rt-on-debian-12) below for the fix.

### 3. Disable Bluetooth USB autosuspend

If BT is affected, disable autosuspend to eliminate wake-up latency:

```bash
# Find BT device path:
lsusb -t | grep btusb
# e.g., Port 7: ... Driver=btusb

# Disable (immediate):
echo on | sudo tee /sys/bus/usb/devices/<BT-PORT>/power/control

# Disable (permanent):
echo 'options btusb enable_autosuspend=0' | sudo tee /etc/modprobe.d/btusb-no-autosuspend.conf
```

### 4. Reduce video adapter bandwidth

Lower resolution or framerate reduces USB bulk transfer volume:

```bash
# 30Hz instead of 60Hz (halves USB traffic):
xrandr --output HDMI-A-2 --mode 1920x1080 --rate 30

# Lower resolution:
xrandr --output HDMI-A-2 --mode 1280x720 --rate 60
```

### 5. Offload the affected function

If the contention is not solvable, offload the affected function entirely:
- **Audio**: Use Spotify Connect, AirPlay, or network audio to another device
- **Network**: Use wired Ethernet on a different bus (PCIe, built-in)
- **Webcam**: Use an IP camera or built-in laptop camera if on different bus

## PipeWire RT on Debian 12

Debian 12 ships PipeWire 0.3.65 with `rt.prio` **commented out** by default.
This version does **not** support `conf.d/` drop-in overrides (requires
0.3.68+).

Fix:
```bash
# 1. Add user to pipewire group (for RT limits):
sudo usermod -aG pipewire $USER
# Log out and back in

# 2. Copy and edit config:
mkdir -p ~/.config/pipewire/
cp /usr/share/pipewire/pipewire.conf ~/.config/pipewire/pipewire.conf

# 3. In ~/.config/pipewire/pipewire.conf, find libpipewire-module-rt
#    and uncomment rt.prio:
#
#    nice.level    = -11
#    rt.prio       = 88       <- uncomment this
#    rt.time.soft  = -1       <- uncomment this
#    rt.time.hard  = -1       <- uncomment this

# 4. Restart:
systemctl --user restart pipewire pipewire-pulse wireplumber

# 5. Verify (data loop threads should show FF, not TS):
ps -eLo pid,lwp,cls,rtprio,nice,comm | grep pipewire
```

**Note**: The main process thread always shows `TS` (SCHED_OTHER). Only the
internal data loop threads use `FF` (SCHED_FIFO). Use `ps -eLo` (with `-L`
for threads) to verify.

## Diagnostic Commands

```bash
# 1. Count xHCI controllers:
lspci | grep -c -i xhci

# 2. Map devices to controllers:
lsusb -t

# 3. Check IRQ count and affinity:
cat /proc/interrupts | grep xhci
cat /proc/irq/<IRQ>/effective_affinity_list

# 4. Monitor USB errors:
sudo dmesg -w | grep -i 'urb failed\|timeout\|hci.*command'

# 5. Check PipeWire RT status:
ps -eLo pid,lwp,cls,rtprio,nice,comm | grep -E 'pipewire|wireplumber'

# 6. Check BT autosuspend:
cat /sys/bus/usb/devices/*/product | grep -n .  # find BT port
cat /sys/bus/usb/devices/<BT-PORT>/power/control
```

## Tested Systems

| System | xHCI Controllers | Contention | Notes |
|--------|-----------------|------------|-------|
| Lenovo IdeaPad (7th gen, Sunrise Point-LP) | 1 | **Yes** | BT audio stutters with USB video active |
| Systems with Thunderbolt 3/USB4 | 2+ | No (if video adapter on TB port) | TB has its own xHCI |
