# USB-C for C++ Game Dev — Cross-Platform

> Input (HID), Display (DP Alt Mode), Power (PD) — what USB-C means for your game engine.

## Legend

| Symbol | Meaning |
|--------|--------|
| ✅ | Available via package manager |
| ⚠️ | Available with caveats |
| 🔧 | Manual download / build |
| 🪟 | Windows |
| 🐧 | Linux |
| 🍎 | macOS |

---

## 0. Install — One-Liners Per Platform

### 🪟 Windows (Scoop)

```powershell
# SDL3 + USB/HID tools
scoop install sdl3 pkg-config cmake ninja ccache

# USB analysis
scoop install wireshark                          # USB capture support
scoop install usbview                            # Sysinternals — USB device tree

# ADB (Android handheld testing)
scoop install adb scrcpy
```

### 🐧 Linux (apt — Ubuntu/Debian)

```bash
sudo apt update && sudo apt install -y \
  libsdl3-dev libsdl3-image-dev libsdl3-ttf-dev \
  libudev-dev libusb-1.0-0-dev \
  usbutils hwdata \
  wireshark tcpdump \
  adb scrcpy
```

### 🐧 ALT Linux (epm)

```bash
# ALT Linux — epm package manager
epm install libSDL3-devel libudev-devel libusb-devel
epm install usbutils wireshark
epm install android-tools
```

### 🍎 macOS (Homebrew)

```bash
brew install sdl3 pkg-config cmake ninja ccache

# USB/HID
brew install libusb wireshark

# ADB
brew install android-platform-tools scrcpy
```

### Verify Install

```bash
# SDL3
sdl3-config --version 2>/dev/null || pkg-config --modversion sdl3

# USB tools
lsusb --version          # Linux
system_profiler SPUSBDataType  # macOS

# ADB
adb version
```

---

## 1. Gamepad Input — SDL3 Gamepad API

> **Did you know?** Every OS has its own USB HID API. SDL3 abstracts them all + ships with a database of 1000+ gamepad mappings. One API, every controller.

### Install SDL3

```bash
# One-liner per platform
scoop install sdl3                    # 🪟
brew install sdl3                     # 🍎
sudo apt install libsdl3-dev          # 🐧 Ubuntu/Debian
epm install libSDL3-devel             # 🐧 ALT Linux
```

### CMake Integration

```cmake
find_package(SDL3 REQUIRED)
target_link_libraries(mygame PRIVATE SDL3::SDL3)
```

```bash
# With vcpkg (cross-platform)
vcpkg install sdl3
cmake -B build -DCMAKE_TOOLCHAIN_FILE="$VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake"
```

### Hotplug-Safe Gamepad Code

```cpp
#include <SDL3/SDL.h>

SDL_Gamepad* gamepad = nullptr;

// Handle events (call every frame)
SDL_Event event;
while (SDL_PollEvent(&event)) {
    switch (event.type) {
    case SDL_EVENT_GAMEPAD_ADDED:
        if (!gamepad)
            gamepad = SDL_OpenGamepad(event.gdevice.which);
        break;

    case SDL_EVENT_GAMEPAD_REMOVED:
        if (gamepad && SDL_GetGamepadID(gamepad) == event.gdevice.which) {
            SDL_CloseGamepad(gamepad);
            gamepad = nullptr;
        }
        break;
    }
}

// Read input (poll per-frame)
if (gamepad) {
    if (SDL_GetGamepadButton(gamepad, SDL_GAMEPAD_BUTTON_SOUTH)) {
        // A / Cross pressed
    }
    Sint16 lx = SDL_GetGamepadAxis(gamepad, SDL_GAMEPAD_AXIS_LEFTX);
    float normalized_x = lx / 32767.0f;
}
```

> **Critical**: Xbox, Steam Deck, and Switch **require** hotplug support for certification. Controllers may not be available at startup — always handle `GAMEPAD_ADDED` events.

### Identify Controller Type via USB VID/PID

```cpp
if (gamepad) {
    Uint16 vid = SDL_GetGamepadVendor(gamepad);
    Uint16 pid = SDL_GetGamepadProduct(gamepad);

    // Known VIDs:
    // 0x045e — Microsoft (Xbox)
    // 0x054c — Sony (DualShock/DualSense)
    // 0x057e — Nintendo (Switch Pro / Joy-Con)
    // 0x28de — Valve (Steam Controller)
    // 0x046d — Logitech
    // 0x1532 — Razer
}
```

### Rumble, LED, Gyro

