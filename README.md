# Virtual Pet Guide

This repo holds the reference material for building a Tamagotchi-style virtual pet on the **M5StickC Plus 2** (the small ESP32 device with a screen, buttons, buzzer, microphone, and accelerometer). The code itself lives in a separate project - this repo is just the documentation that explains how and why it all works.

---

## What's inside

| File | What it's for |
|---|---|
| `DEV_ROADMAP.md` | The full list of bonus / extension features you can add once the core pet is working. Web dashboard, pet-to-pet communication over ESP-NOW, voice memos, an RTC clock widget - pick whatever sounds fun. |
| `USEFUL_NOTES.md` | Explanations of the concepts you'll bump into while building the pet. Why the code is split into separate files, how musical notes map to frequencies, what `const` does at the end of a function, how non-blocking timers work, how the state machine works, why sprites look the way they do - all the questions you'd ask after finishing a feature. |
| `WIRELESS_GUIDE.md` | A "how it works" walkthrough of the web dashboard feature - how the ESP32 starts its own WiFi network, builds an HTML page with stat bars, and serves it to your phone. |

---

## Where to start

If you're working through the 10-session curriculum, you probably won't need this repo much - the lesson plans tell you everything you need session by session. Come here when:

- You finish a feature and want to understand *why* it works the way it does -> open `USEFUL_NOTES.md`
- You want to add something extra beyond the core pet and see what's possible -> open `DEV_ROADMAP.md`
- You're building the web dashboard and want the full walkthrough -> open `WIRELESS_GUIDE.md`

---

## License

Public domain - see `LICENSE` for the full text.
