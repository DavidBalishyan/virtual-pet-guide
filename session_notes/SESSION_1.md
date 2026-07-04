# Session 1 - Boot, Screen, First Stat and Sprite

We got the M5StickC Plus 2 to display something. The whole session was about getting from "plug it in" to seeing a little pixel pet on screen with a fullness bar that goes down over time.

## What we did

Setup is pretty standard for these boards - M5.begin() initialises everything, then we call display.init() to get our screen ready. The device shows a "Virtual Pet initialized!" message for 2 seconds when it boots, then the main loop kicks in.

The main loop is where the actual game runs. It does two things every frame: update the pet's stats (specifically fullness, which decays on a timer), then draw the screen. No delay() anywhere in the loop - everything is driven by millis() comparisons, which was a new concept for me. You store the last time something happened, then check if enough milliseconds have passed before doing it again. This way the loop can run as fast as it wants and nothing gets blocked.

The Pet class holds all the pet's data. Right now it only has one stat (fullness) but the header already lists all the others as comments - we'll enable them later. The constructor uses a member initialiser list (the colon syntax after the constructor name) to set the starting values. Fullness starts at 90. Every 6 seconds (or 0.6 seconds if FAST_TEST is defined in time_manager.cpp) it drops by 1.

The display shows a centred sprite (80x80 pixels) and a fullness bar at the bottom. The sprite is drawn from an array stored in PROGMEM - it's raw pixel data in RGB565 format, not an image file. The transparent colour is 0xF81F (magenta), which gets skipped when drawing so the background shows through.

## Things that tripped me up

- The member initialiser list syntax. I kept putting the initialisation in the constructor body instead of before it. The colon after the constructor name and the comma-separated list of values is something I'll need practice with.
- millis() returns unsigned long. If you try to store it in an int it wraps around after 32 seconds and your timers break. I did this and wondered why fullness stopped decaying after a minute.
- The SPRITE_TEST define. If it's uncommented, setup() draws one sprite and returns - the whole game loop is bypassed. I spent a few minutes wondering why nothing was moving before I noticed it was defined at the top of main.cpp.
- const methods. Putting const after a method declaration means it promises not to modify the object. The getters are all const. If you forget the const, the compiler complains when you try to call a getter on a const reference.

## Files we touched

- src/main.cpp
- lib/Pet/pet.h and pet.cpp
- lib/Display/display_manager.h
- lib/Timer/time_manager.h and time_manager.cpp
- lib/Config/scaffold_config.h

The display manager is already pretty big but it's all screen drawing code, so it makes sense to keep it in one place. The TimerManager handles all time-based stat changes. Keeping the timer logic separate from the Pet class was intentional - Pet just stores values and provides methods to change them, it doesn't decide when changes happen.
