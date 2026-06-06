# Mobile Development — Android & iOS

> ADB, devicectl, xcrun, xcodebuild — cross-platform mobile dev one-liners.

## ADB — Android Debug Bridge

### Install

```bash
# Windows (scoop preferred)
scoop install adb

# macOS
brew install android-platform-tools

# Linux
sudo apt install adb        # Debian/Ubuntu
sudo pacman -S android-tools  # Arch
```

### Device Management

```bash
adb devices                     # list connected devices
adb devices -l                  # with model info
adb kill-server && adb start-server  # restart ADB daemon
adb connect 192.168.1.100:5555  # wireless debug
adb usb                         # switch back to USB
```

### App Management

```bash
adb install app.apk             # install
adb install -r app.apk          # reinstall (keep data)
adb install -d app.apk          # downgrade
adb uninstall com.example.app   # uninstall
adb uninstall -k com.example.app  # keep data
adb shell pm list packages      # list installed packages
adb shell pm list packages -3   # third-party only
adb shell pm clear com.example.app  # clear app data
adb shell am force-stop com.example.app  # force stop
adb shell am start -n com.example.app/.MainActivity  # launch activity
```

### Logs

```bash
adb logcat                      # all logs
adb logcat -c                   # clear
adb logcat *:E                  # errors only
adb logcat -s MyTag             # filter by tag
adb logcat | grep "myapp"       # grep filter
adb logcat -v time              # with timestamps
adb logcat -d > crash.log       # dump to file
adb logcat --pid=$(adb shell pidof com.example.app)  # app PID only
```

### Files

```bash
adb push local.txt /sdcard/     # PC → device
adb pull /sdcard/file.txt .     # device → PC
adb shell ls -la /sdcard/       # list files
adb shell du -sh /data/data/com.example.app  # app size
```

### Screenshots & Recording

```bash
adb exec-out screencap -p > screen.png  # screenshot
adb shell screenrecord /sdcard/rec.mp4  # record (Ctrl+C to stop)
adb pull /sdcard/rec.mp4 .              # pull recording
```

### System Info

```bash
adb shell getprop ro.build.version.sdk     # API level
adb shell getprop ro.product.model         # device model
adb shell getprop ro.build.display.id      # build ID
adb shell dumpsys battery                  # battery info
adb shell dumpsys meminfo com.example.app  # memory usage
adb shell dumpsys cpuinfo                  # CPU usage
adb shell cat /proc/cpuinfo                # CPU details
adb shell wm size                          # screen resolution
adb shell wm density                       # screen DPI
```

---

## ADB Root & Advanced

> For rooted devices, custom ROMs, engineering builds.

### Root access

```bash
adb root                        # restart adbd as root
adb shell                       # now you're root
adb unroot                      # back to normal

# If "adbd cannot run as root in production builds":
# → Need eng/userdebug build OR Magisk
# → With Magisk: enable "ADB shell is root" in Magisk settings
```

### SELinux

```bash
adb shell getenforce            # check: Enforcing/Permissive/Disabled
adb shell setenforce 0          # → Permissive (allows everything, logged)
adb shell setenforce 1          # → Enforcing

# Check SELinux denials
adb shell dmesg | grep avc      # kernel log denials
adb shell cat /proc/kmsg | grep avc  # live denials

# Generate allow rules from denials
adb shell dmesg | audit2allow -p policy
```

### System modification (root required)

```bash
# Remount system as writable
adb shell mount -o rw,remount /system

# Edit hosts file
adb push hosts /system/etc/hosts

# Install system app
adb push app.apk /system/app/
adb shell chmod 644 /system/app/app.apk

# Remove bloatware
adb shell pm uninstall --user 0 com.bloatware.app

# Backup/restore system partition
adb shell dd if=/dev/block/bootdevice/by-name/system of=/sdcard/system.img
adb pull /sdcard/system.img .
```

### Google APIs (no Play Services image)

> Use AOSP images + Google APIs addon for clean testing without Play Services.

```bash
# List available system images
sdkmanager --list | grep "system-images"

# Install AOSP image (no Google Play Services)
sdkmanager "system-images;android-34;default;arm64-v8a"

# Install Google APIs addon (has Google APIs, NOT Play Services)
sdkmanager "system-images;android-34;google_apis;arm64-v8a"

# Create emulator with Google APIs (no Play Store)
avdmanager create avd -n pixel_test \
  -k "system-images;android-34;google_apis;arm64-v8a" \
  -d "pixel_6"

# Launch emulator
emulator -avd pixel_test -writable-system -no-snapshot

# For Play Services testing:
sdkmanager "system-images;android-34;google_apis_playstore;arm64-v8a"
```

