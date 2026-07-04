# Session 5 - Persistence: Save and Load

The pet finally remembers its state after you unplug it. Before this session, every time the device lost power the pet reset to default stats - a fresh 90 fullness, 0 sickness, etc. Now you can save and the next boot picks up exactly where you left off.

## What we did

StorageManager uses the ESP32's NVS (Non-Volatile Storage) through the Arduino Preferences library. It's a key-value store that lives in flash memory and survives power-off. The interface is simple: open a namespace, read/write keys as integers, close it. Writes are committed when you call end().

The save() method writes all eight stats plus the pet's name under the "virtual-pet" namespace. Each stat gets its own key - "fullness", "happy", "tired", etc. The load() method reads them back. If a key doesn't exist yet (first boot), it falls back to the DEFAULT_* constants from the Pet class, so a fresh device starts healthy.

The Save action appears in the action menu as a new entry between Drink and Back, but only when ENABLE_PERSISTENCE is defined. The menu array and NUM_ACTIONS constant adjust automatically through #ifdef so there's never an empty gap or a phantom entry. When persistence is off, the array has 7 entries and no Save button.

Save is triggered through the action menu like any other action - the user scrolls to Save, presses A, and confirmAction() calls storage.save(pet). There's no auto-save. I thought about adding one but the manual save makes the concept more concrete - you choose when to persist.

Load happens in setup(), after display.init() and before the main loop starts. If the pet died before the last power-off, you load a dead pet's stats and the death screen shows immediately. The clear() method wipes all saved data, and it's called when the pet is reset after death so a fresh game starts on the next boot.

## Things that tripped me up

- Preferences needs different modes for read and write. prefs.begin(NAMESPACE, false) opens in read/write mode for saving. prefs.begin(NAMESPACE, true) opens read-only for loading. If you open read/write when you only need to read, it still works but it's wasteful.
- The namespace name is limited to 15 characters on ESP32. "virtual-pet" is exactly 11, so it fits.
- The default values in load() must match the Pet constructor's defaults. If they drift apart, a fresh device and a loaded save would start with different stats. The code uses Pet::DEFAULT_FULLNESS etc. directly in the getInt() calls so they can't diverge.
- Save is only compiled when ENABLE_PERSISTENCE is on, but the Save action in the menu also needs ENABLE_ACTION_MENU. There's a compile-time check in scaffold_config.h that prevents enabling persistence without the action menu, because Save lives inside the menu.

## Files we touched

- lib/Storage/storage_manager.h and storage_manager.cpp
- lib/Actions/action_menu.cpp (added ACTION_SAVE case in confirmAction())
- lib/Actions/action_menu.h (ACTION_SAVE in the enum, adjusted NUM_ACTIONS)
- src/main.cpp (added storage.load() in setup(), storage.clear() in death handler)
