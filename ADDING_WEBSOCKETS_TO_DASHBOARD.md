# Adding WebSockets to the Dashboard

This guide builds on the web dashboard from `WIRELESS_GUIDE.md` and replaces the clunky `<meta http-equiv="refresh">` with a live-updating WebSocket connection. The bars slide smoothly, the page never flickers, and the pet name/mood update in real time.

---

## Part 1 - What is wrong with `<meta refresh>`?

The original dashboard used a meta tag to reload the entire page every three seconds:

```html
<meta http-equiv="refresh" content="3">
```

This works, but it has problems:

- **White flash.** The browser tears down the old page and renders the new one. You see a blink on every refresh.
- **Wasted bandwidth.** The browser re-downloads the CSS, the layout, the mood badge everything when only eight numbers have changed.
- **No transitions.** The CSS `transition: width 0.3s` on the bars does nothing, because every refresh replaces the DOM from scratch.

A real-time dashboard should update the numbers in place, leaving the rest of the page untouched. That is where WebSockets come in.

---

## Part 2 - What is a WebSocket?

A WebSocket is a persistent two-way connection between the browser and the server. Unlike HTTP (where the client asks and the server answers, then the connection closes), a WebSocket stays open. Either side can send a message at any time.

```
HTTP:  client asks -> server answers -> connection closes
WS:    client connects -> both sides can send freely -> connection stays open
```

For the dashboard, this means:

1. The browser connects once (when the page loads).
2. The server pushes new stats every 500ms without the browser asking.
3. The browser updates the bar widths and mood text in place  no page reload, no flash.

The ESP32 runs a `WebSocketsServer` on port 81, right alongside the normal HTTP server on port 80.

---

## Part 3 - Architecture overview

The new dashboard has five files:

```
lib/Wireless/
  wireless_manager.h        class declaration (now with WebSocket + JSON)
  wireless_manager.cpp      implementation
  dashboard.css             styles (extracted to its own file)
  dashboard.html            page template (extracted to its own file)
  dashboard.js              WebSocket client (extracted to its own file)
```

At build time, the three asset files (css, html, js) are rolled into a single generated header by a Perl script:

```
tools/embed_file.pl         reads text files, writes PROGMEM string constants
lib/Wireless/dashboard_assets.h   generated: DASHBOARD_CSS, DASHBOARD_HTML, DASHBOARD_JS
```

The generated header is listed in `.gitignore` and recreated by `make generate` (which runs automatically before `make build`).

---

## Part 4 - The generated asset header

The old code stored CSS and HTML directly inside C++ string concatenations. That made it hard to edit the page  you had to reflash the device to change a colour or fix a typo.

Now each asset lives in a plain file that you can edit with any text editor:

| Source file | Generated constant | Purpose |
|---|---|---|
| `dashboard.css` | `DASHBOARD_CSS` | Served at `/style.css` |
| `dashboard.html` | `DASHBOARD_HTML` | Served at `/` (with `{{PET_NAME}}` placeholder) |
| `dashboard.js` | `DASHBOARD_JS` | Served at `/script.js` |

The Perl script `tools/embed_file.pl` reads each file and wraps it in a `PROGMEM` string constant:

```perl
# Usage: perl tools/embed_file.pl <dst> <src1>:<var1> <src2>:<var2> ...
# Generates a single header with all variables

static const char DASHBOARD_CSS[] PROGMEM = R"rawliteral(body { ... })rawliteral";
static const char DASHBOARD_HTML[] PROGMEM = R"rawliteral(<!DOCTYPE html>...)rawliteral";
static const char DASHBOARD_JS[] PROGMEM = R"rawliteral((() => { ... })())rawliteral";
```

`PROGMEM` tells the compiler to store the string in flash memory (which the ESP32 has plenty of) instead of precious RAM. The `R"rawliteral(...)rawliteral"` syntax is a C++ raw string literal  everything between the delimiters is treated as literal text, so you do not need to escape quotes or newlines.

The single-header approach means `wireless_manager.cpp` only needs one include:

```cpp
#include "dashboard_assets.h"
```

instead of three separate includes.

---

## Part 5 - Serving the assets

Three HTTP routes are registered in `begin()`:

```cpp
void WirelessManager::begin(Pet& pet) {
    petPtr = &pet;
    WiFi.softAP(AP_SSID);

    server.on("/",         std::bind(&WirelessManager::handleRoot,  this));
    server.on("/style.css", std::bind(&WirelessManager::handleCSS,  this));
    server.on("/script.js", std::bind(&WirelessManager::handleJS,   this));
    server.begin();
    // ...
}
```

