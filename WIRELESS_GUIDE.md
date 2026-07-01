# Web Dashboard Guide

This guide walks you through the web dashboard - a live stats page that the
pet serves over WiFi. Connect your phone or laptop to the pet's own wireless
network, open a browser, and you will see all eight stats as coloured progress
bars that update every few seconds. No internet needed. The device is the server.

---

## Part 1 - What you are building

Right now the pet's stats live on the little 135x240 LCD. That is fine when
you are holding the device, but what if you want to check on the pet from
across the room? Or show a friend what the stats look like without passing
the hardware around?

The web dashboard solves that. The ESP32 starts its own WiFi access point
(a tiny wireless network that comes from the device itself, not from a
router). When you connect your phone to that network and open a web browser,
the pet sends back an HTML page with all the stats drawn as coloured bars.
The page refreshes automatically every three seconds, so you can watch the
numbers change in real time.

This is the same request-response pattern as every website you have ever
visited. Your phone (the client) asks for a page. The pet (the server)
builds the HTML and sends it back. The only difference is that the server
is a microcontroller the size of a stick of gum, running on battery, with
no operating system and no files.

---

## Part 2 - What is a web server, and how does one fit on an ESP32?

A web server is just a program that sits waiting for a connection. When a
browser connects and sends a request (a piece of text that says "GET /"),
the server builds a response (some HTML) and sends it back. That is all.
There is no magic - just a socket, a string, and a send.

On a laptop you would use a framework like Express (Node.js) or Django
(Python). On the ESP32 you use the `WebServer` library, which is part of
the Arduino ecosystem. It handles the low-level TCP networking and HTTP
parsing so you only need to write the part that builds the reply.

The M5StickC Plus 2 has built-in WiFi hardware. It can run in two modes:

- **Station mode** - connects to an existing WiFi network (like your
  home router), the way a phone or laptop normally does.
- **Soft AP mode** - broadcasts its own network, acting as a miniature
  access point. Other devices connect to *it*.

The dashboard uses soft AP mode. The pet broadcasts a network called
`VirtualPet`. Your phone connects to that network directly, device to
device, with no router involved. Once connected, your phone sends HTTP
requests to the pet's IP address (`192.168.4.1`) and the pet replies.

---

## Part 3 - The two files: `wireless_manager.h` and `wireless_manager.cpp`

The dashboard code lives in two files, following the same header/source
split as every other module in this project:

```
lib/Wireless/
  wireless_manager.h    - class declaration (what the module looks like)
  wireless_manager.cpp  - implementation (what it does)
```

The header is the public face. It declares the `WirelessManager` class
with three methods and two private members:

```cpp
class WirelessManager {
public:
    WirelessManager();
    void begin(Pet& pet);
    void handleClient();

private:
    Pet*       petPtr;   // Points to the pet whose stats we serve
    WebServer  server;   // HTTP server on port 80

    void handleRoot();
    String buildStatsPage();
};
```

- **`begin(pet)`** - starts the WiFi access point, stores a pointer to
  the pet so it can read live stats later, and registers the route
  handler. Called once from `setup()`.
- **`handleClient()`** - processes one incoming web request. Called
  once per `loop()` so the server does not stall or drop connections.
- **`handleRoot()`** - the function that runs when a browser requests
  `GET /`. It builds the HTML page and sends it back.
- **`buildStatsPage()`** - a private helper that assembles the whole
  HTML document as a `String`.

The `petPtr` is a pointer (a memory address) rather than a copy of the
pet object. That is important: the stats change every frame, and the
dashboard needs to read the *current* values, not a snapshot from when
the server started. A pointer lets `WirelessManager` reach into the
original `Pet` object in `main.cpp` and read its live fields every time
a request arrives.

The class is gated behind a feature switch, the same as every major
feature in this project:

```cpp
// In scaffold_config.h
#define ENABLE_WIRELESS
```

Everything inside `lib/Wireless/` is wrapped in `#ifdef ENABLE_WIRELESS`
in `main.cpp`, so the module simply does not exist when the switch is off.

---

## Part 4 - Starting the access point

When `begin()` is called, the first thing it does is start the ESP32's
software access point:

```cpp
void WirelessManager::begin(Pet& pet) {
    petPtr = &pet;

    WiFi.softAP(AP_SSID);

    server.on("/", std::bind(&WirelessManager::handleRoot, this));
    server.begin();
}
```

The SSID (the network name) is a constant near the top of the `.cpp`
file:

```cpp
static const char* AP_SSID = "VirtualPet";
```

"Static" here means the constant is only visible inside this file -
no other module can see or change it. It is the C++ equivalent of a
private variable at file level.