```cpp
// Vibration
SDL_RumbleGamepad(gamepad, low_freq_rumble, high_freq_rumble, duration_ms);

// RGB LED (DualSense, some Xbox)
SDL_SetGamepadLED(gamepad, 0, 0, 255);  // blue

// Gyro (DualSense, Switch Pro, Steam Deck)
if (SDL_GamepadHasSensor(gamepad, SDL_SENSOR_GYRO)) {
    SDL_SetGamepadSensorEnabled(gamepad, SDL_SENSOR_GYRO, true);
    // Read via SDL_EVENT_GAMEPAD_SENSOR_UPDATE
}
```

### Quick Test — Is My Gamepad Detected?

```bash
# 🐧 Linux — list all USB HID devices
lsusb | grep -i "game\|controller\|joystick"
cat /proc/bus/input/devices | grep -A 5 "Gamepad\|Joystick"

# 🍎 macOS
system_profiler SPUSBDataType | grep -A 3 -i "controller\|gamepad"

# 🪟 Windows — PowerShell
Get-PnpDevice -Class HIDClass | Where-Object { $_.FriendlyName -match "game|controller|xbox|dual" }

# Cross-platform — SDL3 test utility
sdl3-gamepad-test    # if available, or build from SDL3 test/
```

---

## 2. USB-C Display Output — Handheld → TV

> **Did you know?** Handheld gaming devices (Steam Deck, ROG Ally, Switch) output video over USB-C using **DisplayPort Alternate Mode**. Your game must handle display hotplug and resolution changes.

### Handheld USB-C Specs

| Device | USB-C Port | DP Output | PD Charging | Notes |
|--------|-----------|-----------|-------------|-------|
| **Steam Deck** | USB-C 3.2 Gen 2 (10 Gbps) | DP 1.4 | 45W | Official dock or USB-C → HDMI |
| **ROG Ally / Ally X** | USB-C 3.2 Gen 2 | DP Alt Mode | 100W (need 100W PSU for Turbo) | |
| **Nintendo Switch** | USB-C (5 Gbps) | HDMI via dock (1080p60) | 18-20W | 3rd-party docks may fail |
| **Switch 2** | USB-C | HDMI 4K60 | PD up to 100W | |
| **Legion Go** | USB-C | DP Alt Mode | 65W+ | |

### Handle Display Hotplug

```cpp
// SDL3 — display events
while (SDL_PollEvent(&event)) {
    switch (event.type) {
    case SDL_EVENT_DISPLAY_CONNECTED:
        // New monitor plugged in (dock connected!)
        recreateSwapChain(event.display.displayID);
        break;

    case SDL_EVENT_DISPLAY_DISCONNECTED:
        // Monitor removed (dock disconnected!)
        fallbackToNativeDisplay();
        break;

    case SDL_EVENT_DISPLAY_ORIENTATION:
        // Rotation changed
        handleOrientationChange(event.display.displayID);
        break;
    }
}
```

```cpp
// Windows — WM_DISPLAYCHANGE
case WM_DISPLAYCHANGE:
    int new_w = LOWORD(lParam);
    int new_h = HIWORD(lParam);
    recreateSwapChain(new_w, new_h);
    break;
```

### Test — USB-C Display Output

```bash
# 🐧 Linux — connected displays
xrandr --query
# or Wayland:
swaymsg -t get_outputs

# 🍎 macOS
system_profiler SPDisplaysDataType

# 🪟 Windows
Get-CimInstance -Namespace root\wmi -ClassName WmiMonitorBasicDisplayParams
# or
wmic path Win32_VideoController get CurrentHorizontalResolution,CurrentVerticalResolution
```

### Dock Recommendations for Testing

| Dock | Price | Features | Works With |
|------|-------|----------|------------|
| **UGREEN 6-in-1** | ~$40-60 | 4K@120Hz HDMI 2.1, 100W PD, GbE | Deck, ROG Ally, Legion Go |
| **j5create JCD624** | ~$50 | 4K60, 100W PD, GbE, USB-A | Deck, Switch 2, ROG Ally |
| **Official Steam Dock** | ~$90 | HDMI + DP, USB-A | Steam Deck |
| **Baseus 6-in-1** | ~$10-20 | Budget, basic | Most USB-C DP devices |

---

## 3. USB-C Power Delivery — Battery Management

> **Did you know?** USB PD negotiates voltage/current via the CC pin. Profiles range from 5V/3A (15W) to 48V/5A (240W EPR). Your handheld game must not block charging.

### PD Profiles

| Profile | Voltage | Power | Used By |
|---------|---------|-------|---------|
| 5V / 3A | 5V | 15W | Phones |
| 9V / 3A | 9V | 27W | Switch |
| 15V / 3A | 15V | 45W | Steam Deck |
| 20V / 5A | 20V | 100W | Laptops, ROG Ally |
| 28V / 5A | 28V | 140W | MacBook Pro |
| 48V / 5A | 48V | 240W | EPR (new) |

### Battery Monitoring in Your Game

