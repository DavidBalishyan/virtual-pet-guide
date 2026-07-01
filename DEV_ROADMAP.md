# Virtual Pet - Developer Roadmap

> **Audience:** Students of varying skill levels.
> **Hardware:** M5StickC Plus 2 (ESP32-PICO-V3-02, MPU6886, LCD 135x240, Buzzer, Microphone)
> **Pedagogical rules:** No clever syntax. Full descriptive names. Every function gets a comment.

---

## What this repo is

This repo is the finished, working copy of the virtual pet - the place you look when you
want to see how the whole thing fits together. Think of it as the answer key: every session
of the 10-week curriculum builds toward this code, and reading through it is how you learn
embedded C++ in practice.

The actual lesson plans live in `LESSON_PLANS/SESSION_01.md` through `SESSION_10.md`.
Each session turns on one more feature by flipping its `ENABLE_*` flag in
`lib/Config/scaffold_config.h` (Session 1 has everything off; the final session has it all on).
For a quick view of which features are done, see `COURSE_CHECKLIST.md`. For the big-picture
architecture, see `CLAUDE.md`.

## What this file is for now

This file used to track every task one by one, but that job has moved elsewhere. Its one
remaining job is to hold the **bonus / extension feature designs in Appendix B** - extras
that got trimmed from the main curriculum to keep each session focused, but that make great
deeper-dive material when you have time. The **Session 7-9 lesson challenges point at these
bonus designs**, so they need to stay here as a reference anyone can pick up cold.

Before you dive into the bonuses, there are two conventions the project follows that you
need to know:

- **Fullness model.** The pet's main survival stat is `fullness`. It starts full, drops
  toward 0 over time, and eating refills it. The pet dies when `fullness` hits 0, and it
  *looks* hungry when `fullness` is **low**. The accessors are `getFullness()` /
  `setFullness()`, the default constant is `DEFAULT_FULLNESS`, the NVS key is `"fullness"`,
  the stat enum value is `STAT_FULLNESS`, and the timer rule is `applyFullnessDecay()`.
  (The display-side names are `MOOD_HUNGRY`, `sprite_hungry_placeholder`, "Hungry", and
  `playHungerAlertSound()` - they describe the *look* of hunger, not the stat itself.)
- **Motion.** Tilt-reactive movement lives in `lib/Imu/tilt_motion.h/.cpp` and the
  shake-to-play link is gated behind `ENABLE_IMU_PLAY`.

## Appendix B - Bonus Features

These features were on the main to-do list at first, but got trimmed from the core
curriculum to keep each session manageable. They are **totally optional** - finish the
main pet first, then pick whichever one sounds fun. None of them are required to have
a working Tamagotchi.

Each bonus is documented the same way the main tasks are: why it exists, what you need
to build, a code skeleton that shows the shape of the API, new concepts you will learn,
and a list of ways to stretch it further. The goal is that you (or someone maintaining
this curriculum later) can pick one up cold and have enough to start.

Here is what is available:
1. **RTC Clock Widget** - display the current time on the LCD.
2. **Web Dashboard (Static Stats Page)** - phone connects to the pet, sees stats in a browser.
3. **Pet-to-Pet Stat Swap (ESP-NOW)** - two M5Sticks exchange a happiness boost.
4. **Live-Refreshing Dashboard** - the dashboard updates without page reload.
5. **Phone-Controlled Actions** - buttons on the dashboard trigger pet actions.
6. **Voice Memos** - record short audio clips and play them back through the buzzer.

---

### Bonus Feature 1 - RTC Clock Widget

**Builds on:** nothing - independent.

**Why this is a bonus rather than a critical-path task:**

The M5StickC Plus 2 has a BM8563 RTC chip on its I2C bus, accessible
via the `M5.Rtc` API in the M5Unified library. The original RTC
scope ("RTC overnight logic") wanted to use that chip to apply
accumulated decay during off-time, which would need Unix-timestamp
math, NVS persistence of the last-seen timestamp, capping logic so a
pet does not die after a week off, and a UI for setting the initial
time. Without overnight-decay logic, the only thing left is "show
HH:MM on a screen" - a stand-alone widget that does not integrate with
any other module. This is better as opt-in
bonus content than as a critical-path slug.

**Foundation scope:**

New `lib/Clock/clock_manager.h` and `.cpp`. Module skeleton:

```cpp
class ClockManager {
public:
    void begin();                     // sets a hardcoded initial time
    void getCurrentTime(int& hours, int& minutes) const;

private:
    static const int STARTING_HOUR = 12;
    static const int STARTING_MINUTE = 0;
};
```

