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

# Sweet Potato — Libre Computer AML-S905X-CC-V2

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
# Orange Pi 5 — Rockchip RK3588S

> **Author:** René
>
> A step-by-step guide to building a DIY camera using the Orange Pi 5 SBC with an OV13850 MIPI camera module, Waveshare 3.5" HDMI touchscreen display, and a custom Python camera application.

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Hardware Requirements](#2-hardware-requirements)
3. [Software & System Info](#3-software--system-info)
4. [Reference Documents & Links](#4-reference-documents--links)
5. [Wiring Reference](#5-wiring-reference)
6. [Step 1 — OS Setup](#6-step-1--os-setup)
7. [Step 2 — Camera Setup](#7-step-2--camera-setup)
8. [Step 3 — Display Setup](#8-step-3--display-setup)
9. [Step 4 — Touch Input Setup](#9-step-4--touch-input-setup)
10. [Step 5 — Camera App Installation](#10-step-5--camera-app-installation)
11. [Step 6 — Autostart Configuration](#11-step-6--autostart-configuration)
12. [Troubleshooting](#12-troubleshooting)
13. [Known Limitations](#13-known-limitations)
14. [FAQ](#14-faq)
15. [Changelog](#15-changelog)

---

## 1. Project Overview

This guide documents the full process of building a standalone DIY camera based on the **Orange Pi 5** single-board computer. The system consists of:

- A **13MP OV13850 MIPI camera** for photo and video capture
- A **3.5" Waveshare HDMI LCD** as a live preview display
- **Resistive touchscreen** input via the XPT2046/ADS7846 controller driven by a userspace SPI daemon
- A **custom Python camera application** with live preview, photo capture, and video recording

The goal is a fully self-contained camera that works without a keyboard or mouse, using only touch input for operation.

> 📸 **[Photo placeholder — insert hardware overview photo here]**

> 🎬 **[Video demo placeholder — insert YouTube link here]**

---

## 2. Hardware Requirements

| Component | Model / Details |
|---|---|
| SBC | Orange Pi 5 (RK3588S) |
| Camera Module | OV13850 13MP MIPI — official Orange Pi module |
| Camera Cable | FPC ribbon cable (included with camera module) |
| Display | Waveshare 3.5" HDMI LCD (B) — 800×480 IPS resistive touch |
| Display Connection | HDMI cable + 26-pin GPIO header |
| Power | 5V/4A USB-C power supply |
| Storage | MicroSD card (16GB+) or eMMC |
| Optional | USB keyboard and mouse (for initial setup only) |

### Notes on hardware

- The **FPC ribbon cable** is fragile and a common failure point. If the camera is not detected, try a replacement cable before assuming software issues.
- The Waveshare display uses **HDMI for video** and the **26-pin GPIO header for touch** (SPI). The micro USB port on the display is power only — it does NOT carry touch data.
- The touch controller is **XPT2046** (compatible with ADS7846 driver).

---

## 3. Software & System Info

| Item | Details |
|---|---|
| OS Image | `Orangepi5_1.2.2_ubuntu_jammy_desktop_xfce_linux6.1.99` |
| OS | Ubuntu 22.04 LTS (Jammy) |
| Kernel | `6.1.99-rockchip-rk3588` |
| Desktop | XFCE |
| Python | 3.10 |
| GStreamer | 1.20.3 |
| OpenCV | 4.5.4 (python3-opencv) |

### Download the OS image

The official Orange Pi 5 Ubuntu image can be downloaded from:
- [http://www.orangepi.org/html/hardWare/computerAndMicrocontrollers/service-and-support/Orange-pi-5.html](http://www.orangepi.org/html/hardWare/computerAndMicrocontrollers/service-and-support/Orange-pi-5.html)

Flash using **balenaEtcher** or **Win32DiskImager** to a microSD card.

---

## 4. Reference Documents & Links

### Official Documentation
- **Orange Pi 5 Official Page & Manuals** — [orangepi.org](http://www.orangepi.org/html/hardWare/computerAndMicrocontrollers/service-and-support/Orange-pi-5.html)
- **Waveshare 3.5" HDMI LCD Wiki & FAQ** — [waveshare.com](https://www.waveshare.com/wiki/3.5inch_HDMI_LCD#FAQ)

### Drivers & Tools Used
- **ADS7846-X11-daemon** (userspace SPI touch driver for Orange Pi) — [github.com/Tomasz-Mankowski/ADS7846-X11-daemon](https://github.com/Tomasz-Mankowski/ADS7846-X11-daemon)
- **WiringOP** (GPIO library for Orange Pi) — [github.com/zhaolei/WiringOP](https://github.com/zhaolei/WiringOP)

### Community Resources
- **OV13850 Camera discussion for Orange Pi 5** — [reddit.com/r/OrangePI](https://www.reddit.com/r/OrangePI/comments/109cgz8/camera_for_orange_pi_5/)

### Important Notes on Compatibility
- The **Waveshare LCD-show** scripts ([github.com/waveshareteam/LCD-show](https://github.com/waveshareteam/LCD-show)) and **LCD-show-ubuntu** ([github.com/lcdwiki/LCD-show-ubuntu](https://github.com/lcdwiki/LCD-show-ubuntu)) are **Raspberry Pi specific** and will NOT work on Orange Pi 5.
- The `waveshare-ads7846.dtbo` file distributed by Waveshare targets `brcm,bcm2835` and is **not compatible** with RK3588.
- The ADS7846 kernel driver (`CONFIG_TOUCHSCREEN_ADS7846`) is **disabled** in the Orange Pi 6.1 kernel and cannot be used without recompiling the kernel. The userspace daemon approach used in this guide bypasses this limitation entirely.

---

## 5. Wiring Reference

### 26-Pin GPIO Header — SPI4 Pinout (Touch)

The Waveshare display connects to the Orange Pi 5 26-pin header. The relevant SPI4 (M0) pins are:

| Physical Pin | Function | GPIO | WiringPi# | Display Signal |
|---|---|---|---|---|
| 19 | SPI4_TXD (MOSI) | GPIO1_B1 | — | TP_SI |
| 21 | SPI4_RXD (MISO) | GPIO1_B0 | — | TP_SO |
| 23 | SPI4_CLK | GPIO1_B2 | — | TP_SCK |
| 24 | SPI4_CS1 | GPIO1_B4 | — | (CS bridge — see note) |
| 22 | GPIO2_D4 | GPIO 92 | **13** | TP_IRQ |
| 26 | PWM1 | GPIO 35 | **16** | TP_CS |
| 6/9/14/20/25 | GND | — | — | GND |
| 1/17 | 3.3V | — | — | VCC |

> ⚠️ **Critical CS Pin Note:** The Waveshare display routes TP_CS to **pin 26** on the connector. However, SPI4_CS1 is on **pin 24**. The XPT2046 will not respond without CS being driven correctly.
>
> **Solution implemented in software:** The ADS7846-X11-daemon was modified to manually toggle pin 26 (WiringPi pin 16) as a software CS signal before and after each SPI transaction. This eliminates the need for any hardware bridging.

### Camera Connector

The OV13850 camera module connects to the **CAM1** port on the Orange Pi 5 board using the included FPC ribbon cable.

> ⚠️ **CAM1 requires overlay `ov13850-c1`** — do not use c2 or c3 overlays for this connector.

---

## 6. Step 1 — OS Setup

### 1.1 Flash the image

Flash the Ubuntu 22.04 image to a microSD card. Default credentials:

```
Username: orangepi
Password: orangepi
```

### 1.2 First boot

Connect keyboard, mouse, and HDMI monitor. Complete initial setup and connect to your network.

### 1.3 Update the system

```bash
sudo apt update && sudo apt upgrade -y
```

### 1.4 Enable SPI4 overlay

Edit the boot configuration:

```bash
sudo nano /boot/orangepiEnv.txt
```

Add or modify the `overlays` line. Final value used in this project:

```
overlays=ov13850-c1 spi4-m0-cs1-spidev
```

> ⚠️ Do **not** add `spi4-m1-cs1-spidev` — the M1 mux conflicts with the ethernet interface and will cause the network adapter to disappear.

Save and reboot:

```bash
sudo reboot
```

Verify SPI device appeared:

```bash
ls /dev/spidev4.1
```

---

## 7. Step 2 — Camera Setup

### 7.1 Connect the camera

Connect the OV13850 module to the **CAM1** port using the FPC ribbon cable. Ensure the cable is fully seated and the latch is closed on both ends.

> ⚠️ **FPC cable failure is the most common hardware issue.** If the camera is not detected after correct software setup, replace the cable first.

### 7.2 Verify the overlay is loaded

After reboot with `ov13850-c1` in overlays:

```bash
dmesg | grep ov13850
```

Expected output includes sensor ID `0x138500`:

```
ov13850 7-0010: driver version: 00.01.05
ov13850 7-0010: Detected ov13850 sensor, CHIP ID = 0x138500
```

If you see `Unexpected sensor id(000000), ret(-5)` — the camera is not communicating. Check the FPC cable.

### 7.3 Verify the video device

```bash
v4l2-ctl --list-devices
```

The correct capture device is:

```
rkisp_mainpath (platform:rkisp0-vir2):
    /dev/video11
```

### 7.4 Test video capture

```bash
gst-launch-1.0 v4l2src device=/dev/video11 io-mode=4 ! \
  video/x-raw,format=NV12,width=1920,height=1080,framerate=30/1 ! \
  mpph264enc header-mode=1 bps=20000000 ! \
  h264parse config-interval=-1 ! matroskamux ! \
  filesink location=~/test.mkv
```

Press Ctrl+C to stop. Play with:

```bash
mpv ~/test.mkv
```

### 7.5 Supported resolutions

The camera outputs up to **2112×1568** via the ISP mainpath (V4L2 limit). The physical sensor maximum is 4208×3120 but this requires ISP tuning beyond the scope of this guide.

---

## 8. Step 3 — Display Setup

### 8.1 Physical connections

1. Connect HDMI cable from display to Orange Pi 5
2. Connect display 26-pin connector to Orange Pi 5 26-pin header (align pin 1)
3. The micro USB on the display can be connected for power but is **not required**

### 8.2 Display configuration

The Waveshare 3.5" HDMI LCD is plug-and-play for video on Orange Pi 5. No additional configuration is needed — HDMI is detected automatically at 800×480.

### 8.3 Verify display

The display should show the XFCE desktop after boot. If it shows a white screen or no signal, check the HDMI cable connection.

---

## 9. Step 4 — Touch Input Setup

### 9.1 Install dependencies

```bash
sudo apt install libx11-dev libxtst-dev -y
```

### 9.2 Install WiringOP

```bash
git clone https://github.com/zhaolei/WiringOP.git -b h3
cd WiringOP
chmod +x ./build
./build
```

### 9.3 Clone and patch the ADS7846 daemon

```bash
git clone https://github.com/Tomasz-Mankowski/ADS7846-X11-daemon.git
cd ADS7846-X11-daemon
```

Apply the required patches:

**Patch 1 — Fix C++ pointer comparison (compile error fix):**

```bash
sed -i 's/if (gc < 0)/if (gc == nullptr)/' main.cpp
```

**Patch 2 — Increase calibration timeout (allows enough time to tap calibration points):**

```bash
sed -i 's/int waitTime = 5;/int waitTime = 60;/' main.cpp
```

**Patch 3 — Software CS on pin 26 (critical for Waveshare display):**

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

### 9.4 Compile

```bash
make
```

### 9.5 Calibrate

```bash
sudo DISPLAY=:0.0 XAUTHORITY=/home/orangepi/.Xauthority \
  bash -c 'cd /home/orangepi/ADS7846-X11-daemon && \
  ./ADS7846-X11 --spi /dev/spidev4.1 --pin 13 --cal'
```

A fullscreen calibration window will appear. Tap each of the 3 cross targets accurately. After completion, verify the calibration file has real values (not `8191;8191`):

```bash
cat ~/ADS7846-X11-daemon/calibpoints.cal
```

Expected output (values will differ based on your calibration):

```
3458;3466
2065;808
684;2134
```

### 9.6 Test touch

```bash
sudo DISPLAY=:0.0 XAUTHORITY=/home/orangepi/.Xauthority \
  bash -c 'cd /home/orangepi/ADS7846-X11-daemon && \
  ./ADS7846-X11 --spi /dev/spidev4.1 --pin 13'
```

Touch the screen. The mouse cursor should follow your finger.

---

## 10. Step 5 — Camera App Installation

### 10.1 Install Python dependencies

```bash
sudo apt install python3-opencv -y
```

Verify GStreamer Python bindings:

```bash
python3 -c "import gi; gi.require_version('Gst','1.0'); from gi.repository import Gst; Gst.init(None); print('OK')"
```

### 10.2 Install the camera app

Download `camera_app.py` from this repository and copy to your home directory:

```bash
cp camera_app.py ~/camera_app.py
```

### 10.3 Run the camera app

```bash
python3 ~/camera_app.py /dev/video11
```

### 10.4 Controls

| Key | Action |
|---|---|
| `P` | Take photo |
| `V` | Start / stop video recording |
| `S` | Open settings menu |
| `Q` | Quit |

In the settings menu:

| Key | Action |
|---|---|
| `1–4` | Select resolution |
| `6–8` | Select photo format (JPEG / PNG / TIFF) |
| `A–B` | Select video format (MP4 H.264 / MKV H.264) |
| `S` | Close settings |

### 10.5 Output files

All photos and videos are saved to:

```
~/Pictures/camera/
```

---

## 11. Step 6 — Autostart Configuration

### 11.1 Touch daemon autostart

Create an autostart entry so the touch daemon launches automatically after login:

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

Add sudoers rule so the daemon runs without a password prompt:

```bash
echo 'orangepi ALL=(ALL) NOPASSWD: /home/orangepi/ADS7846-X11-daemon/ADS7846-X11' | \
  sudo tee /etc/sudoers.d/touch
```

The 8-second sleep gives the X session time to fully initialize before the daemon starts.

### 11.2 Verify autostart

Reboot and confirm touch works without manually starting the daemon:

```bash
sudo reboot
```

---

## 12. Troubleshooting

### Camera not detected — `Unexpected sensor id(000000), ret(-5)`

The sensor is not responding to I2C. Causes in order of likelihood:

1. **Faulty FPC cable** — the most common cause. Replace the ribbon cable.
2. **Cable not fully seated** — power off, reseat both ends, close latches fully.
3. **Wrong connector** — the camera must be in **CAM1**, not CAM2 or CAM3.
4. **Wrong overlay** — verify `/boot/orangepiEnv.txt` contains `ov13850-c1`.

### `/dev/spidev4.1` not appearing

Check `/boot/orangepiEnv.txt` contains `spi4-m0-cs1-spidev` in the overlays line.

> ⚠️ Never use `spi4-m1-cs1-spidev` — it conflicts with ethernet and removes the network interface.

### Touch calibration returns `8191;8191`

The XPT2046 is not returning real touch data. This is caused by the CS pin not being driven correctly. Ensure **Patch 3** (software CS on pin 26) was applied before compiling.

### Touch always clicks top-left corner

The calibration file contains bad data (`8191;8191`). Delete it and recalibrate:

```bash
sudo rm ~/ADS7846-X11-daemon/calibpoints.cal
# Then run calibration again (Step 9.5)
```

### Ethernet disappears after reboot

You enabled the wrong SPI overlay. Edit `/boot/orangepiEnv.txt` and change `spi4-m1-cs1-spidev` to `spi4-m0-cs1-spidev`, then reboot.

### Camera app opens but shows no preview

OpenCV cannot open the MIPI camera directly. The app uses a GStreamer pipeline with `io-mode=4`. Make sure `python3-opencv` is installed and GStreamer bindings are working (Step 10.1).

### MP4 video file is corrupt / unplayable

MP4 requires a clean EOS (end-of-stream) signal to finalize the moov atom. The camera app uses `gst-launch-1.0 -e` and sends `SIGINT` to the process group for a clean shutdown. If the process was killed forcefully, the MP4 will be unplayable. Use MKV format for more robust recording.

### Touch daemon doesn't start after reboot

The sleep delay may not be long enough for your system. Increase it:

```bash
nano ~/.config/autostart/touch.desktop
# Change: sleep 8
# To:     sleep 15
```

---

## 13. Known Limitations

| Limitation | Details |
|---|---|
| Max resolution via app | 2112×1568 (ISP V4L2 limit) |
| No H.265 video | `mpph265enc` element not available in this GStreamer build |
| No touch kernel driver | `CONFIG_TOUCHSCREEN_ADS7846` is disabled in the 6.1 kernel. Userspace daemon is used instead |
| No kernel headers | `linux-headers` package not available for this custom kernel, preventing external module compilation |
| Preview and recording share camera | A GStreamer `tee` pipeline is used so both preview and recording can use the camera simultaneously |
| Touch calibration GUI | The calibration tool requires X11 and must be run with correct `DISPLAY` and `XAUTHORITY` environment variables |

---

## 14. FAQ

**Q: Can I use a Raspberry Pi LCD-show script to set up the display?**
No. Those scripts modify `/boot/config.txt` and install RPi-specific device tree overlays that are incompatible with Orange Pi / RK3588.

**Q: Can I use the Waveshare `waveshare-ads7846.dtbo` overlay file?**
No. That file targets `brcm,bcm2835` (Raspberry Pi SoC) and will not load on RK3588.

**Q: Why is the ADS7846 kernel module not available?**
The Orange Pi 6.1 kernel was compiled with `CONFIG_TOUCHSCREEN_ADS7846=n`. Enabling it requires recompiling the kernel from source, which also requires kernel headers that are not distributed for this image. The userspace daemon approach bypasses this entirely.

**Q: Can I connect to CAM2 or CAM3 instead of CAM1?**
Yes, but you must change the overlay accordingly: `ov13850-c2` for CAM2 or `ov13850-c3` for CAM3.

**Q: Why does video recording freeze the preview?**
It doesn't — the app uses a GStreamer `tee` to split the stream so preview and recording run simultaneously from the same camera pipeline.

**Q: Can I add a physical shutter button?**
Yes. The camera app includes optional GPIO button support via the `OPi.GPIO` library. Connect a button between physical pin 7 (GPIO1_C6, WiringPi 2) and any GND pin. Install `python3-opi.gpio` and the app will detect it automatically.

---

## 15. Camera App Source Code

Save the following as `camera_app.py` in your home directory:

```python
#!/usr/bin/env python3
"""
OrangePi 5 Camera App
- Live preview always on via GStreamer tee
- Recording branches off tee without interrupting preview
- Photos: JPEG, PNG, TIFF
- Video:  MP4 H.264, MKV H.264
- Keys:   P=photo  V=record/stop  S=settings  Q=quit
- GPIO:   Pin 7 shutter (optional)
"""

import gi
gi.require_version('Gst', '1.0')
gi.require_version('GstApp', '1.0')
from gi.repository import Gst, GstApp, GLib

import cv2
import numpy as np
import threading
import os
import time
import signal
from datetime import datetime

Gst.init(None)

# ── Config ─────────────────────────────────────────────────────────────────────

DEVICE   = "/dev/video11"
SAVE_DIR = os.path.expanduser("~/Pictures/camera")

RESOLUTIONS = [
    ("2112x1568", 2112, 1568, 15),
    ("1920x1080", 1920, 1080, 30),
    ("1280x720",  1280,  720, 30),
    ("720x576",    720,  576, 30),
]

PHOTO_FMTS = ["JPEG", "PNG", "TIFF"]
VIDEO_FMTS = ["MP4 H.264", "MKV H.264"]

# Colors BGR
DARK   = (35,  35,  35)
PANEL  = (50,  50,  50)
WHITE  = (240, 240, 240)
GRAY   = (120, 120, 120)
GREEN  = (60,  200, 80)
RED    = (40,  40,  220)
YELLOW = (30,  210, 230)

# ── GPIO ───────────────────────────────────────────────────────────────────────

gpio_ok = False
GPIO_PIN = 7
try:
    import OPi.GPIO as GPIO
    GPIO.setmode(GPIO.BOARD)
    GPIO.setup(GPIO_PIN, GPIO.IN, pull_up_down=GPIO.PUD_UP)
    gpio_ok = True
except Exception:
    pass

# ── GStreamer pipeline ─────────────────────────────────────────────────────────

class Camera:
    def __init__(self, device, res_i):
        self.device  = device
        self.res_i   = res_i
        self.frame   = None
        self.lock    = threading.Lock()
        self.rec     = False
        self.rec_bin = None
        self.pipeline= None
        self.appsink = None
        self.tee     = None
        self.preview_queue = None
        self._build()

    def _build(self):
        label, w, h, fps = RESOLUTIONS[self.res_i]

        pipe_str = (
            f"v4l2src device={self.device} io-mode=4 ! "
            f"video/x-raw,format=NV12,width={w},height={h},framerate={fps}/1 ! "
            f"tee name=t "
            f"t. ! queue max-size-buffers=2 leaky=downstream ! "
            f"videoconvert ! video/x-raw,format=BGR ! "
            f"appsink name=preview emit-signals=true drop=true max-buffers=2 "
            f"t. ! queue name=rec_queue max-size-buffers=2 leaky=downstream ! "
            f"fakesink name=rec_sink"
        )

        self.pipeline = Gst.parse_launch(pipe_str)
        self.appsink  = self.pipeline.get_by_name("preview")
        self.tee      = self.pipeline.get_by_name("t")
        self.rec_sink = self.pipeline.get_by_name("rec_sink")
        self.rec_queue= self.pipeline.get_by_name("rec_queue")

        self.appsink.connect("new-sample", self._on_frame)

        bus = self.pipeline.get_bus()
        bus.add_signal_watch()
        bus.connect("message::error", self._on_error)

        self.pipeline.set_state(Gst.State.PLAYING)

    def _on_frame(self, sink):
        sample = sink.emit("pull-sample")
        if sample:
            buf  = sample.get_buffer()
            caps = sample.get_caps()
            w    = caps.get_structure(0).get_value("width")
            h    = caps.get_structure(0).get_value("height")
            ok, mapinfo = buf.map(Gst.MapFlags.READ)
            if ok:
                arr = np.frombuffer(mapinfo.data, dtype=np.uint8).reshape((h, w, 3)).copy()
                buf.unmap(mapinfo)
                with self.lock:
                    self.frame = arr
        return Gst.FlowReturn.OK

    def _on_error(self, bus, msg):
        err, dbg = msg.parse_error()
        print(f"[GST ERROR] {err}: {dbg}")

    def get_frame(self):
        with self.lock:
            return self.frame.copy() if self.frame is not None else None

    def start_recording(self, path, fmt):
        if self.rec:
            return

        if "MP4" in fmt:
            enc_mux = ("mpph264enc header-mode=1 bps=20000000 ! "
                       "h264parse config-interval=-1 ! mp4mux fragment-duration=1000")
        else:
            enc_mux = ("mpph264enc header-mode=1 bps=20000000 ! "
                       "h264parse config-interval=-1 ! matroskamux")

        label, w, h, fps = RESOLUTIONS[self.res_i]

        bin_str = (
            f"videoconvert ! "
            f"video/x-raw,format=NV12,width={w},height={h} ! "
            f"{enc_mux} ! "
            f"filesink location={path}"
        )

        try:
            rec_bin = Gst.parse_bin_from_description(bin_str, True)
            self.pipeline.add(rec_bin)
            tee_src = self.tee.get_request_pad("src_%u")
            rec_sink_pad = rec_bin.get_static_pad("sink")
            tee_src.link(rec_sink_pad)
            rec_bin.sync_state_with_parent()
            self.rec_bin  = rec_bin
            self.tee_pad  = tee_src
            self.rec      = True
            print(f"[CAM] Recording started: {path}")
        except Exception as e:
            print(f"[CAM] Record start error: {e}")

    def stop_recording(self):
        if not self.rec or not self.rec_bin:
            return
        try:
            self.tee_pad.send_event(Gst.Event.new_eos())
            time.sleep(0.5)
            self.tee.release_request_pad(self.tee_pad)
            self.rec_bin.set_state(Gst.State.NULL)
            self.pipeline.remove(self.rec_bin)
        except Exception as e:
            print(f"[CAM] Record stop error: {e}")
        finally:
            self.rec_bin = None
            self.tee_pad = None
            self.rec     = False
            print("[CAM] Recording stopped")

    def restart(self, res_i):
        self.pipeline.set_state(Gst.State.NULL)
        self.pipeline = None
        self.res_i    = res_i
        self._build()

    def stop(self):
        if self.rec:
            self.stop_recording()
        if self.pipeline:
            self.pipeline.set_state(Gst.State.NULL)

# ── App ────────────────────────────────────────────────────────────────────────

class App:
    def __init__(self):
        os.makedirs(SAVE_DIR, exist_ok=True)

        self.res_i   = 1
        self.pfmt_i  = 0
        self.vfmt_i  = 0
        self.flash   = 0
        self.status  = "Ready"
        self.status_t= 0
        self.settings= False
        self.prev_btn= False
        self.rec_t   = 0

        self.cam = Camera(DEVICE, self.res_i)

        cv2.namedWindow("Camera", cv2.WINDOW_NORMAL)
        cv2.resizeWindow("Camera", 1280, 760)

    def msg(self, text, secs=3):
        self.status   = text
        self.status_t = time.time() + secs
        print("[APP]", text)

    def ts(self):
        return datetime.now().strftime("%Y%m%d_%H%M%S")

    def snap(self):
        frame = self.cam.get_frame()
        if frame is None:
            self.msg("No frame!")
            return
        fmt  = PHOTO_FMTS[self.pfmt_i]
        exts = {"JPEG": ".jpg", "PNG": ".png", "TIFF": ".tiff"}
        path = os.path.join(SAVE_DIR, f"photo_{self.ts()}{exts[fmt]}")
        params = []
        if fmt == "JPEG":
            params = [cv2.IMWRITE_JPEG_QUALITY, 95]
        elif fmt == "PNG":
            params = [cv2.IMWRITE_PNG_COMPRESSION, 1]
        cv2.imwrite(path, frame, params)
        self.msg(f"Photo: {os.path.basename(path)}")
        self.flash = time.time() + 0.2

    def toggle_rec(self):
        if self.cam.rec:
            self.cam.stop_recording()
            secs = int(time.time() - self.rec_t)
            self.msg(f"Video saved ({secs}s)")
        else:
            fmt  = VIDEO_FMTS[self.vfmt_i]
            ext  = ".mp4" if "MP4" in fmt else ".mkv"
            path = os.path.join(SAVE_DIR, f"video_{self.ts()}{ext}")
            self.cam.start_recording(path, fmt)
            self.rec_t = time.time()
            self.msg(f"Recording {fmt}...", 99999)

    def set_res(self, i):
        if self.cam.rec:
            self.msg("Stop recording first!")
            return
        self.res_i = i
        self.cam.restart(i)
        self.msg(f"Resolution: {RESOLUTIONS[i][0]}")

    def draw(self, frame):
        fh, fw = frame.shape[:2]
        pw     = 270
        canvas = np.full((fh, fw + pw, 3), DARK, dtype=np.uint8)
        canvas[:, :fw] = frame

        if time.time() < self.flash:
            alpha = (self.flash - time.time()) / 0.2
            white = canvas.copy()
            white[:, :fw] = 255
            cv2.addWeighted(white, alpha * 0.8, canvas, 1 - alpha * 0.8, 0, canvas)

        if self.cam.rec:
            secs = int(time.time() - self.rec_t)
            m, s = divmod(secs, 60)
            if int(time.time() * 2) % 2 == 0:
                cv2.circle(canvas, (18, 18), 8, RED, -1)
            cv2.putText(canvas, f"REC {m:02d}:{s:02d}", (32, 24),
                        cv2.FONT_HERSHEY_SIMPLEX, 0.65, RED, 2)

        cv2.rectangle(canvas, (fw, 0), (fw + pw, fh), PANEL, -1)
        cv2.line(canvas, (fw, 0), (fw, fh), GREEN, 2)

        x, y = fw + 12, 30

        def t(text, col=WHITE, sc=0.52, bold=False):
            nonlocal y
            cv2.putText(canvas, text, (x, y), cv2.FONT_HERSHEY_SIMPLEX,
                        sc, col, 2 if bold else 1)
            y += int(sc * 44 + 4)

        def hr():
            nonlocal y
            cv2.line(canvas, (x, y), (x + pw - 18, y), GRAY, 1)
            y += 8

        t("OPI5  CAMERA", GREEN, 0.62, bold=True)
        hr()
        t("RESOLUTION", GRAY, 0.42)
        t(RESOLUTIONS[self.res_i][0], WHITE, 0.52)
        t("PHOTO FORMAT", GRAY, 0.42)
        t(PHOTO_FMTS[self.pfmt_i], WHITE, 0.52)
        t("VIDEO FORMAT", GRAY, 0.42)
        t(VIDEO_FMTS[self.vfmt_i], WHITE, 0.52)
        hr()
        t("KEYS", GRAY, 0.42)
        t("[P]  Photo", WHITE, 0.47)
        t("[V]  Rec / Stop", RED if self.cam.rec else WHITE, 0.47)
        t("[S]  Settings", WHITE, 0.47)
        t("[Q]  Quit", WHITE, 0.47)
        if gpio_ok:
            t("[BTN] Shutter", YELLOW, 0.47)
        hr()
        t("~/Pictures/camera", GRAY, 0.40)

        if time.time() < self.status_t:
            hr()
            msg = self.status
            while msg:
                t(msg[:26], YELLOW, 0.44)
                msg = msg[26:]

        if self.settings:
            canvas = self.draw_settings(canvas, fw, fh)

        return canvas

    def draw_settings(self, canvas, fw, fh):
        ow = int(fw * 0.55)
        oh = int(fh * 0.78)
        ox = (fw - ow) // 2
        oy = (fh - oh) // 2
        overlay = canvas.copy()
        cv2.rectangle(overlay, (ox, oy), (ox + ow, oy + oh), (15, 15, 15), -1)
        cv2.addWeighted(overlay, 0.93, canvas, 0.07, 0, canvas)
        cv2.rectangle(canvas, (ox, oy), (ox + ow, oy + oh), GREEN, 2)

        x, y = ox + 16, oy + 30

        def s(text, col=WHITE, sc=0.50, bold=False):
            nonlocal y
            cv2.putText(canvas, text, (x, y), cv2.FONT_HERSHEY_SIMPLEX,
                        sc, col, 2 if bold else 1)
            y += 26

        s("SETTINGS", GREEN, 0.60, bold=True)
        y += 6
        s("Resolution  [1-4]", GRAY, 0.43)
        for i, (label, w, h, fps) in enumerate(RESOLUTIONS):
            s(f"  [{i+1}] {label}  @{fps}fps", GREEN if i == self.res_i else WHITE, 0.43)
        y += 4
        s("Photo Format  [6-8]", GRAY, 0.43)
        for i, f in enumerate(PHOTO_FMTS):
            s(f"  [{i+6}] {f}", GREEN if i == self.pfmt_i else WHITE, 0.43)
        y += 4
        s("Video Format  [A-B]", GRAY, 0.43)
        for i, f in enumerate(VIDEO_FMTS):
            s(f"  [{chr(65+i)}] {f}", GREEN if i == self.vfmt_i else WHITE, 0.43)
        y += 8
        s("[S]  Close", YELLOW, 0.48)
        return canvas

    def key(self, k):
        if k == -1:
            return True
        if self.settings:
            if ord('1') <= k <= ord('4'):
                self.set_res(k - ord('1'))
            elif ord('6') <= k <= ord('8'):
                self.pfmt_i = k - ord('6')
            elif k in (ord('a'), ord('A')):
                self.vfmt_i = 0
            elif k in (ord('b'), ord('B')):
                self.vfmt_i = 1
            if k == ord('s'):
                self.settings = False
            return True
        if k == ord('q'):
            return False
        elif k == ord('p') and not self.cam.rec:
            self.snap()
        elif k == ord('v'):
            self.toggle_rec()
        elif k == ord('s'):
            self.settings = True
        return True

    def check_gpio(self):
        if not gpio_ok:
            return
        pressed = GPIO.input(GPIO_PIN) == GPIO.LOW
        if pressed and not self.prev_btn:
            if self.cam.rec:
                self.toggle_rec()
            else:
                self.snap()
        self.prev_btn = pressed

    def run(self):
        print(f"OPI5 Camera | {SAVE_DIR}")
        print("P=photo  V=video  S=settings  Q=quit")
        last_frame = None
        while True:
            frame = self.cam.get_frame()
            if frame is not None:
                last_frame = frame
            if last_frame is None:
                time.sleep(0.03)
                continue
            self.check_gpio()
            cv2.imshow("Camera", self.draw(last_frame))
            if not self.key(cv2.waitKey(1) & 0xFF):
                break
        self.cam.stop()
        cv2.destroyAllWindows()
        if gpio_ok:
            GPIO.cleanup()
        print("Done.")

# ── Entry ──────────────────────────────────────────────────────────────────────

if __name__ == "__main__":
    import sys
    if len(sys.argv) > 1:
        DEVICE = sys.argv[1]
    if not os.path.exists(DEVICE):
        print(f"Device not found: {DEVICE}")
        os.system("v4l2-ctl --list-devices 2>/dev/null")
        sys.exit(1)
    try:
        App().run()
    except KeyboardInterrupt:
        print("Interrupted.")
```

---

## 16. Changelog

| Version | Date | Notes |
|---|---|---|
| 1.0 | 2026-02-27 | Initial release |

---

## License

This guide and the camera application code are released under the MIT License.

The ADS7846-X11-daemon is by Tomasz Mankowski — see [original repository](https://github.com/Tomasz-Mankowski/ADS7846-X11-daemon) for its license.

---

*Guide maintained by the project author. Contributions and corrections welcome via GitHub Issues.*
