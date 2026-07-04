# What's Next for the Virtual Pet?

A bunch of ideas to make this virtual pet even more fun.

---

### 1. Let the Pet Grow Up

Start as a baby, turn into a child, then an adult. Each stage gets its own set
of sprites. You could unlock the next stage by keeping the pet alive long
enough or hitting stat targets. Gives you a real reason to keep going.

### 2. Mini-Games
>[!NOTE]
>Pretty fun, but it introduces more complexity

- **Reaction Time** - a shape flashes on screen, tap A as fast as you can
- **Memory Sequence** - watch a pattern of button presses, then repeat it
- **Tilt Catch** - move the pet side to side to catch falling food
- Games cost energy but boost happiness. Simple enough to add one at a time.

### 3. More Than One Kind of Food
>[!NOTE]
>Seems tedious

Snacks, meals, treats - each with different effects on fullness, hydration,
and happiness. You'd get an inventory system to track what you have, and
maybe find extra food by playing mini-games.

### 4. A Life Log for Your Pet
>[!NOTE]
>Probably this is the one

How many times has it been fed? Bathed? How many times has it been sick?
A little stats screen with the pet's birthday and lifetime numbers. Makes
it feel less like a simulation and more like a real pet with a history.

### 5. Day and Night

The pet sleeps at night (energy drains slower, but you can't do much).
The screen dims on its own when it gets late. Uses the RTC that's already
in the code. Could even tint the background warmer at night.

### 6. Richer Sickness

Different kinds of illness (tummy ache, cold) that need different medicine.
A little mini-game where you have to pick the right treatment before a timer
runs out. Makes healing more interesting than just pressing a button.

### 7. Multiple Save Slots

Three slots you pick from at boot. Each one holds a different pet with its
own stats, name, and history. Good if more than one person wants to use
the device.

### 8. Achievements
>[!NOTE]
>Maybe the most "doable and fun" idea

Small badges you unlock: "Survivor" (respawned 5 times), "Clean Freak"
(100 baths), "Night Owl" (active past midnight), "Green Thumb" (kept the
pet alive for a week). Displayed somewhere on the stats screen. Stored
in NVS so they stick around.

### 9. Screen Brightness Control
>[!NOTE]
>More of a basic nessecity, than an interesting feature

Stick a brightness slider in the action menu. Auto-dim after 30 seconds
of no button presses. Auto-save if the battery gets low. The hardware
supports it, just needs the code.

### 10. Phone App via Bluetooth
>[!NOTE]
>Related to #15

Instead of connecting to the WiFi dashboard, use Bluetooth Low Energy.
A small companion app (Flutter or React Native) would let you check stats
and feed the pet from your phone. Works anywhere, no network needed.

### 11. QR Code to Share Your Pet
>[!NOTE]
>Maybe check docs for camera?

Encode the pet's full state into a short string, show it as a QR code on
screen. Someone else can scan it with their M5StickC and import your pet.
Great for trading or backing up.

### 12. Step Counter

The IMU can already detect motion. Count steps while the device is worn
and tie it into energy (more steps = hungrier pet). Adds a reason to
actually carry the thing around.

### 13. Random Weather
>[!NOTE]
>Interesting...

Every few hours the game picks a weather: sunny, rainy, stormy. Changes
what the background looks like and nudges the pet's mood. On stormy days
the pet might refuse to play. Small touches that make the world feel alive.

### 14. Two Pets, One Game
>[!NOTE]
>Had this idea before ...

Use IR or ESP-NOW to let two M5StickC devices talk to each other. Pets
can visit, play together, trade items. Both get a happiness boost. Cute
and surprisingly technical.

### 15. A Real Mobile App (WiFi Version)
>[!NOTE]
>Take a look at the #10

Same as idea 10 but over WiFi. A proper app that connects to the web
dashboard, sends push notifications when the pet is hungry or sick, and
looks nicer than a browser page. Could pair this with the QR code for
easy setup.
