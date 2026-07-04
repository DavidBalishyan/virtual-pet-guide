# Session 6 - Multi-Screen and Mood Sprites

Two features in one session because they work together. The Stats screen gives you a data view with all six stat bars and the mood face. The mood sprites give the pet different faces depending on how it feels - happy, hungry, unwell, thirsty, or neutral.

## What we did

### Multi-screen navigation

There are three screens now: Main, Stats, and Interact. NavigationManager handles switching between them. From Main, button B opens the Interact screen (the action menu), and button C opens the Stats screen. From Stats, any button press returns to Main. From Interact, pressing A on "Back" returns to Main.

The Stats screen is a pure data view - stat bars stacked vertically with labels, the pet's name at the top, and a mood word at the bottom. No pet sprite. The bars use the same drawStatusBar() helper as the rest of the project, just positioned differently. Each stat bar has its own colour: green for happiness, red for fullness, blue for energy, cyan for cleanliness, purple for sickness, orange for hydration.

The screen layout zones are defined as constexpr ScreenZone structs in display_manager.h. The Stats screen has its own zone constants (STATS_ZONE, MOOD_ZONE, etc.) that position each bar at specific pixel coordinates on the 135x240 display. The bar positions are tight enough to fit six of them without scrolling.

### Mood sprites

Before this session there was only one face - neutral. computeMood() always returned MOOD_NEUTRAL. With ENABLE_MOOD_SPRITES defined, it uses a priority ladder:

- If sick > 50: UNWELL (purple tint, sick face)
- If fullness < 30: HUNGRY (red tint, hungry face)
- If hydration < 30: THIRSTY (orange tint, thirsty face)
- If happy > 70: HAPPY (green tint, smiling face)
- Otherwise: NEUTRAL (white, default face)

The priority order matters. Being unwell is checked first, then hunger, then thirst, then happiness. A sick pet with low fullness shows as UNWELL, not HUNGRY. The worst problem always wins.

Each mood has its own sprite header under lib/Display/sprites/. They're all 80x80 pixels, single-frame placeholders for now. The spriteForMood() method in DisplayManager maps the MoodSprite enum to the right pixel array. The mood text below the sprite also changes - the showPetMoodText() switch has a case for each mood with a matching colour and label.

The sprites are stored in PROGMEM (flash) as RGB565 pixel arrays. They were created in Piskel and converted using a custom tool in tools/piskel_converter/. The transparency colour is 0xF81F (magenta), which pushImage() skips so the pet's outline isn't a square block.

## Things that tripped me up

- The screen layout zones are all constexpr and computed at compile time. If you change a zone coordinate, you have to make sure it doesn't overlap with another zone. The mood text on the Stats screen (MOOD_ZONE at y=180) sits below the hydration bar (HYDRATION_BAR_ZONE at y=136) - if you added a seventh stat bar, you'd need to shift things around.
- The pet sprite position on Main and Interact is intentionally the same (faceCenterY = 110 on both). This way the pet doesn't jump when you switch between the two screens. The Stats screen has no sprite at all, so there's nothing to jump.
- spriteForMood() uses #ifdef ENABLE_MOOD_SPRITES to guard the four non-neutral cases. When the flag is off, every mood falls through to the default case and returns the neutral sprite. This means the code compiles and runs correctly regardless of the flag setting - you just don't see different faces.
- The nav bar on the Main screen draws two tabs (Stats and Interact), but only if their respective ENABLE_* flags are defined. With only ENABLE_MULTISCREEN on and ENABLE_ACTION_MENU off, you'd only see the Stats tab. The outer #if guard prevents tabWidth from being an unused variable when both are off.
- The AnimationManager was already there from session 1, but now it matters more because different screens need different animation states. It resets to frame 0 whenever the screen changes.

## Files we touched

- lib/Display/screen_layout.h (MoodSprite enum was already there, but now non-NEUTRAL values are actually used)
- lib/Display/display_manager.h and display_manager.cpp (renderStatsScreen(), spriteForMood(), showPetStatus(), showPetMood())
- lib/Display/sprites/ (added happy_placeholder.h, unwell_placeholder.h, hungry_placeholder.h, thirsty_placeholder.h)
- lib/Navigation/navigation_manager.cpp (handleStatsScreenInput(), screen routing)
- lib/Pet/pet.cpp (computeMood() now uses the ENABLE_MOOD_SPRITES block)