The access point has no password. That is a deliberate choice. The pet
broadcasts a short-range network (a few metres at most), so the only
people who can connect are those physically near the device. Adding a
password would mean entering it on your phone every time you want to
check the stats - a significant friction for very little security gain.
If you were building a product for strangers, you would add one. For a
teaching pet that lives on your desk, it is not worth the complexity.

After the AP is running, the next line registers the route handler:

```cpp
server.on("/", std::bind(&WirelessManager::handleRoot, this));
```

This tells the server: "when a browser requests the root path `/`,
call `handleRoot()`." The `std::bind` part is a C++ trick that wraps
a member function (which needs a `this` pointer to know which object
to call it on) into a plain function the `WebServer` library can call.
You can read it as: *"make a callable thing that, when invoked, calls
`handleRoot` on `this` WirelessManager object."*

The final line `server.begin()` tells the web server to start listening.
After this, any browser that connects to the pet's IP will trigger the
handler.

---

## Part 5 - Building the HTML page with string concatenation

When a browser requests the page, `handleRoot()` runs:

```cpp
void WirelessManager::handleRoot() {
    String html = buildStatsPage();
    server.send(200, "text/html", html);
}
```

It asks `buildStatsPage()` to create the whole HTML document as a
`String`, then sends it back with HTTP status code 200 (which means
"OK") and a content type of `text/html` (which tells the browser
"this is a web page, render it as such").

### 5.1 - The `String` type

In normal C++ you would use `std::string`. On the ESP32 with the
Arduino framework, `String` is the standard type for text. It works
the same way: you can concatenate with `+`, append numbers with
`String(42)`, and build up a document one piece at a time.

`String` has one important quirk on microcontrollers: it allocates
memory on the heap (a region of RAM the program can request chunks
from at runtime). Every `+` or append may allocate a new block, copy
the old text over, and free the old block. That is fine for a single
web page built once per request, but you would not want to do it in a
hot loop that runs hundreds of times per second. The dashboard only
builds the page when a browser actually asks for it - once every few
seconds at most - so the cost is negligible.

### 5.2 - The structure of the HTML page

The page `buildStatsPage()` builds has four sections, assembled in
order:

```
1. DOCTYPE + head     - character encoding, title, inline CSS styles
2. Title + pet name   - the page heading
3. Mood badge         - a coloured pill showing the pet's current mood
4. Stat rows          - eight labelled progress bars, one per stat
5. Dead notice        - a red warning if the pet has died (conditional)
6. Footer             - network name and a refresh link
```

### 5.3 - Inline CSS for styling

All the styling is written directly into the HTML inside a `<style>`
tag. There is no separate CSS file, because the pet has no filesystem.
The stylesheet is just another string concatenated into the page:

```cpp
page += "<style>";
page += "body{background:#1a1a2e;color:#eee;font-family:sans-serif;"
        "padding:16px;max-width:400px;margin:0 auto;}";
page += "h1{color:#f1c40f;text-align:center;margin-bottom:4px;}";
// ... more rules ...
page += "</style>";
```

A few style choices worth noticing:

- **Dark background (`#1a1a2e`).** Matches the device screen's black
  background, so the dashboard feels like an extension of the device
  itself, not a separate thing.
- **Max-width 400px.** On a phone in portrait mode the page fills the
  screen. On a laptop it stops at 400px and centres itself, so the
  bars do not stretch across the whole monitor.
- **Transition on the bar width (`transition: width 0.3s`).** When the
  page auto-refreshes (see Part 7), the bars animate smoothly to their
  new width instead of snapping. A small detail that makes the
  dashboard feel alive.

### 5.4 - The stat rows

Each stat is drawn as a labelled progress bar. A helper function called
`statRow()` builds the HTML for one bar:

```cpp
static String statRow(const char* name, int value, const char* color) {
    int clamped = value;
    if (clamped < 0)  { clamped = 0; }
    if (clamped > 100) { clamped = 100; }

    String row = "<div class=\"stat\">";
    row += "<div class=\"stat-label\">";
    row += "<span class=\"stat-name\">" + String(name) + "</span>";
    row += "<span class=\"stat-value\">" + String(value) + "</span>";
    row += "</div>";
    row += "<div class=\"bar-bg\">";
    row += "<div class=\"bar-fill\" style=\"width:" + String(clamped) + "%;background:" + String(color) + ";\"></div>";
    row += "</div></div>";
    return row;
}
```

Each bar has two parts: a label row (name on the left, number on the
right) and a background bar with a coloured fill that represents the
percentage. The `clamped` variable ensures the bar width never goes
below 0% or above 100%, even if a stat value is temporarily out of
range.

The eight stats are rendered individually so each can have its own
colour:

