# Session 3 - Accelerometer: Shake and Tilt

This was the session where the pet started responding to the physical world. Before this, everything was button inputs - you pressed B to cycle the menu, A to confirm. Now you can shake the device to play with the pet, and tilt it to slide the sprite around the screen.

## What we did

The ImuManager class reads acceleration data from the MPU6886 sensor (which is built into the M5StickC Plus 2). It exposes three values: accelX, accelY, and accelZ, each in G-force units. At rest on a flat surface, Z reads about 1.0 G (gravity) and X/Y are near zero.

Shake detection uses a simple threshold. The magnitude of the acceleration vector is sqrt(x^2 + y^2 + z^2). When this exceeds SHAKE_THRESHOLD (1.8 G), the device is considered to be shaking. There's a cooldown of 2000 ms between reported shakes so one waggle doesn't trigger play() a dozen times in a row. The edge detection works the same way as the button handler - prevShakeDetected vs currentShakeDetected, and wasShaken() returns true only on the transition frame.

When a shake is detected, main.cpp calls myPet.play(). This means you can initiate play from any screen without opening the action menu.

The TiltMotion class is separate from ImuManager. Its job is smoothing and clamping the raw accelerometer values into pixel offsets that the display can use. Raw accelerometer data is noisy and jittery - you don't want the sprite jumping around by 20 pixels every frame. TiltMotion applies a low-pass filter that blends the current reading with the previous one, which gives a smooth, eased feel. The offsets are clamped so the sprite can't slide off the edge of the screen.

The display uses spriteOffsetX and spriteOffsetY to nudge the pet sprite away from its normal centre position. These values are passed as parameters to renderDisplay() and then to drawPetSprite(). When tilt is disabled (the TILT_MOVEMENT_ENABLED toggle), they're just 0, 0 and the sprite stays centred like it did before.

## Things that tripped me up

- The IMU needs M5.update() to be called in loop() before it reads fresh data. I had M5.update() in there already (the buttons need it too) but it's worth remembering because without it the accelerometer returns stale values.
- The cooldown is critical. Without it, one shake triggers play() every frame until the acceleration drops below the threshold. I watched my pet's happiness spike and energy drain in about half a second.
- The shake threshold is a float comparison. 1.8 G is fairly sensitive - I found that a gentle flick works. Making it higher means you have to really shake hard.
- The low-pass filter in TiltMotion uses a smoothing factor. Higher values = more smoothing (slower response). Lower values = more responsive but jittery. The default felt about right for sliding the sprite around.
- ImuManager and TiltMotion are two separate classes even though they both deal with the accelerometer. TiltMotion only cares about smoothed offsets - it doesn't need to know about the raw sensor or how shake detection works. The separation makes it possible to use either feature independently.

## Files we touched

- lib/Imu/imu_manager.h and imu_manager.cpp
- lib/Imu/tilt_motion.h and tilt_motion.cpp
- src/main.cpp (added the IMU update and shake check in loop(), wired tilt into rendering)

The shake-to-play feature was already wired into main.cpp's updateLivePet() behind the ENABLE_IMU_PLAY flag. There's also a comment in there about a challenge: make a big tilt trigger an action of your choice. I haven't tried that yet but it should work the same way as the shake check.
