# Sprite Creation Guide

This guide walks you through the full process of creating a sprite for the virtual pet —
from drawing it in Piskel to having it ready to use in the firmware.

For the technical explanation of why images are stored the way they are (RGB565, PROGMEM,
transparent colour keys), see the **"How Sprite Images Are Stored and Drawn"** section
in `USEFUL_NOTES.md`.

---

## The Workflow at a Glance

```
1. Draw your sprite in Piskel       (piskelapp.com, 80×80 canvas)
            |
2. Export as a C file from Piskel   (Export -> C file -> Download)
            |
3. Rename and move the file         (-> assets/sprites/raw/pet_idle.c)
            |
4. Run the converter tool           (tools/piskel_converter/piskel_converter)
            |
5. The .h file appears              (-> lib/Display/sprites/pet_idle.h)
            |
6. #include it in the display code
```

---

## Part 1 — Drawing Your Sprite in Piskel

### Setting up the canvas

1. Go to [piskelapp.com](https://www.piskelapp.com/p/create/sprite) and click **Create Sprite**.
2. In the right-hand panel, click the **Resize** icon (a square with arrows in the corners).
3. Set **Width: 80** and **Height: 80**, then click **Resize**.
4. Set the **Frames** count at the bottom to **1**. We are drawing one static image per
   mood for now — animation frames come later (with the mood sprites).

The canvas is now 80×80 pixels — this project's sprite size. Each small square you paint is
one pixel on the device screen (the renderer draws it at native size, so what you see in
Piskel is what appears on the device).

---

### The anchor point rule

**The pet's body must occupy roughly the same region of the canvas in every sprite.**

This does not mean every sprite must look the same. Different poses, angles, and expressions
are fine — and expected. What you must keep consistent is *where the body sits within the canvas*.

Here is why. When the pet changes state — say from IDLE to EATING — the rendering code
replaces one sprite with another at the exact same screen position. If the body has shifted
within the canvas, the pet appears to jump on screen even though the code is correct.

#### Good example — different poses, consistent anchor

The pet body occupies roughly the same region of the canvas (columns 3–6, rows 1–4)
in every state. The details change freely: eyes, mouth, arms, sleeping pose.

```
     IDLE                EATING              PLAYING
. . . . . . . .      . . . . . . . .      . . . . . . . .
. . . # # # . .      . . . # # # . .      . . # . . # . .   <-- arms raised
. . . # o # . .      . . . # - # . .      . . . # # # . .   <-- body same region
. . . # # # . .      . . . # # \ . .      . . . # o # . .
. . . . | . . .      . . . . | . . .      . . . # # # . .
. . . . . . . .      . . . . . . . .      . . . . | . . .
. . . . . . . .      . . . . . . . .      . . . . . . . .
```

On screen, the transitions look like this. The pet stays in place:

```
         IDLE        EATING      PLAYING

Screen:  [###]  ->   [###]  ->   [# #]
         [#o#]       [#-#]       [###]
         [###]       [##\]       [#o#]
          | |         | |        [###]
                                   |
         No jump. The pet stays in the same screen position every time.
```

#### Bad example — body shifts between states

The EATING sprite was drawn with the pet body shifted to the right.

```
     IDLE                EATING (body shifted — wrong)
. . . . . . . .      . . . . . . . .
. . . # # # . .      . . . . . # # #   <-- same pet, wrong canvas position
. . . # o # . .      . . . . . # - #
. . . # # # . .      . . . . . # # \
. . . . | . . .      . . . . . . | . .
. . . . . . . .      . . . . . . . . .
. . . . . . . .      . . . . . . . . .
```

On screen, the transition looks like this. The pet snaps sideways:

```
         IDLE        EATING

Screen:  [###]           [###]
         [#o#]   ->           [#-#]   <-- jumped right! the code is fine, the art is wrong.
         [###]           [##\]
          |                   |
```

#### Completely different orientations are fine — as long as they are centred

A sleeping pet lies on its side — a totally different orientation. That is fine.
The body still fills roughly the same central region of the canvas.

```
     IDLE                SLEEPING (rotated, but centred)
. . . . . . . .      . . . . . . . .
. . . # # # . .      . . . . . . . .
. . . # o # . .      . . # # # # # .   <-- rotated body, same central region
. . . # # # . .      . . # - . . # .
. . . . | . . .      . . # # # # # .
. . . . . . . .      . . . z . . . .
. . . . . . . .      . . . . . . . .
```

The rule in plain terms: **you can draw anything you want, as long as you do not slide
the whole body across the canvas between states**. Moving limbs, opening mouths, rotating,
adding accessories — all fine. Accidentally placing the pet at a different part of the
canvas in one state — that is what causes the jump.

---

### Transparency — what the background will look like

Empty (unpainted) pixels in Piskel export as fully transparent. The converter replaces
them with a magenta colour key (`0xF81F`). When the rendering code draws the sprite,
it skips any pixel that matches the key — letting the screen background show through.

**The screens in this project use a black background.**

This matters when you design your sprite:

```
What you draw in Piskel:      What appears on the black screen:

. . . . . . . .               # # # # # # # #   <-- black (transparent = background)
. . # # # # . .               # # G G G G # #   <-- green body
. . # . . # . .               # # G # # G # #   <-- black hole (transparent inside)
. . # . . # . .               # # G # # G # #   <-- black hole (transparent inside)
. . # # # # . .               # # G G G G # #   <-- green body
. . . . . . . .               # # # # # # # #   <-- black (transparent = background)

  . = transparent               # = black (background showing through)
  # = painted green             G = green pixel
```

Transparent pixels *inside* the pet body will appear as black holes. Decide deliberately
whether each area should be painted or left transparent. If you want the inside of an eye
to be white, paint it white — do not leave it empty.

---

### Choosing your colours

- **Keep the pet body colour the same across all states.** The player should recognise
  the pet as the same creature whether it is eating, sleeping, or sick.
- **3–4 colours per sprite is usually enough.** At 80×80 on this small screen, subtle colour
  differences are hard to see. Bold, distinct colours read far more clearly than many similar shades.
- **Use your darkest colour for outlines.** A clear outline helps the pet stand out
  against the black background.

---

### Which states need a sprite

> **How the shipped code actually picks a face:** the project draws one sprite per **mood**
> (`MoodSprite`: neutral / happy / unwell / hungry), chosen by `Pet::computeMood()` — see
> **Part 8**. That is four sprites, not one per `PetState`. The richer per-`PetState` set below is
> an *optional* direction (e.g. a distinct eating or sleeping pose); treat this table as drawing
> guidance for that extension, not as a description of the current renderer.

You need one sprite file for each value in the `PetState` enum in `lib/Pet/pet.h`.

| State constant | When the pet enters this state | Suggested look |
|---|---|---|
| `STATE_IDLE` | Default — pet is waiting | Facing forward, neutral or happy expression |
| `STATE_EATING` | After Feed action is confirmed | Mouth open, food nearby |
| `STATE_PLAYING` | While the device is being shaken | Arms raised, energetic pose |
| `STATE_SLEEPING` | After Sleep action is confirmed | Eyes closed, lying down, Zzz |
| `STATE_BATHING` | After Bathe action is confirmed | Bubbles or water drops around the pet |
| `STATE_HEALING` | After Heal action is confirmed | Small cross or sparkles |
| `STATE_SICK` | When sickness stat is high | Drooping posture, unwell expression |

**Start with `STATE_IDLE`.** Confirm the converter produces a valid `.h` file, then draw
the remaining states one at a time.

---

## Part 2 — Exporting from Piskel

When your sprite is ready:

1. Click the **Export** icon in the left-hand panel (arrow pointing right, out of a box).
2. Select the **C file** tab.
3. Leave the settings at their defaults — **4 bytes per pixel**. This is the ARGB8888
   format the converter expects. Do not change it.
4. Click **Download**.
5. Rename the downloaded file to match the state it represents. Use lowercase with underscores:

| State | Rename the file to |
|---|---|
| `STATE_IDLE` | `pet_idle.c` |
| `STATE_EATING` | `pet_eating.c` |
| `STATE_PLAYING` | `pet_playing.c` |
| `STATE_SLEEPING` | `pet_sleeping.c` |
| `STATE_BATHING` | `pet_bathing.c` |
| `STATE_HEALING` | `pet_healing.c` |
| `STATE_SICK` | `pet_sick.c` |

6. Move the renamed file into `assets/sprites/raw/`.

---

## Part 3 — Running the Converter

The converter is a small C++ program that runs on your Mac — not on the ESP32.
Full instructions are in `tools/piskel_converter/README.md`.

**Compile the tool once** (only needed the first time, or if `main.cpp` changes):

```bash
g++ -std=c++17 -o tools/piskel_converter/piskel_converter tools/piskel_converter/main.cpp
```

**Convert one sprite** (run this from the project root):

```bash
tools/piskel_converter/piskel_converter assets/sprites/raw/pet_idle.c lib/Display/sprites/pet_idle.h
```

When it works, you will see:

```
Converted 1 frame(s), 80x80 pixels each.
Output: lib/Display/sprites/pet_idle.h
```

**Convert all sprites at once** using a shell loop:

```bash
for file in assets/sprites/raw/*.c; do
    name=$(basename "$file" .c)
    tools/piskel_converter/piskel_converter "$file" "lib/Display/sprites/${name}.h"
done
```

---

## Part 4 — Rules That Affect Rendering and Animation Later

These rules apply now, at sprite creation time. Breaking them causes visual bugs that
are difficult to diagnose once the rendering code is in place.

### Rule 1 — All sprites must be the same size

Every sprite must be the **same size as the others**. This project uses **80×80 pixels** —
all four mood sprites are 80×80. If neutral is 80×80 and happy is 48×48, the rendering code
needs a special case for every mismatched sprite. Pick one size and use it for every sprite
in the project.

The sprites are drawn at **native size** — the renderer does not scale them (it draws each
80×80 sprite with a single `pushImage()` at 80×80). The converter handles any size — it reads
the dimensions from the Piskel file. But the rendering code expects one consistent size for
every sprite.

### Rule 2 — Anchor point must be consistent (see Part 1)

Already covered above. The single most common sprite bug. Check every sprite before
exporting: does the pet body sit in the same region of the canvas as your IDLE sprite?

### Rule 3 — Sprite data is stored in flash, not RAM

The `.h` files the converter produces include the `PROGMEM` keyword. This keeps the
pixel arrays in the ESP32's flash memory (4 MB) rather than RAM (only ~200 KB available
after the display buffer). See `USEFUL_NOTES.md` — **"How Sprite Images Are Stored and
Drawn"** — for the full explanation of why this matters and how it works.

