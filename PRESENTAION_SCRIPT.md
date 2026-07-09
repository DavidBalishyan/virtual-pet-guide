# Virtual Pet: Project Description

A Tamagotchi-style virtual pet that runs on an M5StickC Plus 2 (ESP32). You feed
it, play with it, put it to sleep, and keep its stats up or it dies. It has a
screen with animated mood faces, a buzzer that plays melodies, saves itself to
flash so it survives a reboot, and hosts its own WiFi network with a web
dashboard so I can check on it and control it from my phone.

> Note: fill in the **Media** section at the bottom with your own photos/clips,
> and adjust anything in the reflection sections to match your own experience.

---

## What did I build?

**The pet firmware, end to end.** It's a PlatformIO / Arduino C++ project split
into single-purpose modules under `lib/`:

- **Pet**: the game model. Eight stats (0 to 100), a state machine (idle,
  sleeping, dead), the care actions (feed, play, sleep, bathe, heal, drink),
  mood, and death. No hardware code lives here, which is why it's the easiest
  part to test.
- **TimerManager**: the rules for how stats decay over time.
- **DisplayManager**: all the drawing. It renders to an off-screen canvas and
  pushes the whole frame at once so the screen never flickers.
- **NavigationManager**, **ActionMenu**, **ButtonHandler**: the menu and button
  input.
- **ImuManager** and **TiltMotion**: shake-to-play, plus tilting the pet with the
  accelerometer.
- **SpeakerManager**: buzzer melodies for every event, plus the song player
  described below.
- **StorageManager**: saves and loads the pet to NVS flash.
- **ClockManager**: the real-time clock.
- **WirelessManager**: a WiFi access point ("VirtualPet"), a web server, and a
  WebSocket that streams the pet's stats to the browser and takes commands back.

The whole thing is a teaching project built up over sessions, so every big
feature sits behind an `ENABLE_*` switch in `scaffold_config.h`. Turning a switch
off compiles that feature out entirely, which is how each session recreates an
earlier stage of the project.

**The feature I added last: a phone-controlled song player.** The device can play
full melodies (Twinkle Twinkle, Ode to Joy, the Mario theme, and Beethoven's Für
Elise), and I can start or stop any of them from a "Music" panel on the phone
dashboard. It plays without freezing the pet: the animation keeps running and
the buttons stay responsive while a song plays.

To make longer songs practical, a song is stored as data (a list of
`{note, length}` pairs plus a tempo) instead of hand-written `tone()` calls:

```cpp
const SongNote SONG_ODE[] = {
    {NOTE_E5, QUARTER}, {NOTE_E5, QUARTER}, {NOTE_F5, QUARTER}, {NOTE_G5, QUARTER},
    // ...
};
```

Adding a new song is three edits: a note table, one row in the song registry, and
one button in the dashboard HTML.

**Tests.** The project has a host test suite that compiles the real modules on my
PC (not the device) using a stub `Arduino.h`. It's at 70 passing tests,
including 9 I wrote for the song player.

---

## What did I learn?

- **A microcontroller has one thread and one main loop.** Everything shares it:
  reading buttons, updating stats, drawing, networking, sound. If any one step
  blocks, the whole pet freezes. That single fact shaped most of my design
  decisions.
- **Non-blocking / cooperative programming.** Instead of "play this song" being
  one long function that hogs the loop, I learned to keep a little bit of state
  (which note, when it started) and do one small step per loop tick. That's a
  state machine, and it's how you fake concurrency on a device with no threads.
- **Separating logic from hardware pays off in testing.** Because `Pet` and the
  song player don't talk to hardware directly, I could test them on my laptop with
  a fake clock. Deterministic `millis()` meant I could jump time forward and check
  that a song advances and eventually ends, without waiting through it or touching
  a real speaker.
- **Feature flags and conditional compilation** (`#ifdef ENABLE_SOUND`). One set
  of source files can produce many different builds. It also taught me discipline:
  a flag has to be respected everywhere (the declaration, every call site, the
  includes) or it won't compile.
