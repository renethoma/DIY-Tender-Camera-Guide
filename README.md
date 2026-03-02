# DIY Tender Camera Build Guide
### Two approaches to building a modular digital camera from a single-board computer

---

## Introduction

This repository documents two independent approaches to building a fully functional DIY digital camera from a single-board computer (SBC), open-source software, and off-the-shelf components. Both builds were developed in parallel as part of an academic project and share the same core goal: a self-contained camera that boots directly into a camera application, is controlled entirely by touch, and is built from transparent, repairable, and replaceable parts.

The project follows **permacomputing principles** — favouring longevity, low resource consumption, repairability, and software transparency over convenience or cutting-edge specs. Every hardware choice is documented, every software dependency is explained, and the entire build process is designed to be replicable.

### What both builds have in common

- Single-board computer running a Linux-based OS
- Small HDMI display with touch input
- Camera module controlled via the Video4Linux2 (V4L2) subsystem
- Python-based application layer
- No cloud dependencies, no proprietary software

### How the two approaches differ

The two builds diverge significantly in their hardware choices, which cascades into very different software and driver complexity.

| | Sweet Potato | Orange Pi 5 |
|---|---|---|
| **SBC** | Libre Computer AML-S905X-CC-V2 | Orange Pi 5 (RK3588S) |
| **OS** | Armbian 25.11 (Ubuntu 24.04) | Ubuntu 22.04 XFCE |
| **Camera** | Sony IMX179 8MP — USB (UVC) | OV13850 13MP — MIPI CSI |
| **Touch** | Capacitive USB — plug & play | Resistive SPI — custom driver |
| **UI framework** | Python + Kivy (planned) | GStreamer + OpenCV + Python |
| **Driver complexity** | Low — USB handles everything | High — overlays, patched daemon |
| **Author** | Luca | René |

> **Which should you build?** If you want the simplest path to a working camera, follow the Sweet Potato build. USB peripherals require zero driver configuration on Linux. If you want more camera resolution, a more powerful SBC, and don't mind significant driver work, follow the Orange Pi 5 build.

---

## Repository Structure

```
/
├── README.md                          ← This file
├── approach-a-sweet-potato/
│   └── LOGBOOK.md                     ← Full development log and reference
├── approach-b-orangepi5/
│   └── SETUP-GUIDE.md                 ← Step-by-step setup guide
└── shared/
    └── packages-and-drivers.md        ← Packages overview for both builds
```

---

---

# the Sweet Potato build — Sweet Potato (Libre Computer AML-S905X-CC-V2)

> **Author:** Luca · `mycamera@sweet-potato`  
> **Project name:** Tender Camera  
> **Project start:** October 2025 · **Last updated:** January 31, 2026  
> **Current phase:** 4 — System Integration & UI Planning

---

## Project Overview

The **Tender Camera** is a sustainable, modular digital camera built from a single-board computer and open-source software. The name "Sweet Potato" refers to the SBC used: the Libre Computer AML-S905X-CC-V2, whose community nickname is "Sweet Potato."

**Goals:**
- Build a fully functional digital camera from modular, off-the-shelf components
- Design a touch-based UI that embodies permacomputing aesthetics
- Implement advanced features (face detection, metadata control, custom capture modes)
- Document the entire process as a replicable, educational resource

**Project links:**

| Resource | URL |
|---|---|
| Documentation site | https://LucaLonga.github.io/Tender-Camera/ |
| GitHub repository | https://github.com/LucaLonga/Tender-Camera |

---

## Hardware

### SBC — Libre Computer Sweet Potato (AML-S905X-CC-V2)

| Specification | Value |
|---|---|
| SoC | Amlogic S905X |
| CPU | Quad-Core ARM Cortex-A53 @ 1.5GHz |
| RAM | 2GB DDR3 |
| GPU | Mali-450 MP5 |
| USB Ports | 4× USB 2.0 (Type A) |
| Video Output | 1× HDMI 2.0 |
| Storage | MicroSD slot |
| Network | 1× 100Mbps Ethernet (RJ45) |
| Power | 5V/2A via USB-C |
| GPIO | 40-pin header (Raspberry Pi compatible layout) |
| Dimensions | 85mm × 56mm |

> ⚠️ The Sweet Potato has **no built-in WiFi or Bluetooth** — all wireless connectivity requires external USB adapters.

**USB port allocation (as of 2026-01-31):**

