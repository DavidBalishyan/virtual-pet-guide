# Adding Brightness Control to the Simplest Web Dashboard

This guide walks you through adding a screen-brightness control to the
**earliest, simplest version of the web dashboard**, the one first introduced
in commit `2b9f63b` ("Added: web dashboard").

That version is deliberately barebones:

- A single HTTP server on port 80 (`WebServer`, **no** WebSocket).
- One route, `GET /`, that builds a whole HTML page as a `String` and sends it.
- The page carries `<meta http-equiv="refresh" content="3">`, so the browser
  reloads it every 3 seconds to show fresh stats.
- It is **read-only**: there is no JavaScript and no way to send a command back
  to the device.

Because there is no WebSocket and no command channel, we **cannot** reuse the
approach the feature eventually shipped with (a `setBrightness` JSON command over
a socket). Instead we add:

1. Brightness methods on `DisplayManager` (this part is identical to the shipped
   version; it is the one place that talks to the backlight).
2. A **new HTTP route** that reads a brightness value from the query string.
3. A **plain HTML form** on the dashboard that submits to that route.

No JavaScript, no build system. It stays true to the simplest version.

> **How this differs from the shipped feature.** The version now on `main`
> (commit `8cc2b12`) sends `{"action":"setBrightness","value":N}` over a
> WebSocket and updates a slider live. This guide targets the pre-WebSocket
> dashboard, so we use an old-fashioned form submit plus page reload instead.
> The `DisplayManager` changes are the same in both.

---

## Prerequisites

Make sure you are working from the simplest dashboard, roughly commit `2b9f63b`.
The key tell is that `lib/Wireless/wireless_manager.cpp` builds the page with
`buildStatsPage()` and contains the line:

```cpp
page += "<meta http-equiv=\"refresh\" content=\"3\">";
```

If instead you see `dashboard.html`, `dashboard.js`, and `WebSocketsServer`,
you are on a later version and should follow the WebSocket approach in commit
`8cc2b12` rather than this guide.

---

## Step 1: Teach `DisplayManager` about brightness

The backlight is the display's responsibility, so all brightness logic lives in
`DisplayManager`. Everything else just calls into it.

### 1a. Header: `lib/Display/display_manager.h`

Add the stored state and default in the `private:` section (near the other
geometry constants is fine):

```cpp
    // Backlight brightness the screen starts at, as a 0-100 percentage.
    // Applied once in init(); the dashboard can change it live afterwards.
    static const uint8_t DEFAULT_BRIGHTNESS_PERCENT = 100;

    // The current backlight level as a 0-100 percentage. Kept here so the
    // dashboard can read back the value it just set (see getBrightness).
    uint8_t brightnessPercent;
```

Add the public methods in the `public:` section (near `clearScreen` /
`fillRect`):

```cpp
    // Backlight brightness control. percent is clamped to 0-100 and mapped to
    // the LCD's native 0-255 range. The dashboard drives these; nothing else in
    // the firmware changes brightness, so this is the one place that talks to
    // the backlight. getBrightness() returns the last value set (as a percent)
    // so the dashboard control can show the device's real level.
    void setBrightness(int percent);
    int  getBrightness() const;
```

### 1b. Implementation: `lib/Display/display_manager.cpp`

Initialise `brightnessPercent` in the constructor:

```cpp
DisplayManager::DisplayManager()
    : lastRenderedScreen(SCREEN_MAIN),
      petWasDeadLastFrame(false),
      brightnessPercent(DEFAULT_BRIGHTNESS_PERCENT) {
}
```

Apply the starting brightness at the end of `init()`, just before the screen is
first cleared and pushed:

```cpp
    // Set the backlight to a known level so getBrightness() reports the true
    // value from the first frame, rather than whatever M5.begin() left it at.
    setBrightness(brightnessPercent);

    clearScreen(TFT_BLACK);
    pushCanvas();
```

Add the two methods themselves (anywhere sensible, e.g. just after
`pushCanvas()`):