- **Keeping modules decoupled by passing plain values** (ints, strings, enums)
  across boundaries instead of handing one manager a pointer to another. It keeps
  each piece understandable on its own.
- **How a device talks to a phone with no app**: a WiFi access point, a tiny web
  server serving an embedded HTML/JS page, and a WebSocket for live two-way
  messages. I learned the protocol is just small JSON strings in both directions.
- **How music is actually represented**: notes are frequencies in Hz, rhythm is
  fractions of a beat, and tempo in BPM ties beats to real milliseconds.
- **Resource limits are real.** The firmware sits around 83% of flash. Every song
  costs bytes, so "add more content" is a budget decision, not a free one.

---

## What was hard, and how I overcame it

**1. Sound blocked the whole device.** Every existing jingle played with
`tone()` then `delay()`. That's fine for a half-second beep, but a full song is
several seconds, and `delay()` would freeze the screen and buttons the entire
time.
*Fix:* I rewrote song playback as a non-blocking state machine. The player
remembers which note it's on and when that note started; an `updateSong()` call at
the top of the main loop plays the next note only once the current one's time is
up. The song plays in the background while the pet carries on.

**2. Testing time-based code without waiting.** How do you test "this song lasts
the right length and then stops" without playing it for real?
*Fix:* the test stub lets me set the clock directly with `setMillis()`. I start a
song, jump the clock forward in steps, and assert the player advances one note at
a time and eventually reports it has finished. Fast and repeatable.

**3. Writing real songs in milliseconds was painful and error-prone.** My first
version put raw durations (`400`, `800`) on every note. Transcribing an actual
piece that way is miserable and easy to get wrong.
*Fix:* I switched to a musical format (note values like `QUARTER` and `EIGHTH`
plus a per-song tempo) and let the player convert beats to milliseconds. Now a
song reads like sheet music, and changing the tempo is one number.

**4. The buzzer can only play one note at a time (it's monophonic).** I wanted a
"real song," but real songs have chords and harmony.
*Fix:* I accepted the limit and treated each song as just its melody line, which
is what you'd hum anyway. Für Elise works well because it's melody-driven.

**5. Keeping the phone and the device in sync.** The dashboard button sends a song
number, and the device looks that number up in its song list, so the button
order and the list order have to match or you play the wrong song.
*Fix:* I kept the id ordering in one place, documented the coupling with a comment
at both ends, and always add new songs at the end so existing ids don't shift.

**6. Parsing commands with no JSON library.** The device has no room for a heavy
JSON parser, so incoming commands are parsed by hand from the string.
*Fix:* I followed the pattern already in the code (find the key, read the value
after it) and added `playSong`/`stopSong` the same way, guarded so it compiles
even when sound is switched off.

---

## Pictures / recording of the device in action

<!--
Add your own media here. Suggested shots:
  1. The pet on the device screen (a mood face + stat bars).
  2. The phone dashboard, especially the Music panel with the song buttons.
  3. A short clip (10 to 20s) of a song playing while the pet keeps animating,
     ideally with sound so the melody is audible.

Put files in a `media/` folder next to this file and reference them like:

    ![Pet on the M5StickC screen](media/pet-screen.jpg)
    ![Phone dashboard, Music panel](media/dashboard-music.png)

    Video (GitHub renders an uploaded .mp4/.mov inline in a PR/issue, or link it):
    [Demo clip: Für Elise playing](media/demo-fur-elise.mp4)

How to capture:
  - Screen/pet: a phone photo of the device is easiest.
  - Dashboard: connect your phone to the "VirtualPet" WiFi network, open
    http://192.168.4.1 in the browser, and screenshot it.
  - Recording: film the device with your phone while tapping a song on the
    dashboard so both the screen and the sound are captured.
-->

_Media to be added. See the notes above for what to capture and how._

---

## Try it yourself

```bash
make run        # build, flash to the device, and open the serial monitor
make test       # run the host unit tests
```

Then connect a phone to the **VirtualPet** WiFi network and open
**http://192.168.4.1** to reach the dashboard.