| Port | Device | USB ID |
|---|---|---|
| USB 1 | Genesys Logic Hub (internal) | `05e3:0610` |
| USB 2 | Sony IMX179 Camera | `0bda:5805` |
| USB 3 | Waveshare Touch Controller | `0eef:0005` |
| USB 4 | Trust Keyboard (dev only) | `145f:02c9` |

---

### Display — Waveshare 4.3inch HDMI LCD (B)

| Specification | Value |
|---|---|
| Size | 4.3 inches |
| Resolution | 800×480 (landscape native) |
| Planned orientation | Portrait (480×800 after rotation) |
| Panel | IPS, 160° viewing angle |
| Touch type | Capacitive, 5-point multitouch |
| Touch interface | USB — appears as `/dev/input/event0` |
| Display interface | HDMI |

> **Note on physical buttons:** The Menu, OK, Direction, Return, and Power buttons on the display are controlled entirely by the RTD2660H display controller. They are **not accessible to the OS** and cannot be repurposed as camera controls. For a physical shutter button, wire a tactile switch to the GPIO header and read it via `libgpiod`.

---

### Camera — Sony IMX179 USB Module

| Specification | Value |
|---|---|
| Model | DSJ-3879-HE |
| Sensor | Sony IMX179 |
| Sensor size | 1/3.2" |
| Resolution | 8 Megapixels (3264×2448) |
| Interface | USB 2.0 |
| Protocol | UVC — driver-free on Linux |
| Lens mount | M12 (interchangeable) |
| USB ID | `0bda:5805` |

**Supported formats (MJPG — recommended):**

| Resolution | FPS | Megapixels | Use case |
|---|---|---|---|
| 3264×2448 | 15 | 8.0 MP | Maximum quality stills |
| 1920×1080 | 30 | 2.1 MP | Full HD video / viewfinder |
| 1280×720 | 30 | 0.9 MP | HD video |
| 640×480 | 30 | 0.3 MP | Fast preview |

> MJPG compresses on the camera module itself — this is why high resolutions are possible over USB 2.0. Raw YUYV is limited to 640×480 at 30fps by USB bandwidth.

---

### Connection Diagram

```
                    ┌──────────────────────┐
                    │   Sweet Potato SBC   │
                    │  (AML-S905X-CC-V2)   │
                    │                      │
    Ethernet ───────┤ RJ45                 │
                    │ HDMI ────────────────┼──── Waveshare Display
                    │ USB1 ────────────────┼──── IMX179 Camera
                    │ USB2 ────────────────┼──── Waveshare Touch (USB)
                    │ USB3 ────────────────┼──── Trust Keyboard (dev only)
                    │ USB4 ────────────────┼──── [Future: WiFi Adapter]
                    │ USB-C (power) ───────┼──── 5V/3A PSU
                    │ MicroSD: 128GB       │
                    └──────────────────────┘
```

---

## Software Stack

### Operating System

| Property | Value |
|---|---|
| Distribution | Armbian 25.11.2 noble |
| Base | Ubuntu 24.04 LTS |
| Kernel | 6.12.58-current-meson64 |
| Architecture | aarch64 |
| Package manager | apt |
| Hostname | sweet-potato |
| User | mycamera |

> **Historical note:** Earlier documentation described the system as running Arch Linux ARM with `pacman`. The system now runs Armbian (Ubuntu-based). All `pacman` commands in older notes should be replaced with `apt` equivalents.

---

### Packages to Install

```bash
sudo apt update

# Camera capture
sudo apt install fswebcam

# Input device debugging
sudo apt install evtest

# Framebuffer image viewer
sudo apt install fbi

# UI framework (Phase 2)
sudo apt install python3-kivy

# Computer vision (Phase 4)
sudo apt install python3-opencv

# WiFi driver compilation (when needed)
sudo apt install linux-headers-$(uname -r)
```

**What is already present in Armbian (no installation needed):**
- `v4l-utils` — camera querying tools (`v4l2-ctl`)
- `uvcvideo` — kernel driver for USB cameras
- `gcc`, `make`, `git` — build tools
- `evtest` — input device testing

---

### Key Package Explanations

**`uvcvideo` (kernel built-in)**
The USB Video Class driver. Any camera that follows the UVC standard (virtually all modern USB webcams) is supported automatically. The Sony IMX179 is UVC-compliant, so it appeared as `/dev/video1` immediately on boot with no configuration.

