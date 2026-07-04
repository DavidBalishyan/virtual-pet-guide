# Session 4 - Sound and Buzzer Melodies

Gave the pet a voice. It plays short melodies when you feed it, put it to sleep, bathe it, etc., and it beeps at you when it's hungry, sick, or thirsty. Also plays a death sound when the pet dies and a little fanfare when you reset it.

## What we did

The SpeakerManager class wraps M5.Speaker.tone() which generates a square wave at a given frequency (in Hz) for a given duration (in milliseconds). Each public method corresponds to one sound event.

The melodies are hardcoded sequences of notes. Each note is a (frequency, duration) pair followed by a small delay for the gap between notes. For example, the feed sound is C5 (523 Hz) for 120 ms, then E5 (659 Hz) for 180 ms. The play sound ascends C5 -> G5 -> C6. The sleep sound descends A4 -> F4 -> D4. The death sound is a slow descent from A4 down to A3 with longer durations to make it feel heavy.

The alert sounds are different - they're two short beeps with a gap in between. The hunger alert is two high A5 beeps, the sickness alert is two low D4 beeps, and the thirst alert rises from G5 to C6. Each has a distinct pitch pattern so you can tell them apart by ear without looking at the screen.

The alerts are triggered from inside Pet::updateState(), which runs every loop. It checks if fullness is below 20, sickness is above 80, or hydration is below 20, and if enough time has passed since the last alert (15 seconds), it plays the matching sound through the speaker reference that was passed in from main.cpp.

SpeakerManager::init() just sets the volume to 128. M5.Speaker.setVolume() takes a value from 0 to 255. 128 is about half volume, which is comfortable for the built-in buzzer.

## Things that tripped me up

- The delay() calls inside the sound methods are intentional. Normally we avoid delay() everywhere because it blocks the loop, but for sound playback it's fine because the melody only takes a few hundred milliseconds and nothing else can run during a sound anyway. The alternative would be a non-blocking sound system with millis() timers, but that's a lot more complexity for something that plays notes sequentially.
- M5.Speaker.stop() needs to be called after the last note. Without it, the last tone keeps playing indefinitely. The alert sounds have a stop() between beeps to create the silent gap.
- The note frequencies follow the standard A4 = 440 Hz convention. Each octave doubles the frequency. So A3 = 220 Hz, A4 = 440 Hz, A5 = 880 Hz. C5 is 523 Hz because the C note is a specific ratio above A. I don't have these memorised - the code uses the standard values directly.
- The speaker reference is passed around through multiple layers. main.cpp creates the speaker and passes it to menu.confirmAction() (for action sounds), to Pet::updateState() (for alerts and death sound), and to Pet::reset() (for the restart fanfare). Each level takes the speaker as a parameter only when ENABLE_SOUND is defined, using #ifdef so the speaker parameter disappears when sound is disabled. This means the function signatures change depending on the build flags.
- The loudness matters. The buzzer on the M5StickC is small and can sound harsh at high volumes. We set it to 128 and that feels about right for a desktop setting.

## Files we touched

- lib/Speaker/speaker_manager.h and speaker_manager.cpp
- lib/Pet/pet.h and pet.cpp (the ENABLE_SOUND ifdefs in updateState() and reset())
- lib/Actions/action_menu.cpp (sound calls in confirmAction())
- src/main.cpp (speaker creation, init, wiring)
