# Controls & Input — Game Dev Reference

> Steam Input Golden Rules, gamepad handling, input architecture patterns.

## Steam's 5 Golden Rules of Input

From [Valve's Steam Input documentation](https://partner.steamgames.com/doc/features/steam_controller/getting_started_for_devs):

### 1. On-screen icons must match the input device
Show Xbox glyphs for Xbox controllers, PS glyphs for PS. Steam Input emulates Xbox for all controllers → detect real device, swap glyphs at runtime.

```cpp
EInputActionOrigin origin = SteamInput()->GetActionOriginFromXboxOrigin(controller, k_EXboxOrigin_A);
const char* glyphPath = SteamInput()->GetGlyphForActionOrigin(origin);
// Load glyphPath as texture for button prompt
```

### 2. Mouse cursor must match the input device
- Visible OS cursor → mouse/trackpad active
- Hidden cursor → gamepad-only mode
- Never show a static cursor when no mouse is being used

### 3. All devices must work 100% of the time
Test edge cases: disconnect/reconnect mid-game, Steam Remote Play, Steam Deck dock/undock, multiple controllers.

### 4. D-pad, analog stick, AND mouse must all navigate menus
Menu traversal must work with every input method. Never force a specific device for UI navigation.

### 5. Disconnected gamepad must pause the game
Single player → pause. Multiplayer → mark disconnected, don't freeze.

### Valve's Bonus Rule: Simultaneous input
Gamepad + mouse must work **simultaneously**. This is the #1 cause of Steam Input compatibility issues.

---

## Steam Input API

### Setup

```cpp
// Initialize once at startup
SteamInput()->Init();

// Update every frame (SteamAPI_RunCallbacks calls RunFrame internally)
SteamInput()->RunFrame();

// Get controller
InputHandle_t controller = SteamInput()->GetControllerForGamepadIndex(0);

if (controller != 0) {
    // Controller is mapped through Steam Input
    ESteamInputType type = SteamInput()->GetInputTypeForHandle(controller);
    // k_ESteamInputType_PS4Controller, k_ESteamInputType_XBoxOneController, etc.
}
```

### Read actions (native mode)

```cpp
// Digital action (button press)
InputDigitalActionData_t data = SteamInput()->GetDigitalActionData(actionHandle, controller);
if (data.bActive && data.bState) {
    // Button is pressed
}

// Analog action (stick/trigger)
InputAnalogActionData_t data = SteamInput()->GetAnalogActionData(analogHandle, controller);
float x = data.x;  // -1.0 to 1.0
float y = data.y;

// Camera/aim — ALWAYS use absolute_mouse action type
// Steam Input converts joystick, gyro, trackpad → absolute_mouse deltas
// You CANNOT convert joystick_move back to mouse
```

### In-Game Actions file (IGA)

Create `controller_config/game_actions_<appid>.vdf`:

```vdf
"actions" {
    "default" {
        "title" "#Action_Default_Title"

        "buttons" {
            "Action_Jump" "#Action_Jump"
            "Action_Interact" "#Action_Interact"
        }

        "analogtriggers" {
            "Action_Accelerate" "#Action_Accelerate"
            "Action_Brake" "#Action_Brake"
        }

        "stickpadgyro" {
            "Action_Camera" {
                "title" "#Action_Camera"
                "input_mode" "absolute_mouse"    // ← CRITICAL for FPS/TPS
            }
            "Action_Move" {
                "title" "#Action_Move"
                "input_mode" "joystick_move"
            }
        }
    }
}
```

> **Critical**: For FPS/TPS games, `absolute_mouse` action type is mandatory. Steam Input converts gyro, trackpad, and joystick into `absolute_mouse` deltas. Without it, Steam Deck and controller players cannot aim properly.

---

## Input Architecture Patterns

### Action-based input (recommended)

```
Physical Input → Action Mapping → Game Action → Game Logic
   (A button)    (config file)     (Jump)        (apply force)
```

```cpp
// DON'T: hardcode buttons
if (gamepad.IsButtonPressed(BUTTON_A)) { player.Jump(); }

// DO: use action mapping
if (input.IsActionTriggered("jump")) { player.Jump(); }
```

### Input buffer (fighting games, platformers)

```cpp
struct InputBuffer {
    struct Entry { Action action; int framesAgo; };
    std::deque<Entry> buffer;
    static constexpr int BUFFER_SIZE = 8; // frames

    void addFrame(Action a) {
        buffer.push_back({a, 0});
        for (auto& e : buffer) e.framesAgo++;
        while (buffer.size() > BUFFER_SIZE) buffer.pop_front();
    }

    bool wasPressedRecently(Action a, int withinFrames = 5) {
        for (auto& e : buffer)
            if (e.action == a && e.framesAgo <= withinFrames) return true;
        return false;
    }
};
```

### Coyote time (platformers)

```cpp
// Allow jumping for a few frames after walking off a ledge
bool canJump = isGrounded || (framesSinceGrounded < COYOTE_FRAMES);
// COYOTE_FRAMES = 5-8 at 60fps feels right
```

### Input priority system

```cpp
// Priority: UI > Cutscene > Gameplay > Background
enum class InputContext { UI, Cutscene, Gameplay, Background };

class InputManager {
    InputContext current = InputContext::Gameplay;

    bool isActionAllowed(Action a) {
        switch (current) {
            case InputContext::UI:       return a.isUIAction();
            case InputContext::Cutscene: return a.isSkipAction();
            case InputContext::Gameplay: return true;
            case InputContext::Background: return false;
        }
    }
};
```

---

## Gamepad Best Practices

### Dead zones

```cpp
// Apply dead zone to analog stick
float applyDeadzone(float value, float deadzone = 0.15f) {
    if (std::abs(value) < deadzone) return 0.0f;
    // Remap to full range
    float sign = value > 0 ? 1.0f : -1.0f;
    return sign * (std::abs(value) - deadzone) / (1.0f - deadzone);
}
```

### Radial dead zone (better for 2D sticks)

```cpp
Vec2 applyRadialDeadzone(Vec2 stick, float deadzone = 0.15f) {
    float magnitude = length(stick);
    if (magnitude < deadzone) return {0, 0};
    return normalize(stick) * ((magnitude - deadzone) / (1.0f - deadzone));
}
```

### Response curves

```cpp
// Linear — direct mapping
float linear(float x) { return x; }

// Quadratic — more precision near center (recommended default)
float quadratic(float x) { return x * x * sign(x); }

// Cubic — even more center precision
float cubic(float x) { return x * x * x; }
```

### Haptic feedback

```cpp
// Steam Input haptics
SteamInput()->TriggerHaptic(controller, leftFrequency, rightFrequency, duration);

// XInput (Xbox controllers on Windows)
XINPUT_VIBRATION vib;
vib.wLeftMotorSpeed = 65535;   // 0-65535 (low frequency rumble)
vib.wRightMotorSpeed = 32768;  // 0-65535 (high frequency rumble)
XInputSetState(0, &vib);

// DualSense (PS5) — use SDL or hidapi
// Adaptive triggers, haptic feedback, speaker
// SDL: SDL_GameControllerRumbleTriggers()
```

---

## Keyboard + Mouse Patterns

### Raw input (Windows, bypasses acceleration)

```cpp
// Register for raw mouse input
RAWINPUTDEVICE rid;
rid.usUsagePage = 0x01;
rid.usUsage = 0x02;  // mouse
rid.dwFlags = RIDEV_NOLEGACY;  // disable legacy mouse messages
rid.hwndTarget = hwnd;
RegisterRawInputDevices(&rid, 1, sizeof(rid));

// In WM_INPUT handler:
RAWINPUT raw;
UINT size = sizeof(raw);
GetRawInputData((HRAWINPUT)lParam, RID_INPUT, &raw, &size, sizeof(RAWINPUTHEADER));
int dx = raw.data.mouse.lLastX;  // raw delta, no acceleration
int dy = raw.data.mouse.lLastY;
```

### Key rebinding

```cpp
// Store bindings as data, not code
struct KeyBinding {
    Action action;
    Key primary;
    Key secondary;
};

std::vector<KeyBinding> bindings = {
    {Action::Jump, Key::Space, Key::W},
    {Action::Crouch, Key::LControl, Key::C},
    {Action::Reload, Key::R, Key::None},
};

bool isActionPressed(Action a) {
    for (auto& b : bindings) {
        if (b.action == a) {
            return isKeyPressed(b.primary) || isKeyPressed(b.secondary);
        }
    }
    return false;
}
```

---

## Touch Input (Mobile)

### Virtual joystick

```cpp
struct VirtualJoystick {
    Vec2 center;       // touch down position
    Vec2 current;      // current touch position
    float radius = 100.0f;

    Vec2 getDirection() {
        Vec2 delta = current - center;
        float dist = length(delta);
        if (dist < 10.0f) return {0, 0};
        return normalize(delta) * std::min(dist / radius, 1.0f);
    }
};
```

### Multi-touch gesture detection

```cpp
// Pinch to zoom
float pinchScale(Touch t0, Touch t1) {
    return distance(t0.position, t1.position) / distance(t0.startPosition, t1.startPosition);
}

// Two-finger rotate
float rotationAngle(Touch t0, Touch t1) {
    Vec2 current = t1.position - t0.position;
    Vec2 start = t1.startPosition - t0.startPosition;
    return atan2(cross(start, current), dot(start, current));
}
```

---

## Big Picture / Steam Deck

```cpp
// Detect Big Picture / Steam Deck mode
bool isTenfoot() {
    return getenv("SteamTenfoot") != nullptr;
}

// Auto-fullscreen in Big Picture
if (isTenfoot()) setFullscreen(true);

// Font size: minimum 24px at 1920×1080
// UI must be readable from couch distance (~3 meters)

// Steam Deck specifics:
// - Screen: 1280×800 (16:10)
// - Touchscreen + trackpads + gyro
// - Default to showing controller glyphs, not KB+M
```

---

## Quick Reference

```
INPUT PRIORITY:   UI > Cutscene > Gameplay > Background
GOLDEN RULES:     1.Match icons  2.Match cursor  3.All devices work
                  4.All navigate  5.Disconnect→pause
DEAD ZONE:        0.12-0.20 (test per controller)
RESPONSE CURVE:   Quadratic (default) for sticks
COYOTE TIME:      5-8 frames (platformers)
INPUT BUFFER:     5-8 frames (fighting/action games)
CAMERA ACTION:    absolute_mouse (Steam Input, mandatory)
BIG PICTURE:      getenv("SteamTenfoot"), fullscreen, 24px+ fonts
```

→ **Steam Input API**: [partner.steamgames.com/doc/api/ISteamInput](https://partner.steamgames.com/doc/api/ISteamInput)
→ **Steam Deck**: [partner.steamgames.com/doc/steam_deck](https://partner.steamgames.com/doc/steam_deck)
→ **Related**: [README.md](README.md) section 26 — full Steam dev standards