Each handler sends its `PROGMEM` constant with the right content type:

```cpp
void WirelessManager::handleCSS() {
    server.send_P(200, "text/css", DASHBOARD_CSS);
}

void WirelessManager::handleJS() {
    server.send_P(200, "application/javascript", DASHBOARD_JS);
}
```

The `send_P` method (note the `_P` suffix) sends data directly from flash memory, avoiding a copy through RAM. This is important for the CSS file (about 1.3KB) and the JS file (about 2.5KB)  copying them through RAM would eat into the heap.

---

## Part 6 - The HTML template

The page template is stored in `dashboard.html` as a plain HTML file:

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>Virtual Pet Dashboard</title>
  <link rel="stylesheet" href="/style.css">
  <script src="/script.js" defer></script>
</head>
<body>
  <h1>Virtual Pet</h1>
  <p class="pet-name" id="petName">{{PET_NAME}}</p>
  <div class="mood">
    <span class="mood-badge" id="moodBadge">Loading...</span>
  </div>
  <div id="stats"></div>
  <p id="deathNotice" class="death-notice">Your pet has died!</p>
  <div class="footer">
    Connected to <strong>VirtualPet</strong> -
    <span class="status-led" id="statusLed"></span>
    <span id="wsStatus">connecting...</span>
  </div>
</body>
</html>
```

A few things to notice:

- **`<script src="/script.js" defer>`**  the JavaScript is loaded from a separate file, not inlined. The `defer` attribute tells the browser to download the script in parallel with the page and run it after the HTML is parsed.
- **`{{PET_NAME}}`**  a placeholder that `buildStatsPage()` replaces with the real pet name at request time:

```cpp
String WirelessManager::buildStatsPage() {
    String page = DASHBOARD_HTML;
    page.replace("{{PET_NAME}}", String(petPtr->getPetName()));
    return page;
}
```

- **No `<meta refresh>`**  the page loads once and stays put. Updates come through the WebSocket, not page reloads.
- **No initial stat rows**  the `<div id="stats">` is empty. The JavaScript populates it when the first WebSocket message arrives.

---

## Part 7 - The WebSocket server on the C++ side

The `wireless_manager.h` header adds a `WebSocketsServer` member alongside the existing `WebServer`:

```cpp
class WirelessManager {
    // ...
private:
    Pet*              petPtr;
    WebServer         server;         // HTTP on port 80
    WebSocketsServer  webSocket;      // WS on port 81
    unsigned long     lastBroadcastMs;

    void onWebSocketEvent(uint8_t num, WStype_t type,
                          uint8_t* payload, size_t length);
    String buildStatsJson();
    String moodToJson(MoodSprite mood);
};
```

### 7.1 - Starting the WebSocket server

In `begin()`, after the HTTP routes are registered:

```cpp
webSocket.begin();
webSocket.onEvent(std::bind(&WirelessManager::onWebSocketEvent, this,
                            std::placeholders::_1, std::placeholders::_2,
                            std::placeholders::_3, std::placeholders::_4));