`begin()` calls `M5.Rtc.setTime({STARTING_HOUR, STARTING_MINUTE, 0})`.
`getCurrentTime()` calls `M5.Rtc.getTime(&timeStruct)` and writes the
hours/minutes into the out-parameters.

Display the time as a new info row on the Stats screen, formatted as
`HH:MM`. Use the existing `printText()` helper. No new font, no new
colour.

**Verification path:** power on the device, see `12:00` on the Stats
screen, wait a minute, watch it tick to `12:01`.

**Stretch within this bonus:**

- **Setting the initial time via buttons.** Long-press Button A on the
  Stats screen to enter "set time" mode; B/C increment hours/minutes;
  A confirms.
- **NVS-persisted time.** Save the last-known time on every minute
  rollover so the pet remembers what time it was last on.
- **Overnight decay logic.** The maximal RTC scope -
  apply accumulated decay during off-time. Requires Unix-timestamp
  math, capping logic, and the NVS-persisted time stretch above.
- **NTP time sync.** Requires the Wireless Access Point primitive + WiFi-client mode (not the AP
  mode it actually delivers). Adds a real-world clock without
  manual time-set UI.

**New concepts introduced:**

- I2C peripheral access via the `M5.Rtc` wrapper (one-line API call,
  no protocol details exposed)
- Time as a struct (`m5::rtc_time_t` with hours/minutes/seconds fields)
- Out-parameters (`int& hours`) - or skip this by returning a small
  struct; pedagogical choice when implementing.

**Files touched:** new `lib/Clock/clock_manager.h` and `.cpp`,
`src/main.cpp` (instantiate and call `begin()`),
`lib/Display/display_manager.cpp` (info row on Stats), `CLAUDE.md`
(architecture map row for the new Clock module).

---

### Bonus Feature 2 - Web Dashboard (Static Stats Page)

**Builds on:** the Wireless Access Point primitive.

**Why this is a bonus rather than a critical-path task:**

A web server adds another module of new concepts on top of the WiFi AP primitive -
HTTP request/response, route registration, named callback functions,
`String` HTML building. We keep the
critical path at "the pet broadcasts WiFi" and let the dashboard be
an opt-in follow-on for students curious about how their phone's
browser actually talks to their pet.

**Foundation scope:**

Extend `lib/Wireless/wireless_manager.h` (or add a new file
`web_dashboard.h`) with a `WebServer server(80)` instance. Register
one route handler:

```cpp
void handleRoot() {
    String html = "<!DOCTYPE html><html><body>";
    html += "<h1>Pet Stats</h1>";
    html += "<p>Fullness: " + String(pet.getFullness()) + "</p>";
    html += "<p>Happy: "  + String(pet.getHappy())  + "</p>";
    html += "<p>Energy: " + String(pet.getEnergised()) + "</p>";
    // ... rest of the stats ...
    html += "</body></html>";
    server.send(200, "text/html", html);
}

void WirelessManager::beginDashboard(Pet& pet) {
    server.on("/", handleRoot);
    server.begin();
}
```

`server.handleClient()` runs once per `loop()` in `main.cpp`.

**Verification path:** phone connects to `PetPet-XXXX`, opens
`http://192.168.4.1/` in a browser, sees the pet's stats as plain
HTML. Refresh manually to see updated values.

**Stretch within this bonus:** the next three bonus features (3, 4,
5) all build on this one.

**New concepts introduced:**

- HTTP request/response (one route, one response - minimal slug)
- Named callback registration (`server.on(path, handler)`), which
  mirrors `addEventListener("click", handler)` from Programming II
- `String` concatenation in C++ (new - Programming II only uses
  JavaScript strings)
- "Server" vs. "client" mental model

**Important pedagogical note:** the C++ Arduino lambda form
`server.on("/", []() { ... })` is everywhere on the web but is **not
allowed** in this project (see `CLAUDE.md` - "No clever syntax").
Always use named handler functions.

**Files touched:** `lib/Wireless/wireless_manager.h` and `.cpp` (extend),
`src/main.cpp` (call `server.handleClient()` in loop), no new modules.

---

### Bonus Feature 3 - Pet-to-Pet Stat Swap (ESP-NOW)

**Builds on:** IMU shake detection (Session 3) + the Wireless AP primitive (for
the radio being initialised). Note that ESP-NOW does not actually
require WiFi to be connected - it just needs the radio to be on. The
`softAP` mode is sufficient.