**`v4l-utils`**
Video4Linux utilities. The `v4l2-ctl` command lets you list detected cameras, query supported resolutions and pixel formats, and set camera parameters from the terminal. Essential for verifying the camera is working before writing any application code.

**`fswebcam`**
A minimal command-line tool for capturing still images from a V4L2 camera. Handles format negotiation and JPEG saving in a single command. Ideal for the early MVP phase — no Python or GUI required.

**`evtest`**
Reads raw Linux input events from `/dev/input/event*` devices. Used to verify the touchscreen is sending correct coordinates (`ABS_X`, `ABS_Y`) and touch events (`BTN_TOUCH`) before writing any UI code.

**`fbi`**
Framebuffer image viewer. Displays images directly on screen without needing a desktop environment — important for a headless kiosk device.

**`python3-kivy`**
A touch-native Python UI framework. Can render directly to the framebuffer without X11. UI layouts are written in declarative `.kv` files (similar to CSS), separating design from logic. This is the planned framework for the camera application.

---

## Camera Setup

### Device mapping

| Device | Description | Use? |
|---|---|---|
| `/dev/video0` | Amlogic hardware video decoder (SoC built-in) | ❌ NOT the camera |
| `/dev/video1` | Sony IMX179 camera (uvcvideo) | ✅ USE THIS |
| `/dev/video2` | Camera metadata stream | Rarely needed |

> ⚠️ Always use `/dev/video1`. The SoC's hardware video decoder also creates a `/dev/video0` device — it is not the camera.

### Verified working commands

```bash
# List all video devices
v4l2-ctl --list-devices

# Check supported formats and resolutions
v4l2-ctl -d /dev/video1 --list-formats-ext

# Capture a full 8MP still
fswebcam -d /dev/video1 -r 3264x2448 --no-banner photo.jpg

# Capture with timestamp filename
fswebcam -d /dev/video1 -r 1920x1080 --no-banner "photo_$(date +%Y-%m-%d_%H-%M-%S).jpg"

# View image on the display
sudo fbi -T 1 -a photo.jpg   # press q to quit
```

**First capture milestone:** January 31, 2026, 17:58 — `test_8mp.jpg` — 2.8MB — 3264×2448.

---

## Touchscreen Setup

### Device mapping

| Device | Name | Use? |
|---|---|---|
| `/dev/input/event0` | WaveShare WS170120 | ✅ Touchscreen |
| `/dev/input/event1–3` | SIGMACHIP Trust Keyboard | Development only |
| `/dev/input/event4` | meson-ir (IR receiver) | Not used |

### Testing touch input

```bash
# List input devices
sudo evtest

# Select device 0 (WaveShare WS170120)
# Touch the screen — you should see ABS_X, ABS_Y, and BTN_TOUCH events
# Press Ctrl+C to stop
```

The touchscreen was confirmed fully working on 2026-01-31.

---

## Network Notes (FHNW University Network)

ICMP (ping) is blocked by the university firewall. **Do not use `ping` to test internet connectivity.** Use these instead:

```bash
sudo apt update              # Confirms DNS + HTTP — best test
resolvectl query google.com  # Tests DNS only
curl -I https://google.com   # Tests HTTP only
```

---

## UI/UX Plan

### Feature phases

| Phase | Features |
|---|---|
| **Phase 1 — MVP** | Touch capture button, status indicators, kiosk boot, CLI escape hatch |
| **Phase 2** | Live viewfinder, photo gallery, delete controls |
| **Phase 3** | Video mode, timelapse, settings panel |
| **Phase 4** | Face detection, object recognition, custom metadata, filters |

### UI framework decision

After evaluating PyGame, GTK4, Qt/QML, and a web kiosk approach, **Python + Kivy** was selected:
- Native touch support, designed for embedded interfaces
- Renders directly to framebuffer — no X11 or desktop environment needed
- `.kv` layout files allow UI design without touching Python code
- Integrates well with OpenCV for Phase 4 features

### Screen layout (portrait 480×800)

Screens to design: Capture (main), Gallery, Photo View, Settings, Boot/Splash.

Touch targets must be at least 44×44px. Aesthetic: permacomputing-inspired — minimal, functional, deliberate.

---

## Design Decisions