```cpp
page += statRow("Fullness",    petPtr->getFullness(),    "#e74c3c");  // red
page += statRow("Happy",       petPtr->getHappy(),       "#2ecc71");  // green
page += statRow("Energy",      petPtr->getEnergised(),   "#3498db");  // blue
page += statRow("Cleanliness", petPtr->getCleanliness(), "#1abc9c");  // teal
page += statRow("Sick",        petPtr->getSick(),        "#9b59b6");  // purple
page += statRow("Hydration",   petPtr->getHydration(),   "#e67e22");  // orange
page += statRow("Tired",       petPtr->getTired(),       "#95a5a6");  // grey
page += statRow("Sad",         petPtr->getSad(),         "#7f8c8d");  // darker grey
```

Picking a distinct colour per stat makes the dashboard scannable at a
glance. Red means danger (low fullness), green means good (high happy),
grey is informational (tiredness, sadness).

### 5.5 - The mood badge

The mood is shown as a coloured pill using a second helper:

```cpp
static String moodBadgeHtml(MoodSprite mood) {
    const char* label;
    const char* bgColor;

    switch (mood) {
        case MOOD_HAPPY:  label = "Happy";   bgColor = "#2ecc71"; break;
        case MOOD_UNWELL: label = "Unwell";  bgColor = "#9b59b6"; break;
        case MOOD_HUNGRY: label = "Hungry";  bgColor = "#e74c3c"; break;
        case MOOD_THIRSTY: label = "Thirsty"; bgColor = "#e67e22"; break;
        case MOOD_NEUTRAL:
        default:          label = "Neutral"; bgColor = "#95a5a6"; break;
    }

    String html = "<span class=\"mood-badge\" style=\"background:";
    html += bgColor;
    html += ";\">";
    html += label;
    html += "</span>";
    return html;
}
```

This uses the same `MoodSprite` enum and the same colour associations
as `showPetMoodText()` in `display_manager.cpp`. The mood badge on
the web page and the mood word on the device screen always agree,
because both read from the same `computeMood()` function.

### 5.6 - The dead-state notice

After the stat bars, the page checks whether the pet is dead. If so,
it appends a bold red notice:

```cpp
if (petPtr->isInDeadState()) {
    page += "<p style=\"color:#e74c3c;text-align:center;font-weight:bold;"
            "margin-top:16px;\">Your pet has died!</p>";
}
```

This is the web equivalent of the death screen on the device. The
condition is checked fresh every time the page is built, so the notice
disappears as soon as the player presses Button A to start a new life.

### 5.7 - The forward declarations

You may notice at the top of `wireless_manager.cpp` that the two helper
functions (`statRow` and `moodBadgeHtml`) are declared before they are
used:

```cpp
static String statRow(const char* name, int value, const char* color);
static String moodBadgeHtml(MoodSprite mood);
```

These are **forward declarations**. In C++, a function must be declared
before it can be called. Without these lines, `buildStatsPage()` would
try to call `statRow()` and `moodBadgeHtml()` before the compiler had
seen their definitions, and the build would fail.

The two functions are marked `static`, which means they are **file-
scope** - only visible inside `wireless_manager.cpp`, not callable
from any other file. They are internal helpers, not part of the class,
so they do not appear in the header. Declaring them at file level (not
inside a class) is the standard C++ way to write a small helper that
belongs to one implementation file and nothing else.

---

## Part 6 - Auto-refresh

The page includes a `<meta>` tag in the `<head>` that tells the browser
to reload the entire page every three seconds:

```cpp
page += "<meta http-equiv=\"refresh\" content=\"3\">";
```

This is the simplest possible way to make a page update itself. The
browser treats it as if the user had pressed F5 - it requests the same
URL again, gets the updated HTML, and replaces the current page.

Three seconds is a compromise. Too fast and the page flickers constantly
(the browser has to tear down the old page and render the new one, which
takes a moment). Too slow and the stats feel stale. At three seconds,
the bars tick down smoothly enough to feel alive without being
distracting.

This approach has a visible cost: the page flashes white briefly on
every refresh. That is fine for a dashboard you glance at occasionally.
If you wanted a truly smooth live view, the next step would be a JSON
endpoint and JavaScript that updates the numbers in place (see the
Extension section at the end of this guide).

---

## Part 7 - Wiring it into `main.cpp`

Three lines connect `WirelessManager` to the rest of the program:
create, initialise, and poll.

### 7.1 - Creating the instance

Near the top of `main.cpp`, alongside the other manager objects:

```cpp
#ifdef ENABLE_WIRELESS
WirelessManager wireless;
#endif
```

The `#ifdef` guard means this line only compiles when the feature
switch is on. With the switch off, `wireless` does not exist, and any
code that references it is also compiled out (by its own matching
`#ifdef`).