```cpp
// SDL3 — check power state
int seconds, percent;
SDL_PowerState state = SDL_GetPowerInfo(&seconds, &percent);

switch (state) {
case SDL_POWERSTATE_ON_BATTERY:
    if (percent < 20) {
        showLowBatteryWarning();
        reduceGraphicsQuality();  // save battery
    }
    break;
case SDL_POWERSTATE_CHARGING:
    // Charging — don't throttle
    break;
case SDL_POWERSTATE_CHARGED:
    // Plugged in, full
    break;
}
```

### Test — PD Negotiation

```bash
# 🐧 Linux — battery info
upower -i $(upower -e | head -1)
cat /sys/class/power_supply/BAT*/capacity

# 🍎 macOS
pmset -g batt
system_profiler SPPowerDataType

# 🪟 Windows
(Get-WmiObject Win32_Battery).EstimatedChargeStatus
powercfg /batteryreport
```

---

## 4. USB Device Enumeration — Debugging

### List All USB Devices

```bash
# 🐧 Linux — full USB tree
lsusb -tv
# Filter for game controllers
lsusb | grep -iE "game|control|joystick|xbox|playstation|nintendo"

# 🍎 macOS
system_profiler SPUSBDataType

# 🪟 Windows — PowerShell
Get-PnpDevice -Class USB | Format-Table Status, Class, FriendlyName -AutoSize
# Detailed
Get-PnpDevice -Class HIDClass | Select-Object InstanceId, FriendlyName | Format-List
```

### USB Packet Capture

```bash
# 🐧 Linux — usbmon kernel module
sudo modprobe usbmon
# Capture with Wireshark or tcpdump
sudo tcpdump -i usbmon1 -w usb_capture.pcap

# 🍎 macOS — requires root
sudo tcpdump -i usb -w usb_capture.pcap

# 🪟 Windows — Wireshark with USBPcap
# Install Wireshark with USBPcap option
# Wireshark → Capture → USBPcap → select controller
```

### USB Descriptor Dump

```bash
# 🐧 Linux — full device descriptor
lsusb -v -d 045e:0b12    # Xbox controller by VID:PID

# 🍎 macOS
ioreg -p IOUSB -w0 -l | grep -A 20 "VendorID.*0x045e"

# 🪟 Windows — USBView (Sysinternals)
scoop install usbview
usbview
```

---

## 5. ADB — Android Handheld Testing

> For testing on Android-based handhelds, or any Android device via USB-C.

```bash
# Install
scoop install adb scrcpy                    # 🪟
brew install android-platform-tools scrcpy  # 🍎
sudo apt install adb scrcpy                 # 🐧 Ubuntu/Debian
epm install android-tools                   # 🐧 ALT Linux

# Quick workflow
adb devices -l                              # list connected devices
adb install -r mygame.apk                   # install/update
adb logcat -s MyGame *:E                    # error logs only

# Screen mirror + control
scrcpy --max-size 1080 --max-fps 60

# Root (eng/userdebug builds)
adb root
adb shell setenforce 0                      # disable SELinux
adb shell getprop ro.build.type             # user / userdebug / eng
```

---

## 6. Cross-Platform Input Libraries