| Decision | Choice | Reasoning |
|---|---|---|
| Screen orientation | Portrait (480×800) | Distinctive camera feel |
| Photo storage | Internal SD card | 111GB available; USB ports needed for peripherals |
| File naming | Timestamp (`YYYY-MM-DD_HH-MM-SS.jpg`) | Sortable, no conflicts |
| Photo directory | `/home/mycamera/photos/` | Simple, within user home |
| UI framework | Python + Kivy | Touch-native, low resources, framebuffer mode |
| Boot mode | Kiosk (direct to app) | Feels like a dedicated camera |
| Debug access | Keyboard at boot → CLI | Keeps the device hackable |

---

## Known Issues

| Issue | Status |
|---|---|
| WiFi adapter not yet installed | Pending — needs `linux-headers` + driver compilation |
| No UI yet | In planning — framework confirmed, spec written |
| `ping` blocked on FHNW network | Resolved — use `apt update` instead |

---

---

# the Orange Pi 5 build — Orange Pi 5 (Rockchip RK3588S)

> **Author:** René
> 
> A step-by-step guide to building a DIY camera using the Orange Pi 5 SBC with an OV13850 MIPI camera module, Waveshare 3.5" HDMI touchscreen display, and a custom Python camera application.

---

## Project Overview

This build uses the **Orange Pi 5** as its SBC and a **13MP OV13850 MIPI camera** for higher resolution capture. The main additional complexity over the Sweet Potato build comes from two places: the MIPI camera requires a Device Tree overlay to be recognised by the kernel, and the resistive touchscreen requires a custom userspace driver since the kernel ADS7846 module is disabled in this image.

The result is a fully self-contained camera with live preview, still capture, and H.264 video recording — all controllable by touch.

---

## Hardware Requirements

| Component | Details |
|---|---|
| SBC | Orange Pi 5 (RK3588S) |
| Camera | OV13850 13MP MIPI — official Orange Pi module |
| Camera cable | FPC ribbon cable (included) |
| Display | Waveshare 3.5" HDMI LCD (B) — 800×480 resistive touch |
| Display connection | HDMI cable + 26-pin GPIO header |
| Power | 5V/4A USB-C |
| Storage | MicroSD 16GB+ or eMMC |

> ⚠️ The **FPC ribbon cable** is fragile and the most common hardware failure. If the camera is not detected, replace the cable before assuming a software issue.

> ⚠️ The display uses **HDMI for video** and the **26-pin GPIO header for touch** (SPI). The micro USB port on the display is power only — it does not carry touch data.

---

## Software & System Info

| Item | Details |
|---|---|
| OS image | `Orangepi5_1.2.2_ubuntu_jammy_desktop_xfce_linux6.1.99` |
| OS | Ubuntu 22.04 LTS (Jammy) |
| Kernel | `6.1.99-rockchip-rk3588` |
| Desktop | XFCE |
| Python | 3.10 |
| GStreamer | 1.20.3 |
| OpenCV | 4.5.4 |