```

`onEvent` registers a callback that fires on every WebSocket event: connect, disconnect, text message, and so on. The `std::bind` and placeholder syntax wraps the member function into a plain callback the library can call.

### 7.2 - Polling the WebSocket

In `handleClient()`, the WebSocket server needs its own polling call alongside the HTTP server:

```cpp
void WirelessManager::handleClient() {
    server.handleClient();
    webSocket.loop();
}
```

Without `webSocket.loop()`, the server would never process incoming WebSocket data or send outgoing messages.

### 7.3 - Handling events

The event handler responds to three event types:

```cpp
void WirelessManager::onWebSocketEvent(uint8_t num, WStype_t type,
                                        uint8_t* payload, size_t length) {
    switch (type) {
        case WStype_DISCONNECTED:
            break;
        case WStype_CONNECTED: {
            String json = buildStatsJson();
            webSocket.sendTXT(num, json);
            break;
        }
        case WStype_TEXT:
            break;
        default:
            break;
    }
}
```

When a browser connects (`WStype_CONNECTED`), the server immediately sends one JSON snapshot so the page renders without waiting for the next broadcast tick. This makes the initial load feel instant.

### 7.4 - Broadcasting stats on a timer

Stats are pushed to every connected browser at a regular interval using `broadcastStats()`:

```cpp
void WirelessManager::broadcastStats() {
    unsigned long now = millis();
    if (now - lastBroadcastMs < BROADCAST_INTERVAL) return;
    lastBroadcastMs = now;

    String json = buildStatsJson();
    webSocket.broadcastTXT(json);
}
```

`BROADCAST_INTERVAL` is 500ms  twice per second. This is fast enough to feel smooth when a stat is ticking down, but slow enough to keep the CPU free for the main loop. The `broadcastTXT()` call sends the JSON string to every connected WebSocket client at once.

`broadcastStats()` is called from `loop()` in `main.cpp`:

```cpp
#ifdef ENABLE_WIRELESS
wireless.broadcastStats();
#endif
```

### 7.5 - Building the JSON

The JSON payload is built with a helper that reads the eight stats and the pet's mood:

```cpp
String WirelessManager::buildStatsJson() {
    String json = "{";
    json += "\"pet\":{";
    json += "\"name\":\"" + String(petPtr->getPetName()) + "\",";
    json += "\"mood\":\"" + moodToJson(petPtr->computeMood()) + "\",";
    json += "\"alive\":" + String(petPtr->isInDeadState() ? "false" : "true");
    json += "},";
    json += "\"stats\":{";
    json += "\"fullness\":"     + String(petPtr->getFullness()) + ",";
    json += "\"happy\":"        + String(petPtr->getHappy()) + ",";
    // ... six more stats ...
    json += "}";
    json += "}";
    return json;
}
```

The result looks like this:

```json
{
  "pet": {
    "name": "Bunny",
    "mood": "happy",
    "alive": true
  },
  "stats": {
    "fullness": 72,
    "happy": 88,
    "energy": 45,
    "cleanliness": 90,
    "sick": 0,
    "hydration": 60,
    "tired": 12,
    "sad": 3
  }
}
```

The mood is encoded as a string using `moodToJson()`, which maps the `MoodSprite` enum to lowercase strings:

```cpp
String WirelessManager::moodToJson(MoodSprite mood) {
    switch (mood) {
        case MOOD_HAPPY:  return "happy";
        case MOOD_UNWELL: return "unwell";
        case MOOD_HUNGRY: return "hungry";
        case MOOD_THIRSTY: return "thirsty";
        case MOOD_NEUTRAL:
        default:          return "neutral";
    }
}
```

---

## Part 8 - The JavaScript client

The JavaScript file `dashboard.js` runs in the browser. It connects to the WebSocket, receives JSON updates, and patches the DOM in place.

### 8.1 - Connecting

```js
const host = location.hostname;
const ws = new WebSocket('ws://' + host + ':81/');
```

`location.hostname` is the IP address the browser used to load the page (usually `192.168.4.1`). The WebSocket connects to the same host on port 81.

The initial connection state is shown in the footer  the status LED glows red with the text "disconnected" until the WebSocket opens.

### 8.2 - Connection state

```js
const setConnected = function(ok) {
    const led = document.getElementById('statusLed');
    const txt = document.getElementById('wsStatus');
    if (ok) {
        led.style.background = '#2ecc71';
        txt.textContent = 'connected';
    } else {
        led.style.background = '#e74c3c';
        txt.textContent = 'disconnected';
    }
};
```

This gives immediate visual feedback. If the LED stays red, you know the WebSocket did not open  check that the device is broadcasting and that your browser supports WebSockets (all modern browsers do).

On disconnect, the page tries to reconnect after three seconds:

```js
ws.onclose = function() {
    setConnected(false);
    if (reconnectTimer) clearTimeout(reconnectTimer);
    reconnectTimer = setTimeout(function() {
        location.reload();
    }, 3000);
};
```

A full page reload on reconnect ensures the JavaScript state is clean (no stale timers, no corrupted DOM). The three-second delay prevents a reconnect storm if the WiFi is flaky.

### 8.3 - Receiving and rendering data

The `onmessage` handler is the heart of the client:

```js
ws.onmessage = function(e) {
    const data = JSON.parse(e.data);
    document.getElementById('petName').textContent = data.pet.name;
    const mb = document.getElementById('moodBadge');
    const m = data.pet.mood;
    mb.textContent = m.charAt(0).toUpperCase() + m.slice(1);
    mb.style.background = moodColors[m] || '#95a5a6';
    const dn = document.getElementById('deathNotice');
    dn.style.display = data.pet.alive ? 'none' : 'block';
    // ... render stat bars ...
};
```

It updates four things:

1. **Pet name**  in case it changes (the pet keeps its name for life, so this is mostly future-proofing).
2. **Mood badge**  the text (capitalised: "Happy", "Unwell", etc.) and the background colour.
3. **Death notice**  shown or hidden based on `data.pet.alive`.
4. **Stat bars**  rebuilt from scratch every message.

The stat bars are rebuilt by clearing the container and creating new DOM elements in a loop:

```js
const sc = document.getElementById('stats');
sc.innerHTML = '';
for (let i = 0; i < statConfigs.length; i++) {
    const s = statConfigs[i];
    let v = data.stats[s.key];
    if (v < 0) v = 0;
    if (v > 100) v = 100;
    const d = document.createElement('div');
    d.className = 'stat';
    d.innerHTML =
        '<div class="stat-label">' +
        '<span class="stat-name">' + s.label + '</span>' +
        '<span class="stat-value">' + data.stats[s.key] + '</span>' +
        '</div>' +
        '<div class="bar-bg">' +
        '<div class="bar-fill" style="width:' + v + '%;background:' + s.color + ';"></div>' +
        '</div>';
    sc.appendChild(d);
}
```

This is fast enough for eight bars at 500ms intervals, even on a phone. Rebuilding the innerHTML from a template string is actually faster than creating individual text nodes, because the browser parses and inserts the whole fragment in one pass.

### 8.4 - Config arrays

The mood colours and stat configurations are stored as arrays rather than inlined in the rendering code:

```js
const moodColors = {
    happy: '#2ecc71',
    unwell: '#9b59b6',
    hungry: '#e74c3c',
    thirsty: '#e67e22',
    neutral: '#95a5a6'
};