### NDK / JNI debugging

```bash
# Enable JDWP for native debugging
adb forward tcp:5039 tcp:5039

# Get native crash logs
adb logcat | ndk-stack -sym build/android/app/obj/local/arm64-v8a

# Dump native backtrace
adb shell debuggerd -b <pid>

# Symbolize native crash
ndk-stack -sym obj/local/arm64-v8a < crash.log
```

### Emulator console

```bash
emulator -avd my_avd -writable-system     # writable system
emulator -avd my_avd -no-snapshot         # fresh boot
emulator -avd my_avd -gpu swiftshader_indirect  # software GPU
emulator -avd my_avd -netdelay 3g -netspeed 3g  # network simulation

# Console commands (telnet localhost 5554)
geo fix -122.4194 37.7749     # GPS location (SF)
sms send 123456 "test message"
power capacity 15              # battery level
network speed edge             # throttle network
```

---

## Xcode, xcrun, devicectl (iOS/macOS)

> macOS only. Requires Xcode + Command Line Tools.

### Install

```bash
xcode-select --install           # command line tools
# Full Xcode from App Store or developer.apple.com
```

### xcrun — run developer tools

```bash
xcrun simctl list devices        # list simulators
xcrun simctl boot "iPhone 15"    # boot simulator
xcrun simctl shutdown all        # shutdown all
xcrun simctl erase all           # reset all simulators

# Install app on simulator
xcrun simctl install booted MyApp.app

# Launch app
xcrun simctl openurl booted "myapp://deeplink"

# Screenshot
xcrun simctl io booted screenshot screen.png

# Record video
xcrun simctl io booted recordVideo video.mp4

# Set appearance
xcrun simctl ui booted appearance dark

# Push notification
xcrun simctl push booted com.example.app payload.json
```

### xcodebuild — build from CLI

```bash
# List schemes
xcodebuild -list -project MyApp.xcodeproj

# Build for simulator
xcodebuild -project MyApp.xcodeproj -scheme MyApp \
  -sdk iphonesimulator -destination 'platform=iOS Simulator,name=iPhone 15' \
  build

# Build for device
xcodebuild -project MyApp.xcodeproj -scheme MyApp \
  -sdk iphoneos -destination 'generic/platform=iOS' \
  build CODE_SIGN_IDENTITY="Apple Development" CODE_SIGNING_REQUIRED=YES

# Build workspace (CocoaPods/SPM)
xcodebuild -workspace MyApp.xcworkspace -scheme MyApp \
  -sdk iphonesimulator build

# Run tests
xcodebuild test -project MyApp.xcodeproj -scheme MyApp \
  -destination 'platform=iOS Simulator,name=iPhone 15'

# Archive for distribution
xcodebuild archive -project MyApp.xcodeproj -scheme MyApp \
  -archivePath build/MyApp.xcarchive

# Export IPA from archive
xcodebuild -exportArchive -archivePath build/MyApp.xcarchive \
  -exportPath build/ipa -exportOptionsPlist ExportOptions.plist

# Show SDKs
xcodebuild -showsdks

# Clean
xcodebuild clean -project MyApp.xcodeproj -scheme MyApp
```

### devicectl — physical iOS devices (Xcode 15+)

```bash
# List connected devices
xcrun devicectl list devices

# Get device info
xcrun devicectl get info apps --device <device-id>

# Install app on physical device
xcrun devicectl manage pair --device <device-id>
xcrun devicectl device install app --device <device-id> MyApp.app

# Launch app
xcrun devicectl device process launch --device <device-id> \
  com.example.myapp

# Launch with environment variables
xcrun devicectl device process launch --device <device-id> \
  --environment-variables '{"DYLD_INSERT_LIBRARIES"="/path/to/lib"}' \
  com.example.myapp

# Terminate app
xcrun devicectl device process terminate --device <device-id> --pid 1234

# List running processes
xcrun devicectl device info processes --device <device-id>

# Copy files to/from device
xcrun devicectl device copy to --device <device-id> \
  --source local.txt --destination /path/on/device

# Device logs
xcrun devicectl device syslog --device <device-id>
```

---

## LLDB — iOS/macOS/Android NDK Debugging

### Install

```bash
# macOS — included with Xcode CLT
xcode-select --install

# Linux
sudo apt install lldb           # Debian/Ubuntu
sudo pacman -S lldb             # Arch

# Windows
scoop install llvm              # includes lldb
```