Download the OS image from the [official Orange Pi 5 page](http://www.orangepi.org/html/hardWare/computerAndMicrocontrollers/service-and-support/Orange-pi-5.html). Flash with balenaEtcher or Win32DiskImager.

---

## Wiring Reference — Touch (SPI4)

The Waveshare display connects to the Orange Pi 5 26-pin header:

| Physical Pin | Function | WiringPi # | Display Signal |
|---|---|---|---|
| 19 | SPI4_TXD (MOSI) | — | TP_SI |
| 21 | SPI4_RXD (MISO) | — | TP_SO |
| 23 | SPI4_CLK | — | TP_SCK |
| 22 | GPIO2_D4 | **13** | TP_IRQ |
| 26 | PWM1 | **16** | TP_CS |
| 6/9/14/20/25 | GND | — | GND |
| 1/17 | 3.3V | — | VCC |

> ⚠️ **Critical:** TP_CS is on **pin 26**, not pin 24 (where SPI4_CS1 is). The XPT2046 will not respond without CS being driven on pin 26. This is handled in software via Patch 3 below.

---

## Step 1 — OS Setup

### 1.1 Flash and boot

Flash the Ubuntu image to a microSD card. Default credentials: `orangepi` / `orangepi`.

### 1.2 Update the system

```bash
sudo apt update && sudo apt upgrade -y
```

### 1.3 Enable Device Tree overlays

```bash
sudo nano /boot/orangepiEnv.txt
```

Set the overlays line to:

```
overlays=ov13850-c1 spi4-m0-cs1-spidev
```

> ⚠️ **Never use `spi4-m1-cs1-spidev`** — the M1 mux conflicts with the ethernet interface and will make the network adapter disappear after reboot.

```bash
sudo reboot
```

Verify SPI device appeared:

```bash
ls /dev/spidev4.1
```

---

## Step 2 — Camera Setup

### 2.1 Connect the camera

Connect the OV13850 to the **CAM1** port using the FPC ribbon cable. Ensure the cable is fully seated and latches are closed on both ends.

### 2.2 Verify the overlay loaded

```bash
dmesg | grep ov13850
```

Expected output:

```
ov13850 7-0010: Detected ov13850 sensor, CHIP ID = 0x138500
```

If you see `Unexpected sensor id(000000), ret(-5)` — replace the FPC cable.

### 2.3 Verify the video device

```bash
v4l2-ctl --list-devices
```

The correct capture device is `/dev/video11` (under `rkisp_mainpath`).

### 2.4 Test capture

```bash
gst-launch-1.0 v4l2src device=/dev/video11 io-mode=4 ! \
  video/x-raw,format=NV12,width=1920,height=1080,framerate=30/1 ! \
  mpph264enc header-mode=1 bps=20000000 ! \
  h264parse config-interval=-1 ! matroskamux ! \
  filesink location=~/test.mkv
```

Press `Ctrl+C` to stop. Play with `mpv ~/test.mkv`.

> Maximum capture resolution via the app is **2112×1568** (ISP V4L2 limit). The physical sensor supports up to 4208×3120 but requires ISP tuning.

---

## Step 3 — Display Setup

The Waveshare 3.5" HDMI LCD is plug-and-play for video — HDMI is detected automatically at 800×480. No configuration needed.

> **Do not use Raspberry Pi LCD-show scripts.** They modify `/boot/config.txt` and install RPi-specific overlays that are incompatible with Orange Pi / RK3588.

---

## Step 4 — Touch Input Setup

The resistive touch controller (XPT2046) has no kernel driver in this build — `CONFIG_TOUCHSCREEN_ADS7846` is compiled out. Instead, a userspace daemon reads raw SPI data and injects X11 pointer events.

### 4.1 Install build dependencies

```bash
sudo apt install libx11-dev libxtst-dev -y
```

### 4.2 Install WiringOP

```bash
git clone https://github.com/zhaolei/WiringOP.git -b h3
cd WiringOP
chmod +x ./build
./build
```

### 4.3 Clone and patch the ADS7846 daemon

```bash
git clone https://github.com/Tomasz-Mankowski/ADS7846-X11-daemon.git
cd ADS7846-X11-daemon
```

**Patch 1 — Fix C++ pointer comparison error (required to compile):**

```bash
sed -i 's/if (gc < 0)/if (gc == nullptr)/' main.cpp
```

**Patch 2 — Extend calibration timeout to 60 seconds:**

```bash
sed -i 's/int waitTime = 5;/int waitTime = 60;/' main.cpp
```

**Patch 3 — Add software chip-select on pin 26 (critical — without this, touch never responds):**

```bash
python3 - << 'EOF'
content = open('ADS7846.cpp').read()
content = content.replace(
    '#include "ADS7846.h"',
    '#include "ADS7846.h"\n#include <wiringPi.h>\n#define CS_PIN 16'
)
content = content.replace(
    '\tioctl(spiHandler_, SPI_IOC_MESSAGE(1), spi_message);',
    '\tpinMode(CS_PIN, OUTPUT);\n\tdigitalWrite(CS_PIN, LOW);\n\tioctl(spiHandler_, SPI_IOC_MESSAGE(1), spi_message);\n\tdigitalWrite(CS_PIN, HIGH);'
)
open('ADS7846.cpp', 'w').write(content)
print('Done')
EOF
```

### 4.4 Compile

```bash
make
```

### 4.5 Calibrate

```bash
sudo DISPLAY=:0.0 XAUTHORITY=/home/orangepi/.Xauthority \
  bash -c 'cd /home/orangepi/ADS7846-X11-daemon && \
  ./ADS7846-X11 --spi /dev/spidev4.1 --pin 13 --cal'
```

Tap each of the 3 crosshair targets. Verify calibration succeeded (values should not be `8191;8191`):

```bash
cat ~/ADS7846-X11-daemon/calibpoints.cal
```

### 4.6 Test touch

```bash
sudo DISPLAY=:0.0 XAUTHORITY=/home/orangepi/.Xauthority \
  bash -c 'cd /home/orangepi/ADS7846-X11-daemon && \
  ./ADS7846-X11 --spi /dev/spidev4.1 --pin 13'
```

Touch the screen — the cursor should follow your finger.

---

## Step 5 — Camera App

### 5.1 Install Python dependencies

```bash
sudo apt install python3-opencv -y
```

Verify GStreamer bindings:

```bash
python3 -c "import gi; gi.require_version('Gst','1.0'); from gi.repository import Gst; Gst.init(None); print('OK')"
```

### 5.2 Run the app

```bash
python3 ~/camera_app.py /dev/video11
```

### 5.3 Controls

| Key | Action |
|---|---|
| `P` | Take photo |
| `V` | Start / stop video recording |
| `S` | Open settings |
| `Q` | Quit |

Output is saved to `~/Pictures/camera/`.

---

## Step 6 — Autostart

Create an autostart entry for the touch daemon:

```bash
mkdir -p ~/.config/autostart
cat > ~/.config/autostart/touch.desktop << 'EOF'
[Desktop Entry]
Type=Application
Name=Touch Daemon
Exec=bash -c 'sleep 8 && DISPLAY=:0.0 XAUTHORITY=/home/orangepi/.Xauthority cd /home/orangepi/ADS7846-X11-daemon && sudo ./ADS7846-X11 --spi /dev/spidev4.1 --pin 13'
Hidden=false
NoDisplay=false
X-GNOME-Autostart-enabled=true
EOF
```

Allow it to run without a password prompt:

```bash
echo 'orangepi ALL=(ALL) NOPASSWD: /home/orangepi/ADS7846-X11-daemon/ADS7846-X11' | \
  sudo tee /etc/sudoers.d/touch
```

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `Unexpected sensor id(000000)` | Faulty FPC cable | Replace the ribbon cable |
| `/dev/spidev4.1` missing | Wrong overlay | Check `orangepiEnv.txt` for `spi4-m0-cs1-spidev` |
| Touch returns `8191;8191` | CS pin not driven | Ensure Patch 3 was applied before compiling |
| Touch always clicks top-left | Bad calibration file | `sudo rm calibpoints.cal` and recalibrate |
| Ethernet disappears after reboot | Wrong SPI overlay | Change `spi4-m1` → `spi4-m0` in `orangepiEnv.txt` |
| No camera preview in app | GStreamer issue | Verify `io-mode=4` in pipeline; reinstall `python3-opencv` |
| Touch daemon doesn't start | Sleep too short | Increase `sleep 8` to `sleep 15` in autostart |

---

## Known Limitations

| Limitation | Details |
|---|---|
| Max app resolution | 2112×1568 (ISP V4L2 limit) |
| No H.265 | `mpph265enc` not available in this GStreamer build |
| No kernel touch driver | `CONFIG_TOUCHSCREEN_ADS7846=n` — userspace daemon required |
| No kernel headers | Cannot compile external kernel modules on this image |
| Touch requires X11 | Calibration and daemon require a running X session |

---

## FAQ

**Q: Can I use Raspberry Pi LCD-show scripts?**
No — they target `brcm,bcm2835` and are incompatible with RK3588.

**Q: Why is the ADS7846 kernel module unavailable?**
It was compiled out (`CONFIG_TOUCHSCREEN_ADS7846=n`). Enabling it requires recompiling the kernel, which also requires kernel headers that aren't distributed for this image. The userspace daemon bypasses this entirely.

**Q: Can I use CAM2 or CAM3?**
Yes — use overlay `ov13850-c2` or `ov13850-c3` accordingly.

**Q: Does recording freeze the preview?**
No — the app uses a GStreamer `tee` element to split the stream so preview and recording run simultaneously.

**Q: Can I add a physical shutter button?**
Yes — connect a button between pin 7 (GPIO1_C6) and any GND pin, install `python3-opi.gpio`, and the app detects it automatically.

---

---

## Licence

The documentation in this repository is released under **CC BY 4.0** — feel free to adapt and share with attribution.

The ADS7846-X11-daemon is by Tomasz Mankowski — see the [original repository](https://github.com/Tomasz-Mankowski/ADS7846-X11-daemon) for its licence.

---

*Contributions and corrections welcome via GitHub Issues.*