const statConfigs = [
    { key: 'fullness', label: 'Fullness', color: '#e74c3c' },
    { key: 'happy', label: 'Happy', color: '#2ecc71' },
    // ...
];
```

This separation makes it easy to add a new stat later  just add an entry to `statConfigs` and a field to the JSON on the C++ side.

---

## Part 9 - How it all flows

Compare this with the old HTTP request-response cycle from Part 9 of `WIRELESS_GUIDE.md`:

- **Old:** browser requests full HTML page -> server builds everything -> browser tears down and rerenders -> white flash.
- **New:** browser loads page once -> server pushes 300 bytes of JSON -> browser updates six DOM nodes -> smooth, no flash.

The JSON payload (~300 bytes) is about 5% of the full HTML page (~6KB). Less data over the air, less work for the browser, less battery drain on your phone.

---

## Part 10 - Changes from the original dashboard

Here is a summary of everything that changed from the `<meta refresh>` version:

| Area | Before | After |
|---|---|---|
| Page refresh | `<meta http-equiv="refresh" content="3">` | WebSocket push every 500ms |
| CSS location | Inline in C++ string | `dashboard.css`  `DASHBOARD_CSS` via `PROGMEM` |
| JS location | Inline in HTML `<script>` | `dashboard.js`  `DASHBOARD_JS` via `PROGMEM` |
| HTML location | Inline in C++ string | `dashboard.html`  `DASHBOARD_HTML` via `PROGMEM` |
| Asset delivery | One route (`/`) | Three routes (`/`, `/style.css`, `/script.js`) |
| Stat rendering | Server-built HTML on every request | Client-built DOM from JSON |
| Initial page load | Full HTML with all 8 bars | Empty page, bars appear when WS connects |
| Data format | HTML | JSON |
| Bandwidth per update | ~6KB (full page) | ~300 bytes (JSON only) |
| Server port | 80 (HTTP) | 80 (HTTP) + 81 (WebSocket) |

The server-side `statRow()` and `moodBadgeHtml()` helper functions are no longer needed  the client renders stats and mood from JSON. They have been removed from `wireless_manager.cpp`.

---

## Part 11 - Testing the WebSocket dashboard

Testing is the same as before, but the experience is different:

1. **Flash the device** with `make build` or `pio run -e m5stick-c -t upload`.
2. **Connect your phone** to the `VirtualPet` WiFi network.
3. **Open `http://192.168.4.1/`** in a browser.
4. **Watch the bars slide.** They update smoothly without any page flash.
5. **Check the status LED** in the footer  it should glow green with "connected" next to it.
6. **Let the pet's stats decay.** The bars tick down in real time, following the same decay rate as the device screen.

### If something goes wrong

- **The page loads but bars are empty.** The WebSocket didn't connect. Open the browser's developer console (Chrome: F12  Console) and look for a WebSocket error. Common causes: connecting to the wrong WiFi, or a firewall blocking port 81.
- **The status LED stays red.** Same diagnosis  the WebSocket `onopen` never fired.
- **The bars update but feel jerky.** Your phone may be on a congested 2.4 GHz channel. The device and phone are the only two devices on the `VirtualPet` network, so interference from nearby WiFi networks is the usual culprit.
- **The browser shows "failed to load script.js".** The ESP32 has limited flash and RAM  if the firmware is too large, the JS file may not have been served correctly. Check the serial monitor for error messages during the page request.
