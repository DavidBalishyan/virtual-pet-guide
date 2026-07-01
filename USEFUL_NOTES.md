# Virtual Pet - Teaching Notes

This file explains the "why" behind concepts you will run into while building the pet.
These are not step-by-step build instructions - those live in `DEV_ROADMAP.md`. This is
for when you finish a feature and catch yourself wondering *"ok, but how does that actually
work?"*

---

## Why the Code is Split Into Separate Modules

### The problem with putting everything in one file

Imagine the whole virtual pet - display, buttons, sounds, timers, pet logic, storage - crammed into one giant `main.cpp`. It would be thousands of lines long. Want to fix a bug in the buzzer? You would have to scroll past display code and timer code just to find it. Want to add a new action? You would have to understand the entire file before touching a single line. Every change would risk breaking something unrelated.

This is a real problem that professional engineers deal with every day. The fix is **modularisation**: split the code into separate files, where each file has exactly one job and stays out of everyone else's business.

### The Single Responsibility Principle

Each `lib/` module in this project follows a ridiculously simple rule:

> **One module. One job. One reason to change.**

This idea has a formal name - the **Single Responsibility Principle** - and it is one of the most important ideas in software design. It is worth remembering because you will see it everywhere once you know to look.

| Module | Its one job |
|---|---|---|
| `Pet` | Own the pet's stats, its state machine, and its mood (`computeMood()`); decide when its alert, death, and reset sounds should fire |
| `DisplayManager` | Draw things to the screen - nothing else |
| `AnimationManager` | Decide which sprite frame to show right now (timing only) - nothing else |
| `TiltMotion` | Turn live accelerometer tilt into a smoothed pet-sprite offset - nothing else |
| `ButtonHandler` | Detect when buttons are pressed - nothing else |
| `ActionMenu` | Track which action is selected and confirm it - nothing else |
| `TimerManager` | Apply automatic stat changes over time - nothing else |
| `ImuManager` | Detect shake gestures from the accelerometer - nothing else |
| `SpeakerManager` | Play sounds and melodies - nothing else |
| `StorageManager` | Save and load pet stats to flash memory - nothing else |
| `NavigationManager` | Track which screen is active and handle top-level navigation - nothing else |

`AnimationManager` and `TiltMotion` are small **display helpers**: instead of being created in `main.cpp` like the others, `DisplayManager` owns the `AnimationManager`, and `main.cpp` owns the `TiltMotion` and feeds its offset into `DisplayManager`. They are still single-responsibility modules - they just happen to be owned by the part of the program that uses them.

Having one job does **not** mean a module never calls another. `Pet` decides *when* its death, reset, and hunger/sickness sounds should fire - but it does not know *how* to make a tone, so it borrows a `SpeakerManager` (passed into `updateState()` and `reset()`) and asks it to play. In the same way, `ActionMenu`'s `confirmAction()` carries out the chosen action by calling `Pet`, `SpeakerManager`, and `StorageManager`. Each module still owns exactly one responsibility; it simply *delegates* the parts that belong to other modules, borrowing them for the duration of a single call (see "How the modules link together" below).

When a module has one job, you know exactly where to look when something goes wrong. Buzzer broken? Open `speaker_manager.cpp`. Pet not saving? Open `storage_manager.cpp`. You never have to read unrelated code to fix a focused problem.

### How the modules link together

None of the modules talk to each other directly. They each do their own job, and `src/main.cpp` acts as the **orchestrator** - it owns the modules and passes them to each other when needed.

```
                        src/main.cpp
                       (orchestrator)
                            |
         +------------------+------------------+
         |                  |                  |
         v                  v                  v
   ButtonHandler      NavigationManager    TimerManager
   (reads buttons)    (tracks screen)     (decay timers)
         |                  |                  |
         |            +-----+                  |
         |            v                        |
         |       ActionMenu              +-----+
         |       (cycles actions)        |
         |            |                  v
          |            +---------->      Pet
         |                        (stats + state)
         |                              |
         v                              v
   ImuManager                    DisplayManager
   (shake gesture)                (draws screen)
         |                              |
         +------------------------------+
                       |
                       v
                 SpeakerManager
                 (plays sounds)
                       |
                       v
                 StorageManager
                 (saves to flash)
```

*(The two display helpers are left out of this diagram to keep the core flow readable: `DisplayManager` owns the `AnimationManager`, and `main.cpp` owns the `TiltMotion` and passes its offset to `DisplayManager`.)*