**REQUIRES TWO M5StickC Plus 2 DEVICES.** This bonus cannot be tested
in a single-device setup. It is the most demanding bonus in terms of
hardware availability, but also the highest-impact in terms of "wow
factor" - two students can demo "hey, our pets just said hi" to each
other.

**Why this is a bonus rather than a critical-path task:**

Two reasons. First, the second-device requirement makes it unsuitable
as a mandatory task. Second, ESP-NOW introduces three new concepts
(MAC addresses, peer-to-peer protocol, struct-as-bytes serialisation)
that do not compose with the rest of the curriculum - students who
finish the critical path may find this content rewarding, but it is
not foundational.

**Foundation scope:**

New `lib/Wireless/peer_link.h` and `.cpp` (or extend
`WirelessManager`). Module skeleton:

```cpp
struct PetGreeting {
    uint8_t senderMac[6];     // 6 bytes
    uint8_t happinessGift;    // 1 byte - how much happy to give the receiver
};

class PeerLink {
public:
    void begin();                                  // esp_now_init, register callback
    void addPeer(const uint8_t peerMac[6]);        // hardcode the friend's MAC for now
    void sendGreeting(uint8_t happinessGift);      // esp_now_send to all peers
    bool greetingReceived() const;                 // one-frame pulse, mirrors wasShaken()
    uint8_t getLastReceivedGift() const;

private:
    static void onReceive(const uint8_t* mac, const uint8_t* data, int len);
    bool flagReceived;
    uint8_t lastGift;
};
```

In `main.cpp`'s loop: when `imu.wasShaken()` AND a peer is configured,
call `peerLink.sendGreeting(5)` (or whatever amount). On the receiving
side: if `peerLink.greetingReceived()`, call
`myPet.setHappy(myPet.getHappy() + peerLink.getLastReceivedGift())`.

**Pairing model:** for the foundation, hardcode each device's peer
MAC address into the source. Stretch: discover peers automatically by
broadcasting, or show the device's own MAC on the LCD so the
opposite student can type it in.

**Verification path (requires two devices):** flash both devices with
the appropriate peer MAC; shake device A; device B's happiness goes
up; and vice versa. Both students see "the other pet said hi."

**Stretch within this bonus:**

- **Gift items.** Send different gift types (food, toy, medicine) and
  apply different effects on the receiver.
- **Visible peer list on the LCD.** Show which peers are configured.
- **Automatic peer discovery.** Periodic broadcast packet that all
  devices respond to.

**New concepts introduced:**

- MAC address (six bytes, displayed as `XX:XX:XX:XX:XX:XX`)
- Peer-to-peer vs. client-server mental model
- Struct-as-bytes serialisation (sending `sizeof(PetGreeting)` raw
  bytes - students need to understand that this only works because
  both devices have the *same* struct definition)
- Callback registration with a C-style function pointer
  (`esp_now_register_recv_cb`)

**Files touched:** new `lib/Wireless/peer_link.h` and `.cpp` (or
extension of `WirelessManager`), `src/main.cpp` (instantiate, wire
shake -> send, wire received -> pet.setHappy), `CLAUDE.md`
(architecture map row).

---

### Bonus Feature 4 - Live-Refreshing Dashboard

**Builds on:** Bonus Feature 2 (Web Dashboard).

**Why this is a bonus:**

Bonus Feature 2 lands a static HTML page that the user must manually
refresh to see updated stats. That is fine as a teaching milestone but
not what a real dashboard feels like. This bonus adds client-side
JavaScript that polls the server every two seconds and updates the
page in place. Introduces two new concepts (browser-side JavaScript
running on the M5Stick-served page, JSON as a data-exchange format)
without changing the server's architecture much.

**Foundation scope:**

Add a second route - `/stats.json` - that returns the pet's current
state as JSON:

```cpp
void handleStatsJson() {
    String json = "{";
    json += "\"fullness\":"   + String(pet.getFullness()) + ",";
    json += "\"happy\":"    + String(pet.getHappy())  + ",";
    json += "\"energised\":"+ String(pet.getEnergised());
    // ... etc ...
    json += "}";
    server.send(200, "application/json", json);
}
```

Update the HTML page from Bonus 2 to include a small inline JS block
that calls `fetch('/stats.json')` every 2 seconds and overwrites the
DOM elements. Students have already seen `fetch` and DOM manipulation
in Programming II's DOM section, so this is curriculum-aligned.

**Stretch within this bonus:** WebSocket-based push instead of polling
(introduces a new protocol and is a lot of new code for marginal
benefit on the M5Stick).

**New concepts introduced:**

- JSON as a data-exchange format (Programming II students have seen
  JS objects but may not have used JSON for transport)