### Basic commands

```bash
# Start
lldb ./myapp
lldb -n MyApp                   # attach to running process
lldb -p 1234                    # attach by PID

# Breakpoints
(lldb) b main                   # break at function
(lldb) b file.cpp:42            # break at line
(lldb) rb .                     # regex breakpoint (all matches)
(lldb) br list                  # list breakpoints
(lldb) br del 1                 # delete breakpoint

# Execution
(lldb) run                      # run
(lldb) run -- arg1 arg2         # run with args
(lldb) continue                 # continue
(lldb) next                     # step over
(lldb) step                     # step into
(lldb) finish                   # step out
(lldb) thread step-inst         # step one instruction

# Inspection
(lldb) print var                # print variable
(lldb) po obj                   # print object (calls description)
(lldb) frame variable           # all locals
(lldb) bt                       # backtrace
(lldb) thread list              # all threads
(lldb) thread backtrace all     # all thread backtraces

# Memory
(lldb) memory read -count 64 0x7fff0000  # read memory
(lldb) memory write 0x7fff0000 0x00       # write memory
(lldb) watchpoint set variable myvar       # break on change

# Expression evaluation
(lldb) expr myFunc(42)          # call function
(lldb) expr @import UIKit       # import module
(lldb) expr (void)[[UIApplication sharedApplication] performSelector:@selector(openURL:) withObject:url]

# Image/framework inspection
(lldb) image list               # loaded frameworks
(lldb) image lookup -n myFunc   # find function address
(lldb) image lookup -a 0x1234   # address → symbol
```

### Remote debug iOS device

```bash
# 1. Start debugserver on device (via Xcode or devicectl)
xcrun devicectl device process launch --device <id> \
  --start-stopped com.example.myapp

# 2. Connect lldb
lldb
(lldb) platform select remote-ios
(lldb) platform connect connect://<device-ip>:1234
(lldb) process connect connect://<device-ip>:1234
(lldb) process continue
```

### LLDB useful scripts

```bash
# Print view hierarchy (iOS)
(lldb) expr (void)[[[UIApplication sharedApplication] keyWindow] recursiveDescription]

# Force UI update from debugger
(lldb) expr (void)[CATransaction flush]

# Print all Objective-C classes in a framework
(lldb) image dump objfile UIKit

# Swift expression
(lldb) expr import Foundation
(lldb) expr let data = try! Data(contentsOf: URL(string: "https://example.com")!)
```

---

## Frida — Dynamic Instrumentation

```bash
pip install frida-tools

# List processes on device
frida-ps -U

# Attach and trace
frida -U -n com.example.app -l script.js

# Spawn and attach
frida -U -f com.example.app -l script.js --no-pause

# Trace all calls to a function
frida-trace -U -n com.example.app -m "-[NSURL* *]"

# Bypass SSL pinning
frida -U -n com.example.app -l ssl-bypass.js
```

---

## Quick Reference Card

```
ANDROID:
  DEVICES:    adb devices -l
  INSTALL:    adb install -r app.apk
  LOGS:       adb logcat -s MyTag *:E
  ROOT:       adb root && adb shell setenforce 0
  FILES:      adb push/pull local remote
  SCREENSHOT: adb exec-out screencap -p > screen.png
  RECORD:     adb shell screenrecord /sdcard/rec.mp4
  CRASH:      adb logcat | ndk-stack -sym obj/local/arm64-v8a
  EMULATOR:   emulator -avd name -writable-system -no-snapshot

iOS:
  SIMULATOR:  xcrun simctl list/boot/shutdown
  INSTALL:    xcrun simctl install booted App.app
  SCREENSHOT: xcrun simctl io booted screenshot s.png
  BUILD:      xcodebuild -project X -scheme Y -sdk iphonesimulator build
  DEVICE:     xcrun devicectl list devices
  DEPLOY:     xcrun devicectl device install app --device ID App.app
  DEBUG:      lldb -n MyApp / xcrun devicectl device process launch --start-stopped

LLDB:
  START:      lldb ./app | lldb -p PID
  BREAK:      b main | b file.cpp:42 | rb .
  RUN:        run | continue | next | step | finish
  INSPECT:    print var | po obj | bt | frame variable
  MEMORY:     memory read -count 64 ADDR
  WATCH:      watchpoint set variable myvar
```

→ **Related**: [debugging-profiling.md](debugging-profiling.md) — GDB, LLDB, sanitizers, Tracy
→ **Related**: [tools-install.md](tools-install.md) — install instructions for all tools