| Library | C++ Native | Gamepad DB | Rumble | Gyro | Notes |
|---------|-----------|------------|--------|------|-------|
| **[SDL3](https://libsdl.org)** | ✅ | ✅ 1000+ | ✅ | ✅ | Industry standard, Steam built-in |
| **[MPG (FeralAI)](https://github.com/FeralAI/MPG)** | ✅ | ❌ | ❌ | ❌ | Lightweight, BYO USB stack |
| **[GLFW](https://glfw.org)** | ✅ | ❌ | ❌ | ❌ | Minimal, no gamepad mapping DB |
| **[GameInput](https://learn.microsoft.com/en-us/gaming/gdk/_content/gc/input/overviews/input-overview)** (Microsoft) | ✅ | ✅ | ✅ | ✅ | Windows/Xbox only |
| **Native HID** | ✅ | ❌ | ❌ | ❌ | Full control, maximum pain |

### SDL3 vs Native — When to Go Native?

```
USE SDL3 when:                    GO NATIVE when:
─────────────────                 ─────────────────
✅ Standard gamepad input          ❌ Custom USB device (not HID gamepad)
✅ Multi-controller support        ❌ Need USB bulk transfers
✅ Steam/Deck compatibility        ❌ Custom HID reports (LED matrix, etc.)
✅ Don't want to write 5 APIs      ❌ Embedded/bare-metal
```

---

## 7. Cable & Dock Cheat Sheet

### USB-C Cable Types

| Cable Type | Data | Charging | Video | Price | When |
|------------|------|----------|-------|-------|------|
| USB 2.0 Type-C | 480 Mbps | 60W | ❌ | $5-15 | Charge-only |
| USB 3.2 Gen 1 | 5 Gbps | 100W | DP Alt Mode | $15-25 | Data + video |
| USB 3.2 Gen 2 | 10 Gbps | 100W | DP Alt Mode | $20-35 | Fast data |
| USB4 | 40 Gbps | 100W | DP 1.4 | $30-80 | eGPU, high-speed |
| Thunderbolt 4/5 | 40-80 Gbps | 100-240W | DP 2.1 | $50-300 | Everything |

### How to Identify a Good Cable

```bash
# 🐧 Linux — check connected USB-C device speed
lsusb -v | grep -A 5 "MaxPower\|bcdUSB"
usb-devices | grep -A 10 "USB"

# Check if DP Alt Mode is supported (needs kernel 5.10+)
cat /sys/class/typec/port*/data_role
cat /sys/class/typec/port*/power_role
```

> **Tip**: If a cable says "USB-C" but costs $3 — it's USB 2.0 charge-only. No video, no fast data.

---

## 8. Troubleshooting

### Gamepad Not Detected

```bash
# 🐧 Linux — check udev rules
ls /dev/input/js*                           # joystick devices exist?
cat /proc/bus/input/devices | grep -i game  # kernel sees it?
lsusb | grep -i game                         # USB device present?

# Missing permissions?
sudo chmod 666 /dev/input/js0
# Or add udev rule:
echo 'SUBSYSTEM=="usb", ATTRS{idVendor}=="045e", MODE="0666"' | \
  sudo tee /etc/udev/rules.d/99-gamepad.rules
sudo udevadm control --reload-rules

# 🍎 macOS — check IOKit
system_profiler SPUSBDataType | grep -A 5 -i "controller"
# macOS may need "Input Monitoring" permission in System Settings

# 🪟 Windows — Device Manager
Get-PnpDevice | Where-Object { $_.Status -eq "Error" -and $_.Class -eq "HIDClass" }
# Try: Update driver → Xbox 360 Peripherals
```

### Display Not Outputting via USB-C

```bash
# 🐧 Linux — check DP Alt Mode
cat /sys/class/typec/port*/data_role       # should show "host"
xrandr --listproviders                     # GPU providers
xrandr --listmonitors                      # connected monitors

# 🍎 macOS — check Thunderbolt/USB-C
system_profiler SPThunderboltDataType
system_profiler SPDisplaysDataType

# Common issues:
# 1. Cable is USB 2.0 (no DP Alt Mode) → get USB 3.2+ cable
# 2. Dock doesn't support DP Alt Mode → check dock specs
# 3. Linux kernel too old for USB-C DP → need 5.10+
```

### Charging Not Working While Playing

```bash
# 🐧 Linux — check power
upower -i $(upower -e | head -1)
cat /sys/class/power_supply/AC*/online       # 1 = plugged in

# 🍎 macOS
pmset -g batt

# Common issues:
# 1. PD charger too weak (need 45W+ for Deck, 100W for ROG Ally)
# 2. Dock doesn't support PD passthrough
# 3. Cable missing E-Marker chip (needed for 100W+)
```

---

## 9. Quick Reference Card

```
SDL3 INSTALL:  scoop install sdl3 | brew install sdl3 | apt install libsdl3-dev
SDL3 CMAKE:    find_package(SDL3 REQUIRED) + target_link_libraries(SDL3::SDL3)
GAMEPAD VID:   0x045e=Xbox 0x054c=Sony 0x057e=Nintendo 0x28de=Valve
HOTPLUG:       SDL_EVENT_GAMEPAD_ADDED / SDL_EVENT_GAMEPAD_REMOVED
RUMBLE:        SDL_RumbleGamepad(gp, low, high, ms)
GYRO:          SDL_SetGamepadSensorEnabled(gp, SDL_SENSOR_GYRO, true)
BATTERY:       SDL_GetPowerInfo(&secs, &pct) → SDL_POWERSTATE_*
DISPLAY:       SDL_EVENT_DISPLAY_CONNECTED / _DISCONNECTED
USB LIST:      lsusb -tv | system_profiler SPUSBDataType | Get-PnpDevice -Class USB
USB CAPTURE:   sudo tcpdump -i usbmon1 -w usb.pcap
ADB:           adb devices -l && adb install -r game.apk && adb logcat
MIRROR:        scrcpy --max-size 1080
CABLE CHECK:   USB 3.2+ for video, USB 2.0 = charge-only
PD PROFILES:   15W/27W/45W/100W/140W/240W (Deck=45W, Ally=100W)
```

---

→ **Related**: [README.md](README.md) — full cookbook | [controls.md](controls.md) — Steam Input golden rules | [mobile.md](mobile.md) — ADB deep dive | [debugging-profiling.md](debugging-profiling.md) — GDB/LLDB tricks