### Rule 4 — Flash memory budget

Each mood is one static frame for now — well within budget. Here is the maths at the
project's 80×80 size:

```
4 moods × 1 frame × 80 pixels wide × 80 pixels tall × 2 bytes per pixel
= 51,200 bytes ≈ 50 KB
```

The ESP32 has roughly 2.5 MB of flash available after the firmware, so ~50 KB for the four
single-frame mood sprites is comfortable. Animation multiplies the cost — it is `frames ×
sprites`, so two frames per mood doubles it, and so on — so keep frame counts modest. The cost
table later in this Part shows how quickly the budget grows as you add frames or increase the
sprite size.

### Rule 5 — One frame per mood (for now)

Do not draw animation sequences yet. One clear, well-centred drawing per mood is the
deliverable for this task. Multi-frame animation comes later. Getting the single frames
right first makes the animation work much smoother when you get there.

> **Picking up animation later?** When you add animation, see [Part 6 — Animating Your
> Sprite](#part-6--animating-your-sprite) for the multi-frame drawing rules and a
> full walkthrough of the frame-cycling code.

---

## Part 5 — How Your Sprite Appears on Each Screen

The pet sprite is **80×80 pixels, drawn at native size — the renderer does not scale it**.
It appears on two screens — MAIN and INTERACT — at the **same size and the same position**
on both, so the pet never appears to jump or resize when you switch between them. The third
screen, STATS, shows **no pet sprite** at all: it is a pure data view (five stat bars and the
mood word). Understanding the boundaries now helps you design a sprite that reads clearly at
80×80.

The face-centre coordinates below come straight from the constants in
`lib/Display/display_manager.h`.

---

### Screen 1 — MAIN (the home screen)

The main screen shows a large version of the pet in the centre of the 135x240 display.
There are no stat bars on this screen — it is intentionally uncluttered.

```
|<--------- 135 px -------->|
+---------------------------+  y = 0
|                           |
|    Pet name (centred)     |  TITLE_ZONE: y = 5, height = 19
|                           |
+---------------------------+  y = 24
|                           |
|                           |
|       +-----------+       |
|       |           |       |
|       | 80 x 80px |       |  80×80, drawn at native size (1×)
|       |           |       |  face centre: y = 110  (MAIN_FACE_CENTER_Y)
|       |  native   |       |  top edge: y = 70   (110 - 40)
|       |           |       |  bottom edge: y = 150  (110 + 40)
|       +-----------+       |
|                           |
+---------------------------+  y = 155
|   Mood text (centred)     |  MAIN_MOOD_Y = 155
+---------------------------+  y ~171
|                           |
|       (free space)        |
|                           |
+===========================+  y = 220
|   [Stats]  |  [Interact]  |  MAIN_NAV_ZONE: y = 220, height = 20
+===========================+  y = 240
```

**Key constraints on MAIN:**
- Pet name title occupies y = 5 to y = 24 — pet must not overlap it
- Mood text appears at y = 155 — the bottom edge of the sprite (y = 150) clears it by 5 px
- Nav bar starts at y = 220 and ends at y = 240 — well below the pet

> **Session-1-only note:** in the early no-action-menu build (`ENABLE_ACTION_MENU` off), a single
> fullness bar is drawn in the band between the mood text and the nav bar — `MAIN_STAT_ZONE`
> = {5, 174, 125, 30}, label at y = 177, bar at y = 190. Once the action menu is enabled the
> Interact screen carries the stat bars, so this band is left empty and the Main screen is just
> the pet's face again.

---

### Screen 2 — STATS (the detailed stats screen)

The stats screen is a **pure data view — it draws no pet sprite at all.** It shows the pet
name, the five stat bars, the mood **word**, and a "B/C: Back" hint. There is no pet face on
this screen (an earlier design had one, but it was removed). Nothing here constrains your
sprite — it is included only so you can see where the data lives.

```
|<--------- 135 px -------->|
+---------------------------+  y = 0
|                           |
|    Pet name (centred)     |  TITLE_ZONE: y = 5, height = 19
|                           |
+---------------------------+  y = 24
|  Happy: 80                |  label at y = 26   (HAPPY_BAR_ZONE.labelY)
|  [====================]   |  bar   at y = 36   (HAPPY_BAR_ZONE.barY), height = 10
|  Fullness: 60             |  label at y = 48   (FULLNESS_BAR_ZONE.labelY)
|  [===============      ]  |  bar   at y = 58   (FULLNESS_BAR_ZONE.barY), height = 10
|  Energy: 75               |  label at y = 70   (ENERGY_BAR_ZONE.labelY)
|  [==================   ]  |  bar   at y = 80   (ENERGY_BAR_ZONE.barY), height = 10
|  Clean: 90                |  label at y = 92   (CLEAN_BAR_ZONE.labelY)
|  [=====================]  |  bar   at y = 102  (CLEAN_BAR_ZONE.barY), height = 10
|  Sick: 20                 |  label at y = 114  (SICK_BAR_ZONE.labelY)
|  [====                 ]  |  bar   at y = 124  (SICK_BAR_ZONE.barY), height = 10
+---------------------------+  y = 134  (last bar ends here)
|                           |
|         (no sprite)       |  the STATS screen draws no pet face
|                           |
|    Mood text (centred)    |  MOOD_ZONE: y = 180, height = 18
|                           |
+===========================+  y = 220
|   [B/C: Back]             |  MENU_ZONE: y = 220, height = 20
+===========================+  y = 240
```

**Key constraints on STATS:**
- This screen shows **no pet sprite** — it is a pure data view, so it imposes no size limit
- The five stat bars run from y = 26 down to y = 134 (each bar 10 px tall)
- The mood **word** appears at y = 180 (MOOD_ZONE) — the word only, never a face
- The "B/C: Back" hint sits at the bottom at y = 220 (MENU_ZONE)

---

### Screen 3 — INTERACT (the action menu screen)

The interact screen shows the pet at the top — at the **same 80×80 native size and the same
position as on the MAIN screen** — with a contextual stat bar and the action menu indicator
below it.

```
|<--------- 135 px -------->|
+---------------------------+  y = 0
|                           |
|    Pet name (centred)     |  TITLE_ZONE: y = 5, height = 19
|                           |
+---------------------------+  y = 24
|                           |
|       +-----------+       |
|       |           |       |
|       | 80 x 80px |       |  80×80, drawn at native size (1×)
|       |           |       |  face centre: y = 110  (INTERACT_FACE_CENTER_Y, = MAIN)
|       |  native   |       |  top edge: y = 70   (110 - 40)
|       |           |       |  bottom edge: y = 150  (110 + 40)
|       +-----------+       |
|                           |
+---------------------------+  y = 155
|   Mood text (centred)     |  INTERACT_MOOD_Y = 155
+---------------------------+  y ~171
|                           |
+===========================+  y = 174
|  Fullness: 72             |  INTERACT_STAT_ZONE: x = 5, y = 174, width = 125, height = 30
|  [================    ]   |  label at y = 177, bar at y = 190
+===========================+  y = 204  (174 + 30)
|                           |
|                           |
|                           |
+===========================+  y = 220
|  Action: [Feed]           |  MENU_ZONE: y = 220, height = 20
+===========================+  y = 240
```

**Key constraints on INTERACT:**
- The pet is 80×80 at native size, face centre y = 110 — identical to MAIN, so it never jumps
- Mood text is at y = 155 — the bottom edge of the sprite (y = 150) clears it by 5 px
- The contextual stat bar (INTERACT_STAT_ZONE, y = 174; label y = 177, bar y = 190) shows the
  stat affected by the selected action
- The action menu indicator (y = 220) is at the very bottom — separate from the stat bar

---

### Summary table

All values are taken directly from the constants in `lib/Display/display_manager.h`.

| Screen   | Pet size on screen | Scale       | Face centre (y) | Zone boundaries          |
|----------|--------------------|-------------|-----------------|--------------------------|
| MAIN     | 80 x 80 px         | native (1×) | y = 110         | title above y=24, mood at y=155 |
| STATS    | no sprite          | —           | —               | data only: bars to y=134, mood word at y=180 |
| INTERACT | 80 x 80 px         | native (1×) | y = 110         | title above y=24, mood at y=155 |

**The key takeaway:** the pet sprite is **80×80, drawn at native size** on the MAIN and
INTERACT screens — the **same size and the same place on both**, so it never jumps when you
switch screens. The STATS screen shows **no sprite** at all — only the stat bars and the mood
word. So you have exactly one size to design for: draw bold shapes and clear outlines that
read well at 80×80.

---

### Sprite size and flash memory cost

Every pixel in a sprite is stored as a 16-bit (2-byte) RGB565 value in flash memory.
The total cost of a complete set of sprites is:

```
total bytes = width × height × frames_per_sprite × number_of_sprites × 2
```

This project's chosen size is **80×80**, and it ships four single-frame mood sprites — the
80×80 row below, in its "×1 frame" column, is the one that matches the firmware. The wider
table is kept as a general reference: it shows flash cost at each base size from 32×32 up to
135×135 (the largest square that fits the 135 px screen width), and how cost grows as you add
animation frames and more sprites. (The columns are headed "7 states" because the original
design had one sprite per pet *state*; read them as "7 sprites" — the per-pixel maths is the
same whatever you are drawing.)

| Base size | Per frame | 7 states × 1 frame | 7 states × 4 frames | 7 states × 8 frames | Budget |
|-----------|-----------|--------------------|---------------------|---------------------|--------|
| 32 × 32   |    2 KB   |    14 KB           |    56 KB            |   112 KB            | ✓      |
| 48 × 48   |    4.5 KB |    31 KB           |   126 KB            |   252 KB            | ✓      |
| 64 × 64   |    8 KB   |    56 KB           |   224 KB            |   448 KB            | ✓      |
| 80 × 80   |   12.5 KB |    88 KB           |   350 KB            |   700 KB            | ⚠      |
| 96 × 96   |   18 KB   |   126 KB           |   504 KB            | 1,008 KB            | ✗      |
| 112 × 112 |   24.5 KB |   172 KB           |   686 KB            | 1,372 KB            | ✗      |
| 128 × 128 |   32 KB   |   224 KB           |   896 KB            | 1,792 KB            | ✗      |
| 135 × 135 |   35.6 KB |   249 KB           |   997 KB            | 1,993 KB            | ✗      |

**Budget key:**
- ✓ safe — 7 states × 8 frames fits well within the ~812 KB of free flash
- ⚠ tight — 7 states × 8 frames uses ~700 KB; reduce frame count or state count to stay safe
- ✗ over budget for 8 frames — keep to 4 frames per state max (96×96 at 4 frames = 504 KB ✓)

**Flash budget explained:** The firmware partition on the M5StickC Plus 2 is ~1,260 KB.
The compiled firmware binary currently uses ~448 KB, leaving approximately 812 KB for sprite data.
These numbers will shift slightly as more code is added, so treat 700 KB of sprite data as
the practical ceiling.

**Formula reminder:** pixel count grows with the square of the side length — doubling the
sprite from 32×32 to 64×64 quadruples the pixel count (1,024 → 4,096) and quadruples
the flash cost. A jump from 64×64 to 128×128 costs 16× as much as the original 32×32.

---

### How sprite size interacts with each screen zone

The project draws the sprite at **native size — one pixel in the sprite is one pixel on
screen, with no scaling**. The only thing that limits how big a sprite can be is the space
each screen leaves for the pet face. Only two screens draw a sprite (MAIN and INTERACT); the
STATS screen draws none, so it places no limit at all.

Both MAIN and INTERACT centre the face at y = 110, leaving the face room between the title
(which ends at y = 24) and the mood text (at y = 155) — about a **90 px tall** band. A sprite
fits comfortably as long as it stays within that band:

| Base size | MAIN / INTERACT (≈90 px face band) | STATS |
|-----------|------------------------------------|-------|
| 64 × 64   | fits with 26 px spare              | no sprite — no limit |
| 80 × 80   | fits with 10 px spare (the chosen size) | no sprite — no limit |
| 96 × 96   | clips (6 px over)                  | no sprite — no limit |
| 112 × 112 | clips (22 px over)                 | no sprite — no limit |
| 128 × 128 | clips (38 px over)                 | no sprite — no limit |

"Clips" means the sprite is taller than the face band, so its top and bottom edges would
overrun the title or the mood text. Because the renderer draws at native size, the fix is to
draw a smaller sprite — there is no render-time scaling step in this project to shrink an
oversized one. The shipped **80×80** size fits the band with 10 px to spare, which is why it
was chosen.

> **Want a larger pet anyway?** M5GFX does offer a scaling draw (`pushImageAffine()`), so a
> bigger sprite *could* be shrunk at render time — but this project deliberately does not use
> it. Everything here is drawn 1:1 at native size, which keeps the rendering code simple and
> the pet pixel-crisp.

---

## Part 6 — Animating Your Sprite

Everything above produces a **still** picture: one frame per state, drawn once and held.
Animation brings the pet to life by showing several frames in turn — frame 0, then frame 1,
then back again — so it bounces, blinks, or breathes. This part covers both halves of that
work: **drawing** a multi-frame sprite, and the small piece of **code** that decides which
frame to show at each moment.

### 6.1 — Drawing a multi-frame sprite

A multi-frame sprite is just a normal Piskel drawing with **more than one frame** on the
timeline (use the **Frames** dropdown at the bottom of the Piskel canvas to add frames).
The converter already understands this — it reads however many frames you export and emits a
`sprite_name[FRAME_COUNT][W*H]` array. Nothing about the export or converter step changes.

Three art rules make an animation read well on a small 135×240 screen:

**Rule A — Anchor the silhouette across every frame.**
This is the [anchor point rule](#the-anchor-point-rule) from Part 1, and it matters even more
for animation. If the pet's body sits in a different part of the canvas from one frame to the
next, it will appear to *teleport* instead of move. Keep the body in the same region; move only
the part that should animate (eyes blinking, a small bob, ears twitching).

**Rule B — Move at least two pixels.**
At our starting speed of 5 frames per second, a one-pixel change is almost invisible. If you
want motion to read, move the animated part by **two pixels or more** between frames. Our
placeholder bounce, for example, shifts the whole body down by two pixels on the second frame.

**Rule C — Loop cleanly.**
The animation plays forever: after the last frame it jumps straight back to frame 0. Design the
frames so that jump looks smooth. Two reliable patterns:
- **Simple loop:** frame 0 → 1 → 2 → 0 → 1 → 2 … Best when the last frame flows naturally back
  into the first (e.g. a breathing cycle).
- **Ping-pong:** draw frames that go out and back — 0 → 1 → 2 → 1 → 0 — by exporting the
  middle frames twice. Best for a back-and-forth motion like a bounce or a wag, because there is
  no sudden jump at the loop point.

**Frame budget.** Keep to the **8-frames-per-state** ceiling from [Rule 4](#rule-4--flash-memory-budget).
More frames means smoother motion but more flash used; the maths in Rule 4 shows how quickly it
adds up.

### 6.2 — How frames cycle at runtime: the `AnimationManager`

Drawing the frames is only half the job. At runtime, *something* has to decide **which frame is
on screen right now** and change it at a steady speed. That is the entire job of
`lib/Display/animation_manager.{h,cpp}`. It does **not** draw anything — `DisplayManager` asks it
"which frame?" and draws that one. Splitting it this way keeps each class doing exactly one job
(the most important design rule in this codebase).

#### The trap to avoid: `delay()`

The tempting beginner approach is:

```cpp
drawFrame(0);
delay(200);
drawFrame(1);
delay(200);
```

**Never do this.** `delay(200)` freezes the *whole* program for 200 ms — buttons stop responding,
sound stops, the stat timers stop. On a microcontroller running one big `loop()`, `delay()` inside
the loop is almost always a bug (see Hardware Gotcha #2 in `CLAUDE.md`).

#### The pattern to use: "check the clock"

Instead of *waiting*, we *check the clock* every time around the loop and ask: **"has enough time
passed since I last changed the frame?"** `millis()` is the clock — it returns the number of
milliseconds since the device powered on, counting up forever. This is the exact same
non-blocking pattern the students already met in `TimerManager` (fullness falling over time); here it
is reused to advance an animation frame.

#### What the class remembers (its member variables)

```cpp
int frameCount;                     // how many frames the sprite has (e.g. 2)
unsigned long frameDurationMs;      // how long to show each frame (200 ms = 5 fps)
int currentFrame;                   // which frame is showing now (0 .. frameCount-1)
unsigned long lastFrameAdvanceTime; // the millis() value when we last switched frame
```

- `unsigned long` is used for every time value because `millis()` returns a large, always-positive
  number — the same type `TimerManager` uses.
- `currentFrame` is the one piece of state the rest of the program reads; everything else exists
  to keep it correct.

#### The constructor — setting up

```cpp
AnimationManager::AnimationManager(int frameCount, unsigned long frameDurationMs)
    : frameCount(frameCount),
      frameDurationMs(frameDurationMs),
      currentFrame(0),
      lastFrameAdvanceTime(0) {
}
```

It is told **how many frames** and **how fast** (with `FRAME_DURATION_MS = 200` as the default, so
`AnimationManager petAnimation(2);` gives 5 fps automatically). We start on frame 0, and start
`lastFrameAdvanceTime` at **0** on purpose: on the very first loop `millis()` is already larger
than 0, so "has 200 ms passed since time 0?" is true immediately and the animation starts right
away instead of pausing first. `TimerManager` uses the identical trick.

> The `: frameCount(frameCount), …` syntax is a C++ **member initializer list** — the standard way
> to set member variables when an object is created. The member and the parameter share a name, but
> the part in parentheses is always the parameter, so there is no ambiguity.

#### `update()` — the heart of it

Called **once every loop**:

```cpp
void AnimationManager::update() {
    if (frameCount <= 1) {
        return;                                    // (A)
    }

    unsigned long currentTime = millis();          // (B)

    if (currentTime - lastFrameAdvanceTime >= frameDurationMs) {   // (C)
        currentFrame = (currentFrame + 1) % frameCount;            // (D)
        lastFrameAdvanceTime = currentTime;                        // (E)
    }
}
```

- **(A) The guard.** A single-frame sprite (a still image) has nothing to cycle through, so we
  leave immediately. This also prevents a crash on line (D): you can never do `% 0`.
- **(B) Read the clock once** into a clearly named variable.
- **(C) The question.** "Current time − the time I last switched ≥ how long a frame should last?"
  If not, the `if` is false, the body is skipped, and `update()` returns instantly — no waiting.
- **(D) Advance the frame, with wrap-around.** This is the line to slow down on:

  ```cpp
  currentFrame = (currentFrame + 1) % frameCount;
  ```

  `currentFrame + 1` moves to the next frame; `% frameCount` (**modulo** — the remainder after
  division) wraps it back to 0 after the last frame. With 2 frames: `(0+1)%2 = 1`, then
  `(1+1)%2 = 0`. Modulo is the clean, idiomatic way to make a counter loop. A good 2-minute
  exercise: "what does `% 3` give for 0,1,2,3,4,5?" → 0,1,2,0,1,2.
- **(E) Remember when** we switched, so the next interval is measured from now.

#### The two helpers

```cpp
int AnimationManager::getCurrentFrame() const {   // which frame should I draw?
    return currentFrame;
}

void AnimationManager::reset() {                   // start the animation over
    currentFrame = 0;
    lastFrameAdvanceTime = millis();
}
```

`getCurrentFrame()` is what `DisplayManager` calls to ask which frame to draw. The `const` promises
it only *reads* the object, never changes it — a good habit to teach for "getter" functions.
`reset()` jumps back to frame 0 and restarts the timer; we call it when switching screens so the
pet does not appear to resume mid-bounce.

#### A worked timeline (good for the whiteboard)

Frame duration = 200 ms, 2 frames. The loop runs hundreds of times per second:

| `millis()` | `update()` asks: passed ≥ 200? | `currentFrame` after |
|-----------:|-------------------------------|:--------------------:|
| 5          | 5 − 0 = 5 → no                | 0 |
| 150        | 150 − 0 = 150 → no            | 0 |
| 205        | 205 − 0 = 205 → **yes**       | 1  (last = 205) |
| 300        | 300 − 205 = 95 → no           | 1 |
| 410        | 410 − 205 = 205 → **yes**     | 0  (last = 410) |

The frame only changes on the rare loops where enough time has passed; every other loop `update()`
does almost nothing and returns instantly. That is *why* buttons and sound stay responsive while
the pet animates.

#### The architectural lesson worth emphasising

`AnimationManager` knows about **frame numbers and time** — nothing else. It never touches the
screen, M5, or pixels; drawing stays in `DisplayManager`. That clean split — "each module has
exactly one job" — is the single most important design idea in this codebase, and this small class
is a tidy example to point students to.

---

## Part 7 — Sliding Your Sprite With Tilt (optional demo)

Part 6 made the pet change **which frame** is on screen over time (it bounces). This optional extra
makes it change **where on the screen** it is drawn, based on how you physically **tilt the
device** — the pet slides toward the low side, like a marble on a tray. It is a self-contained
demo you can switch on to play with the accelerometer; turning it off returns the pet to dead
centre exactly as before.

The key idea to take away: **animation (which frame) and motion (where on screen) are two separate
jobs.** `AnimationManager` owns the frame timing; a second small helper, `TiltMotion`, owns the
position. They compose without knowing about each other — the pet bounces *and* glides at once.

### 7.1 — Where the tilt signal comes from

The M5StickC has an accelerometer (the MPU6886) that already reports gravity as three numbers —
`accelX`, `accelY`, `accelZ` — in units of **g** (1 g = the pull of gravity). Held flat, gravity
points straight down through the Z axis, so X and Y read ≈ 0. **Tilt** the board and some of that
gravity "leans" into the X and Y axes. That lean is exactly the signal we want: tilt right and
`accelX` grows; tilt away from you and `accelY` grows. `ImuManager` already reads these every loop
for shake detection, and exposes `imu.getAccelX()` / `getAccelY()` — until now those getters were
unused.

### 7.2 — Two problems with raw tilt, and the two fixes

If we drew the pet straight at the raw reading, two things would go wrong. Each fix is one idea in
`lib/Imu/tilt_motion.{h,cpp}`:

1. **Raw data is noisy and jumpy** → the pet would jitter. Fix: a **low-pass filter** (smoothing).
   Instead of snapping to each new reading, ease toward it a little every frame.
2. **A hard tilt would push the pet off the screen** → Fix: a **clamp**, capping the offset to a
   maximum number of pixels.

So `TiltMotion::update()` runs a four-step pipeline every loop: **scale → clamp → smooth → round.**

```cpp
void TiltMotion::update(float accelX, float accelY) {
    // 1 & 2 — scale tilt (g) to pixels, then clamp so the pet stays on screen.
    float targetX = constrain(accelX * TILT_SCALE, -MAX_OFFSET_X, MAX_OFFSET_X);
    float targetY = constrain(accelY * TILT_SCALE, -MAX_OFFSET_Y, MAX_OFFSET_Y);

    // 3 — ease the stored offset a fraction of the way toward the target.
    smoothedX = smoothedX + (targetX - smoothedX) * SMOOTHING_FACTOR;
    smoothedY = smoothedY + (targetY - smoothedY) * SMOOTHING_FACTOR;
}
```

- **`TILT_SCALE`** turns g into pixels — the main *sensitivity* knob (bigger = pet moves further).
- **`constrain(value, low, high)`** is an Arduino helper that returns `value`, or the nearest limit
  if it is out of range. We clamp the **target**, so even a violent shake can only ask for a slide
  of `MAX_OFFSET` pixels.
- **`MAX_OFFSET_X / MAX_OFFSET_Y`** are the limits. The sprite is 80 px wide on a 135 px screen,
  leaving `(135 − 80) / 2 = 27` px of slack each side, so 25 keeps it just inside the edge.

### 7.3 — The smoothing line, slowly

```cpp
smoothedX = smoothedX + (targetX - smoothedX) * SMOOTHING_FACTOR;
```

Read it as: *"move from where I am a fraction of the way toward where I want to be."*
`(targetX − smoothedX)` is the gap left to close; `* SMOOTHING_FACTOR` (0.2) closes 20% of it this
frame, then 20% of the new smaller gap next frame, and so on — fast at first, gently slowing as it
arrives. That is a **low-pass filter**: sudden jitter in the raw reading is averaged away, so the
pet glides instead of twitching.

It is the *same* "a little closer each tick" idea as the `millis()` timers in Part 6 and in
`TimerManager` — there it was applied to **time**, here to **position**. `SMOOTHING_FACTOR` is the
feel knob: 0 would freeze the pet, 1 would snap instantly (all jitter, no smoothing), 0.2 is calm.

#### What "low-pass filter" means

A **low-pass filter** lets *slow* changes through but blocks *fast* ones. A real, sustained tilt
lasts many frames, so it eases all the way through; a one-frame noise spike is gone before the pet
can move 20% toward it, so it gets smoothed away. That is exactly why the pet follows your tilt but
ignores the accelerometer's constant tiny jitter.

#### A worked example

Start the pet at `0` and tilt so `targetX = 100`, with `SMOOTHING_FACTOR = 0.2`. Each frame it
closes 20% of the gap that is *left*:

| Frame | Gap (`target − smoothed`) | Step (gap × 0.2) | New `smoothedX` |
|------:|--------------------------:|-----------------:|----------------:|
| 1 | 100  | 20   | 20   |
| 2 | 80   | 16   | 36   |
| 3 | 64   | 12.8 | 48.8 |
| 4 | 51.2 | 10.2 | 59.0 |

The steps shrink every frame, so the pet *glides in* — fast at first, easing as it arrives — and
never snaps. (It approaches 100 without quite landing on it exactly, which is why we keep
`smoothedX` as a `float`; see 7.4.)

> **Why it recenters by itself.** Hold the device flat and tilt ≈ 0, so `targetX` ≈ 0. The line
> above then keeps easing `smoothedX` toward 0 — the pet drifts back to centre on its own. There is
> no special "go home" code; it falls straight out of the maths.

### 7.4 — Handing out whole pixels

The offsets are kept as `float`s so the easing keeps its precision frame to frame. The screen only
deals in whole pixels, so we round **only when handing the value out**:

```cpp
int TiltMotion::getOffsetX() const { return (int) round(smoothedX); }
int TiltMotion::getOffsetY() const { return (int) round(smoothedY); }
```

### 7.5 — How the offset reaches the screen

`TiltMotion` does pure maths and touches no pixels — just like `AnimationManager`. The offset
travels from `main.cpp` (which owns the IMU) down to the one place that positions the sprite:

```
main.cpp  →  renderDisplay()  →  renderMainScreen()/renderInteractScreen()  →  drawPetSprite()
```

`drawPetSprite()` simply adds the offset to the centre it already computes:

```cpp
int spriteX = (SCREEN_WIDTH - spriteWidth) / 2 + spriteOffsetX;
int spriteY = faceCenterY - (spriteHeight / 2) + spriteOffsetY;
```

When the offset is `0, 0`, this *is* the old centred position — so every screen looks unchanged.
(The Stats screen draws no pet sprite, so it is left out of this entirely.)

### 7.6 — The on/off switch

One flag near the top of `main.cpp` controls the whole effect:

```cpp
static const bool TILT_MOVEMENT_ENABLED = true;
```

When `true`, `main.cpp` feeds the helper live tilt each loop and passes its offset on; when
`false`, it passes `0, 0` and the pet is drawn dead centre exactly as before. Flip it and rebuild —
a safe, self-contained thing for students to experiment with.

### 7.7 — Calibrating on the device (a good exercise)

Which way the pet moves depends on how the accelerometer is physically mounted in *this* board, so
the first build is an educated guess you confirm by looking. All the knobs are named constants in
`tilt_motion.{h,cpp}`, so tuning is a one-file edit:

- **Moves on the wrong axis** (you tilt left/right but it goes up/down): swap which reading feeds
  which target in `update()`.
- **Moves the opposite way** (tilt right, pet goes left): negate that axis's reading —
  `accelX` → `-accelX` (negate the *reading*, not `TILT_SCALE`, which is shared by both axes and
  would flip both).
- **Too sluggish or too twitchy**: raise/lower `TILT_SCALE` (distance) or `SMOOTHING_FACTOR`
  (responsiveness).

### The architectural lesson (again)

`TiltMotion` knows about **tilt and pixels** — nothing else. It never reads the IMU directly (it is
*handed* the numbers) and never draws (it *hands back* an offset). Pair it with `AnimationManager`
from Part 6 and you have two tiny single-job classes — one for *when*, one for *where* — composing
into richer behaviour than either could alone. That is the whole game: small pieces, each with one
job, combined.

## Part 8 — Changing the Pet's Face With Its Mood (the Mood Sprite System)

Part 6 changed **which frame** is drawn (animation); Part 7 changed **where** it is drawn (tilt).
This part changes **which picture** is drawn, based on **how the pet feels**. When the pet is
starving it shows a hungry face; when it is sick it shows an unwell face. Same slot on screen, same
animation timing — a different sprite, chosen from the pet's stats.

The whole feature is three small pieces working in a line:

```
Pet::computeMood()  →  a MoodSprite value  →  DisplayManager::spriteForMood()  →  the picture
```

### 8.1 — Mood is not the same thing as state

The codebase already has a `PetState` enum (`STATE_IDLE`, `STATE_EATING`, …). It is tempting to
think that is the pet's "mood", but they answer different questions:

- **`PetState`** = what the pet is *doing* right now (eating, sleeping, being healed). It is set by
  the action methods.
- **`MoodSprite`** = how the pet *looks* / feels, read straight from its stats. It is computed fresh
  every frame and is never stored.

Keeping them separate means the pet can be `STATE_IDLE` (doing nothing) yet still look `HUNGRY`
because its fullness is low. The face follows the stats, not the activity.

### 8.2 — `computeMood()`: a priority ladder, not a winner-takes-all

`Pet::computeMood()` (in `lib/Pet/pet.cpp`) turns the seven stats into exactly one of four moods:

```cpp
MoodSprite Pet::computeMood() const {
    if (sick > 50) {
        return MOOD_UNWELL;
    }
    if (fullness < 30) {
        return MOOD_HUNGRY;
    }
    if (happy > 70) {
        return MOOD_HAPPY;
    }
    return MOOD_NEUTRAL;
}
```

Read it top to bottom as a **ladder**: the *first* rule that is true wins and we `return`
immediately, so every rule below it is skipped. Order therefore encodes **priority** — we deal with
the worst thing first. Being unwell matters more than being hungry, which matters more than being
happy. If nothing crosses a threshold, the pet is `MOOD_NEUTRAL`.

A worked example with a fresh pet (its starting stats are `fullness 80, happy 70, sick 0`):

| Check | Value | threshold met? | Result |
|---|---|---|---|
| `sick > 50`     | 0  | no  | fall through |
| `fullness < 30` | 80 | no  | fall through |
| `happy > 70`    | 70 | **no** (70 is not *greater than* 70) | fall through |
| (none matched) | — | — | **`MOOD_NEUTRAL`** |

So a brand-new pet shows the **neutral** face, not the happy one — `happy` has to climb *above* 70
(press **Play** and it jumps to ~95) before `MOOD_HAPPY` wins. That `>` vs `>=` detail is exactly
the kind of off-by-one worth checking on the whiteboard.

> **What this replaced.** The old `getDominantMood()` scanned all seven stats and returned whichever
> was *numerically highest*. That answered "which stat is biggest?" — not "what mood should the pet
> be in?" — so a high *good* stat like cleanliness could announce itself as the mood. The priority
> ladder answers the real question and is far easier to extend (see 8.6).

### 8.3 — Where the `MoodSprite` enum lives, and why it is not in `pet.h`

`computeMood()` lives in `Pet`, so you might expect the `MoodSprite` enum to live in `pet.h` next to
`PetState`. It does not — it lives in **`lib/Display/screen_layout.h`**:

```cpp
enum MoodSprite {
    MOOD_NEUTRAL,
    MOOD_HAPPY,
    MOOD_UNWELL,
    MOOD_HUNGRY
};
```

The reason is a dependency rule worth understanding. The enum is a *shared word* that two modules
both need to speak: `Pet` **produces** a mood, `DisplayManager` **consumes** it to pick a picture.
Whoever does not own the enum has to `#include` the file that does. So:

- If the enum lived in `pet.h`, then `DisplayManager` would have to `#include "pet.h"` — dragging
  the entire `Pet` class into the display layer and breaking the rule written at the top of
  `display_manager.h`: *"it does not need to know what a `Pet` is."*
- Instead it lives in `screen_layout.h`, a tiny header with **no includes of its own**. `Pet`
  includes it to *return* a `MoodSprite`; `DisplayManager` already includes it. Neither depends on
  the other — they only share the small vocabulary file.

This is the same pattern `RelevantStat` already uses (produced by the action menu, defined in
`screen_layout.h`). The rule of thumb: **a type shared across a boundary belongs in the lightweight
header both sides can include, not inside either heavy module.** Keep the arrow pointing
`Pet → screen_layout` and `Display → screen_layout`, never `Display → Pet`.

### 8.4 — From a mood to a picture: `spriteForMood()`

`DisplayManager::spriteForMood()` is the single place that maps a mood to its artwork:

```cpp
const uint16_t* DisplayManager::spriteForMood(MoodSprite mood, int frame) {
    switch (mood) {
        case MOOD_HAPPY:   return sprite_happy_placeholder[frame];
        case MOOD_UNWELL:  return sprite_unwell_placeholder[frame];
        case MOOD_HUNGRY:  return sprite_hungry_placeholder[frame];
        case MOOD_NEUTRAL:
        default:           return sprite_neutral_placeholder[frame];
    }
}
```

`drawPetSprite()` then calls it, passing the frame `AnimationManager` picked, so a mood sprite can
*also* animate:

```cpp
const uint16_t* spriteData = spriteForMood(mood, petAnimation.getCurrentFrame());
canvas.pushImage(spriteX, spriteY, spriteWidth, spriteHeight, spriteData, SPRITE_TRANSPARENT_COLOR);
```

Notice `drawPetSprite()` does not receive the pixel data as a parameter — it is *given the mood*
and looks up the picture itself. That switch is the single hook point where a mood becomes a
specific picture.

### 8.5 — The face and the word can never disagree

The same `MoodSprite` value also chooses the **word** under the pet, in `showPetMoodText()`:

```cpp
switch (mood) {
    case MOOD_HAPPY:  moodText = "Happy";   moodColor = TFT_GREEN;  break;
    case MOOD_UNWELL: moodText = "Unwell";  moodColor = TFT_PURPLE; break;
    case MOOD_HUNGRY: moodText = "Hungry";  moodColor = TFT_RED;    break;
    case MOOD_NEUTRAL:
    default:          moodText = "Neutral"; moodColor = TFT_WHITE;  break;
}
```

Because **one** `mood` value flows down the whole render chain
(`renderDisplay → renderMainScreen / renderInteractScreen → drawPetSprite` *and* `showPetMoodText`),
the sprite and the label are always computed from the same source. They cannot drift apart and say
"Happy" while drawing a hungry face. Passing the typed `MoodSprite` (rather than a bare `int`) is
what lets the compiler keep everyone honest.

### 8.6 — One picture per mood, and the single-frame placeholder

Each mood has its own sprite header in `lib/Display/sprites/`
(`neutral_placeholder.h`, `happy_placeholder.h`, `unwell_placeholder.h`, `hungry_placeholder.h`),
made with the exact workflow from Parts 1–3. They are **single-frame** for now, so `AnimationManager`
is created with a frame count of **1**:

```cpp
AnimationManager petAnimation = AnimationManager(SPRITE_NEUTRAL_PLACEHOLDER_FRAME_COUNT); // = 1
```

A one-frame animator simply holds frame 0 (its `update()` does nothing — see Part 6), so the pet is
still and `spriteForMood(mood, 0)` always returns frame 0. To make the moods animate later: draw a
second frame for each (Part 6's anchor-point rule still applies), re-run the converter, and bump
that frame count to 2. Nothing else in the chain changes — the `frame` argument is already threaded
through `spriteForMood()` waiting for it.

On flash cost: four 80×80 sprites at 12.8 KB each is ~51 KB, and even an animated set (eight frames)
is ~102 KB — a rounding error against the ESP32's 4 MB of flash (Part 4, Rule 4). Moods are cheap.

### 8.7 — Add your own mood (a three-step exercise)

The system is built so a new mood is a small, bounded change in three named places:

1. **Add the value** to `MoodSprite` in `screen_layout.h` (e.g. `MOOD_SAD`).
2. **Add a rule** in `Pet::computeMood()` — a new `if` block, placed at the priority height you want
   (remember: higher in the ladder = higher priority).
3. **Add a sprite** header and one `case` in `spriteForMood()` (and a word/colour in
   `showPetMoodText()`).

A natural first exercise: the four shipped moods deliberately give **no** face to the two stats that
can *kill* the pet at their low end — `happy → 0` and `energised → 0` (see `Pet::isDead()`). As those
fall, the face stays `NEUTRAL` right up until the pet dies. Adding a `MOOD_SAD` (for low `happy`)
and/or a `MOOD_TIRED` (for low `energised`) closes that gap and is the perfect way to practise the
three steps above.

### The architectural lesson (once more)

The mood system is three single-job pieces in a line: `Pet` decides the *mood* from its stats and
knows nothing about pictures; `screen_layout.h` holds the shared *word* both sides speak;
`DisplayManager` turns a mood into *pixels* and knows nothing about stats. The dependency arrows all
point inward to the small shared enum, never module-to-module. Add the typed value flowing through
one render chain so the face and the label can't disagree, and you have a feature that is both easy
to read today and easy to extend tomorrow — the same "small pieces, each with one job" idea as the
animator and the tilt helper before it.