### 7.2 - Starting in `setup()`

Inside `setup()`, after the pet's stats have been loaded from storage
so the dashboard shows the real saved values:

```cpp
#ifdef ENABLE_WIRELESS
wireless.begin(myPet);
#endif
```

The order matters. `storage.load(myPet)` restores the pet's saved
fullness, happiness, and so on from NVS flash. If `wireless.begin()`
ran before the load, the first page request would show default stats
instead of the saved ones. By running `begin()` after `load()`, the
very first request always shows the correct state.

### 7.3 - Polling in `loop()`

Inside `loop()`, on every frame:

```cpp
#ifdef ENABLE_WIRELESS
wireless.handleClient();
#endif
```

`handleClient()` does almost nothing when no browser is connected.
It checks once whether a new client has arrived, and if not, returns
immediately. The cost is negligible - a single function call and a
quick check, not a blocking wait.

When a browser *is* connected, `handleClient()` reads the request,
calls the registered handler (`handleRoot()`), builds the HTML, and
sends the response. That whole process takes a few milliseconds, which
is fast enough that the pet's loop still runs smoothly.

---

## Part 8 - Testing it on real hardware

Once the firmware is on the device, testing the dashboard takes about
thirty seconds:

1. **Flash the device.** Run `pio run -e m5stick-c -t upload`. The
   device reboots automatically when the upload finishes.

2. **Open your phone's WiFi settings.** You should see a network called
   `VirtualPet` in the list. Tap it to connect. There is no password.

3. **Open a browser.** Safari, Chrome, Firefox - any of them work.
   Go to `http://192.168.4.1/`. The stats page appears.

4. **Watch the bars change.** Leave the page open and interact with the
   pet (feed it, play with it, or just wait). Every three seconds the
   page refreshes and the bars update.

5. **If the page does not load:**
   - Make sure you connected to the `VirtualPet` network, not your
     home WiFi.
   - Check that the address is `http://192.168.4.1/` - the `http://`
     part matters on some browsers (typing just `192.168.4.1` may
     perform a search instead).
   - On a classroom network with many devices, the SSID `VirtualPet`
     may be the same for every student's pet. Add your initials to
     distinguish them: change `AP_SSID` to `"VirtualPet-DM"` (or your
     own initials) and rebuild.

6. **Testing without a phone.** If you have a laptop on the same desk,
   you can connect that instead. The M5Stick's soft AP supports one or
   two connected devices at a time, so a single phone is the easiest
   test.

---

## Part 9 - How the pieces fit together

---

## Part 10 - What you can do next

This dashboard is a foundation. Three natural extensions build directly
on it, each adding one new concept:

### Live-refresh without the flash (JSON endpoint)

The current page uses `<meta http-equiv="refresh">`, which reloads the
entire page (including the CSS and layout) every three seconds. A
smoother approach:

1. Add a second route, `/stats.json`, that returns just the numbers
   as JSON instead of a full HTML page.
2. Add a small JavaScript block to the HTML page that calls
   `fetch('/stats.json')` every two seconds and updates the bar widths
   in place.

The bars would slide smoothly (the CSS `transition` property is already
there, waiting) and the page would never flicker. This is Bonus Feature
4 in `DEV_ROADMAP.md`.

### Phone-controlled actions (buttons on the page)

Add routes like `/feed`, `/play`, `/sleep`, `/bathe`, `/heal` that
call the corresponding `Pet` method and redirect back to the dashboard:

```cpp
void handleFeed() {
    pet->feed();
    server.sendHeader("Location", "/");
    server.send(303);
}
```

Add `<a>` links (styled as buttons) to the HTML for each action. Your
phone becomes a remote control for the pet. This is Bonus Feature 5.


---

## Part 11 - A note on the feature switch

The dashboard is the last feature(currently) in the cumulative session staircase.
Like every feature before it, the entire module is gated
behind a single `#define` in `lib/Config/scaffold_config.h`:

```cpp
#define ENABLE_WIRELESS
```

Comment that line out and the wireless module disappears from the
binary. The pet runs exactly as it did before the dashboard existed -
no WiFi, no server, no change in behaviour. The access point is not
broadcast, the server does not run, and the couple of kilobytes of RAM
the module uses are freed for other things.

This is the same pattern you have seen with every feature in this
project. `ENABLE_ACTION_MENU` removes the interact screen.
`ENABLE_SOUND` removes the buzzer melodies. `ENABLE_PERSISTENCE`
removes the save/load logic. Each feature is a self-contained slice
you can flip on and off independently, which is what makes the codebase
work as a teaching staircase - every session starts from the exact
subset of features that session covers, and nothing more.