- Client-side JavaScript embedded in C++ served HTML (a tricky
  "two-languages-in-one-file" experience worth discussing explicitly)
- Polling vs. event-driven update models

**Files touched:** `lib/Wireless/wireless_manager.cpp` (or
`web_dashboard.cpp`) - add `/stats.json` route, update the root HTML
to include the JS polling block.

---

### Bonus Feature 5 - Phone-Controlled Actions

**Builds on:** Bonus Feature 2 (Web Dashboard).

**Why this is a bonus:**

The dashboard from Bonus 2 is read-only. This bonus makes it
interactive - phone-side buttons for Feed, Play, Sleep, Bathe, Heal
that trigger the corresponding `Pet::feed()`, `Pet::play()`, etc.
methods on the device. Demonstrates that an HTTP server can do more
than serve pages; it can be a control surface.

**Foundation scope:**

Add five new routes - `/feed`, `/play`, `/sleep`, `/bathe`, `/heal` -
each of which calls the corresponding pet method and redirects back
to `/`:

```cpp
void handleFeed() {
    pet.feed();
    server.sendHeader("Location", "/");
    server.send(303);
}
```

Update the root HTML to include `<a>` links (or `<form>` POST buttons)
for each action.

**Stretch within this bonus:**

- **Auth.** The dashboard is open to anyone on the AP - a "kid mode"
  password gate would teach basic HTTP auth headers.
- **Action history.** Server stores the last N actions and shows them
  on the page.

**New concepts introduced:**

- HTTP request methods (GET vs. POST - GET via `<a>` is the easier
  intro; POST via `<form>` is more "correct" and a good stretch).
- HTTP redirects (303 See Other).
- The idea that a server can have side effects on hardware state.

**Files touched:** same as Bonus 4 - extend the dashboard module with
five new routes and update the HTML.

---

### Bonus Feature 6 - Voice Memos

**Builds on:** microphone input (detect & react).

**Why this is a bonus:**

This is the **original microphone scope** before it was right-sized to a bonus. Full record + playback needs DMA-style audio buffer
handling, double-buffering, sample-rate matching against the
`M5.Speaker` output rate, and a UI for browsing recordings - three
modules of new concepts in one feature. Better as a bonus that
students can attempt after they have already shipped the simpler
"detect and react" foundation.

**Foundation scope:**

Extend `lib/Microphone/microphone_manager.h` and `.cpp`:

```cpp
class MicrophoneManager {
public:
    // ... existing detect-and-react API ...

    // Recording
    void beginRecording();                // long-press Button A enters this
    void updateRecording();               // call once per loop while recording
    void endRecording();                  // releasing Button A exits

    // Playback (uses M5.Speaker, not Microphone - left here for locality)
    void playLastRecording();
    bool hasRecording() const;

private:
    int16_t* recordingBuffer;             // heap allocated - see Hardware Gotcha 1
    size_t recordingCapacity;             // max samples
    size_t recordingLength;               // actual samples captured
};
```

Record on long-press of Button A from the Main screen. Allocate
`recordingBuffer` on the heap with capacity for ~5 seconds at 16 kHz
(~160 KB - a huge slug of memory; must be heap, never stack - see
Hardware Gotcha 1). Read samples in `updateRecording()` while the
button is held. Play back through `M5.Speaker` when a "Play Memo"
action is selected from the menu.

**Stretch within this bonus:**

- **Multiple memos.** Slot 1, slot 2, slot 3 - store in heap, browse
  with B/C.
- **NVS persistence.** Save recordings across power-off.
- **Pitch-shifted playback.** Pet "speaks" the memo at a higher pitch
  for comedic effect.
- **Memo-triggered reactions.** Pet plays back the memo when it is
  hungry / lonely.

**New concepts introduced:**

- Long-press button detection (held vs. edge - students have only
  used edge detection so far)
- Large heap allocations (~160 KB) and the importance of freeing them
- Sample-rate matching between mic and speaker
- Double-buffering (only needed at the stretch level - the foundation
  can record once, then play once)

**Critical pedagogical warning:**

This bonus is **the most heap-intensive feature in the project.** It
makes Hardware Gotcha 1 (audio buffers must use the heap) into a
concrete problem the student will hit if they ignore it. Use it as
the canonical "this is why we use `malloc()` and `free()`" lesson.

**Files touched:** `lib/Microphone/microphone_manager.h` and `.cpp`
(extend), `lib/Actions/action_menu.cpp` (add "Record" and "Play"
actions, or repurpose long-press), `src/main.cpp` (wire long-press
detection on Button A).