```cpp
// setBrightness() is the one place that drives the LCD backlight.
// percent is clamped to 0-100 and scaled to the panel's native 0-255 range.
void DisplayManager::setBrightness(int percent) {
    if (percent < 0)   { percent = 0; }
    if (percent > 100) { percent = 100; }
    brightnessPercent = (uint8_t)percent;

    // Map 0-100% onto the 0-255 range the LCD driver expects.
    uint8_t level = (uint8_t)((percent * 255) / 100);
    M5.Lcd.setBrightness(level);
}

// getBrightness() returns the last brightness set, as a 0-100 percentage.
// Lets the dashboard control reflect the device's actual level.
int DisplayManager::getBrightness() const {
    return brightnessPercent;
}
```

---

## Step 2: Give `WirelessManager` a handle on the display

The dashboard needs to reach the display to change and report brightness.

### 2a. Header: `lib/Wireless/wireless_manager.h`

Include the display header at the top:

```cpp
#include "../Display/display_manager.h"
```

Add a setter to the `public:` section and a pointer to `private:`:

```cpp
public:
    // ... existing members ...

    // Gives the dashboard a handle on the screen so the /brightness route can
    // adjust the backlight and the page can report the current level.
    void setDisplay(DisplayManager& display);

private:
    Pet*            petPtr;    // Points to the pet whose stats we serve
    DisplayManager* displayPtr; // Points to the screen for brightness control
    WebServer       server;    // HTTP server on port 80
```

Also declare the new route handler alongside `handleRoot()`:

```cpp
    // Handles GET /brightness?value=N. Sets the backlight, then redirects
    // back to / so the refreshed page shows the new level.
    void handleBrightness();
```

### 2b. Implementation: `lib/Wireless/wireless_manager.cpp`

Initialise the new pointer in the constructor:

```cpp
WirelessManager::WirelessManager()
    : petPtr(nullptr), displayPtr(nullptr), server(HTTP_PORT) {
}
```

Add the setter (near the top of the file, after the constructor):

```cpp
void WirelessManager::setDisplay(DisplayManager& display) {
    displayPtr = &display;
}
```

---

## Step 3: Add the `/brightness` HTTP route

Register the route in `begin()`, right next to where `"/"` is registered:

```cpp
    server.on("/", std::bind(&WirelessManager::handleRoot, this));
    server.on("/brightness", std::bind(&WirelessManager::handleBrightness, this));
```

Then add the handler. It reads the `value` query parameter, clamps it via
`setBrightness`, and sends a `303 See Other` redirect back to `/` so the browser
lands on the freshly rebuilt page:

```cpp
// Handles GET /brightness?value=N.
// Reads the requested percentage, applies it to the screen, then redirects
// back to the dashboard so the reloaded page shows the confirmed level.
void WirelessManager::handleBrightness() {
    // server.arg() returns "" for a missing parameter; toInt() then yields 0,
    // which setBrightness() safely clamps, so no explicit "missing" check is
    // needed. But only touch the display if we actually have one wired up.
    if (displayPtr && server.hasArg("value")) {
        int percent = server.arg("value").toInt();
        displayPtr->setBrightness(percent);
    }

    // 303 plus Location sends the browser back to / with a plain GET, so a
    // manual reload won't re-submit the brightness change.
    server.sendHeader("Location", "/");
    server.send(303, "text/plain", "");
}
```

---

## Step 4: Add the control to the dashboard page

Now surface a control in `buildStatsPage()`. Because this version has no
JavaScript, we use a plain HTML `<form method="get" action="/brightness">`.
A range slider plus a Set button works and needs zero script.

First, a little CSS. Find the `<style>` block in `buildStatsPage()` and add a
couple of rules near the footer rule:

```cpp
    // Brightness control
    page += ".brightness{margin-top:20px;}";
    page += ".brightness form{display:flex;align-items:center;gap:10px;}";
    page += ".brightness input[type=range]{flex:1;accent-color:#f1c40f;}";
    page += ".brightness button{background:#f1c40f;border:none;border-radius:6px;"
            "padding:6px 12px;font-weight:bold;cursor:pointer;}";
    page += ".brightness h2{font-size:15px;color:#f1c40f;margin-bottom:6px;}";
```

Then add the control's markup. A good spot is just before the footer `<div>`.
Note we seed the slider's `value` from `getBrightness()` so it shows the
device's real level on every 3-second refresh:

```cpp
    // Brightness control. A plain GET form so no JavaScript is needed.
    // The slider's starting position reflects the device's current level.
    int currentBrightness = displayPtr ? displayPtr->getBrightness() : 100;
    page += "<div class=\"brightness\">";
    page += "<h2>Display Brightness</h2>";
    page += "<form method=\"get\" action=\"/brightness\">";
    page += "<input type=\"range\" name=\"value\" min=\"0\" max=\"100\" value=\""
            + String(currentBrightness) + "\">";
    page += "<button type=\"submit\">Set</button>";
    page += "</form>";
    page += "</div>";
```

> **Why a Set button and not "live" dragging?** Live updates would need
> JavaScript and a socket, which is exactly what the WebSocket version added
> later. On this simplest dashboard, the user drags the slider and presses
> **Set**; the form submits to `/brightness`, the device applies it, and the
> redirect brings back a page whose slider is already at the new value.

---

## Step 5: Wire it up in `main.cpp`

In `setup()`, after `wireless.begin(myPet);`, hand the display to the dashboard.
This must come after `display.init()` (which sets the initial brightness):

```cpp
    wireless.begin(myPet);
    // Let the dashboard drive the screen backlight and read its current level.
    wireless.setDisplay(display);
```

Both `display` and `wireless` are already global instances in `main.cpp`, so
there is nothing new to construct.

---

## Step 6: Update the test stub

The native unit tests compile against a fake M5 library in
`test/stubs/M5StickCPlus2.h`. Add `setBrightness` to the fake `Lcd` so the
tests still build:

```cpp
        void setRotation(int) {}
        void setBrightness(uint8_t) {}   // <-- add this line
        uint16_t color565(uint8_t, uint8_t, uint8_t) { return 0; }
```

Without this, any test that pulls in `DisplayManager` will fail to compile
because `M5.Lcd.setBrightness()` is undefined in the stub.

---

## Step 7: Build and test

1. **Native tests** (fast sanity check that nothing broke):

   ```bash
   pio test -e native
   ```

2. **Flash the device**:

   ```bash
   pio run -e <your-device-env> -t upload
   ```

3. **On the device**:
   - Connect your phone/laptop to the `VirtualPet` WiFi network.
   - Open `http://192.168.4.1`.
   - Drag the **Display Brightness** slider and press **Set**.
   - The screen backlight should change immediately, and the reloaded page's
     slider should sit at the value you chose.
   - Try `0` (screen goes dark) and `100` (full brightness) as edge cases.

---

## How the pieces fit together

```
Browser                         ESP32 (device)
  │  GET /brightness?value=40
  ├───────────────────────────────►  handleBrightness()
  │                                     └─ displayPtr->setBrightness(40)
  │                                          └─ M5.Lcd.setBrightness(102)  // 40% of 255
  │  303 Location: /
  ◄───────────────────────────────┤
  │  GET /
  ├───────────────────────────────►  handleRoot() → buildStatsPage()
  │                                     └─ slider value = getBrightness() = 40
  │  200 text/html (slider at 40%)
  ◄───────────────────────────────┤
```

The single source of truth for brightness is `DisplayManager::brightnessPercent`.
The dashboard only ever *asks* the display to change and *reads back* what it
reports. That is the same discipline the shipped WebSocket version follows, just
over plain HTTP.

---

## Summary of files touched

| File | Change |
|------|--------|
| `lib/Display/display_manager.h` | Declare `setBrightness`/`getBrightness`, add `brightnessPercent` + default |
| `lib/Display/display_manager.cpp` | Init brightness in ctor + `init()`, implement both methods |
| `lib/Wireless/wireless_manager.h` | Include display header, add `setDisplay` + `displayPtr` + `handleBrightness` |
| `lib/Wireless/wireless_manager.cpp` | Init `displayPtr`, register route, add handler + form markup + CSS |
| `src/main.cpp` | Call `wireless.setDisplay(display)` in `setup()` |
| `test/stubs/M5StickCPlus2.h` | Add `setBrightness(uint8_t)` to the fake `Lcd` |