When `main.cpp` needs `ButtonHandler` to tell `ActionMenu` about a button press, it passes the `ButtonHandler` object into `ActionMenu::update()` as a parameter. The top-level managers never hold permanent references to each *other* - they only borrow what they need for the duration of one function call. (The one exception is a module owning a small private helper of its own, like `DisplayManager`'s `AnimationManager` - that is composition, not two managers reaching into each other.) This keeps the dependencies loose and easy to follow.

### What is Object-Oriented Programming?

Each module is a **class** - think of it as a blueprint that describes:
- What data it holds (its **member variables**)
- What it can do (its **member functions**, also called methods)

When `main.cpp` creates a `Pet` object, it is using that blueprint to build a real pet. And you can build as many as you want from the same blueprint - if you wanted two pets, you would just create two `Pet` objects from the same `Pet` class.

```cpp
// The class is the blueprint
class Pet {
    int fullness;     // data it holds
    void feed();      // things it can do
};

// The object is the real thing built from the blueprint
Pet myPet;           // one pet
Pet rivalPet;        // another pet - same blueprint, separate data
```

The key idea is **encapsulation**: the data inside a class is hidden from the outside world. Other code cannot reach in and change `Pet::fullness` directly - it has to use the methods the `Pet` class provides (`getFullness()`, `setFullness()`). That way the `Pet` class stays in control of its own data and can enforce rules like "fullness can never go below 0 or above 100."

### What is Inheritance - and why this project does not use it

Inheritance is a way to build a new class by starting from an existing one. The new class **inherits** all the data and behaviour of the original, then adds or changes things on top.

```cpp
// A general Animal class
class Animal {
public:
    int fullness;
    void eat() { fullness = fullness + 10; }
};

// Dog inherits everything from Animal, and adds its own behaviour
class Dog : public Animal {
public:
    void bark() { /* play bark sound */ }
};

// Dog objects can eat() (inherited) AND bark() (their own)
Dog myDog;
myDog.eat();   // inherited from Animal
myDog.bark();  // Dog's own method
```

Inheritance is useful when you have several things that are mostly the same but differ in specific ways - for example, `Cat`, `Dog`, and `Bird` all sharing an `Animal` base class.

**This project intentionally avoids inheritance.** All of these modules are completely independent - a `DisplayManager` is not a kind of `Pet`, and a `SpeakerManager` is not a kind of `TimerManager`. Forcing inheritance where there is no real "is-a" relationship just makes code harder to read and harder to change. Using separate, unrelated classes connected through `main.cpp` (called **composition**) is the right approach here, and most professional embedded C++ code does the same thing for exactly this reason.

A useful rule of thumb:
- Use **inheritance** when one thing genuinely *is a kind of* another thing (a `Dog` is an `Animal`).
- Use **composition** (separate classes connected through `main.cpp`) when things *work together* but are not the same kind of thing.

In this project, everything is composition. You will run into inheritance in future projects - now you know what it looks like and why it exists.

---

## How Musical Notes Work (`M5.Speaker.tone`)

### Sound is vibration

When the buzzer makes a sound, it is literally vibrating - a tiny component is wiggling back and forth really fast. The speed of that wiggling is called the **frequency**, and it is measured in **Hertz (Hz)**, which just means "times per second".

Your ear detects that vibration speed and your brain interprets it as a pitch:

- A slow vibration (low Hz) sounds like a low, deep note.
- A fast vibration (high Hz) sounds like a high, sharp note.

When you write `M5.Speaker.tone(440, 200)`, you are saying:
> "Vibrate at 440 times per second, for 200 milliseconds."

Your ear hears that as the musical note **A4** - the A above middle C, the note that orchestras tune to before a concert.

### The notes used in this project

| Frequency | Note name | Where it appears |
|---|---|---|
| 220 Hz | A3 | End of death melody (very low, very final) |
| 294 Hz | D4 | Sickness alert beeps |
| 349 Hz | F4 | Middle of sleep melody |
| 392 Hz | G4 | Start of heal melody |
| 440 Hz | A4 | Start of death melody, part of heal melody |
| 523 Hz | C5 | Feed melody, bathe melody, heal melody, play melody |
| 659 Hz | E5 | Feed melody, heal melody, reset fanfare |
| 784 Hz | G5 | Play melody, bathe melody, reset fanfare |
| 880 Hz | A5 | Hunger alert beeps (high, urgent) |
| 1047 Hz | C6 | End of play melody and reset fanfare (bright, triumphant) |

### The doubling rule - what an octave is

Here is the most important pattern in music: **doubling the frequency raises the pitch by exactly one octave**.

- A4 = 440 Hz
- A5 = 880 Hz (double 440 - one octave higher, same note but brighter)
- A6 = 1760 Hz (double again - two octaves above A4)

Your ear recognises the ratio between two frequencies, not the raw numbers. That is why 440 Hz and 880 Hz feel like "the same note" even though the numbers are different. This is also why the hunger alert (880 Hz) and the sickness alert (294 Hz) sound so different - they are nearly three octaves apart.

### How duration and gaps shape a melody

`M5.Speaker.tone(frequency, duration)` plays one note for `duration` milliseconds. The `delay()` call after it creates the gap before the next note starts.

```cpp
M5.Speaker.tone(523, 100);  // C5, very short - snappy
delay(120);
M5.Speaker.tone(659, 180);  // E5, slightly longer - feels like it "arrives"
delay(200);
```

Changing just the durations changes the whole character of a melody:
- Short notes (80-120 ms) feel quick and energetic.
- Longer notes (200-400 ms) feel calm, dramatic, or final.
- A longer final note gives a sense of resolution - the melody has "landed".

Try changing the numbers in `lib/Speaker/speaker_manager.cpp` and flashing the device to hear the difference. Experimenting is the fastest way to develop an intuition for this.

### Why the sound code uses `delay()` - and why that is acceptable here

You will notice that the functions in `speaker_manager.cpp` use `delay()` between each note. The next section explains why `delay()` is normally forbidden inside `loop()`, so this might look like a contradiction. It is not - and understanding the difference is important.

The sound functions only ever run inside `confirmAction()`, which is called when the user presses Button A to confirm a menu choice. That is a deliberate, **one-shot response** to something the user just did - not a background event - so a short melody of a few hundred milliseconds is exactly the kind of brief, intentional pause the user expects as acknowledgement of their button press. The melody's own `delay()` calls between notes are the only blocking here, and they last only as long as the notes play.

The rule is more precisely stated as: **do not use `delay()` for background, automatic events that should happen silently without interrupting the user**. The stat decay timers are a perfect example - if fullness decreased every 5 seconds and the screen froze for a moment each time, the device would feel broken. The melodies are different: they are a direct, intentional response to something the user just did.

The hunger and sickness alert beeps are a minor exception - they do call `delay()` from the main loop for about 300 ms each time they fire. That brief freeze is a deliberate trade-off: the beep needs two separate tones to be recognisable, and the simplicity of the code is worth the very short interruption that happens at most once every 15 seconds.

---

## How Non-Blocking Timers Work (`millis()`)

### The problem with `delay()`

`delay(1000)` pauses the **entire program** for one second. While it is paused, nothing runs. Buttons are not being checked. The display is not updating. No stat changes are happening. To a user, the device just feels broken.

For a one-off startup message this is fine. But inside `loop()` - which runs hundreds of times per second - using `delay()` means the pet's face freezes every time a timer fires. That is where the rule comes from: **never use `delay()` inside `loop()`**.

### The solution: ask the clock instead of stopping the clock

`millis()` returns the number of milliseconds that have passed since the device was switched on. It never pauses anything - it just reads a counter that the hardware is always updating in the background.

The pattern for a non-blocking timer is always these same three ideas:

```cpp
// 1. Remember the last time this event fired.
//    'static' means this variable keeps its value between loop() calls.
static unsigned long lastHappinessDecayTime = 0;

// 2. Decide how long to wait between firings.
unsigned long happinessDecayInterval = 5000;  // 5000 ms = 5 seconds

// 3. Check if enough time has passed. If yes, act and reset the clock.
if (millis() - lastHappinessDecayTime >= happinessDecayInterval) {
    myPet.setHappy(myPet.getHappy() - 1);  // do the timed action
    lastHappinessDecayTime = millis();      // reset the timer
}
```

Every time `loop()` runs, this check takes almost zero time - it is just subtraction and comparison. If not enough time has passed, the `if` is false and the code skips straight past it. The program never stops.

### Where to find this pattern in the project

Every decay timer in `lib/Timer/time_manager.cpp` uses exactly this pattern - `applyFullnessDecay()`, `applyHappinessDecay()`, `applyEnergyDrain()`, and so on. The hunger alert and sickness alert in `src/main.cpp` use it too. Once you recognise the three-part structure, you will see it everywhere in embedded code.

---

## How to Completely Reset an ESP32 Device

Reflashing the firmware does **not** wipe NVS or any other data partition - only the firmware image is overwritten. If you need a guaranteed clean slate, you have two options.

### Option 1 - Erase via PlatformIO (inside a project)

Run this command from inside any PlatformIO project directory:

```bash
pio run --target erase
```

This wipes the entire flash chip - firmware, NVS, filesystem, everything. The next upload will start completely fresh. This works in any PlatformIO project without any extra setup.

### Option 2 - Erase via esptool.py (outside a project)

`esptool.py` is the underlying tool PlatformIO uses, and it is installed automatically as part of the ESP32 platform. You can call it directly without a PlatformIO project:

```bash
esptool.py --port /dev/tty.usbserial-XXXX erase_flash
```

Replace `/dev/tty.usbserial-XXXX` with the port your device is connected to. On macOS you can find it by running:

```bash
ls /dev/tty.usb*
```

This is useful when sharing a reset procedure with someone who has the ESP32 toolchain installed but is not using PlatformIO, or when working on a device outside of any specific project.

### What each option erases

| What | `pio run --target erase` | `esptool.py erase_flash` |
|---|---|---|
| Firmware (`app0` / `app1`) | [x] | [x] |
| NVS (Preferences data) | [x] | [x] |
| Filesystem (SPIFFS / LittleFS) | [x] | [x] |
| Partition table | [x] | [x] |

Both commands erase everything. After either one, you must re-upload your firmware before the device will run again.

---

## What `const` Means at the End of a Function Declaration

You will notice that some functions in the class headers end with the word `const`:

```cpp
PetState getState()    const;
int      getFullness() const;
bool     isDead()      const;
```

This `const` is a **promise to the compiler** that the function will not modify any member variables of the object. Functions marked this way are called **const member functions**.

### Why it matters - intent and enforcement

`getState()` only reads `currentState` and returns it. It changes nothing. Marking it `const` communicates that clearly to anyone reading the code, and the compiler enforces the promise. If you accidentally wrote `currentState = STATE_IDLE` inside a `const` function, the compiler would refuse to build and tell you exactly why.

Scanning a class header becomes much easier once you know this rule:
- Functions with `const` at the end are **read-only** - safe to call with no side effects.
- Functions without it **may change the object's state** - setters and action methods like `feed()` and `play()`.

### Why it matters - const references

When a function receives an object as `const Pet&`, it is promising not to modify that pet. The compiler then only allows `const` member functions to be called on it. `StorageManager::save()` is a real example of this in the project:

```cpp
// The const Pet& parameter means: "I promise not to modify this pet."
void StorageManager::save(const Pet& pet) {
    prefs.putInt("fullness", pet.getFullness());  // allowed - getFullness() is const
    pet.setFullness(50);                          // compiler error - setFullness() is not const
}
```

The `const` on the parameter and the `const` on each getter work together. The compiler can verify that `save()` truly does not modify the pet - every function it calls on `pet` has made the same promise.

### The general rule

Mark a member function `const` whenever it only **reads** from the object and never changes it. In practice this means all getters should be `const`, and action methods that change stats should not be.

---

## How the Scope Resolution Operator Works (`::`)

`::` is called the **scope resolution operator**. It means "look inside this class (or namespace) for this name."

### Static members vs instance members

When a member is declared `static` in a class, it belongs to the **class itself** - not to any individual object created from that class. There is only one copy of it, shared everywhere.

```cpp
class Pet {
public:
    static const int DEFAULT_FULLNESS = 80;  // belongs to Pet the class
    int getFullness() const;                  // belongs to each Pet object
};
```

The way you access each type of member reflects this difference:

```cpp
// Instance member - accessed via a dot on a specific object
pet.getFullness()        // pet is an object; getFullness() belongs to that object

// Static member - accessed via :: on the class name, no object needed
Pet::DEFAULT_FULLNESS    // no object required; it belongs to the class itself
```

### Why :: is needed outside the class

Inside `pet.cpp`, the compiler already knows it is working inside the `Pet` class context, so `DEFAULT_FULLNESS` on its own is enough. But in a different file like `storage_manager.cpp`, the compiler has no idea where `DEFAULT_FULLNESS` comes from unless you tell it:

```cpp
// Compiler does not know which class to look in - fails
prefs.getInt("fullness", DEFAULT_FULLNESS);

// Compiler knows to look inside Pet - works
prefs.getInt("fullness", Pet::DEFAULT_FULLNESS);
```

### Plain enums vs `enum class` - when you need ::

This project uses **plain enums** for its option lists, with a prefix on every value:

```cpp
ACTION_FEED    // a value of the plain enum ActionType - written directly, no ::
SCREEN_STATS   // a value of the plain enum ScreenState - also written directly
```

A plain enum's values live in the surrounding scope, so you write them by name with no `::`. The prefix (`ACTION_`, `SCREEN_`) is what keeps them from clashing with other names. If these had instead been declared `enum class ActionType { FEED, ... }`, the values would live *inside* the type and you would have to write `ActionType::FEED` - the same `::` you already use for `Pet::DEFAULT_FULLNESS`. This project chooses plain enums precisely so beginners can write the value directly.

Think of `::` as saying *"open this container and find the name inside it."* The container can be a class, an `enum class`, or a namespace - `::` works the same way in all three cases.

### Defining class functions outside the class body

In this project, every class is split across two files: a header (`.h`) that declares what the class has, and a source file (`.cpp`) that defines what each function actually does. When you write a function body in the `.cpp` file, you use `::` to tell the compiler which class that function belongs to.

```cpp
// In pet.h - the declaration (what exists)
class Pet {
public:
    void feed();
};

// In pet.cpp - the definition (what it does)
void Pet::feed() {
    fullness = fullness + 20;
    happy    = happy    + 15;
    constrainValues();
}
```

`Pet::feed()` means "this is the implementation of the `feed` function that belongs to the `Pet` class." Without the `Pet::` prefix, the compiler would treat it as a standalone free function with no connection to the class at all.

Once you are inside the function body - between the `{` and `}` - the compiler knows you are in `Pet` context. That is why you can write `fullness` and `constrainValues()` directly without any prefix. The `Pet::` on the function name is the door; everything inside is already in the room.

You can see this pattern used consistently throughout the project:

```cpp
void StorageManager::save(const Pet& pet) { ... }
void TimerManager::update(Pet& pet)       { ... }
void ActionMenu::confirmAction(...)       { ... }
```

Every one of those `ClassName::functionName` lines is the same thing: "here is the body for a function I declared inside this class."

---

## How the State Machine Works

### What is a "state"?

At any moment, the pet is doing exactly one thing - eating, sleeping, playing, being sick, or just waiting around. That "one thing it is currently doing" is called its **state**.

A state machine is a system that:
1. Keeps track of the current state.
2. Defines rules for when to switch from one state to another.
3. Can run different behaviour depending on which state is active.

### The three functions that split the responsibility

In `lib/Pet/pet.h` and `pet.cpp`, three functions handle all the state logic:

| Function | Job |
|---|---|
| `setState(newState)` | Changes the current state. Called by action methods like `feed()` and `play()`. |
| `getState()` | Returns the current state. Used by the display and any code that needs to know what the pet is doing. |
| `updateState()` | Called once per `loop()`. Checks the current state and decides if it is time to switch to a new one. |

### How a state change flows through the code

Here is what happens when the user presses Button A to feed the pet:

```
Button A pressed
  -> menu.confirmAction() is called in main.cpp
    -> executePetAction() calls pet.feed()
      -> feed() calls setState(STATE_EATING)   <- state changes here
      -> feed() adjusts fullness, happy, etc.
  -> next loop() iteration
    -> pet.updateState() runs
      -> sees STATE_EATING
      -> immediately calls setState(STATE_IDLE) <- returns to idle
```

Eating happens instantly in this project - `feed()` adjusts stats right away - so `updateState()` returns to `STATE_IDLE` on the very next frame. In a more advanced version, you could hold the pet in `STATE_EATING` for several seconds and show an eating animation before returning to idle.

### Why not just use a variable and an `if` statement?

You could write `if (isFeedingNow) { ... }`. For one action this works. But as soon as you have six states, you have six separate boolean variables that can accidentally all be `true` at once. The state machine using an `enum` guarantees exactly one state is active at a time - the compiler enforces it. It also makes the code much easier to read: `getState() == STATE_SLEEPING` is clearer than `isSleeping && !isEating && !isPlaying`.

The state list lives in `lib/Pet/pet.h` as the `PetState` enum. Reading through it gives you a complete picture of everything the pet can do.

---

## Why `static const` Structs Need a Definition in the `.cpp` File

### The short version

`static const int` members are special - the compiler treats them as compile-time numbers and substitutes them directly into the machine code. `static const` members of any other type (like a `struct`) have to exist in memory as real objects, so they need a definition in a `.cpp` file or the linker will complain.

### What "declaration" vs "definition" means

Every piece of code in C++ exists in two parts:

- A **declaration** (in the `.h` file) is a *promise* - it tells the compiler "this thing exists and here is what it looks like."
- A **definition** (in the `.cpp` file) is where the thing is *actually created* - memory is allocated, values are assigned.

For most class members, the constructor handles this automatically. But `static` members belong to the class itself, not to any object, so the constructor never runs for them. You have to provide the definition manually.

### Why `static const int` gets away without a `.cpp` definition

When you write this in a header:

```cpp
static const int SCREEN_WIDTH = 135;
```

The compiler treats `135` as a compile-time constant - like a smarter `#define`. Wherever `SCREEN_WIDTH` appears in your code, the compiler substitutes `135` directly into the machine code at compile time. It never needs to find a memory address for it, because it never puts it in memory at all.

This is called **constant folding**, and it is why `static const int` is a special case that does not need a `.cpp` definition.

### Why `static const ScreenZone` is different

Suppose one of the screen zones were declared the plain `static const` way:

```cpp
static const ScreenZone TITLE_ZONE = { 0, 5, 135, 19 };
```

`ScreenZone` is a struct - it has four separate `int` fields. The compiler cannot substitute it as a single number. It has to exist somewhere in memory as a real object with an address, so that code can read `.x`, `.y`, `.width`, and `.height` from it.

If you only declare it in the header and never define it in a `.cpp` file, the compiler will not warn you - it trusts your promise. But when the linker tries to build the final program, it searches all the compiled files looking for "where is `DisplayManager::TITLE_ZONE` actually stored?" and finds nothing. That produces a **linker error** - a subtly different kind of failure from a normal compile error.

### The fix - one line in the `.cpp` file

```cpp
// In display_manager.cpp
const ScreenZone DisplayManager::TITLE_ZONE = { 0, 5, 135, 19 };
```

This is the definition. It tells the linker: "here is where `TITLE_ZONE` actually lives in memory, and here are its values." The `DisplayManager::` prefix (the scope resolution operator, explained in an earlier section) tells the compiler this belongs to the `DisplayManager` class - the same pattern used for every function body in the project.

(In this project the zones do not actually need this `.cpp` line - they are declared `constexpr`, which sidesteps the problem entirely. That shortcut is the next section. The `static const` long form shown here is what you would need for any `static const` object that is not `constexpr`-friendly.)

### The analogy

Think of the header file as a restaurant menu and the `.cpp` file as the kitchen.

A `static const int` is like a price printed directly on the menu - it is just a number, right there on the page, no kitchen required.

A `static const ScreenZone` is like a dish listed on the menu - the menu describes it, but someone in the kitchen has to actually prepare it before a customer can have it. The `.cpp` definition is the kitchen preparing the dish.

### When you will see this pattern

Any time you add a `static const` class member that is not a plain integer or boolean, you will need a matching definition in the `.cpp` file. This applies to structs, strings, arrays, and any other non-trivial type. The pattern is always the same:

```cpp
// Header - the promise
static const YourType MEMBER_NAME;

// .cpp - the definition
const YourType ClassName::MEMBER_NAME = { ... };
```

### The shortcut - `constexpr`

There is a smarter way to write struct constants that skips the extra `.cpp` line entirely. Just swap `static const` for `static constexpr`:

```cpp
// This works on its own in the header - no .cpp line needed
static constexpr ScreenZone TITLE_ZONE = { 0, 5, 135, 19 };
```

`constexpr` is short for "constant expression". It is your way of telling the compiler: **"I promise this value will never change - ever."**

When the compiler sees that promise, it works out the value while it is building your program, before the program ever runs on the device. It then bakes the answer directly into the code.

After the compiler finishes, a second tool called the **linker** runs. The linker's job is to take all the separate compiled pieces and join them into one final program. Without `constexpr`, the linker would go searching for your struct in memory and fail - because you never put it in a `.cpp` file. With `constexpr`, the value was already baked in during the build step, so the linker never needs to search for it.

This is what the zone constants in `display_manager.h` use. Whenever you add a struct constant to a class, use `constexpr` and you will not need the extra `.cpp` line at all.

---

## Why Header Files Start With `#ifndef`

Open any `.h` file in this project and you will see the same three lines at the very top and bottom:

```cpp
#ifndef DISPLAY_MANAGER_H
#define DISPLAY_MANAGER_H

// ... all the actual code ...

#endif
```

These three lines are called an **include guard**. Here is why they exist.

### The problem - including a file twice

When your program is built, the compiler reads each `.cpp` file one at a time. In `main.cpp` you write `#include "pet.h"`. That tells the compiler: "copy everything in `pet.h` into this file right now."

But what if two different files both include `pet.h`? What if `main.cpp` includes `pet.h` and also includes `action_menu.h`, and `action_menu.h` also includes `pet.h`? The compiler would end up seeing the `Pet` class definition twice in the same file. In C++, defining the same thing twice is an error - the compiler will refuse to build and print a confusing message.

### How include guards fix it

`#ifndef` means **"if not defined"**. The three lines together say:

1. `#ifndef DISPLAY_MANAGER_H` - "have I seen the name `DISPLAY_MANAGER_H` before?"
2. `#define DISPLAY_MANAGER_H` - "if not, remember this name now"
3. `#endif` - "skip everything between these lines if the name was already known"

The first time the compiler reads `display_manager.h`, it has never seen `DISPLAY_MANAGER_H`, so it reads all the code inside and remembers the name. If anything tries to include the same file a second time, the compiler sees the name is already defined and skips the whole file. The class only gets defined once, no matter how many times the file is included.

### The name in the guard

The name used in the guard - `DISPLAY_MANAGER_H` - is just a convention. It matches the filename in ALL CAPS with the `.` replaced by `_`. As long as the name is unique across your whole project, it works. That is why every header uses a different name:

```cpp
#ifndef PET_H            // pet.h
#ifndef DISPLAY_MANAGER_H  // display_manager.h
#ifndef ACTION_MENU_H    // action_menu.h
```

### The short version

Every `.h` file needs an include guard. Without one, including the same header twice causes a build error. The `#ifndef` / `#define` / `#endif` pattern is the standard way to prevent that. You will see it at the top and bottom of every header file you ever write.

---

## How Multi-Screen Apps Are Structured

### What is a "screen" in embedded code?

In a web app or phone app, a "screen" is usually a whole new page loaded from a server. On a tiny device like the M5StickC Plus 2, there is no server and no pages - there is just one 135x240 pixel display that you draw on directly.

A "screen" in embedded code is just a **set of decisions about what to draw**. The Stats screen draws five stat bars and a mood word (no pet sprite). The Interact screen draws a pet sprite, one stat bar, and an action menu. The Main screen draws a pet sprite and a tab bar. The hardware never changes - only what gets painted on it does.

### The `ScreenState` enum

The simplest way to track which screen is active is with an enum:

```cpp
enum ScreenState {
    SCREEN_MAIN,
    SCREEN_STATS,
    SCREEN_INTERACT
};
```

An enum is just a named list of options. Using an enum instead of a plain integer (0, 1, 2) makes the code readable: `SCREEN_STATS` tells you exactly what it means, while `1` means nothing on its own.

`ScreenState` lives in `lib/Display/screen_layout.h` because it is a display concept - it describes what the screen is showing. Any file that needs to know the current screen can include this header.

### How the three screens are structured

| Screen | What it shows | How you leave it |
|---|---|---|
| `SCREEN_MAIN` | Pet sprite, mood, nav bar (Stats / Interact) | Press B -> Interact, Press C -> Stats |
| `SCREEN_STATS` | All five stat bars + mood word (no pet sprite) | Press any button -> back to Main |
| `SCREEN_INTERACT` | Pet sprite, mood, one contextual stat bar, action menu | Confirm "Back" action -> Main |

---

## How Screen Layouts Are Designed

### Thinking in zones

Before writing a single line of drawing code, it helps to divide the screen into named **zones** - rectangular regions, each with one job. This is the same idea a graphic designer uses when laying out a page: title area, content area, navigation area.

For this project, the Stats screen zones look like this:

```
+-----------------+  y: 0
|   TITLE_ZONE    |     5 - 24    pet name
|   STATS_ZONE    |    26 - 124   five stat bars
|                 |  (124 - 180   left empty - the Stats screen has no pet sprite)
|   MOOD_ZONE     |   180 - 198   mood word
|   MENU_ZONE     |   220 - 240   Back hint
+-----------------+  y: 240
```

Each zone is stored as a `ScreenZone` struct - four numbers that describe its top-left corner and its size:

```cpp
static constexpr ScreenZone TITLE_ZONE = { 0, 5, 135, 19 };
//                                         x  y  w    h
```

Using `constexpr` (explained in the earlier section) means these values are baked in at compile time and do not need a `.cpp` definition.

### Why define layout as data, not code

Without zone structs, drawing code looks like this:

```cpp
M5.Lcd.setCursor(5, 26);   // where is 26? no idea without comments
M5.Lcd.setCursor(5, 48);   // and 48?
M5.Lcd.setCursor(5, 70);   // and 70?
```

These **magic numbers** have no meaning to a reader. Change the layout and you have to hunt for every hardcoded pixel value across the whole file.

With zone structs:

```cpp
M5.Lcd.setCursor(STATS_ZONE.x, HAPPY_BAR_ZONE.labelY);
M5.Lcd.setCursor(STATS_ZONE.x, FULLNESS_BAR_ZONE.labelY);
M5.Lcd.setCursor(STATS_ZONE.x, ENERGY_BAR_ZONE.labelY);
```

Now the intent is visible. If you want to move all the stat bars down by 10 pixels, you change one constant - not six scattered lines.

### Each screen can have its own zones

Not every screen shows the same things in the same places. The Main and Interact screens show the pet sprite; the Stats screen does not - it uses that vertical space for the five stat bars instead. Each screen's render method decides which zones it draws.

The pet sprite is positioned by a single `*_FACE_CENTER_Y` value (the sprite is always centred horizontally and drawn at a fixed 80x80 size). The Main and Interact screens deliberately use the *same* centre, so the pet does not appear to jump when you switch between them:

```cpp
// Main screen - face sits in the centre, no stat bars above it
static constexpr int MAIN_FACE_CENTER_Y = 110;

// Interact screen - same centre as Main, so the pet stays put across screens
static constexpr int INTERACT_FACE_CENTER_Y = 110;  // = MAIN_FACE_CENTER_Y
```

This is not duplication for its own sake - it is each screen owning its own layout while choosing to line up where it matters. The Stats screen owns a different layout entirely: no sprite, more room for data.

---

## Why DisplayManager Does Not Decide What to Draw

### The tempting wrong approach

One tempting design lets `DisplayManager` track its own internal `DisplayState` enum:

```cpp
enum class DisplayState {
    STATUS_VIEW,
    MENU_INDICATOR,
    DEAD
};
```

This meant `DisplayManager` was making two decisions at once: *what to draw* and *how to draw it*. That is two jobs - a violation of the Single Responsibility Principle.

Worse, `STATUS_VIEW` and `MENU_INDICATOR` were not really different screens - they were both the same stats layout, just with or without the action indicator drawn over the top. As the number of real screens grew, this approach would have needed a new `DisplayState` value for every combination of layout and overlay.

### The approach this project uses - the caller decides, DisplayManager executes

Instead, `renderDisplay()` receives a `ScreenState` from the caller:

```cpp
void DisplayManager::renderDisplay(..., ScreenState screenState) {
    switch (screenState) {
        case SCREEN_MAIN:     renderMainScreen(...);     break;
        case SCREEN_STATS:    renderStatsScreen(...);    break;
        case SCREEN_INTERACT: renderInteractScreen(...); break;
    }
}
```

`DisplayManager` no longer guesses what the user is looking at. `NavigationManager` tracks that, and passes the answer in. `DisplayManager` just draws.

This separation - **one module decides, another executes** - is a pattern called **separation of concerns**. Each module concerns itself with exactly one question:

- *What screen is the user on?* -> `NavigationManager`'s job
- *How do I draw that screen?* -> `DisplayManager`'s job

---

## How NavigationManager Keeps `loop()` Clean

### The problem without it

Without `NavigationManager`, all the screen-switching logic would live inside `loop()` as a large `switch` statement. Every time a new screen was added, `loop()` would grow longer. After three or four screens, `loop()` would be difficult to read and easy to break - it would be doing too many jobs at once.

### The solution - delegate to a module

`NavigationManager` owns one piece of state: which screen the user is currently on. Its `update()` method reads button input and switches screens when needed. It is deliberately kept unaware of `ActionMenu` - rather than receiving the menu, it is told only whether the "Back" entry is currently selected, as a plain `bool`. `loop()` stays short:

```cpp
bool backSelected = menu.isBackSelected();
navManager.update(buttons, backSelected);

if (navManager.shouldConfirmAction()) {
    menu.confirmAction(myPet, display, speaker, storage);
}
```

The switch statement still exists - it just lives inside `NavigationManager::update()` where it belongs, rather than mixed into the main program loop.

### One handler per screen

Inside `NavigationManager`, each screen gets its own private handler method:

```cpp
void handleMainScreenInput(const ButtonHandler& buttons);
void handleStatsScreenInput(const ButtonHandler& buttons);
void handleInteractScreenInput(const ButtonHandler& buttons, bool backSelected);
```

Each handler only reads the buttons that are relevant on that screen. `handleStatsScreenInput()` has no idea what `handleInteractScreenInput()` does. If you need to understand what the buttons do on the Stats screen, you open that one method and read four lines. Nothing else to search through.

When a new screen is added in the future, you add one new handler method and one new `case` in `update()`. Everything else stays the same.

---

## The One-Shot Flag Pattern

### The problem

When the user presses Button A to feed the pet, `confirmAction()` should fire **once**. But `loop()` runs hundreds of times per second. If you stored "A was pressed" as a boolean and never cleared it, `confirmAction()` would fire on every single frame until the user released the button - feeding the pet hundreds of times in one press.

### The solution - reset at the top of every frame

`NavigationManager` tracks this with a `confirmActionRequested` flag. The key is where it gets reset:

```cpp
void NavigationManager::update(const ButtonHandler& buttons, bool backSelected) {
    confirmActionRequested = false;  // reset FIRST, before checking anything

    switch (currentScreen) {
        ...
    }
}
```

By resetting `confirmActionRequested` to `false` at the very start of `update()`, the flag can only ever be `true` for **one loop iteration** - the single frame where the user's button press was detected. On the next frame, `update()` resets it back to `false` before anything else runs.

This is called a **one-shot flag**: it fires once and immediately resets itself. You will see this pattern used in `ButtonHandler` (`wasButtonAPressed()` returns `true` only once per press) and in `NavigationManager` (`confirmActionRequested`). A close cousin lives in `Pet`: it plays the death melody exactly once by checking `currentState != STATE_DEAD` just before it switches into `STATE_DEAD`, so the sound fires only on the transition, not on every frame the pet is dead. It is one of the most useful patterns for any program that responds to hardware inputs inside a polling loop.

---

## How Flicker Is Avoided - the Off-Screen Canvas (Double-Buffering)

### The problem with drawing straight to the screen

The M5StickC Plus 2 LCD is driven over SPI - a serial connection that sends pixel data one chunk at a time. If you clear the real screen and then draw each element directly onto it, there is a brief moment where the screen is half-finished: cleared to black, with only some of the new frame painted on. Repeated every loop, that half-drawn moment shows up as **screen flicker**, and it makes the device feel broken.

### The solution - draw to an invisible canvas, then push it in one shot

`DisplayManager` never draws directly to the LCD. Instead it owns a full-screen `M5Canvas` - an off-screen buffer the same size as the display (135x240). Every frame it clears that canvas, draws the *entire* screen onto it (pet sprite, stat bars, mood word, nav bar - everything), and then copies the finished image to the real LCD in a single `pushSprite()` call:

```cpp
void DisplayManager::renderDisplay(..., ScreenState screenState) {
    // Draw the whole screen onto the off-screen canvas...
    switch (screenState) {
        case SCREEN_MAIN:     renderMainScreen(...);     break;
        case SCREEN_STATS:    renderStatsScreen(...);    break;
        case SCREEN_INTERACT: renderInteractScreen(...); break;
    }
    pushCanvas();   // ...then copy the finished canvas to the LCD in one operation
}
```

Because the LCD is only ever updated by that one push of a fully-drawn frame, the user never sees a half-drawn intermediate state - there is no flicker, even though the **whole screen is redrawn every single loop**. This is what **double-buffering** means: one buffer (the canvas) is drawn to while the other (the LCD) is shown, then they swap.

### Why redraw everything every frame instead of only what changed?

One tempting optimisation tries to be clever: throw most frames away with a multi-second redraw throttle and only repaint small regions (like the menu strip) when they change, tracking each fast-changing element with its own "did this change?" variable. That saves SPI traffic, but it has two costs - stat changes can sit stale on screen for several seconds, and every animated element needs its own bookkeeping.

The off-screen canvas removes the need for all of that. Pushing one pre-drawn 135x240 buffer is fast enough to do on every loop without flicker, so there is no throttle and no per-region fast-path. The rule is simply: **redraw the whole canvas every frame, then push it once.** Stat changes and the sprite animation both appear instantly, and there are no `last...`-changed flags to keep in sync.

The one piece of `last...` tracking that remains is `lastRenderedScreen`, and it has nothing to do with flicker - it just lets `DisplayManager` notice when the user has switched screens so it can restart the sprite animation cleanly from frame 0.

---

## How Sprite Images Are Stored and Drawn

Before reading this section, make sure you have read `SPRITE_GUIDE.md`. That guide
covers how to create and convert sprites step by step. This section explains the
*why* behind the technical choices - why images cannot just be loaded from a file,
what RGB565 and ABGR8888 actually mean, and why the `PROGMEM` keyword matters.

### Why not just load a PNG or JPEG?

On a laptop or phone, loading an image is simple: the OS reads the file, a decoder
library unpacks it into pixels, and the display driver draws them. The M5StickC Plus 2
has none of those layers. There is no OS, no filesystem by default, and no image decoder
in memory. The only storage available during a normal program run is the code itself -
sitting in flash memory - and a small amount of RAM.

The solution is to convert the image into a raw array of pixel values *before* the
program is compiled, and bake that array directly into the firmware. When the program
runs, the pixel data is already sitting in flash, ready to draw without any decoding step.
That is what the `piskel_converter` tool does.

### Two colour formats: ABGR8888 and RGB565

Every digital colour is made up of red, green, and blue channels mixed together.
The difference between formats is how many bits are used to describe each channel,
and whether an alpha (transparency) channel is included.

**ABGR8888 - what Piskel exports**

Piskel stores each pixel as a 32-bit number: 8 bits for alpha, then 8 for blue, 8 for
green, and 8 for red. Watch the order - it is **ABGR**, not the more familiar ARGB, so
the red and blue bytes sit in opposite positions. (Green is in the middle either way,
which is why a red/blue mix-up is easy to miss until you draw a red or blue pixel.)

```
One pixel in ABGR8888 - 32 bits (4 bytes):

Bit: 31      24 23      16 15       8 7        0
     +---------+----------+----------+----------+
     |  Alpha  |   Blue   |  Green   |   Red    |
     |  8 bits |  8 bits  |  8 bits  |  8 bits  |
     +---------+----------+----------+----------+

Example: 0xFF689858
  Alpha = 0xFF = 255 -> fully opaque
  Blue  = 0x68 = 104
  Green = 0x98 = 152
  Red   = 0x58 = 88
  Result: a sage green colour
```

**RGB565 - what the M5StickC Plus 2 LCD needs**

The LCD controller uses a 16-bit format: 5 bits for red, 6 bits for green (green gets
an extra bit because human eyes are most sensitive to it), and 5 bits for blue.
No alpha channel - the LCD has no concept of transparency in hardware.
That is 2 bytes per pixel instead of 4, which halves the memory cost.

```
One pixel in RGB565 - 16 bits (2 bytes):

Bit: 15    11 10     5 4      0
     +-------+--------+--------+
     |  Red  | Green  |  Blue  |
     | 5 bits| 6 bits | 5 bits |
     +-------+--------+--------+

Example: 0x5CCD
  Red   = 0b01011 = 11
  Green = 0b100110 = 38
  Blue  = 0b01101 = 13
  Result: the same sage green, using half the memory
```

### How the converter translates between the two formats

The conversion happens in three steps for each pixel:

```
Step 1 - Check alpha.
  If alpha == 0, the pixel is transparent.
  Write the colour key (0x1FF8) and stop.

Step 2 - Extract the colour channels from the 32-bit ABGR value.
  Blue  = (pixel >> 16) & 0xFF   -> 8-bit value, 0-255
  Green = (pixel >>  8) & 0xFF   -> 8-bit value, 0-255
  Red   = (pixel >>  0) & 0xFF   -> 8-bit value, 0-255

Step 3 - Shrink each channel to fit the RGB565 bit widths.
  Red5   = Red   >> 3   (keep top 5 bits, discard bottom 3)
  Green6 = Green >> 2   (keep top 6 bits, discard bottom 2)
  Blue5  = Blue  >> 3   (keep top 5 bits, discard bottom 3)

Step 4 - Pack the three channels into one 16-bit value.
  rgb565 = (Red5 << 11) | (Green6 << 5) | Blue5

Step 5 - Swap the two bytes before storing.
  stored = (rgb565 >> 8) | (rgb565 << 8)
  (Why this is necessary is explained in the byte-ordering section below.)

Example with 0xFF689858:
  Blue  = 0x68 = 104 -> 104 >> 3 = 13  -> 0b01101
  Green = 0x98 = 152 -> 152 >> 2 = 38  -> 0b100110
  Red   = 0x58 = 88  -> 88  >> 3 = 11  -> 0b01011

  Pack: (Red 11 << 11) | (Green 38 << 5) | (Blue 13) = 0x5CCD [x]
```

You can see this exact logic in `tools/piskel_converter/main.cpp` inside the
`convertAbgrToRgb565()` function.

### The transparent colour key

The LCD has no hardware transparency. When the drawing code calls `pushImage()`, it
draws every pixel it is given - there is no automatic "skip this pixel" behaviour.

To get transparency, the converter replaces every pixel with alpha == 0 with a special
**colour key** value. The drawing call then receives that value as the "transparent"
argument and skips any pixel that matches it.

The logical transparent colour is **magenta** - `0xF81F` in RGB565. But because of the
byte-ordering requirement explained in the next section, the value actually stored in
the sprite array and passed to `pushImage()` is `0x1FF8` - the byte-swapped form of
`0xF81F`. Both values refer to the same colour; they are just the same two bytes in
opposite order.

```
Why magenta?

  Magenta (0xF81F) in RGB565:
    Red   = 0b11111 = 31  (maximum)
    Green = 0b000000 = 0  (none)
    Blue  = 0b11111 = 31  (maximum)

  It is the most visually distinct colour from anything likely to appear in a
  pet sprite. Using black (0x0000) as the key would make any genuinely black
  pixel in the sprite also transparent - undesirable. Magenta is the convention.
```

If a magenta pixel ever appears in your finished sprite where you did not intend
transparency, check your source artwork - you may have an unintentionally transparent
pixel that the converter has replaced with the key colour.

### PROGMEM - storing sprite data in flash, not RAM

The ESP32-PICO-V3-02 has two kinds of memory:

```
Flash memory:  4 MB   - large, read-only, stores your firmware and sprite arrays
RAM:         ~200 KB  - small, read-write, used by running code, the display
                        buffer, the stack, and any variables your program creates
```

A single 80x80 sprite at 2 bytes per pixel = 12,800 bytes. The four mood sprites
(NEUTRAL/HAPPY/UNWELL/HUNGRY) together are ~50 KB. That is manageable in flash, but the
moment you add animation frames - each mood gaining a second frame doubles its cost - the
total grows quickly. That is exactly why sprite arrays live in flash by default, not RAM.

The `PROGMEM` keyword (from `<pgmspace.h>`) marks an array for flash storage:

```cpp
// Without PROGMEM - may end up in RAM
static const uint16_t sprite_pet_idle[1][6400] = { ... };  // [frames][80*80]

// With PROGMEM - guaranteed to live in flash
static const uint16_t PROGMEM sprite_pet_idle[1][6400] = { ... };
```

On the ESP32, flash memory is memory-mapped - the CPU can read it using normal array
syntax, the same as RAM. You do not need any special function calls to access the data.
`PROGMEM` is included as a clear signal of intent: this data is read-only and should
not be copied into RAM.

### Why pixel values are stored byte-swapped

When you first test a sprite, you may notice colours appear wrong - a green sprite
shows as light pink, or a blue sprite shows as red. This is caused by a byte-ordering
mismatch between the ESP32 and the LCD controller.

**The problem: two different byte orders**

A 16-bit RGB565 colour is two bytes. There is a question of which byte comes first:

```
Colour: 0x5CCD  (a sage green in RGB565)

Big-endian order    - high byte first: [0x5C] [0xCD]
Little-endian order - low byte first:  [0xCD] [0x5C]
```

- The **ESP32** stores data in **little-endian** order. When the CPU writes the
  `uint16_t` value `0x5CCD` into memory, it stores byte `0xCD` at the lower address
  and byte `0x5C` at the higher address.
- The **SPI driver** sends bytes in address order - low address first. So it sends
  `[0xCD, 0x5C]`.
- The **LCD controller** is **big-endian**. It expects the high byte first and
  interprets whatever arrives as `[first byte] [second byte]`. Receiving `[0xCD, 0x5C]`
  it reads this as the colour `0xCD5C` - completely wrong.

```
Without byte-swap:

  Stored in array:   0x5CCD   (sage green)
  ESP32 memory:      [0xCD] [0x5C]    (little-endian)
  SPI sends:         [0xCD] [0x5C]    (address order)
  LCD receives:      0xCD5C
  Displayed colour:  R=200, G=170, B=230  ->  light lavender-pink  (wrong!)
```

**The fix: pre-swap the bytes in the converter**

By storing the byte-swapped value (`0xCD5C`) in the array instead:

```
With byte-swap:

  Logical colour:    0x5CCD   (sage green)
  Stored in array:   0xCD5C   (bytes pre-swapped)
  ESP32 memory:      [0x5C] [0xCD]    (little-endian storage of 0xCD5C)
  SPI sends:         [0x5C] [0xCD]    (address order)
  LCD receives:      0x5CCD           (big-endian interpretation)
  Displayed colour:  sage green  (correct!)
```

The bytes arrive at the LCD in the right order because the pre-swap in the converter
cancels out the reversal caused by little-endian storage. The `piskel_converter` tool
does this automatically for every pixel, including the transparent colour key. You do
not need to think about it when drawing sprites - but if you ever work with raw pixel
data directly in firmware code, remember that `pushImage()` on this device expects
pre-swapped (byte-swapped) RGB565 values.

---

## Button Guards - Only Respond When Visible

### The hidden side-effect problem

`menu.update(buttons)` reads Button B and Button C to cycle through the action list. But Button B and C are also used on the Main screen to switch between screens.

If `menu.update(buttons)` ran on every frame regardless of which screen was active, pressing B on the Main screen would do two things at once:
1. Switch to the Interact screen (handled by `NavigationManager`)
2. Silently advance the action selection in the menu (handled by `menu.update()`)

The user would arrive at the Interact screen with a different action already selected - and have no idea why.

### The fix - check the screen before passing input

In `loop()`, `menu.update()` is wrapped in a guard:

```cpp
if (navManager.getCurrentScreen() == SCREEN_INTERACT) {
    menu.update(buttons);
}
```

The action menu only reads buttons when it is actually visible. On any other screen, the menu ignores input entirely.

This is a general rule for multi-screen apps: **UI elements should only respond to input when they are on screen**. An invisible menu, dialog, or button that silently responds to input is a bug waiting to happen. Always guard input handling with a screen check.
